## Ejercicio 1
Asumiendo que ya existen las funciones malloco, esMemoriaReservada, chau y la tarea garbage_collector
lo que se tendría que hacer para que pueda reservar es que tengamos las syscalls para
que la tarea pueda llamarlas pudiendo reservar y liberar memoria. Como se tratan de syscalls estas estarán dentro del rango de INT 48-255, las cuales se tiene que definir sus respectivas entradas en la idt, agregando las lineas 'IDT_ENTRY3(XX)' dentro de 'idt_init()' en 'idt.c', luego definir sus rutinas de atención dentro de 'isr.asm'

En isr.asm:
```x86asm
global isr_90
global isr_91

;malloco
isr_90:
    pushad
    push edi
    call malloco
    add esp, 4
    popad
    iret
;chau
isr_91:
    pushad
    push edi
    call chau
    add esp, 4
    popad
    iret
```
En idt.c:

```C
void idt_init() {
    //...
    // Syscalls
    IDT_ENTRY3(88);
    //Agregamos las syscalls de malloco y chau
    IDT_ENTRY3(90); // malloco
    IDT_ENTRY3(91); // chau
    //...
}
```

Cuando la tarea intente acceder a uno de sus bloques reservados por primera vez, 
tendremos que modificar la funcion del page_fault_handler(), para que revise en
el array de reservas de la tarea, si la dirección que se trato acceder esta dentro del rango de InicioDelBloquevirt =< direccionALaQueSeIntentoAcceder < InicioDelBloquevirt + PAGE_SIZE, se hace el mapeo a la siguiente pagina libre de usuario con el inicio del bloque, si el acceso no fue dentro de las reservas, no se hace el mapeo y se devuelve FALSE, ahi se modificará el _isr14 en 'isr.asm' que en el caso de dar FALSE, se deshabilita la tarea con 'sched_free_task'(similar a sched_disable_task pero esta la marca como TASK_SLOT_FREE, podríamos tratar de eliminar su entrada de sched_tasks si es necesario) y se llama una funcion que marque todas las reservas para liberar, luego se salta a la siguiente tarea.


En mmu.c:
```C

// COMPLETAR: devuelve true si se atendió el page fault y puede continuar la ejecución 
// y false si no se pudo atender
bool page_fault_handler(vaddr_t virt) {
  print("Atendiendo page fault...", 0, 0, C_FG_WHITE | C_BG_BLACK);
  // Chequeemos si el acceso fue dentro del area on-demand
  // En caso de que si, mapear la pagina
  if (virt >= ON_DEMAND_MEM_START_VIRTUAL && virt < ON_DEMAND_MEM_END_VIRTUAL){
    mmu_map_page(rcr3(), virt, ON_DEMAND_MEM_START_PHYSICAL, (MMU_P|MMU_U|MMU_W));
    return true;
  } 
  uint32_t task_id = current_task_id()
  reservas_por_tarea* reservas = dameReservas(task_id);
  uint32_t cantReservas = reservas->reservas_size;
  reserva_t* array = reservas->array_reservas; //como es un puntero se accede a sus atributos con '->'
  if (esMemoriaReservada(virt)){
    for (int i = 0; i < cantReservas; i++){
       if(array[i].virt <= virt && virt < (array[i].virt + array[i].tamanio) && array[i].estado == 1){
        vaddr_t base = virt & 0xFFFFF000;  // Alinear a 4 KB
        paddr_t next = mmu_next_free_user_page();
        mmu_map_page(rcr3(), base, next, (MMU_P|MMU_U| MMU_W));
        zero_page(next);
        //array[i].estado = 1; no me queda claro si 1 indica que ya fue mapeado
        break;
       } 
    }
  } else{
    //Trato de acceder a una dir. virtual que no reservo, hay que marcar todas sus reservas para borrar, luego van a tener que deshabilitarla
    for(int i = 0; i < cantReservas; i++){
        chau(array[i].virt);
    }
  }
  return false;
}
```
```x86asm
global _isr14

_isr14:
	; Estamos en un page fault.
	pushad
    ; COMPLETAR: llamar rutina de atención de page fault, pasandole la dirección que se intentó acceder
    mov EAX, CR2
    push EAX
    call page_fault_handler
    cmp EAX, 0
    jne .fin
    .ring0_exception:
	; Si llegamos hasta aca es que cometimos un page fault fuera del area bloque reservado.
    ;ya marcamos todas para borrar, nos falta invalidar la tarea
    call acceso_invalido 
    call sched_next_task ;solicitamos la sig. tarea
    cmp ax, 0
    je .fin

    str bx ;guardamos el TR en bx
    cmp ax, bx ;comparamos si son los mismos
    je .fin

    mov word [sched_task_selector], ax
    jmp far [sched_task_offset]
    jmp $

    .fin:
    pop EAX
	popad
	add esp, 4 ; error code
	iret
```
Agrego las sig. funciones a sched.c
```C
//Parcial malloco
void acceso_invalido(void){
  int8_t tarea = current_task;
  sched_kill_task(tarea);
  //sched_disable_task(tarea); 
  //No me queda del todo claro la parte "el kernel debe desalojar tarea inmediatamente y asegurarse de que no vuelva a correr", podría dejarla pausada y que luego el garbage_collector termine de limpiar todo y despúes de eso marcarla como TASK_SLOT_FREE por que corro el riesgo de que la reemplacen (en el proximo sched_add_task(X)) y el garbage_colector nunca se fije la memoria que debio limpiar
}
void sched_kill_task(int8_t task_id) {
  kassert(task_id >= 0 && task_id < MAX_TASKS, "Invalid task_id");
  sched_tasks[task_id].state = TASK_SLOT_FREE; //podría crear un nuevo estado, marcada para matar y luego de que el garbage_colector termine lo suyo, setearla como TASK_SLOT_FREE
}
```

