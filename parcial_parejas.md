# Recuperatorio 2do Parcial de Arquitectura y Organización del Computador - 1c2025

## Enunciado
Como muchas actividades son mas divertidas de hacer en parejas, como bailar un tango o jugar al truco, se pensó que las tareas podrían trabajar en conjunto para poder solucionar problemas de forma cooperativa.
Nuestro sistema implementará una nueva funcionalidad llamada _parejas_ con las siguientes características:
- Toda tarea puede pertenecer **como máximo** a una pareja.
- Toda tarea puede _abandonar_ a su pareja.
- Las tareas pueden formar parejas cuantas veces quieran, pero solo pueden pertenecer a una en un momento dado.
- Las tareas de una pareja comparten un area de memoria de 4MB ($2^{22}$ bytes) a partir de la direccion virtual `0xC0C00000`. Esta región se asignará bajo demanda. Acceder a memoria no asignada dentro de la región reservará sólo las páginas necesarias para cumplir ese acceso. Al asignarse memoria ésta estará limpia (todos ceros). **Los accesos de ambas tareas de una pareja en ésta región deben siempre observar la misma memoria física.**
- Sólo la tarea que **crea** una pareja puede **escribir** en los 4MB a partir de `0xC0C00000`.
- Cuando una tarea abandona su pareja pierde acceso a los 4MB a partir de `0xC0C00000`.

Para poder soportar el mecanismo de _parejas_, el sistema debe implementar las tres syscalls que serán detalladas a continuación.

### `crear_pareja()`
Para crear una pareja una tarea deberá invocar `crear_pareja()`. Como resultado de esto se deben cumplir las siguientes condiciones:
- Si **ya pertenece a una pareja** el sistema ignora la solicitud y retorna de inmediato.
- Sino, la tarea no volverá a ser ejecutada hasta que otra tarea se una a la pareja.
- La tarea que crea la pareja será llamada la **líder** de la misma y será la única que puede escribir en el área de memoria especificado.

La firma de la syscall es la siguiente:
```C
void crear_pareja();
```

### `juntarse_con(id_tarea)`
Para unirse a una pareja una tarea deberá invocar `juntarse_con(<líder-de-la-pareja>)`. Aquí se deben considerar las siguientes situaciones:
- Si **ya pertenece a una pareja** el sistema ignora la solicitud y retorna `1` de inmediato.
- Si la tarea identificada por `id_tarea` no estaba **creando** una pareja entonces el sistema ignora la solicitud y retorna `1` de inmediato.
- Si no, se conforma una pareja en la que `id_tarea` es **líder**. Luego de conformarse la pareja, esta llamada retorna `0`.

La firma de la syscall es la siguiente:
```C
int juntarse_con(int id_tarea);
```

Notar que una vez **creada** la pareja se debe garantizar que ambas tareas tengan acceso a los 4MB a partir de `0xC0C00000` a medida que lo requieran.
Sólo la líder podrá escribir y **ambas podrán leer**.

### `abandonar_pareja()`
Para abandonar una pareja una tarea deberá invocar `abandonar_pareja()`, hay tres casos posibles a los que el sistema debe responder:
- Si la tarea no pertenece a ninguna pareja: No pasa nada.
- Si la tarea pertenece a una pareja y **no es su líder**: La tarea abandona la pareja, por lo que pierde acceso al área de memoria especificada.
- Si la tarea pertenece a una pareja y **es su líder**: La tarea queda bloqueada hasta que la otra parte de la pareja abandone. Luego pierde acceso a al área de memoria especificada.

La firma de la syscall es la siguiente:
```C
void abandonar_pareja();
```

### Ejercicios
#### Parte 1: implementación de las syscalls (80 pts)
1. **(50 pts)** Definir el mecanismo por el cual las syscall `crear_pareja`, `juntarse_con` y `abandonar_pareja` recibirán sus parámetros y retornarán sus resultados según corresponda. Dar una implementación para cada una de las syscalls. Explicitar las modificaciones al kernel que sea necesario realizar, como pueden ser estructuras o funciones auxiliares.

