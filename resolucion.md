Nuestro sistema tiene tareas que producen y solicitan recursos.
- Si se solicita quedamos pausados hasta que una tarea que termine de preparar dicho recurso nos despierte, puede suceder que ese recurso ya este listo y podamos tomarlo directamente.
- Si producimos un recurso tenemos que buscar si alguien lo solicitó, despertarlo y copiarselo en su parte de memoria, puede ser que nadie nos solicitará
nuestro recurso, por lo que podemos dejarlo disponible para cualquiera que lo solicite.

Nos resulta practico tener los datos de que datos producimos, que solicitamos, si estamos esperando por un recurso o si nuestro recurso ya está disponible.
Podríamos modificar la estructura del scheduler para tener dicha información.

Considerando que puede ocurrir que una tarea solicito un recurso antes que otra me conviene tener una cola de solicitudes de dicho recurso, podría simplemente
dejar un campo para solicitante y así mantener cierta prioridad.

```C
/**
 * Estructura usada por el scheduler para guardar la información pertinente de
 * cada tarea.
 */
typedef struct {
  int16_t selector;
  task_state_t state;
  recurso_t solicito;
  recurso_t MiRecurso; //Recurso que produzco, asumo que esto ya viene inicializado
  bool esperandoRecurso;
  bool recursoListo;
  task_id_t quienMeSolicito; //Asumimos que se inicializa en 0
} sched_entry_t;
```

### Ejercicio 1 (50pts):
Ahora tenemos que definir como implementamos la syscall. Toda interrupción que querramos agregar es necesario que tenga su entrada en la IDT en la 'idt.c' con su respectivo nivel y definir su comportamiento en 'isr.asm'.
Las Syscalls son las interrupciones del 48-255 con entradas de nivel 3 pues estas deben ser accedidas por aplicaciones.
Las interrupciones externas del 32-47 con entradas de nivel 0.

# En idt.c
```C
void idt_init() {
    //....
  // COMPLETAR: Interrupciones de reloj y teclado
  IDT_ENTRY0(32);
  IDT_ENTRY0(33);

  IDT_ENTRY0(41); //iniciar produccion
  // COMPLETAR: Syscalls
  IDT_ENTRY3(88);
  
  IDT_ENTRY3(90); //solicitar
  IDT_ENTRY3(91); //recurso_listo
  
  IDT_ENTRY3(98);
}
```

## Mecanismo de copia
Cada vez que una tarea llame a recurso_listo, está ya habrá dejado su recurso en la dirección virt '0x0AAAA000' (que en este momento debe estar mapeado).
al que solicito su recurso, se le tiene que copiar la pagina en la que esta el recurso en la dirección virt '0x0BBBB000' El cual definitivamente no va estar mapeada.

Desde recurso_listo, miramos nuestro solicitante, de tenerlo tenemos que prepara el mapeo, limpieza de pagina y después la copia, en ese orden, ya que si no
tenemos la direccion virtual mapeada se debería generar un page fault.
- 1) Solicitamos una pagina nueva para la tarea solicitante mediante 'destino_PHY = mmu_next_free_user_page()'
- 2) Mapeamos con mmu_map_page(cr3_del_solicitante, 0x0BBBB000, destino_PHY, (MMU_P|MMU_U|MMU_W)), asumo que debe ser writable 
- 3) Ahora que mapeamos podemos limpiar dicha pagina en caso de que tenga basura 'zero_page(0x0BBBB000)'
- 4) Hay un inconveniente al respecto, la dirección fisica del destino la tenemos, pero no la de fuente, por lo que tenemos que hacer
    'destino_PHY = virt_to_phy(cr3_del_productor, FUENTE)' antes del copy_page.
- 5) Finalmente realizamos la copia, recordemos que copy_page recibe direcciones fisicas, 'copy_page(destino_PHY, fuente_PHY)'

Desde solicitar, suponiendo que hay tarea libre cuyo recurso esta disponible, tenemos el mismo modelo, solicitar una pagina libre, mapear para el solicitante,
limpiar la pagina, copiar, es exactamente igual. Solo cambia como obtenemos el cr3 del solicitante (con rcr3()) y del productor es con su task_id tomar el selector, 'selector_Prod = sched_tasks[task_id_del_productor].selector', 'cr3_del_productor = task_selector_to_CR3(selector_Prod)'.

Problema que tenemos que tomar en cuenta, cuando la tarea productora quiera escribir su recurso en '0x0AAAA000' se va a cometer un page fault, tenemos que preparar nuestro page_fault_handler(virt)

