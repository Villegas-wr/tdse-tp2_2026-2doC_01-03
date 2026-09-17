A continuación, presento un análisis detallado del funcionamiento de los tres archivos de código fuente proporcionados, correspondientes a un proyecto típico de microcontroladores STM32 utilizando la librería HAL (Hardware Abstraction Layer) de STMicroelectronics. Luego, se detalla la evolución de las variables/sistemas `SysTick` y `SystemCoreClock`.

### Análisis de los archivos fuente

#### 1. `startup_stm32f103rbtx.s` (Archivo de arranque)

Este archivo está escrito en lenguaje ensamblador y es el primer código que se ejecuta cuando el microcontrolador se enciende o se reinicia (Reset). Sus funciones principales son:

* **Definición de la Tabla de Vectores:** Contiene la dirección inicial del puntero de pila (`_estack`) y las direcciones de todas las rutinas de servicio de interrupción (ISR), como `Reset_Handler`, `SysTick_Handler`, etc.


* **Rutina `Reset_Handler`:** Es el verdadero punto de entrada de la ejecución. Al dispararse, realiza lo siguiente:


* Llama a la función `SystemInit` (usualmente definida en un archivo `system_stm32f1xx.c` no adjunto) para realizar la configuración inicial de los relojes del sistema a su estado por defecto.


* Copia los valores iniciales de las variables globales inicializadas desde la memoria Flash hacia la memoria RAM (sección `.data`).


* Inicializa en cero las variables globales no inicializadas en la RAM (sección `.bss`).


* Llama a la inicialización de la biblioteca de C (`__libc_init_array`) y, finalmente, hace un salto (`bl main`) a la función principal en C.





#### 2. `main.c` (Programa Principal)

Contiene la lógica de inicialización y el bucle infinito de la aplicación.

* **Inicialización (`main`)**:
* Ejecuta `HAL_Init()`, la cual reinicia los periféricos e inicializa el temporizador SysTick.


* Llama a `SystemClock_Config()` para establecer la frecuencia de reloj del sistema. Configura el oscilador interno (HSI) y el PLL (Multiplicador de Fase) para elevar la frecuencia.


* Llama a funciones de inicialización de periféricos: `MX_GPIO_Init()` (configura pines, el LED `LD2` y el botón `B1` con interrupciones) y `MX_USART2_UART_Init()` (configura la comunicación serial).


* Ejecuta la función `app_init()` que inicializa la lógica de usuario de la aplicación.




* **Bucle principal (`while (1)`)**: Es un bucle infinito que llama constantemente a `app_update()`, donde reside la lógica continua de la aplicación.



#### 3. `stm32f1xx_it.c` (Rutinas de Servicio de Interrupción)

Este archivo gestiona las interrupciones del hardware.

* Contiene funciones "Handler" para fallos del sistema (como `HardFault_Handler`), las cuales entran en un bucle infinito `while(1)` si ocurre un error grave.


* **`SysTick_Handler()`:** Es la interrupción del temporizador del sistema. Se dispara periódicamente (generalmente cada 1 ms) y llama a `HAL_IncTick()`, la cual incrementa un contador global utilizado por la HAL para generar retardos (como `HAL_Delay`) y gestionar tiempos de espera (timeouts).


* **`EXTI15_10_IRQHandler()`:** Maneja las interrupciones externas para los pines del 10 al 15. En este caso, atiende la interrupción generada por el pin del botón (`B1_Pin`) delegando el manejo a `HAL_GPIO_EXTI_IRQHandler`.



---

### Evolución de `SysTick` y `SystemCoreClock`

A continuación, se traza cómo cambian el estado del temporizador `SysTick` y la variable global `SystemCoreClock` (que almacena la frecuencia de la CPU en Hz) desde el reinicio hasta el bucle principal:

1. **En `Reset_Handler` (`startup_stm32f103rbtx.s`):**
* `SysTick`: Está deshabilitado por hardware y no cuenta.


* `SystemCoreClock`: Al llamarse a `SystemInit`, esta variable se inicializa con la frecuencia por defecto del oscilador interno (HSI), que en la familia STM32F1 suele ser de 8 MHz (8,000,000 Hz).




2. **Al ejecutar `HAL_Init()` en `main.c`:**
* `SysTick`: Se configura y se enciende. La HAL lo programa para que genere una interrupción exactamente cada 1 milisegundo basándose en el reloj actual (que en este instante sigue siendo el inicial de 8 MHz). El contador interno (`uwTick`) comienza en 0.


* `SystemCoreClock`: Sigue siendo el valor por defecto (8 MHz).




3. **Al ejecutar `SystemClock_Config()` en `main.c`:**
* `SystemCoreClock`: Se modifica drásticamente. El código configura el oscilador HSI encendido (`RCC_HSI_ON`), toma su señal dividida por 2 (`RCC_PLLSOURCE_HSI_DIV2`, que resulta en 4 MHz) y la multiplica por 16 en el PLL (`RCC_PLL_MUL16`). Esto da como resultado **64 MHz** (64,000,000 Hz). La variable `SystemCoreClock` se actualiza internamente en la librería HAL a este nuevo valor.


* `SysTick`: Como la frecuencia del procesador cambió de 8 MHz a 64 MHz, la función `HAL_RCC_ClockConfig()` (llamada al final de la configuración) reconfigura automáticamente el temporizador `SysTick` para que siga interrumpiendo cada 1 ms a la nueva velocidad de 64 MHz.




4. **En el bucle principal `while (1)` en `main.c`:**
* `SystemCoreClock`: Se mantiene constante en 64 MHz (64,000,000 Hz) durante toda la ejecución normal de la aplicación.


* `SysTick`: Se mantiene en continuo funcionamiento en segundo plano. Cada 1 milisegundo interrumpe la ejecución de `app_update()`, salta a `SysTick_Handler()`, incrementa el valor de la variable del contador de "ticks" mediante `HAL_IncTick()` y regresa al bucle `while (1)`.
