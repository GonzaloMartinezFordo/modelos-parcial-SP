## Ejercicio 1
- a) Programar la rutina que atenderá la interrupción que el lector de cartuchos generará al terminar de llenar el buffer.

Primero armemos la rutinas de atencion en el isr.asm
El cartucho avisará que esta lleno mediante la interrupcion IRQ(40)
```x86asm
global isr_40

isr_40:
    pushad
    call pic_finish1
    call deviceready
    popad
    iret
```
Vamos a modificar y agregar ciertas estructuras dentro de sched.c
```c
//nuevo
typedef enum {
  NO_ACCESS,
  ACCESS_DMA,
  ACCESS_COPIA
} task_access_mode_t;

// Modificamos
typedef enum {
  TASK_SLOT_FREE,
  TASK_RUNNABLE,
  TASK_PAUSED,
  //agregamos el estado blocked para saber si fue detenido por acceder al buffer
  TASK_BLOCKED
} task_state_t;

typedef struct {
  int16_t selector;
  task_state_t state;
  //Agregamos copydir y modo
  uint32_t copydir;
  uint8_t mode;
} sched_entry_t;
```

Definimos deviceready, dentro de sched.c simplemente para acceder a los structs del scheduler.
Como deviceready es llamada cada vez que el buffer se llena, este se tiene que encargar de comprobar que tareas solicitaron acceso al buffer y actuar acorde a si fue por DMA o por copia, encargandose de mapear o remapear y copiar la pagina del buffer y dejar a la tarea lista para que pueda volver a correr.
```c
void deviceready(void){
  //Recorremos las tareas que en el scheduler
  for(int i = 0; i < MAX_TASKS; i++){
    sched_entry_t * tarea = &sched_tasks[i];
    //Si la tarea no pidio acceso, no hacemos nada con ella, miramos la siguiente
    if(tarea->mode = NO_ACCESS){
      continue;
    }
    //Si la tarea esta bloqueada por que hizo un opendevice, actuamos acorde a su modo.
    if(tarea->state = TASK_BLOCKED){
      if(tarea->mode = ACCESS_DMA){
        //contamos con funciones buffer_dma y buffer_copy
        //Si hizo acceso DMA se mapea la dir virt 0xBABAB000
        buffer_dma(CR3_TO_PAGE_DIR(task_selector_to_CR3(tarea->selector)));
      }
      if(tarea->mode = ACCESS_COPIA){
        buffer_copy(CR3_TO_PAGE_DIR(task_selector_to_CR3(tarea->selector)), mmu_next_free_user_page(), tarea->copydir);
      }
      tarea->state = TASK_RUNNABLE; //la tarea ya puede correr ahora que tiene su mapeo (y su copia si es el caso)
    } else{
      //Si la tarea no estaba bloqueada ya se le hizo su mapeo, es por que hubo otra
      // interrupcion donde se lleno el buffer y la tarea sigue accediendo al mismo por que no hizo
      // closedevice aún, tenemos que actualizar la copia
        if(tarea->mode = ACCESS_COPIA){ 
          //Hay que actualizar la copia.
          paddr_t destino = virt_to_phy(task_selector_to_CR3(tarea->selector), tarea->copydir);
          copy_page(destino, (paddr_t) 0xF151C000); 
          //el 2do argumento es la fuente de donde vamos a copiar el contenido
          //el 1er argumento es el destino, donde vamos a pegar el contenido a copiar.
        }
        //en el caso de DMA, no es necesario re-mapear ya que seguimos apuntando a la misma dir. fisica desde la 
        // misma dir. virtual
    }
  }
}
```

- b) Programar las syscalls opendevice y closedevice.
- Cuentan con las siguientes funciones ya implementadas:
	- void buffer_dma(pd_entry_t* pd) que dado el page directory de una tarea realice el mapeo del buffer en modo DMA.
	- void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt) que dado el page directory de una tarea realice la copia del buffer a la dirección física pasada por parámetro y realice el mapeo a la dirección virtual pasada por parámetro.


