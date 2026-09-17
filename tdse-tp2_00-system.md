**Funcionamiento General del Sistema**

El conjunto de archivos implementa la tarea de control central (`Task System`) dentro de una arquitectura guiada por eventos. Su función es servir de intermediario entre los sensores y los actuadores: desacopla el procesamiento recibiendo eventos desde una cola circular FIFO (`event_task_system_queue`) y, según el estado actual de la máquina de estados (FSM), notifica a la tarea del actuador invocando funciones de interfaz (`put_event_task_actuator`).

---

**Evolución de Variables de `Task System**`
*(Unidad de medida de `tick`: milisegundos / ticks del sistema)*.

* **`index`**: Durante `task_system_init()`, toma el valor `0` para iterar sobre la lista `task_system_dta_list` (de tamaño `SYSTEM_DTA_QTY = 1`). Durante `task_system_update()`, se utiliza el índice correspondiente a `NORMAL` (`0`).


* **`task_system_dta_list[index].tick`**: Permanece en `0` (asignado por la sección `.bss`) durante la inicialización. Solo se actualiza al valor `DEL_SYS_MIN` (`0`) si la FSM ingresa al caso `default`.


* **`task_system_dta_list[index].state`**:
* En `task_system_init()`: Se inicializa en `ST_SYS_IDLE` (`0`).


* En `task_system_update()`: Cambia a `ST_SYS_ACTIVE` (`1`) al recibir un evento `EV_SYS_ACTIVE`, y retorna a `ST_SYS_IDLE` (`0`) al recibir `EV_SYS_IDLE`.




* **`task_system_dta_list[index].event`**:
* En `task_system_init()`: Se inicializa en `EV_SYS_IDLE` (`0`).


* En `task_system_update()`: Toma el valor del evento desencolado de la estructura FIFO (`EV_SYS_ACTIVE` o `EV_SYS_IDLE`) mediante `get_event_task_system()`.




* **`task_system_dta_list[index].flag`**:
* En `task_system_init()`: Se inicializa en `false`.


* En `task_system_update()`: Pasa a `true` al detectar un evento disponible con `any_event_task_system()`. Se restablece a `false` inmediatamente después de procesar la transición de estado correspondiente.





---

**Comportamiento de la Función `task_system_normal_statechart()**`

(Nota: En `task_system.c`, la lógica de la máquina de estados para el modo normal está implementada bajo la función `task_system_normal_statechart`).

1. Verifica si hay eventos disponibles en la cola mediante `any_event_task_system()`.


2. Si existe un evento, enciende la bandera (`flag = true`) y lo desencola guardándolo en la variable `event` mediante `get_event_task_system()`.


3. Evalúa el estado actual (`state`):


* **`ST_SYS_IDLE`**: Si `flag == true` y `event == EV_SYS_ACTIVE`, limpia la bandera (`flag = false`), envía el evento `EV_LED_ACTIVE` al actuador `ID_LED_A` mediante `put_event_task_actuator()` y cambia el estado a `ST_SYS_ACTIVE`.


* **`ST_SYS_ACTIVE`**: Si `flag == true` y `event == EV_SYS_IDLE`, limpia la bandera (`flag = false`), envía el evento `EV_LED_IDLE` al actuador `ID_LED_A` mediante `put_event_task_actuator()` y cambia el estado a `ST_SYS_IDLE`.


* **`default`**: Reinicia los atributos del sistema a `tick = 0`, `state = ST_SYS_IDLE`, `event = EV_SYS_IDLE` y `flag = false`.





---

**Evolución de Variables de la Cola `event_task_system_queue**`

* **En `task_system_init()` (invoca a `init_event_task_system()`)**:


* `i`: Incrementa de `0` a `15` en el bucle `for` para recorrer todo el arreglo.


* `event_task_system_queue.head` = `0`.


* `event_task_system_queue.tail` = `0`.


* `event_task_system_queue.count` = `0`.


* `event_task_system_queue.queue[0..15]` = `EMPTY` (`255`).




* **En `task_system_update()**`:


* Si no hay eventos (`head == tail`), la estructura no sufre cambios.


* Al desencolar un evento con `get_event_task_system()`:


* `count`: Se decrementa en `1`.


* `queue[tail]`: Se lee el evento contenido y luego la posición se sobrescribe con `EMPTY` (`255`).


* `tail`: Incrementa en `1` (vuelve a `0` si alcanza `QUEUE_LENGTH` / `16`).


* `head`: Permanece inalterado durante la lectura (solo es modificado al encolar eventos desde la tarea sensora).







---

**Evolución de Variables del Actuador (`task_actuator_dta_list`)**

* **En `task_system_init()**`: No son modificadas dentro del ámbito de este módulo (su inicialización ocurre dentro de `task_actuator_init()`).


* **En `task_system_update()**`:
* `identifier`: Recibe el identificador `ID_LED_A` (`0`) enviado desde `task_system_normal_statechart()` hacia `put_event_task_actuator()`.


* `task_actuator_dta_list[ID_LED_A].event`: Se actualiza a `EV_LED_ACTIVE` (`1`) al ingresar a `ST_SYS_ACTIVE`, o a `EV_LED_IDLE` (`0`) al regresar a `ST_SYS_IDLE`.


* `task_actuator_dta_list[ID_LED_A].flag`: Se establece en `true` indicando a la tarea del actuador que existe un evento pendiente de ejecución.