Se puede asumir como implementadas las siguientes funciones, que se resolverán en puntos siguientes. Su uso es opcional pero recomendado:
- `task_id pareja_de_actual()`: si la tarea actual está en pareja devuelve el `task_id` de su pareja, o devuelve 0 si la tarea actual no está en pareja.
- `bool es_lider(task_id tarea)`: indica si la tarea pasada por parámetro es lider o no.
- `bool aceptando_pareja(task_id tarea)`: si la tarea pasada por parámetro está en un estado que le permita formar pareja devuelve 1, si no devuelve 0.
- `void conformar_pareja(task_id tarea)`: informa al sistema que la tarea actual y la pasada por parámetro deben ser emparejadas. Al pasar `0` por parámetro, se indica al sistema que la tarea actual está disponible para ser emparejada.
- `void romper_pareja()`: indica al sistema que la tarea actual ya no pertenece a su pareja actual. Si la tarea actual no estaba en pareja, no tiene efecto.

Estas funciones únicamente interactuan con lo que sería el "sistema de asignación de parejas"; **no tienen efecto sobre el resto del comportamiento** que deben implementar las syscalls (manejo de memoria compartida, desalojo de tarea en ejecución, etc).

A modo de ayuda el siguiente es un diagrama de estados en el que se puede ver todas las posibles circunstancias en las que una tarea puede encontrarse. Se omiten las transiciones en las que no se hace nada (una tarea sin pareja intenta abandonar a su pareja ó una tarea con pareja intenta crear otra pareja).
![Diagrama de estados descrito por el enunciado anterior](img/state-machine.svg)

2. **(20 pts)** Implementar `conformar_pareja(task_id tarea)` y `romper_pareja()`
3. **(10 pts)** Implementar `pareja_de_actual()`, `es_lider(task_id tarea)` y `aceptando_pareja(task_id tarea)`

#### Parte 2: monitoreo de la memoria de las parejas (20 pts)

4. Escribir la función `uso_de_memoria_de_las_parejas`, la cual permite averiguar cuánta memoria está siendo utilizada por el sistema de parejas.
Ésta función debe ser implementada por medio de código ó pseudocódigo que corra en nivel 0.

La firma esperada de la función es la siguiente:
```c
uint32_t uso_de_memoria_de_las_parejas();
```

Tengan en consideración los siguientes puntos:
- Cuando `abandonar_pareja` deja a una pareja sin participantes los recursos asociados cuentan como liberados.
- Una pareja sólo usa la memoria a la que intentó acceder, no se contabiliza toda la memoria a la que podría acceder en el futuro.
- Una página _es usada por el sistema de parejas_ si es accesible dentro de la región compartida para parejas por alguna de las tareas que actualmente se encuentran en pareja (o son líderes solitarios de una pareja que aún no se terminó de disolver).
- En el caso en que ambas tareas de una pareja utilizan, por ejemplo, el primer megabyte del área compartida de su pareja, la función debe retornar que hay **un megabyte** de memoria en uso por el sistema de parejas. Es decir, no se debe contabilizar dos veces el mismo espacio de memoria si ambas tareas de la pareja lo están utilizando.

## A tener en cuenta para la entrega
- Se parte de un sistema igual al del TP-SP terminado, sin ninguna funcionalidad adicional.
- Indicar todas las estructuras de sistema que deben ser modificadas para implementar las soluciones.
- Está permitido utilizar las funciones desarrolladas en las guías y TPs, siempre y cuando no utilicen recursos con los que el sistema no cuenta.
- Es necesario que se incluya una **explicación con sus palabras** de la idea general de las soluciones.
- Es necesario explicitar todas las asunciones que hagan sobre el sistema.
- Es necesaria la entrega de **código que implemente las soluciones**.
- `zero_page()` modifica páginas basándose en una **dirección virtual** (no física!)