Las tareas pueden solicitar acceso al buffer mediante la syscall opendvice y avisar cuando dejan de acceder con closedevice, estas syscalls son interrupciones de software (int 48-255), definimos las que están disponibles, elijo 90 y 91 para estas 2.
```x86asm
global isr_90
;OPENDEVICE
isr_90:
    pushad
    ;los parametros se pasan por pila y en ECX estará la dir. virtual en donde realizar la copia.
    push ECX
    call opendevice
    add esp, 4 ;aumentamos el stack pointer por el n° de argumentos que recibio opendevice (N=1), N*4
    call sched_next_task
    ;Como esta solicitando acceso al buffer, su ejecución debe detenerse hasta que el lector de cartuchos
    ; termine de llenar el buffer, solo ahi podrá volver a ejecutarse, por eso pedimos la siguiente tarea
    ; a ejecutar con sched_next_task.
    
    ; Aca nos comportamos como lo hace la interrupcion de reloj al momento de saltar a la sig. tarea
    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]

    popad
    iret
```
```x86asm
global isr_91
;CLOSEDEVICE
isr_91:
    pushad
    call closedevice
    popad
    iret
```

Ahora vamos a inicializar sus entradas en la idt. Dentro de idt.c tenemos la funcion idt_init()
```c
void idt_init() {
    //...

    //Interrupciones de hardware
    IDT_ENTRY0(32);
    IDT_ENTRY0(33);
    IDT_ENTRY0(40); //Interrupcion del lector de cartuchos

    // Syscalls
    IDT_ENTRY3(88);
    //Agregamos las syscalls de opendevice y closedevice
    IDT_ENTRY3(90); // opendevice
    IDT_ENTRY3(91); // closedevice
    IDT_ENTRY3(98);
 }
```


Las implementaciones de opendevice y closedevice:
```c
void opendevice(vaddr_t copyDirVirt){
  sched_tasks[current_task].state = TASK_BLOCKED;
  //Cada tarea configura su modo de acceso en esa dir. virtual,
  // el enunciado nos dice que asumamos que esa direccion ya esta mapeada a una dir. fisica
  sched_tasks[current_task].mode = *(uint8_t*)0xACCE500; 
  sched_tasks[current_task].copydir = copyDirVirt;
}

void closedevice(void){
  //Aca desmapeamos y dejamos la tarea con modo no accedido
  if(sched_tasks[current_task].mode == ACCESS_DMA){
    //Si hizo acceso DMA se mapea la dir virt 0xBABAB000
    mmu_unmap_page(rcr3(), (vaddr_t)0xBABAB000);
  }
  if(sched_tasks[current_task].mode == ACCESS_COPIA){
    mmu_unmap_page(rcr3(), sched_tasks[current_task].copydir);
  }
  sched_tasks[current_task].mode = NO_ACCESS;
}
```

## Ejercicio 2:
- a) Programar la función void buffer_dma(pd_entry_t* pd)
- b) Programar la función void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt)

Vamos a implementar estas funciones dentro de mmu.c

```c
void buffer_dma(pd_entry_t* pd){
  mmu_map_page((uint32_t)pd, (vaddr_t)0xBABAB000, (paddr_t)0XF151C000, (MMU_P | MMU_U));
  //Los accesos por DMA, se mapean de la dir Virt 0xBABAB000 direccionada a la dir. fisica 0XF151C000
  // el acceso es de usuario y solo lectura, lo marcamos como presente
}

void buffer_copy(pd_entry_t* pd, paddr_t phys, vaddr_t virt){
    //virt es la direccion virtual a la que vamos a mapear
    // phys es la proxima pagina de usuario disponible
    // y pd es el cr3 de la tarea
  mmu_map_page((uint32_t)pd, virt, phys, (MMU_P | MMU_U | MMU_W));
  copy_page(phys, (paddr_t)0xF151C00);
}
```

funciones auxiliares:
en tasks.c
```c
#include "tasks.h"
#include "tss.h"
//Funciones auxiliar.
uint32_t task_selector_to_CR3(uint16_t selector) {
  uint16_t index = selector >> 3; // Sacamos los atributos
  gdt_entry_t* taskDescriptor = &gdt[index]; // Indexamos en la gdt
  tss_t* tss = (tss_t*)((taskDescriptor->base_15_0) | (taskDescriptor->base_23_16 << 16) |
  (taskDescriptor->base_31_24 << 24));
  return tss->cr3;
}
```
En mmu.c
```c
//funcion auxiliar

#include "defines.h"
#include "mmu.h"
paddr_t virt_to_phy(uint32_t cr3, vaddr_t virt) {

 uint32_t* cr3 = task_selector_to_cr3(task_id);
 uint32_t pd_index = VIRT_PAGE_DIR(virtual_address);
 uint32_t pt_index = VIRT_PAGE_TABLE(virtual_address);
 pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);

 pt_entry_t* pt = pd[pd_index].pt << 12;
 return (paddr_t) (pt[pt_index].page << 12);
}
```

