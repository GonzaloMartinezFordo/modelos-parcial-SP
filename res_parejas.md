Se nos pide que armemos un sistema de parejas, en la cual pueden unirse y abandonar a voluntad pero con ciertas restricciones que detallare más adelante.
Las parejas sirven para que las tareas compartan paginas fisicas (Hasta 4MB) que se mapean bajo demanda (es decir que solo se mapea mediante el page_fault_handler), solo el lider (el que creó la pareja) puede escribir el otro solo lee.

# Restricciones:
- Las tareas solo pueden pertenecer a una pareja maximo.
- Toda tarea puede abandonar su pareja, si no es lider actúa sin problemas, si es lider queda pausada hasta que su compañera abandone la pareja.
- Las tareas pueden formar parejas cuantas veces quieran pero solo pueden pertenecer a una.
- Cuando una tarea abandona su pareja pierde acceso a los 4MB (Tenemos que desmapear)
- Si crea una pareja queda pausada hasta que alguien se una

La direccion compartida empieza a partir de la dirección virtual '0xC0C00000'.

Claramente es necesario tener un registro de las parejas que están formadas para garantizar que no pertenecen a otra, a su vez es necesario 
guardar cuantas paginas tienen mapeadas cada tarea, de esa forma poder desmapear correctamente.

- 1) Se nos pide implementar los mecanismos para unirse, crear y abandonar pareja
- 2) Implementar un monitoreo del uso de memoria de las parejas, en el que no se contabilizan 2 veces el acceso al mismo megabyte


Antes de implementar las syscalls, primero armamos las estructuras necesarias.
Claramente necesitamos una lista para el estado de las parejas en el cual denotamos quién es el lider, su compañera y la cant. de paginas accedidas.
Se implementarán dentro de 'sched.c'
```C
typedef struct {
    task_id_t lider;
    task_id_t compañera;
    uint16_t cantPags; //No va solicitar mas de 1024 paginas de 4kb
} parejas_t;

parejas_t* lista_parejas[MAX_TASKS];

typedef struct {
  int16_t selector;
  task_state_t state;
  bool en_pareja; //agregamos este campo, para facilitar su acceso
} sched_entry_t;

static sched_entry_t sched_tasks[MAX_TASKS] = {0};

```

## Ejercicio 1 (50 pts):
Para definir syscalls es necesario inicializar sus entradas de nivel 3 (de ese modo pueden acceder las tareas de nivel usuario) de en la IDT.

Dentro de 'idt.c' 
```C
void idt_init() {
    //...

    //Interrupciones de hardware
    IDT_ENTRY0(32);
    IDT_ENTRY0(33);

    // Syscalls
    IDT_ENTRY3(88);
    //Agregamos las syscalls de parejas
    IDT_ENTRY3(90); // crear_pareja()
    IDT_ENTRY3(91); // juntarse_con(id)
    IDT_ENTRY3(92); // abandonar_pareja()
    IDT_ENTRY3(98);
}
```
Ahora definimos las rutinas de atención en 'isr.asm'
```ASM
global _isr90
global _isr91
global _isr92

;crear_pareja()
_isr90:
    pushad
    call crear_pareja
    ;La tarea queda pausada hasta que alguien se una, saltamos a la siguiente tarea
    ;Recordatorio eax == 0 es False, eax != 0 es True.
    cmp ax, 0 ; si ax == 0, quiere decir que no pudo formar pareja
    je .fin ;Si no pudo crear pareja, ignoramos solicitud y retornamos inmediatamente

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

;juntarse_con(taskid)
_isr91:
    pushad
    push ax ;pasamos los argumentos por pila según la convencion de C
    call juntarse_con
    add esp, 4 ;sumamos esp la cant.Argumentos * 4
    cmp ax, 1 ;si retorno 1 es que no puede formar pareja
    je .fin

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

.fin
    popad
    iret

;abandonar_pareja()
_isr92:
    pushad
    call abandonar_pareja ;como el comportamiento de la syscall depende de si es lider o no, devuelve bool
    cmp ax, 1 ;Si NO es lider(o si es lider y su pareja abandono), sigue su ejecución sin problemas. 
    jne .fin

;Si es lider y todavía tiene compañera, queda pausada hasta que su compañera abandone.
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



En 'sched.c' defino las implementaciones de las syscalls
```C
typedef struct {
    task_id_t lider; //inicializada en 0
    task_id_t compañera; //inicializada en 0
    // uint16_t cantPags; //No va solicitar mas de 1024 paginas de 4kb
} parejas_t;

parejas_t* lista_parejas[MAX_TASKS];

typedef struct {
  int16_t selector;
  task_state_t state;
  bool lider; //Inicializa en False
  task_id_t compañera; //inicializada en 0, si es 0, no tiene compañera
} sched_entry_t;