```C
#define FUENTE = 0x0AAAA000
#define DESTINO = 0x0BBBB000

// COMPLETAR: devuelve true si se atendió el page fault y puede continuar la ejecución 
// y false si no se pudo atender
bool page_fault_handler(vaddr_t virt) {
  print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
  // Chequeemos si el acceso fue dentro del area on-demand
  // En caso de que si, mapear la pagina
  if (virt >= ON_DEMAND_MEM_START_VIRTUAL && virt <= ON_DEMAND_MEM_END_VIRTUAL){
    mmu_map_page(rcr3(), virt, ON_DEMAND_MEM_START_PHYSICAL, (MMU_P|MMU_U|MMU_W));
    return true;
  } if(virt >= FUENTE && virt <= (FUENTE + PAGE_SIZE)) {
    //Si la tarea productora trato de acceder a la dir 0x0AAAA000 dentro de los 4KB, le damos la pagina
    paddr_t phy = mmu_next_free_user_page();
    mmu_map_page(rcr3(), virt, phy, (MMU_P|MMU_U|MMU_W));
    zero_page(FUENTE); //Ahora que esta mapeada podemos limpiarla
    return true;
  } else {
    return false;
  }
}

#include "defines.h"
#include "mmu.h"
paddr_t virt_to_phy(uint32_t cr3, vaddr_t virt) {
    uint32_t* cr3 = task_selector_to_cr3(task_id);
    uint32_t pd_index = VIRT_PAGE_DIR(virt);
    uint32_t pt_index = VIRT_PAGE_TABLE(virt);
    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    pt_entry_t* pt = pd[pd_index].pt << 12;
    return (paddr_t) (pt[pt_index].page << 12);
}

//Funciones auxiliares.
uint32_t task_selector_to_CR3(uint16_t selector) {
  uint16_t index = selector >> 3; // Sacamos los atributos
  gdt_entry_t* taskDescriptor = &gdt[index]; // Indexamos en la gdt
  tss_t* tss = (tss_t*)((taskDescriptor->base_15_0) | (taskDescriptor->base_23_16 << 16) |
  (taskDescriptor->base_31_24 << 24));
  return tss->cr3;
}
```

Sabemos que solicitar recibe por parametro el recurso a solicitar (asumo que la pasan por EAX)
- Si nadie tiene ese recurso disponible y las tareas que lo producen ya tienen un solicitante, quedará pausada
- Si alguien lo tiene y no tiene solicitante, tenemos que hacer el mecanismo de copia y resetear a la que nos produjo dicho recurso y seguimos corriendo.
- Si alguien tiene dicho recurso y tiene solicitante es un bug, no debe ser posible ya que siempre que una tarea tenga su recurso listo se lo da a su solicitante.

Como necesitamos saber si nos quedamos pausados o podemos seguir corriendo será necesario devolver nuestro estado mediante True o False

## solicitar(recurso)

# En isr.asm
```x86asm
global _isr90

_isr90:
    pushad
    push EAX ;por convencion pasamos los argumentos por pila de der. a izq.
    call solicitar
    add esp, 4 ;sumamos al esp (cant. argumentos)*4
    cmp ax, 1
    jne .fin

.sig_Tarea:
    call sched_next_task
    cmp ax, 0 ;si la sig. es la idle no saltamos
    je .fin

    str bx
    cmp ax, bx ;si la sig. es la actual no saltamos
    je .fin

.salto:
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]

.fin:
    popad
    iret
```

# En sched.c
```C

#define FUENTE = 0x0AAAA000
#define DESTINO = 0x0BBBB000


bool solicitar(recurso_t recurso){
    //Asumo que me devuelve la tarea que produce dicho recurso y no tiene solicitante (no necesariamente tiene el recurso listo) 
    task_id_t tareaProdDisponible =  hay_tarea_disponible_para_recurso(recurso);

    if(tareaProdDisponible != 0){
        //Encontramos una tarea que puede producir nuestro recurso, tenemos que ver si esta listo o nos ponemos como su solicitante.
        if(sched_tasks[tareaProdDisponible].recursoListo){
            //Tiene recurso listo y no tiene solicitante
            (paddr_t) destino_PHY = mmu_next_free_user_page(); //pedimos la prox. pagina disponible
            mmu_map_page(rcr3(), DESTINO, destino_PHY, (MMU_P|MMU_U|MMU_W)); // mapeamos
            zero_page(DESTINO); //Recordemos que toma direcciones virtuales y como esta mapeada no va a fallar, además es MI dir virt, no del productor
            uint16_t selector_del_productor = sched_tasks[tareaProdDisponible].selector;
            uint32_t cr3_del_productor = task_selector_to_CR3(selector_del_productor);
            paddr_t fuente_PHY = virt_to_phy(cr3_del_productor, FUENTE);
            copy_page(destino_PHY, fuente_PHY);
            restaurar_tarea(tareaProdDisponible) //No nos olvidemos de restaurarla
            return true;
        }
        // No tiene su recurso preparado, nos ponemos como su solicitante
        sched_tasks[tareaProdDisponible].quienMeSolicito = current_task;
    }

    //Si no hay tarea sin solicitantes que lo produzca o no hay recurso a nuestra disposición nos pausamos e indicamos que estamos solicitando
    sched_tasks[current_task].esperandoRecurso = true;
    sched_tasks[current_task].solicito = recurso;
    sched_tasks[current_task].state = TASK_PAUSED;
    return false;
}

```

