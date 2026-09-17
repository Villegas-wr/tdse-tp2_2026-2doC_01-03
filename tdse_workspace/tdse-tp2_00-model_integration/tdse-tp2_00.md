Para traducir un diagrama de estados a código C, el patrón más utilizado y robusto es estructurar una **Máquina de Estados Finitos (FSM)** mediante enumeraciones (`enum`) y sentencias `switch-case`.

Este enfoque divide el problema en tres partes fundamentales: los estados posibles, los eventos que disparan cambios, y la lógica que conecta ambos.

## 3 Pasos para la Implementación

1. **Definir Estados y Eventos:** Utiliza `enum` para que el código sea legible y auto-documentado. No uses números mágicos (0, 1, 2).
2. **Variable de Estado:** Crea una variable que almacene el estado actual del sistema.
3. **Lógica de Transición:** Construye una función que reciba el evento actual. Utiliza un `switch` para evaluar en qué estado estás, y dentro de cada caso, usa `if` para evaluar el evento y definir hacia qué nuevo estado saltar.

---

## Ejemplo Práctico: Molinete de Subte

Imagina el clásico diagrama de un molinete.

* **Estados:** `BLOQUEADO`, `DESBLOQUEADO`.
* **Eventos (Inputs):** `MONEDA` (insertar cospel), `EMPUJAR` (pasar por el molinete).

Aquí tienes la plantilla base en C de cómo se codifica este comportamiento:

```c
#include <stdio.h>

// 1. Definición de Estados y Eventos
typedef enum {
    ESTADO_BLOQUEADO,
    ESTADO_DESBLOQUEADO
} EstadoMolinete;

typedef enum {
    EVENTO_MONEDA,
    EVENTO_EMPUJAR
} Evento;

// Prototipo de la función que maneja la máquina
void procesarEvento(EstadoMolinete *estadoActual, Evento evento);

int main() {
    // 2. Inicializar el sistema en su estado base
    EstadoMolinete estado = ESTADO_BLOQUEADO;

    // Simulando una secuencia de uso real
    printf("--- Iniciando simulación ---\n");
    procesarEvento(&estado, EVENTO_EMPUJAR); // Intenta pasar sin pagar
    procesarEvento(&estado, EVENTO_MONEDA);  // Paga
    procesarEvento(&estado, EVENTO_EMPUJAR); // Pasa correctamente
    procesarEvento(&estado, EVENTO_EMPUJAR); // Intenta pasar un segundo sin pagar

    return 0;
}

// 3. Lógica de Transición
void procesarEvento(EstadoMolinete *estadoActual, Evento evento) {
    switch (*estadoActual) {
        
        case ESTADO_BLOQUEADO:
            if (evento == EVENTO_MONEDA) {
                printf("Accion: Moneda aceptada. Molinete DESBLOQUEADO.\n");
                *estadoActual = ESTADO_DESBLOQUEADO; // Transición de estado
            } 
            else if (evento == EVENTO_EMPUJAR) {
                printf("Error: El molinete esta bloqueado. Inserte moneda.\n");
                // No hay transición, se mantiene en BLOQUEADO
            }
            break;

        case ESTADO_DESBLOQUEADO:
            if (evento == EVENTO_EMPUJAR) {
                printf("Accion: Persona paso. Molinete BLOQUEADO.\n");
                *estadoActual = ESTADO_BLOQUEADO; // Transición de estado
            } 
            else if (evento == EVENTO_MONEDA) {
                printf("Aviso: Ya esta desbloqueado. Moneda devuelta.\n");
                // No hay transición, se mantiene en DESBLOQUEADO
            }
            break;
    }
}

```

> **Consejo clave:** Para trabajos prácticos universitarios, los profesores suelen evaluar que no haya lógica de eventos fuera del bloque `switch`. Toda decisión sobre qué hacer con una entrada *debe* depender del estado en el que se encuentra el sistema.