## EXTRA
Nuestro kernel "Orga Génesis" es funcional, pero ineficiente. Hemos detectado que algunas tareas de Nivel 3
solicitan acceso al dispositivo (opendevice) pero luego nunca leen los datos.
Esto es un desperdicio crítico de recursos. En particular:
● En Modo Copia, se gastan 4KB de RAM valiosa para una copia que no se usa.
● El deviceready (IRQ 40) gasta ciclos de CPU actualizando copias.
Para solucionar esto, implementaremos una nueva tarea de nivel 0 llamada task_killer. El codigo de esta tarea
reside en el area de kernel, se ejecutará como las demas tareas en round robin y debera deshabilitar cualquier
tarea de nivel 3 que haya desperdiciado recursos por mucho tiempo.
Una tarea se considera ociosa y debe ser deshabilitada si cumple todas las siguientes condiciones:
1. Ha estado en estado TASK_RUNNABLE por más de 100 "ticks" de reloj.
2. Tiene un acceso activo al dispositivo (es decir, task[i].mode != NO_ACCESS y task[i].status !=
TASK_BLOCKED).
3. No ha leído la memoria del buffer (DMA o Copia) desde la última vez que el "killer" la inspeccionó.

3) Describa todos los pasos para crear e inicializar la nueva tarea task_killer:
Detalle los cambios necesarios en:
- 1. GDT: ¿Cómo se crea la nueva entrada de TSS (Task State Segment) en la GDT?
- 2. TSS: ¿Qué campos críticos de la struct tss (TSS) de esta nueva tarea debe
inicializar? (Mencione al menos EIP, CS, DS, ESP0, SS0 y CR3).
- 3. Scheduler: ¿Cómo se agrega esta nueva tarea Nivel 0 al array sched_tasks?

Respuestas:
- 1. Necesitaríamos crear la tarea con TSS_Create_User_Task(paddr_t code_start) (o su version equivalente)
    que nos crea una TSS, la cual usamos su direccion de memoria para TSS_GDT_ENTRY_FOR_TASK(tss_t* tss), estas se 
    encuentran en tss.c, por suerte contamos con una funcion que se encarga de hacer estas cosas llamada 
    create_task(tipo_e tipo) en tasks.c, la cual podríamos adaptar para esta tarea particular.
- 2. el eip tendría la direccion donde comienza el codigo, cs sería GDT_CODE_0_SEL y ds,es,fs,gs,ss,ss0 serían GDT_DATA_0_SEL y el cr3 sería la la proxima pagina libre del kernel en la cual no haya basura (usando zero page), tenga identity mapping (por que sigue siendo una tarea que vive dentro del area libre del kernel), esp0 es la proxima pagina libre del kernel y le sumamos a esa direccion PAGE_SIZE ya que "crece" hacia abajo.
- 3. sched_add_task(uint16_t selector) es una funcion del sched.c con el cual podemos agregarla al scheduler