## recurso_listo()
recurso_listo, mas allá de tener que seguir el mecanismo de copia indicado previamente, este tiene que reanudar la tarea que lo solicito o la que esté esperando su recurso y saltar a la siguiente tarea. SE TIENE QUE DEJAR A LA PRODUCTORA PAUSADA


# En isr.asm
```x86asm
global _isr91

_isr91:
    pushad
    call recurso_listo
    ;Nos piden que desalojemos la tarea, así que avanzamos a la siguiente.
.sig_Tarea:
    call sched_next_task
    cmp ax, 0 ;si la sig. es la idle no saltamos
    je .fin

    str bx
    cmp ax, bx ;si la sig. es la actual no saltamos
    je .fin

;Independientemente de que nos dejemos como pausada, saltamos a la siguiente tarea
.salto:
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset] ;Se hace cambio de contexto, se te va a pisar la tss si la modificaste

    
.fin:
    ;como nos tenemos que restaurar cuando suceda el iret, se va a restaurar eip viejo, en lugar del inicial
    ;Recordemos que una tarea de nivel 3 fue interrumpida, hay cambio de privilegio
    mov [esp + 8], TASK_STACK_BASE ;cambiamos el esp
    mov [esp + 20], CODE_VIRT ; cambiamos el eip

    popad
    iret
```

# En sched.c
```C
void recurso_listo(){
    //Comprobemos si alguien nos solicitó
    task_id_t solicitante_id = sched_tasks[current_task].quienMeSolicito;

    if(solicitante_id != 0){
        //Como es distinto de 0, hay solicitante (y no es el de la interrupción externa)
        uint16_t selector_del_solicitante = sched_tasks[solicitante_id].selector;
        uint32_t cr3_del_solicitante = task_selector_to_CR3(selector_del_solicitante);
        paddr_t destino_PHY = mmu_next_user_page();
        mmu_map_page(cr3_del_solicitante, DESTINO, destino_PHY, (MMU_P|MMU_U|MMU_W)); // mapeamos
        zero_page(DESTINO); //Recordemos que toma direcciones virtuales y como esta mapeada no va a fallar
        uint32_t cr3_del_productor = rcr3();
        paddr_t fuente_PHY = virt_to_phy(cr3_del_productor, FUENTE);
        copy_page(destino_PHY, fuente_PHY);
        sched_tasks[solicitante_id].state = TASK_RUNNABLE; //Ya puede correr el solicitante
        restaurar_tarea(current_task); //No nos olvidemos de restaurarla
        return;
    } else {
        solicitante_id = hay_consumidora_esperando(sched_tasks[current_task].MiRecurso);
        if(solicitante_id != 0){
            //Como es distinto de 0, hay solicitante esperando mi recurso
            uint16_t selector_del_solicitante = sched_tasks[solicitante_id].selector;
            uint32_t cr3_del_solicitante = task_selector_to_CR3(selector_del_solicitante);
            paddr_t destino_PHY = mmu_next_user_page();
            mmu_map_page(cr3_del_solicitante, DESTINO, destino_PHY, (MMU_P|MMU_U|MMU_W)); // mapeamos
            //Recordemos que toma direcciones virtuales y como esta mapeada no va a fallar PERO debe ser con el cr3 del solicitante, como cambiar mi cr3 
            //register no conviene, vamos mapear temporalmente la dirección fisica del solicitante y así limpiarla.
   
            mmu_map_page(rcr3(), DESTINO, destino_PHY, (MMU_P|MMU_U|MMU_W)) //mapeamos temporalmente
            zero_page(DESTINO); //(en principio esto no sería necesario, ya habia limpiado mi parte de los 4KB y se lo voy a copiar)
            mmu_unmap_page(rcr3(), DESTINO); //Desmapeamos

            uint32_t cr3_del_productor = rcr3();
            paddr_t fuente_PHY = virt_to_phy(cr3_del_productor, FUENTE);
            copy_page(destino_PHY, fuente_PHY);
            sched_tasks[solicitante_id].state = TASK_RUNNABLE; //Ya puede correr el solicitante
            restaurar_tarea(current_task); //No nos olvidemos de restaurarla
            return;
        }
    }
    //Como nadie esta esperando mi recurso, simplemente informo que estoy listo.
    sched_tasks[current_task].recursoListo = true;
    sched_tasks[current_task].state = TASK_PAUSED; //Queda pausada la tarea
    return;
}
```