### Resolucion:

#### Parte 1.1:

##### `crear_pareja()`

- Primero debemos hacer la funcion de la syscall que llame a una interrupcion
    ``` C
    LS_INLINE void crear_pareja(void) {
    __asm__ volatile("int $90" /* int. de soft. 91 */
                     :
                     : "memory",
                      "cc"); /* announce to the compiler that the memory and
                                condition codes have been modified */
    }
    ```

- Luego la interrupcion debe llamar a un handler
    ``` asm
    global _isr90
    _isr90:
      pushad

      call crear_pareja_handle

      popad
      iret
    ```

- En el handler debemos primero ver si la tarea pertenece ya a una pareja, si no debemos pausar la tarea actual e informar al sistema que esta buscando pareja
    ``` C
    void crear_pareja_handle(void) {
        int8_t task_id = current_task;
        kassert(task_id <= 3, "task_id fuera del rango valido!");

        if (pareja_de_actual(task_id) == 0) return;

        sched_disable_task(task_id);

        conformar_pareja(0);
    }
    ```


##### `juntarse_con()`

- Primero debemos hacer la funcion de la syscall que llame a una interrupcion
    ``` C
    LS_INLINE int juntarse_con(int pareja_id) {
    __asm__ volatile("int $91" /* int. de soft. 91 */
                     :
                     : "a"(pareja_id)
                     : "memory",
                      "cc"); /* announce to the compiler that the memory and
                                condition codes have been modified */
    }
    ```

- Luego la interrupcion debe llamar a un handler
    ``` asm
    global _isr91
    _isr91:
      pushad

      push eax
      call juntarse_handle
      add esp, 4

      mov DWORD [esp + 28], eax ; Para poder devolver el valor de eax, lo modifico en la pila

      popad
      iret
    ```

- En el handler debemos primero ver si la tarea pertenece ya a una pareja, si no debemos verificar que la tarea pasada por parametro esten buscando pareja, en ese caso emparejamos y despausamos la tarea pareja
    ``` C
    int juntarse_handle(int pareja_id) {
        task_id_t task_id = current_task;
        kassert(task_id <= 3, "task_id fuera del rango valido!");

        if (pareja_de_actual(task_id) == 0) return 1;

        if (!aceptando_pareja(pareja_id)) return 1;

        conformar_pareja(pareja_id);

        return 0;
    }
    ```

##### `abandonar_pareja()`

- Primero debemos hacer la funcion de la syscall que llame a una interrupcion
    ``` C
    LS_INLINE void abandonar_pareja(void) {
    __asm__ volatile("int $92" /* int. de soft. 92 */
                     :
                     : "memory",
                      "cc"); /* announce to the compiler that the memory and
                                condition codes have been modified */
    }
    ```

- Luego la interrupcion debe llamar a un handler
    ``` asm
    global _isr92
    _isr92:
      pushad

      call abandonar_pareja

      popad
      iret
    ```

- En el handler debemos primero ver si la tarea pertenece ya a una pareja, si no debemos verificar que la tarea pasada por parametro esten buscando pareja, en ese caso emparejamos y despausamos la tarea pareja
    ``` C
    void abandonar_pareja(void) {
        task_id_t task_id = current_task;
        kassert(task_id <= 3, "task_id fuera del rango valido!");

        uint32_t pareja_id = pareja_de_actual(task_id);

        if (pareja_id == 0) return;

        romper_pareja();
    }
    ```

- Para que las interrupciones se puedan llamar debemos agregar las entradas de nivel 3 a la IDT, usamos `IDT_ENTRY3()` para que las tareas de nivel de usuario puedan llamar la interrupcion.
    ``` C
    ...
    IDT_ENTRY3(90);
    IDT_ENTRY3(91);
    IDT_ENTRY3(92);
    ...
    ```


#### Parte 1.2:

Para poder implementar `conformar_pareja(task_id tarea)` y `romper_pareja()` debemos hacer unas modificaciones en el kernel actual.

- Primero vamos a cambiar el enum de estados de una tarea
    ``` C
    typedef enum {
        TASK_SLOT_FREE,
        TASK_RUNNABLE,
        TASK_PAUSED,

        TASK_BLOCKED,
        TASK_LONELY,
        TASK_MATCHED
    } task_state_t;

    typedef struct {
        int16_t selector;
        task_state_t state;

        bool lider;
        uint32_t pareja_id;
        paddr_t* shared_phys_base;
        bool shared_active;
    } sched_entry_t;

    ```

- Ahora podemos hacer las funciones:

    ``` C
    void conformar_pareja(task_id_t pareja_id) {
        task_id_t task_id = current_task;
        sched_entry_t* task_actual = &sched_tasks[task_id];
        
        if (pareja_id == 0) {
            task_actual->state = TASK_LONELY;
            task_actual->lider = true;

        } else {
            sched_entry_t* task_pareja = &sched_tasks[pareja_id];

            task_pareja->pareja_id = task_id;
            task_actual->pareja_id = pareja_id;

            task_pareja->state = TASK_MATCHED;
            task_actual->state = TASK_MATCHED;

            paddr_t base_fisica = mmu_alloc_region_4mb(); // tu función o equivalente
            task_pareja->shared_phys_base = base_fisica;
            task_pareja->shared_active = true;

            task_actual->shared_phys_base = base_fisica;
            task_actual->shared_active = true;

            // Mapear en ambos CR3
            for (uint32_t offset = 0; offset < 4 * 1024 * 1024; offset += PAGE_SIZE) {
                mmu_map_page(cr3_de(pareja_id), 0xC0C00000 + offset, base_fisica + offset, 1, 0); // solo lectura
                mmu_map_page(cr3_de(task_id),   0xC0C00000 + offset, base_fisica + offset, 1, 1); // lectura/escritura
            }

            // Reanudar a la líder, que estaba pausada
            sched_enable_task(pareja_id);
        }
    }

    void romper_pareja(void) {
        task_id_t task_id = current_task;
        sched_entry_t* task_actual = &sched_tasks[task_id];
        sched_entry_t* task_pareja = &sched_tasks[task_actual->pareja_id];
        
        uint32_t base = actual->shared_phys_base;
        uint32_t cr3 = rcr3();

        for (uint32_t offset = 0; offset < 4 * 1024 * 1024; offset += PAGE_SIZE)
            mmu_unmap_page(cr3, 0xC0C00000 + offset);

        task_actual->shared_active = false;
        task_actual->shared_phys_base = 0;

        if (task_actual->lider) {
            task_actual->state = TASK_BLOCKED;
        } else {
            if (task_pareja->state == TASK_BLOCKED) {
                mmu_free_region_4mb(base);
                task_pareja->shared_active = false;
                task_pareja->shared_phys_base = 0;
                task_pareja->state = TASK_RUNNABLE;
            }
        }
    }
    ```

- En el page fault handler agregamos lo siguiente
    ``` C
    ...
    if (addr >= 0xC0C00000 && addr < 0xC1000000) {
        sched_entry_t* actual = &sched_tasks[current_task];
        if (!actual->shared_active) return;

        uint32_t offset = addr - 0xC0C00000;
        uint32_t phys = mmu_next_free_user_page();

        // Limpio la página (usa una virtual temporal o zero_page)
        zero_page(addr);

        uint32_t base = actual->shared_phys_base;
        uint32_t pareja_id = actual->pareja_id;
        sched_entry_t* pareja = &sched_tasks[pareja_id];

        // Mapear en ambos CR3
        mmu_map_page(cr3_de(current_task), addr, base + offset, 1, actual->lider);
        mmu_map_page(cr3_de(pareja_id), addr, base + offset, 1, 0);
    }
    ...
    ```
