Implementar diagramas de estado (también conocidos como Máquinas de Estados Finitos o FSM por sus siglas en inglés) en C es un patrón de diseño clásico, súper útil y muy común, especialmente en sistemas embebidos, automatización y desarrollo de videojuegos.



Para ayudarte a armar tu Trabajo Práctico, te propongo dividir la explicación en la teoría básica de cómo se traduce un diagrama a código y luego ver un ejemplo práctico completo.

### 1. Conceptos Básicos: Del Diagrama al Código

En cualquier diagrama de estados tienes tres elementos principales, que en C se traducen de una forma muy específica:

* **Estados (States):** Son las "burbujas" en tu diagrama. En C, la mejor forma de representarlos es utilizando un tipo enumerado (`enum`). Esto hace que el código sea muy legible.
* **Eventos (Events):** Son las "flechas" que conectan los estados. Son las acciones que disparan los cambios. También se suelen definir con un `enum`.
* **Transiciones y Acciones:** Es la lógica de "si estoy en el estado X y ocurre el evento Y, entonces hago Z y paso al estado W". Esto se suele implementar mediante una estructura de control `switch-case`.

---

### 2. El Método Estándar: `switch-case` y `enum`

La forma más enseñada en las universidades y más fácil de implementar para máquinas de estado pequeñas/medianas es usar una función principal que evalúa el estado actual con un `switch`, y dentro de cada `case`, evalúa los eventos con `if/else` (o con otro `switch`).

#### Ejemplo Práctico: Un Molinete de Subte (Torno)
Imagina el clásico diagrama de un molinete. 
* **Estados:** `BLOQUEADO` y `DESBLOQUEADO`.
* **Eventos:** `INSERTAR_MONEDA` y `EMPUJAR`.

Aquí tienes cómo se codifica eso en C:

```c
#include <stdio.h>

// 1. Definimos los Estados del sistema
typedef enum {
    ESTADO_BLOQUEADO,
    ESTADO_DESBLOQUEADO
} EstadoMolinete;

// 2. Definimos los Eventos posibles
typedef enum {
    EVENTO_MONEDA,
    EVENTO_EMPUJAR
} EventoMolinete;

// 3. Variable global para guardar el estado actual 
// (En un programa más grande, esto podría ir en un `struct`)
EstadoMolinete estado_actual = ESTADO_BLOQUEADO;

// 4. La Máquina de Estados (La lógica de transición)
void procesar_evento(EventoMolinete evento) {
    
    switch (estado_actual) {
        
        case ESTADO_BLOQUEADO:
            if (evento == EVENTO_MONEDA) {
                // Acción
                printf("Moneda aceptada. Desbloqueando molinete...\n");
                // Transición al nuevo estado
                estado_actual = ESTADO_DESBLOQUEADO; 
            } 
            else if (evento == EVENTO_EMPUJAR) {
                printf("El molinete esta bloqueado. Inserte una moneda.\n");
            }
            break;

        case ESTADO_DESBLOQUEADO:
            if (evento == EVENTO_EMPUJAR) {
                // Acción
                printf("Persona paso. Bloqueando molinete...\n");
                // Transición al nuevo estado
                estado_actual = ESTADO_BLOQUEADO;
            } 
            else if (evento == EVENTO_MONEDA) {
                printf("El molinete ya esta desbloqueado. Moneda devuelta.\n");
            }
            break;
    }
}

int main() {
    printf("--- INICIO DE LA SIMULACION ---\n");
    
    // Simulamos una secuencia de eventos
    procesar_evento(EVENTO_EMPUJAR); // Intenta pasar sin pagar
    procesar_evento(EVENTO_MONEDA);  // Paga
    procesar_evento(EVENTO_MONEDA);  // Intenta pagar de nuevo
    procesar_evento(EVENTO_EMPUJAR); // Pasa
    
    return 0;
}
```

---

### 3. Métodos Avanzados (Para sumar puntos en el T.P.)

Si tu trabajo práctico exige un nivel un poco más avanzado, puedes mencionar o implementar otras arquitecturas en C:

* **Tabla de Transiciones (Matriz de estados):** En lugar de usar `switch-case`, se usa una matriz bidimensional (un array de arrays) donde las filas son los estados y las columnas los eventos. Cada celda contiene el siguiente estado y un puntero a la función que ejecuta la acción. Es ideal para máquinas con muchísimos estados porque el código no crece de forma desordenada.
* **Punteros a Funciones (State Pattern en C):** El "estado actual" no es un `enum`, sino un puntero a la función que representa ese estado. Cuando cambias de estado, simplemente cambias a qué función apunta el puntero.

¿De qué trata específicamente el diagrama de estados que te asignaron para este Trabajo Práctico (por ejemplo, es un semáforo, un cajero automático, un motor)?
