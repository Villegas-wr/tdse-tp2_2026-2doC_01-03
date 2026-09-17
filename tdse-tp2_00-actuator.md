**Evolución de variables (index, tick, state, event y flag)**

* **`index`:** Durante la ejecución de `task_actuator_init()`, la variable `index` inicia en 0 y se incrementa hasta evaluar todos los actuadores configurados, cuyo límite está dado por `ACTUATOR_DTA_QTY` (en este caso, 1). En las sucesivas ejecuciones de `task_actuator_update()`, el `index` vuelve a iterar de la misma manera para procesar la máquina de estados de cada actuador.


* **`task_actuator_dta_list[index].tick`:** Esta variable representa el tiempo en milisegundos (mS). En el código proporcionado, no se inicializa de forma explícita en la función de inicio, pero se reinicia a `DEL_LED_MIN` (0 ul) si la máquina de estados entra en su condición por defecto (`default`).


* **`task_actuator_dta_list[index].state`:** En el inicio (`task_actuator_init()`), el estado se fuerza al valor inicial `ST_LED_IDLE`. Durante el loop de `task_actuator_update()`, este estado evolucionará a `ST_LED_ACTIVE` (y viceversa) únicamente cuando se presenten y validen los eventos correspondientes.


* **`task_actuator_dta_list[index].event`:** Se inicializa por defecto en `EV_LED_IDLE`. Su valor permanece intacto durante los ciclos de actualización hasta que una llamada externa lo modifique.


* **`task_actuator_dta_list[index].flag`:** Al iniciar, se establece en `false`. Su propósito es indicar si hay un evento sin procesar; durante el loop, la máquina de estados lo vuelve a fijar en `false` inmediatamente después de consumir un evento válido.



**Comportamiento de la función `task_actuator_statechart(uint32_t index)**`

Esta función implementa una Máquina de Estados Finitos (FSM) no bloqueante que controla el comportamiento del hardware (LED) según eventos externos:

* **Estado `ST_LED_IDLE`:** Si el actuador está inactivo, verifica si hay una bandera activa (`flag == true`) y si el evento reportado es `EV_LED_ACTIVE`. De ser así, consume el evento (`flag = false`), enciende el LED utilizando `HAL_GPIO_WritePin`, y transiciona al estado `ST_LED_ACTIVE`.


* **Estado `ST_LED_ACTIVE`:** Si el actuador está activo, evalúa si existe un evento pendiente (`flag == true`) del tipo `EV_LED_IDLE`. Al cumplirse la condición, baja la bandera (`flag = false`), apaga el LED, y retorna el sistema al estado `ST_LED_IDLE`.


* **Caso `default`:** Actúa como un mecanismo de seguridad; si el estado se corrompe, reinicia los parámetros clave (`tick`, `state`, `event`, `flag`) a sus valores mínimos o de reposo.



**Evolución de variables asociadas al `identifier` (Interacción con la Interfaz)**

* **`identifier`:** Cumple el rol análogo al `index`, identificando qué actuador específico se desea alterar desde un punto externo a la tarea principal (por ejemplo, `ID_LED_A`).


* **`task_actuator_dta_list[identifier].event` y `flag`:** Aunque `task_actuator_init()` y `task_actuator_update()` no generan eventos por sí solos de manera cíclica, la aplicación cuenta con la función `put_event_task_actuator()`. Al ejecutarse, esta función actualiza la variable `event` del identificador solicitado con un nuevo estado y fuerza su variable `flag` a `true`.


* **Resolución:** Una vez que el componente externo altera el `event` y levanta el `flag` usando el `identifier`, el próximo ciclo del loop principal (`task_actuator_update()`) detectará la bandera, ejecutará la transición del hardware y volverá a fijar el `flag` en `false` para esperar un nuevo evento.
