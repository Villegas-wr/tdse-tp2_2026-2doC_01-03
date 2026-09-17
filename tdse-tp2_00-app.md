### Análisis del Código Fuente

El conjunto de archivos implementa un sistema operativo a nivel básico (bare-metal) basado en el disparo por eventos/tiempo (Event/Time-Triggered System), donde se miden métricas de rendimiento para cada tarea.

* **`app.c`**: Es el núcleo de la aplicación. Define una lista de tareas (`task_cfg_list`) y arreglos de datos estadísticos (`task_dta_list`) para cada una. Contiene `app_init()` para preparar el sistema y `app_update()` que se encarga de despachar y temporizar las tareas iterativamente.


* **`app_it.c`**: Aloja las rutinas de atención de interrupciones a nivel de aplicación. En particular, maneja el callback del temporizador del sistema (`HAL_SYSTICK_Callback`), el cual simplemente incrementa la variable `g_app_tick_cnt`.


* **`logger.c` y `logger.h**`: Implementan un sistema de registro de eventos (logging). `logger.h` provee macros como `LOGGER_INFO` que desactivan las interrupciones globales temporalmente, formatean el texto con `snprintf` en un búfer interno, y luego utilizan `printf` (vía *semihosting*) para enviar el mensaje a la consola de depuración, antes de volver a habilitar las interrupciones.


* **`systick.c`**: Proporciona una función de retardo bloqueante (`systick_delay_us`) calculando ciclos directamente sobre los registros de hardware del temporizador SysTick.


* **`dwt.h`**: Proporciona una capa de abstracción para el periférico DWT (Data Watchpoint and Trace) del núcleo ARM Cortex. Utiliza su contador de ciclos de hardware de 32 bits (`DWT->CYCCNT`) para medir tiempos de ejecución con precisión de microsegundos (`cycle_counter_get_time_us`) sin depender de interrupciones.



---

### Evolución de Variables en el Sistema

#### Durante la inicialización (`app_init` de `app.c`)

Al arrancar, la función `app_init()` establece el estado inicial del sistema:

* **`index`**: Opera como iterador del bucle `for` e incrementa desde `0` hasta `TASK_QTY - 1` para recorrer la lista de tareas configuradas.


* **Variables de la estructura `task_dta_list[index]**`:
* `NOE` (Número de ejecuciones): Se inicializa en `0`.


* `LET` (Último tiempo de ejecución, en microsegundos): Se inicializa en `0`.


* `BCET` (Mejor tiempo de ejecución, en microsegundos): Se preestablece en `1000` simulando un infinito inicial.


* `WCET` (Peor tiempo de ejecución, en microsegundos): Se inicializa en `0`.




* **`g_app_tick_cnt`**: Se inicializa en `0` cuando `app_init()` invoca a `app_it_init()`.



#### Durante las sucesivas ejecuciones (`app_update` de `app.c`)

La función `app_update()` es ejecutada perpetuamente en el bucle infinito. Evoluciona de la siguiente manera:

1. **`g_app_tick_cnt`**: Cuando la interrupción del SysTick ocurre, este valor es incrementado a espaldas del programa principal. En `app_update()`, si `g_app_tick_cnt > 0`, se decrementa bajo una sección crítica (interrupciones deshabilitadas) y se habilita la ejecución de las tareas (`b_time_update_required = true`).


2. **`g_app_runtime_us`**: Antes de despachar las tareas, se reinicia a `0`.


3. **`index`**: Vuelve a iterar desde `0` hasta `TASK_QTY - 1` ejecutando la función `task_update` correspondiente de cada tarea.


4. **Actualización de `task_dta_list[index]**`:


* `NOE`: Se incrementa en `1` para reflejar una nueva ejecución de la tarea.
* `LET`: Toma el valor retornado por `cycle_counter_get_time_us()`, reflejando exactamente cuántos microsegundos tardó la tarea en ejecutarse.
* `BCET`: Se compara con `LET`. Si `LET < BCET`, `BCET` se sobrescribe con el valor de `LET`.
* `WCET`: Se compara con `LET`. Si `LET > WCET`, `WCET` se sobrescribe con el valor de `LET`.


5. **`g_app_runtime_us`**: Acumula iterativamente el `LET` de cada tarea evaluada (`g_app_runtime_us += task_dta_list[index].LET`). Al finalizar el ciclo `for`, representa el tiempo total de procesamiento de la aplicación en microsegundos para ese "tick".



---

### Impacto de utilizar `LOGGER_INFO()` en el rendimiento

Llamar a `LOGGER_INFO()` dentro de una tarea de la aplicación perturbará drásticamente las métricas de rendimiento temporales del sistema debido a su arquitectura subyacente.

La macro `LOGGER_INFO` deshabilita las interrupciones para evitar problemas de concurrencia, utiliza procesamiento pesado (`snprintf`) para el manejo de cadenas, y realiza una llamada a `printf` apoyándose en el mecanismo de depuración "semihosting" (`LOGGER_CONFIG_USE_SEMIHOSTING = 1`). El *semihosting* detiene virtualmente la ejecución normal de la CPU para que el depurador de hardware (ST-Link, J-Link) extraiga la cadena de caracteres hacia la PC host.

Dado que la medición del tiempo en `app_update()` se realiza evaluando contadores de hardware directos del microcontrolador (DWT) antes y después de la tarea, este mecanismo contabilizará todo el tiempo bloqueado por el semihosting y el procesamiento de cadenas. Como consecuencia:

* **Impacto en `task_dta_list[index].WCET**`: El `LET` resultante de la tarea será desproporcionadamente largo. Al ser comparado con el máximo histórico, sobrescribirá el `WCET`, registrando un "Peor Tiempo de Ejecución" distorsionado y gigantesco.


* **Impacto en `g_app_runtime_us**`: Dado que acumula el `LET` de todas las tareas, presentará un salto enorme en el ciclo en el que se imprimió el registro. Esto destruye la evaluación determinística del sistema, indicando falsamente que la aplicación no puede cumplir con sus plazos estrictos en tiempo real.
