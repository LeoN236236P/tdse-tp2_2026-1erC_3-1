Esta estructura es el esqueleto clásico de un proyecto generado por **STM32CubeMX** para un microcontrolador de la familia STM32F1 (específicamente un Cortex-M3 como el STM32F103).

---

### 1. Análisis de los archivos

#### `startup_stm32f103rbtx.s` (Código de inicio en Ensamblador)
Este es el primer código que se ejecuta en el microcontrolador cuando se enciende o se reinicia. Sus funciones principales son:
* **Definición de la Tabla de Vectores (`g_pfnVectors`):** Es una estructura en la dirección `0x0000 0000` (o la dirección mapeada de la Flash) que le dice al procesador dónde encontrar el puntero de la pila (`_estack`) y las direcciones de las rutinas de interrupción (como el `Reset_Handler`, `SysTick_Handler`, etc.).
* **`Reset_Handler`:** Es el punto de entrada real. Cuando ocurre un reset:
    1. Llama a `SystemInit` (una función externa que usualmente configura aspectos básicos del hardware de bajo nivel).
    2. Copia los datos inicializados (sección `.data`) de la memoria Flash a la memoria RAM.
    3. Llena con ceros la memoria no inicializada (sección `.bss`) en la RAM.
    4. Llama a inicializadores de librerías C estándar (`__libc_init_array`).
    5. Finalmente, hace un salto (branch) a la función `main()` en C.

#### `main.c` (Lógica principal de la aplicación)
Este archivo contiene la configuración de los periféricos y el bucle principal:
* **`HAL_Init()`:** Inicializa la librería HAL (Hardware Abstraction Layer), configurando el temporizador SysTick para que genere interrupciones cada 1 milisegundo.
* **`SystemClock_Config()`:** Configura el árbol de relojes (Clock Tree) del microcontrolador. Activa el oscilador interno (HSI), lo divide a la mitad y lo multiplica por 16 usando el PLL para acelerar el procesador.
* **`MX_GPIO_Init()` y `MX_USART2_UART_Init()`:** Inicializan los pines de entrada/salida (como el pin de un LED `LD2` y un botón `B1`) y el puerto serie (USART2 a 115200 baudios).
* **`app_init()` y `app_update()`:** Funciones de usuario. `app_init()` se ejecuta una vez antes del bucle, y `app_update()` se ejecuta repetitivamente dentro del `while(1)`.

#### `stm32f1xx_it.c` (Manejadores de Interrupciones)
Aquí aterrizan las excepciones del procesador y las interrupciones de los periféricos:
* **`SysTick_Handler(void)`:** Esta función es llamada automáticamente por hardware (usualmente cada 1 milisegundo). Llama a `HAL_IncTick()`, la cual incrementa una variable global interna (generalmente llamada `uwTick`) que el sistema usa para medir el paso del tiempo y manejar los *delays* (como `HAL_Delay()`).
* **`EXTI15_10_IRQHandler(void)`:** Es el manejador para las interrupciones externas de los pines 10 al 15. En este caso, atiende al botón `B1_Pin`.

---

### 2. Evolución de `SysTick` y `SystemCoreClock`

Es importante aclarar un detalle técnico: `SysTick` es físicamente un temporizador (un periférico de hardware dentro del núcleo Cortex-M3), mientras que `SystemCoreClock` es una variable global (definida por el estándar CMSIS) que almacena la frecuencia actual del procesador en hercios (Hz). 

Así es como evolucionan desde el encendido hasta llegar al `while(1)`:

**Fase 1: El reinicio (`Reset_Handler` en `startup_stm32f103rbtx.s`)**
* **`SystemCoreClock`:** El procesador arranca usando su oscilador interno de alta velocidad (HSI) por defecto. En los STM32F1, el HSI funciona a **8 MHz**. Por lo tanto, si consultáramos `SystemCoreClock` aquí, valdría `8000000`.
* **`SysTick`:** Está **apagado/deshabilitado**. Sus registros están en estado de reset.

**Fase 2: Entrada a `main()` e inicialización de HAL (`main.c`)**
* El código ejecuta `HAL_Init()`.
* **`SystemCoreClock`:** Sigue en **8 MHz**.
* **`SysTick`:** `HAL_Init()` enciende el SysTick. Calcula el valor de recarga basado en el reloj actual (8 MHz) para que se desborde exactamente cada 1 milisegundo. A partir de este momento, el `SysTick_Handler` empieza a dispararse 1000 veces por segundo.

**Fase 3: Configuración del Reloj (`SystemClock_Config()` en `main.c`)**
* Esta función cambia drásticamente la velocidad del sistema. Toma el oscilador HSI (8 MHz), lo divide por 2 (`RCC_PLLSOURCE_HSI_DIV2` = 4 MHz) y lo multiplica por 16 (`RCC_PLL_MUL16`). El resultado es 64 MHz.
* **`SystemCoreClock`:** Se actualiza internamente a **64 MHz** (`64000000`).
* **`SysTick`:** Como la velocidad del procesador se multiplicó por 8 (de 8 a 64 MHz), si el SysTick no se tocara, empezaría a contar 8 veces más rápido. Sin embargo, la función `HAL_RCC_ClockConfig()` (dentro de la configuración del reloj) **re-configura el SysTick** automáticamente, actualizando su valor de recarga (ahora será 64,000 ciclos de reloj por milisegundo en lugar de 8,000). Esto garantiza que siga interrumpiendo exactamente cada 1 ms a pesar del cambio de velocidad.

**Fase 4: El bucle principal (`while (1)` en `main.c`)**
* **`SystemCoreClock`:** Se mantiene estable en **64 MHz**.
* **`SysTick`:** Se mantiene funcionando de fondo. Su valor actual (registro `SysTick->VAL`) cuenta regresivamente desde 64,000 hasta 0 a una velocidad de 64 MHz. Cada vez que llega a cero, genera una interrupción, salta a `SysTick_Handler` en `stm32f1xx_it.c`, incrementa la variable interna de milisegundos (`uwTick`) y vuelve al `while(1)`.