bool crear_pareja(){
    if(sched_tasks[current_task].compañera != 0 || (sched_tasks[current_task].lider)){
        //No tiene sentido que un lider de una pareja quiera crear de nuevo otra pareja
        return false;
    } else {
        //Si no tiene pareja nos pausamos y nos ponemos como lider
        sched_tasks[current_task].state = TASK_PAUSED;
        sched_tasks[current_task].lider = true;
        // sched_tasks[current_task].compañera = 0; //esto no es necesario, ya comprobamos que es 0
        return true;
    }
}

//Si la tarea pasada por parametro esta en pareja o no es lider
int juntarse_con(task_id_t task_id){
    if((sched_tasks[task_id].lider && sched_tasks[task_id].compañera != 0) ||
         !(sched_tasks[task_id].lider) || (sched_tasks[current_task].compañera != 0) ){
            //Si me quiero juntar con alguien que no es lider FALLA
            // SI me quiero juntar con alguien, cuando la tarea llamadora tiene compañera, FALLA
            // SI me quiero juntar con alguien que es lider pero ya tiene compañera, FALLA
        return 1;
    } else {
        sched_tasks[task_id].compañera = current_task;
        sched_tasks[current_task].compañera = task_id;
        return 0;
    }

}

//Falta abandonar tarea

```


Para mantener registro de la cant. de paginas reservadass tenemos que armar una lista de los accesos a memoria.
Como se que a los sumo habrá MAX_TAREAS/2 armo la siguiente lista.


```C
typedef struct {
    uint16_t pags_acccedidas; //Se incrementa en 1 si cada vez que ocurre un page_fault por intentar acceder a la pagina, se comprueba si su compañero tiene la pagina presente en su page table (de ser el caso no se incrementa)-
} reservas_t;

reservas_t* reservas_de_parejas[MAX_TAREAS/2] = {0};

#define VIRT_BASE_PAREJAS = 0xC0C00000
#define VIRT_LIMIT_PAREJAS = (VIRT_BASE_PAREJAS + PAGE_SIZE * PAGE_SIZE * 4)
//Preparamos el page fault handler



// COMPLETAR: devuelve true si se atendió el page fault y puede continuar la ejecución 
// y false si no se pudo atender
bool page_fault_handler(vaddr_t virt) {
    print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
  // Chequeemos si el acceso fue dentro del area on-demand
  // En caso de que si, mapear la pagina
    if (virt >= ON_DEMAND_MEM_START_VIRTUAL && virt <= ON_DEMAND_MEM_END_VIRTUAL){
        mmu_map_page(rcr3(), virt, ON_DEMAND_MEM_START_PHYSICAL, (MMU_P|MMU_U|MMU_W));
        return true;
    } else {
        return false;
    }
  //Falta completar para las parejas
    if(VIRT_BASE_PAREJAS =<  virt &&  virt <= VIRT_LIMIT_PAREJAS){
        if(es_lider(current_task)){
            //ACA hay que comprobar si esta pagina esta presente para la compañera para
            // saber si aumentamos la cantidad de paginas accedidas por la pareja
            paddr_t phy = next_free_user_page()
            mmu_map_page(rcr3(), virt, phy, (MMU_P|MMU_U|MMU_W));
            return true;
        } else {
            //Comprobemos si esta en pareja, de no estarlo, retornar false.
            if(sched_tasks[current_task].compañera != 0){
                //ACA hay que comprobar si esta pagina esta presente para la compañera para
            // saber si aumentamos la cantidad de paginas accedidas por la pareja

                // si pertenece a una pareja y no es lider, no puede escribir
                paddr_t phy = next_free_user_page()
                mmu_map_page(rcr3(), virt, phy, (MMU_P|MMU_U));   
                return true;
            }
            return false; //No tiene pareja y trata de acceder a las direcciones de la pareja
            
        }
        
    } else {
        return false;
    }
  
}

uint32_t task_selector_to_CR3(uint16_t selector) {
    uint16_t index = selector >> 3; // Sacamos los atributos
    gdt_entry_t* taskDescriptor = &gdt[index]; // Indexamos en la gdt
    tss_t* tss = (tss_t*)((taskDescriptor->base_15_0) |
    (taskDescriptor->base_23_16 << 16) |
    (taskDescriptor->base_31_24 << 24));
    return tss->cr3;
}

bool pagina_presente(uint32_t task_id, vaddr_t virt) {
    uint32_t selector = sched_tasks[task_id].selector;
    uint32_t cr3 = task_selector_to_cr3(selector);
    uint32_t pd_index = VIRT_PAGE_DIR(virt);
    uint32_t pt_index = VIRT_PAGE_TABLE(virt);
    pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

    pt_entry_t* pt = pd[pd_index].pt << 12;
    uint32_t attrs = pt[pt_index].attrs;
    return ((attrs & MMU_P) != 0);
}
```
