Este conjunto de archivos implementa la abstracción de una tarea sensora para lectura de pulsadores y la gestión de eventos hacia el sistema mediante una cola circular FIFO.

**Análisis de los Archivos Fuente**

* `task_sensor_attribute.h` y `task_system_attribute.h`: Definen las enumeraciones de eventos (`EV_BTN_UP`, `EV_BTN_DOWN`, `EV_SYS_IDLE`, `EV_SYS_ACTIVE`) y estados (`ST_BTN_IDLE`, `ST_BTN_ACTIVE`), además de las estructuras de configuración (`task_sensor_cfg_t`) y datos dinámicos (`task_sensor_dta_t`, `task_system_dta_t`).


* `task_sensor.c`: Contiene las funciones de inicialización (`task_sensor_init`) y actualización (`task_sensor_update`), manejando el estado del pulsador mediante `task_sensor_statechart`.


* `task_system_interface.c`: Implementa una cola circular FIFO de 16 elementos (`event_task_system_queue`) para enviar y recibir eventos entre tareas de forma desacoplada (`put_event_task_system`, `get_event_task_system`).



**Comportamiento de `task_sensor_statechart(uint32_t index)**`

1. Lee el pin GPIO del pulsador configurado en `task_sensor_cfg_list[index]`.


2. Determina el evento: asigna `EV_BTN_DOWN` si el botón está presionado o `EV_BTN_UP` si está liberado.


3. Evalúa la máquina de estados (`p_task_sensor_dta->state`):


* En `ST_BTN_IDLE`: Si detecta `EV_BTN_DOWN`, envía la señal `signal_down` (`EV_SYS_ACTIVE`) a la cola del sistema mediante `put_event_task_system()` y cambia el estado a `ST_BTN_ACTIVE`.


* En `ST_BTN_ACTIVE`: Si detecta `EV_BTN_UP`, envía la señal `signal_up` (`EV_SYS_IDLE`) a la cola del sistema y pasa al estado `ST_BTN_IDLE`.


* En caso `default`: Reinicia `tick` a 0 (`DEL_BTN_MIN`), `state` a `ST_BTN_IDLE` y `event` a `EV_BTN_UP`.





**Evolución de Variables de la Tarea Sensor**
Unidades de medida: `index` (adimensional / entero), `tick` (milisegundos / ticks de temporización), `state` (adimensional / enum `task_sensor_st_t`), `event` (adimensional / enum `task_sensor_ev_t`).

* **Durante `task_sensor_init()**`:
* `index`: Toma el valor `0` (recorre desde 0 hasta `SENSOR_DTA_QTY - 1`).


* `task_sensor_dta_list[0].state`: Se inicializa en `ST_BTN_IDLE` (0).


* `task_sensor_dta_list[0].event`: Se inicializa en `EV_BTN_UP` (0).


* `task_sensor_dta_list[0].tick`: Permanece en `0` (no se inicializa explícitamente en la función, conservando el valor de la memoria `.bss`).




* **Durante ejecuciones de `task_sensor_update()**`:
* `index`: Mantiene el valor `0` en cada ciclo.


* **Pulsador en reposo (suelto)**: `event = EV_BTN_UP`, `state = ST_BTN_IDLE`, `tick = 0`.


* **Al presionar el pulsador**: `event` pasa a `EV_BTN_DOWN` y `state` cambia a `ST_BTN_ACTIVE`.


* **Pulsador mantenido presionado**: `event = EV_BTN_DOWN` y `state` permanece en `ST_BTN_ACTIVE`.


* **Al soltar el pulsador**: `event` pasa a `EV_BTN_UP` y `state` regresa a `ST_BTN_IDLE`.





**Evolución de Variables de la Cola `event_task_system_queue**`

* **En la inicialización (`init_event_task_system()`)**:
* `event_task_system_queue.head` = `0`.


* `event_task_system_queue.tail` = `0`.


* `event_task_system_queue.count` = `0`.


* `event_task_system_queue.queue[0..15]` = `EMPTY` (`255`).




* **Durante ejecuciones de `task_sensor_update()**`:
* **Sin cambios en el pulsador**: Las variables de la cola se mantienen inalteradas.
* **Evento Presionar Botón (`ST_BTN_IDLE` → `ST_BTN_ACTIVE`)**:
* `queue[head]` almacena el evento `EV_SYS_ACTIVE` (`1`).


* `count` se incrementa en `1` (p. ej., de `0` a `1`).


* `head` incrementa en `1` (p. ej., de `0` a `1`), volviendo a `0` si alcanza `QUEUE_LENGTH` (`16`).


* `tail` no se modifica durante la inserción.




* **Evento Soltar Botón (`ST_BTN_ACTIVE` → `ST_BTN_IDLE`)**:
* `queue[head]` almacena el evento `EV_SYS_IDLE` (`0`).


* `count` se incrementa en `1`.


* `head` se incrementa en `1` (módulo 16).


* `tail` permanece sin cambios hasta que la tarea del sistema invoque `get_event_task_system()` para consumir los eventos encolados.