## Ejercicio 2
Como nos piden que se ejecute garbage_colector cada 100 ticks, no necesariamente agregarla al scheduler, tenemos que modificar la interrupción de reloj isr32 en 'isr.asm'

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

static uint8_t contador_de_ticks = 0;

void add_tick_to_task(){
  contador_de_ticks++;
  if(contador_de_ticks == 100){
    garbage_collector();
  }
}
```
en mmu.c
```c
void garbage_collector(){
  for (int8_t i = 0; i < MAX_TASKS; i++) {
    reservas_por_tarea* reser = dameReservas(i);
    reserva_t* array = reser->array_reservas;
    uint32_t size = reser->reservas_size;
    sched_entry_t* task = &sched_tasks[i];
    uint32_t cr3_tarea = task_selector_to_CR3(task->selector);
    for(uint32_t j = 0; j  < size; j++){
      if(array[j].estado == 2){
        uint32_t bytes_restantes = array[j].tamanio;
        uint32_t offset = 0;
        while(bytes_restantes > 0){
          paddr_t phy = mmu_unmap_page(cr3_tarea,array[j].virt + offset);
          //en caso de que nos pidan tambien que las dejemos en zero, con desmapear sería suficiente
          //paddr_t phy = virt_to_phy(cr3_tarea, array[j].virt + offset);
          zero_page(phy);
          bytes_restantes -= PAGE_SIZE;
          offset += PAGE_SIZE; //tenemos que desmapear las siguientes paginas.
        }
      }
    }

  }
}

//Auxiliares
#include "tss.h"
#include "gdt.h"
#include "i386.h"  // donde está rtr()

uint8_t current_task_id() {
    uint16_t selector = rtr(); // lee el Task Register (TR)
    // >>3 para eliminar los 3 bits de privilegio y TI, y restamos la base del bloque de tareas
    return (selector >> 3) - GDT_IDX_TASKS_START;
}


uint8_t task_id_from_selector(uint16_t selector) {
    for (uint8_t i = 0; i < MAX_TASKS; i++) {
        if (sched_tasks[i].selector == selector) {
            return i;
        }
    }
    return 0xFF; // no encontrada
}

```

## Ejercicio 3
- a) Nuestras estructuras podrían guardarse en donde este definido malloco, chau esmemoriaReservada y dameReservas (al igual que el garbage_colector) en mmu.c, sin embargo podríamos guardarlas dentro de sched.c

```c
typedef struct {
  uint32_t virt; //direccion virtual donde comienza el bloque reservado
  uint32_t tamanio; //tamaño del bloque en bytes
  uint8_t estado; //0 si la casilla está libre, 1 si la reserva está activa, 2 si la reserva está marcada para liberar, 3 si la reserva ya fue liberada
} reserva_t; 

typedef struct {
  uint32_t task_id;
  reserva_t* array_reservas;
  uint32_t reservas_size;
} reservas_por_tarea; 
```

- b) Implementación de malloco en mmu.c
- c) Implementación de chau en mmu.c
```c
#define VIRT_BASE   0xA10C0000
#define VIRT_LIMIT  0xA14C0000
#define MAX_TAM_MEMORIA (4 * 1024 * 1024) // 4 MiB

void* malloco(size_t size) {
    uint32_t task_id = current_task_id();
    reservas_por_tarea* res = dameReservas(task_id);

    // // Alineamos el tamaño al múltiplo de 4 KB
    // if (size % PAGE_SIZE != 0)
    //     size = ((size / PAGE_SIZE) + 1) * PAGE_SIZE;

    //Verificar que la tarea no supere su límite de 4 MiB
    uint32_t total_actual = 0;
    for (uint32_t i = 0; i < res->reservas_size; i++) {
        if (res->array_reservas[i].estado == 1) // activa
            total_actual += res->array_reservas[i].tamanio;
    }

    if (total_actual + size > MAX_TAM_MEMORIA)
        return NULL; // supera el límite

    //Buscar la última reserva activa para continuar desde allí
    uint32_t next_virt = VIRT_BASE;
    for (uint32_t i = 0; i < res->reservas_size; i++) {
        if (res->array_reservas[i].estado == 1) {
            uint32_t fin = res->array_reservas[i].virt + res->array_reservas[i].tamanio;
            if (fin > next_virt)
                next_virt = fin;
        }
    }

    //Verificar si hay espacio disponible
    if (next_virt + size > VIRT_LIMIT)
        return NULL; // fuera del área virtual

    // Registrar la nueva reserva
    reserva_t nueva = {
        .virt = next_virt,
        .tamanio = size,
        .estado = 1 // activa
    };

    res->array_reservas[res->reservas_size++] = nueva;

    //Devolver la dirección virtual reservada
    return (void*) next_virt;
}

void chau(vaddr_t virt) {
    // Si la dirección no pertenece a una reserva válida, no hacemos nada
    if (!esMemoriaReservada(virt))
        return;

    uint32_t task_id = current_task_id();
    reservas_por_tarea* res = dameReservas(task_id);
    reserva_t* arrayR = res->array_reservas;
    uint32_t cantReservas = res->reservas_size;

    for (uint32_t i = 0; i < cantReservas; i++) {
        // Si el bloque está activo y la dirección está dentro del rango reservado
        if (virt >= arrayR[i].virt && virt < (arrayR[i].virt + arrayR[i].tamanio)) {
            if (arrayR[i].estado == 1) {
                arrayR[i].estado = 2; // marcar para liberar
            }
            break; // ya encontramos la reserva correspondiente
        }
    }
}


```