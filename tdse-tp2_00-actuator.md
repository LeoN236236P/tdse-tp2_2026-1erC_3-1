Este sistema implementa un controlador de actuador (específicamente un LED) utilizando una arquitectura de **Máquina de Estados Finitos (FSM)** no bloqueante, diseñada para ejecutarse en sistemas embebidos (STM32/HAL).

A continuación, el análisis detallado del funcionamiento y la evolución de sus variables.

---

## 1. Análisis de los Archivos

* **`task_actuator_attribute.h`**: Define las estructuras de datos. Separa la **configuración** (constante: puerto, pin, tiempos) del **estado dinámico** (variable: estado actual, evento, bandera).
* **`task_actuator.c`**: Contiene la lógica principal. Implementa la inicialización (`init`), el ciclo de actualización (`update`) y la lógica de la máquina de estados (`statechart`).
* **`task_actuator_interface.c`**: Provee una función puente (`put_event_task_actuator`) para que otras tareas envíen eventos al actuador de forma segura.

---

## 2. Comportamiento de `task_actuator_statechart(uint32_t index)`

Esta función es el "corazón" del actuador. Evalúa el estado actual del componente identificado por `index` y decide si debe transicionar a otro estado basándose en los eventos recibidos.


### Lógica de Transición:
1.  **Si está en `ST_LED_IDLE`**: Espera a que `flag` sea `true` y el evento sea `EV_LED_ACTIVE`. Al cumplirse, apaga la bandera, enciende el LED físicamente y cambia a `ST_LED_ACTIVE`.
2.  **Si está en `ST_LED_ACTIVE`**: Espera a que `flag` sea `true` y el evento sea `EV_LED_IDLE`. Al cumplirse, apaga la bandera, apaga el LED físicamente y vuelve a `ST_LED_IDLE`.
3.  **Default**: Si el estado es inválido, resetea todas las variables a sus valores por defecto (Idle).

---

## 3. Evolución de Variables: `task_actuator_init` y `update`

A continuación se detalla cómo cambian las variables desde el arranque. Se asume un único actuador (`index = 0`, `ID_LED_A`).

### Fase de Inicio: `task_actuator_init()`
Al ejecutarse una sola vez al principio:

| Variable | Valor / Estado | Comentario |
| :--- | :--- | :--- |
| `index` | 0 | Identificador del primer (y único) actuador. |
| `dta_list[0].tick` | No inicializado explícitamente* | La unidad es **milisegundos (mS)**. |
| `dta_list[0].state` | `ST_LED_IDLE` | El sistema comienza en reposo. |
| `dta_list[0].event` | `EV_LED_IDLE` | No hay eventos pendientes. |
| `dta_list[0].flag` | `false` | No hay señales de cambio pendientes. |

> *Nota: El hardware se fuerza a `led_off` mediante `HAL_GPIO_WritePin`.

### Fase de Ejecución: `task_actuator_update()`
Esta función se llama repetidamente en el loop principal.

* **Escenario A (En espera):** Mientras no lleguen eventos externos, las variables se mantienen constantes (`state: ST_LED_IDLE`, `flag: false`).
* **Escenario B (Recepción de evento `EV_LED_ACTIVE`):**
    1. Una tarea externa llama a `put_event_task_actuator`.
    2. `flag` pasa a `true` y `event` a `EV_LED_ACTIVE`.
    3. En la siguiente ejecución de `update`, el statechart detecta la bandera, enciende el LED y cambia `state` a `ST_LED_ACTIVE`. `flag` vuelve a `false`.

---

## 4. Evolución de Variables de Interfaz (`identifier`, `event`, `flag`)

La función `put_event_task_actuator` es la que altera el flujo desde afuera. Su evolución depende de la interacción con otras tareas:

| Momento | `identifier` | `dta_list[id].event` | `dta_list[id].flag` | Acción resultante |
| :--- | :--- | :--- | :--- | :--- |
| **Inicio** | ID_LED_A | EV_LED_IDLE | false | Sistema en reposo. |
| **Petición Encendido** | ID_LED_A | **EV_LED_ACTIVE** | **true** | El statechart encenderá el LED. |
| **Post-Procesamiento** | ID_LED_A | EV_LED_ACTIVE | **false** | El statechart consume la bandera. |
| **Petición Apagado** | ID_LED_A | **EV_LED_IDLE** | **true** | El statechart apagará el LED. |

### Resumen de Unidades y Tipos:
* **`tick`**: La unidad de medida es **milisegundos (mS)**, derivada de `HAL_GetTick()`, aunque en este código actual el `tick` se actualiza principalmente en el caso `default` o se define en la configuración, no se ve un incremento activo dentro del `statechart` (parece preparado para una lógica de tiempo que aún no se usa para parpadeos en este fragmento).
* **`flag`**: Booleano que actúa como un semáforo para indicar que hay un evento nuevo "fresco" para ser procesado.
