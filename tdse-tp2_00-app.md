A continuación, se presenta el análisis del funcionamiento de los archivos proporcionados y la evolución temporal de las variables del sistema.

### 1. Análisis del funcionamiento de los archivos

Este conjunto de archivos conforma una arquitectura de software *Bare Metal* basada en un planificador cooperativo disparado por eventos temporales (Event-Triggered System). 

* **`app.c`**: Es el núcleo de la aplicación. Implementa un planificador (scheduler) básico que administra una lista de tareas (sensor, sistema y actuador). Su función es inicializar estas tareas (`app_init`) y luego ejecutarlas de forma secuencial y periódica (`app_update`), midiendo al mismo tiempo cuánto tarda en ejecutarse cada una para tener un perfil de rendimiento (profiling).
* **`logger.c` y `logger.h`**: Proporcionan un módulo para imprimir mensajes de depuración (logs). Utilizan la técnica de *semihosting* (`printf` redirigido a través de la interfaz de depuración JTAG/SWD hacia la PC del desarrollador) y deshabilitan temporalmente las interrupciones (`__asm("CPSID i")`) para armar de forma segura el mensaje (usando `snprintf`) antes de enviarlo.
* **`systick.c`**: Contiene una utilidad de retardo bloqueante (`systick_delay_us`). A diferencia del clásico `HAL_Delay()` que usa milisegundos, esta función lee directamente el registro del hardware del SysTick (`SysTick->VAL`) para lograr retardos muy precisos en el orden de los microsegundos.
* **`dwt.h`** *(Inferido por su uso en app.c)*: Es la interfaz para el hardware DWT (Data Watchpoint and Trace) del núcleo Cortex-M. Provee las funciones `cycle_counter_*` que cuentan los ciclos exactos de reloj de la CPU, permitiendo medir tiempos de ejecución en microsegundos de manera extremadamente precisa sin depender de interrupciones externas.

---

### 2. Evolución de las variables clave

A continuación se detalla cómo cambian estas variables desde el inicio (`app_init()`) y a lo largo del bucle infinito (`app_update()`):

* **`g_app_tick_cnt`** (Unidad: **Ticks / Numeral puro**)
    * *Inicio*: Se inicializa en 0 (`G_APP_TICK_CNT_INI`) dentro de `app_init()`.
    * *Evolución*: Aumenta en 1 cada milisegundo de manera asíncrona dentro de `HAL_SYSTICK_Callback()` (llamada por la interrupción de hardware). En el bucle de `app_update()`, si este contador es mayor a 0, se decrementa en 1 y se habilita la bandera `b_time_update_required`, lo que desencadena la ejecución de todas las tareas. Funciona como un token de sincronización.
* **`index`** (Unidad: **Numeral / Índice puro**)
    * *Inicio y Evolución*: Es una variable local que sirve como contador para los bucles `for`. En cada pasada, iterará desde 0 hasta 2 (ya que `TASK_QTY` es 3), permitiendo acceder a los datos y funciones de cada tarea secuencialmente.
* **`g_app_runtime_us`** (Unidad: **Microsegundos, $\mu s$**)
    * *Inicio*: No está explícitamente inicializada en `app_init` (por ser global arranca en 0).
    * *Evolución*: Al comienzo del bloque de ejecución en `app_update()`, se reinicia a `0`. Luego, por cada tarea ejecutada, se le va sumando el tiempo que tardó esa tarea (`LET`). Al finalizar el ciclo, contiene el tiempo total de procesamiento que tomó correr *todas* las tareas en esa iteración.
* **`task_dta_list[index].NOE`** (Number of Executions) - (Unidad: **Numeral / Cantidad**)
    * *Inicio*: Se inicializa en 0 (`TASK_X_NOE_INI`).
    * *Evolución*: Cada vez que la tarea correspondiente es llamada y finaliza su actualización en `app_update()`, este valor se incrementa en `1`. Lleva el registro histórico de cuántas veces corrió la tarea.
* **`task_dta_list[index].LET`** (Last Execution Time) - (Unidad: **Microsegundos, $\mu s$**)
    * *Inicio*: Se inicializa en 0 (`TASK_X_LET_INI`).
    * *Evolución*: En `app_update()`, justo antes de llamar a la tarea se resetea el contador de ciclos (`cycle_counter_reset`). Al volver de la tarea, se lee el tiempo (`cycle_counter_get_time_us()`) y se guarda en `LET`. Representa cuánto tardó la tarea en su *última* ejecución.
* **`task_dta_list[index].BCET`** (Best-Case Execution Time) - (Unidad: **Microsegundos, $\mu s$**)
    * *Inicio*: Se inicializa en un valor intencionalmente alto, típicamente `1000` (`TASK_X_BCET_INI`).
    * *Evolución*: Si en el ciclo actual el `LET` medido resulta ser *menor* que el `BCET` histórico, el `BCET` se sobreescribe con el nuevo `LET` (`BCET = LET`). Sirve para registrar el tiempo más rápido que la tarea ha logrado ejecutarse.
* **`task_dta_list[index].WCET`** (Worst-Case Execution Time) - (Unidad: **Microsegundos, $\mu s$**)
    * *Inicio*: Se inicializa en `0` (`TASK_X_WCET_INI`).
    * *Evolución*: Inversamente al BCET, si el `LET` actual es *mayor* que el `WCET` histórico, este se actualiza (`WCET = LET`). Sirve para atrapar el pico máximo de tiempo (el caso más lento) que demoró la tarea. Es vital para análisis de tiempos de respuesta en sistemas críticos.

---

### 3. Impacto de usar `LOGGER_INFO()` en el código

Utilizar la macro `LOGGER_INFO()` dentro de la función de actualización de una tarea (`task_update`) tiene un **impacto drástico y muy perjudicial** sobre los tiempos medidos del sistema.

Esto ocurre por dos motivos:
1.  **Semihosting (`printf`)**: Para enviar el texto hacia la PC del depurador, el procesador debe pausar la ejecución del programa normal y comunicarse a través de un protocolo complejo (JTAG/SWD). Esto toma muchísimo tiempo en la escala del microcontrolador.
2.  **Formateo de strings**: Utiliza `snprintf`, una función estándar de C que es inherentemente lenta y consume muchos ciclos de procesamiento.

**Efecto directo en las variables:**
* **`task_dta_list[index].WCET`**: Dado que el envío del texto toma una cantidad enorme de microsegundos, el tiempo de ejecución en ese ciclo particular (`LET`) será altísimo. En consecuencia, el **`WCET` atrapará este pico** y quedará permanentemente arruinado, mostrando un valor enorme que no representa la verdadera carga de procesamiento lógico de la tarea, sino la latencia de la herramienta de *debug*.
* **`g_app_runtime_us`**: Como esta variable suma los tiempos de ejecución de todas las tareas del ciclo actual, experimentar un retardo enorme por la impresión en pantalla causará que el `g_app_runtime_us` de ese ciclo específico se dispare, consumiendo casi con seguridad toda la ventana de tiempo estipulada (ej. los 1 ms del *tick*), lo que podría causar que el planificador pierda los plazos (deadlines) temporales del sistema y la aplicación se desincronice.