```c
/** tasks.c
 * Crea una tarea en base a su tipo e indice número de tarea en la GDT
 */
static int8_t create_task(tipo_e tipo) {
  size_t gdt_id;
  for (gdt_id = GDT_TSS_START; gdt_id < GDT_COUNT; gdt_id++) {
    if (gdt[gdt_id].p == 0) {
      break;
    }
  }
  kassert(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");

  int8_t task_id = sched_add_task(gdt_id << 3);
  tss_tasks[task_id] = tss_create_user_task(task_code_start[tipo]);
  gdt[gdt_id] = tss_gdt_entry_for_task(&tss_tasks[task_id]);
  return task_id;
}


/** tss.c
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
    .esp = stack + PAGE_SIZE,
    .ebp = stack + PAGE_SIZE,
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

//tasks.c
static int8_t create_task_kill(tipo_e tipo) {
  size_t gdt_id;
  for (gdt_id = GDT_TSS_START; gdt_id < GDT_COUNT; gdt_id++) {
    if (gdt[gdt_id].p == 0) {
      break;
    }
  }
  kassert(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");

  int8_t task_id = sched_add_task(gdt_id << 3);
  tss_tasks[task_id] = tss_create_kernel_task(&killer_task_loop);
  gdt[gdt_id] = tss_gdt_entry_for_task(&tss_tasks[task_id]);
  return task_id;
}

//tss.c
tss_t tss_create_kernel_task(paddr_t code_start) {
  //COMPLETAR: asignar valor inicial de la pila de la tarea
  vaddr_t stack = mmu_next_free_kernel_page();
  return (tss_t) {
    .cr3 = create_cr3_for_kernel_task(),
    .esp = stack + PAGE_SIZE,
    .ebp = stack + PAGE_SIZE,
    .eip = (vaddr_t) code_virt,
    .cs = GDT_CODE_0_SEL,
    .ds = GDT_DATA_0_SEL,
    .es = GDT_DATA_0_SEL,
    .fs = GDT_DATA_0_SEL,
    .gs = GDT_DATA_0_SEL,
    .ss = GDT_DATA_0_SEL,
    .ss0 = GDT_DATA_0_SEL,
    .esp0 = stack + PAGE_SIZE,
    .eflags = EFLAGS_IF,
  };

}

// mmu.c
paddr_t create_cr3_kernel_task(){
  //inicializamos directorio de paginas
  paddr_t task_page_dir = mmu_next_free_kernel_page();
  zero_page(task_page_dir); //no debe haber basura en esa dirección

  //hacemos identity mapping
  for (uint32_t i = 0; i < end_identity_map; i += PAGE_SIZE){
    mmu_map_page(task_page_dir, i,i, MMU_W);
  }
  return task_page_dir;
}
```
Modificamos la rutina de reloj para contabilizar los ticks

```x86asm
;; Rutina de atención del RELOJ
;; -------------------------------------------------------------------------- ;;
global _isr32
; COMPLETAR (Parte 2: Interrupciones): La rutina se encuentra escrita parcialmente. Completar la rutina
  
_isr32:
    pushad
    ; 1. Le decimos al PIC que vamos a atender la interrupción
    call pic_finish1
    call next_clock

    call add_tick_to_task ; agregamos esto a la rutina

    
    call sched_next_task
    ;...
    popad
    iret

```

Dentro de sched.c agregamos la sig. función
```c
//EXTRA
static uint8_t contador_de_ticks[MAX_TASKS] = {0};

void add_tick_to_task(){
  contador_de_ticks[current_task]++;
}
```
ahora implementamos task_killer en tasks.c
```c

//Extra
void killer_main_loop(){
  while(true){
    //iteramos por las tareas y comprobamos si debemos matarlas
    for(int i = 0; i < MAX_TASKS; i++){
      if(i == current_task) continue;

      sched_entry_t* task = &sched_tasks[i];

      if(task->state == TASK_PAUSED) continue;
      if(task->mode != NO_ACCESS && task->state != TASK_BLOCKED) continue;
      if(contador_de_ticks[i] <= 100) continue;
      vaddr_t vaddr = 0;
      if(task->mode == ACCESS_DMA){
        vaddr_t vaddr_to_check = 0xBABAB000;
      } else {
        vaddr_t vaddr_to_check = task->copydir;
      }
      pte_t* pte = mmu_get_pte_for_task(task->selector, vaddr);
      if(pte.accessed == 0){
        task->state = TASK_KILLED;
      } else {
        contador_de_ticks[i] = 0;
        pte.accesed = 0;
      }
    }
  }
}

//en mmu.c
pt_entry_t* mmu_get_pte_for_task(uint16_t task_id, vaddr_t virtual_address){
  uint32* cr3 = task_selector_to_cr3(task_id);
  uint32_t pd_index = VIRT_PAGE_DIR(virtual_address);
  uint32_t pt_index = VIRT_PAGE_TABLE(virtual_address);

  pd_entry_t* pd = (pd_entry_t*)CR3_TO_PAGE_DIR(cr3);
  pt_entry_t* pt = pd[pd_index].pt << 12;
  return (pt_entry_t*) &(pt[pt_index]);
}
```