## Ejercicio 2 (25pts):
restaurar_tarea(task_id), tiene que restaurar la tarea a su estado inicial, es decir su eip debe apuntar al inicio de su codigo, su esp = ebp = inicio del stack de las tareas + PAGE_SIZE (recordemos que crecen hacia abajo), su esp0 debe apuntar al inicio del stack del kernel + page_size ,restaurar su EFLAGS, No es necesario limpiar sus paginas FUENTE y DESTINO por que FUENTE se limpia con el page_fault_handler y DESTINO se mapea y se limpia una vez que llame a la syscall solicitar(recurso).
Como hacemos todo esto? Necesitamos acceder a la tss de la tarea y solo tenemos su task_id, no es un problema por que podemos usar ese task_id para obtener
el selector de la tarea desde el scheduler, con el cual podemos obtener el indice de la GDT, el tendrá el descriptor de la TSS
Con el cual pdoremos acceder y modificar sus campos, hay un problema y es que, cambiarle la tss a una tarea que no es la actual no hay problema.
El tema es que al momento de hacer esto con nuestra propia tarea, el iret, nos restaura el eip y deja de ser el del code_start


```C
void restaurar_tarea(task_id_t task_id){
    //CAMBIAR LA TSS ES INNECESARIO
    uint16_t selector_de_la_tarea = sched_tasks[task_id].selector;
    tss_t* tss_de_la_tarea = &gdt[(selector_de_la_tarea >> 3)];

    tss_de_la_tarea->esp = TASK_STACK_BASE;
    tss_de_la_tarea->ebp = TASK_STACK_BASE;
    tss_de_la_tarea->eip = code_virt;
    tss_de_la_tarea->eflags = EFLAGS_IF;

    if(task_id == current_task){       
        


        sched_tasks[task_id].state = TASK_RUNNABLE; //la dejamos runnable
        sched_tasks[task_id].quienMeSolicito = 0;
        sched_tasks[task_id].solicito = 0; //asumo que 0 no es un recurso.
        sched_tasks[task_id].esperandoRecurso = false;
        sched_tasks[task_id].recursoListo = false;
        mmu_unmap_page(rcr3(), FUENTE);
        mmu_unmap_page(rcr3(), DESTINO);
        return;
        //Si es la tarea actual el iret nos va a restaurar el eip cuando salgamos de la interrupción por lo que tendremos que solucionarlo desde el asm (si hay cambio de nivel de privilegio tambien cambia el esp)
    } else {
        uint16_t selector = sched_tasks[task_id].selector;
        uint32_t cr3 = task_selector_to_CR3(selector);
        mmu_unmap_page(cr3, FUENTE);
        mmu_unmap_page(cr3, DESTINO);
    }
    

}



/**
 * Crea una tss con los valores por defecto y el eip code_start
 */
tss_t tss_create_user_task(paddr_t code_start) {

  //COMPLETAR: es correcta esta llamada a mmu_init_task_dir?
  uint32_t cr3 = mmu_init_task_dir(code_start);
  //COMPLETAR: asignar valor inicial de la pila de la tarea
  vaddr_t stack = TASK_STACK_BASE;
  //COMPLETAR: dir. virtual de comienzo del codigo
  vaddr_t code_virt = TASK_CODE_VIRTUAL;
  //COMPLETAR: pedir pagina de kernel para la pila de nivel cero
  vaddr_t stack0 = mmu_next_free_kernel_page();
  //COMPLETAR: a donde deberia apuntar la pila de nivel cero?
  vaddr_t esp0 = stack0 + PAGE_SIZE;  // DECRECE HACIA ABAJO, VA RESTANDO
  return (tss_t) {
    .cr3 = cr3,
    .esp = stack,
    .ebp = stack,
    .eip = code_virt,
    .cs = GDT_CODE_3_SEL,
    .ds = GDT_DATA_3_SEL,
    .es = GDT_DATA_3_SEL,
    .fs = GDT_DATA_3_SEL,
    .gs = GDT_DATA_3_SEL,
    .ss = GDT_DATA_3_SEL,
    .ss0 = GDT_DATA_0_SEL,
    .esp0 = esp0,
    .eflags = EFLAGS_IF,
  };

}
```

## Ejercicio 3 (15pts):
Interrupcion externa 41, la atiende al pic 2, pone a correr todas las tareas que producen su recurso, si están pausadas las pone runnable