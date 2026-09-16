Para codificar una Máquina de Estados Finitos (FSM) en C existen dos enfoques principales según la complejidad del diagrama: la estructura basada en **switch-case** (ideal para sistemas sencillos) y la tabla de transiciones mediante **punteros a funciones** (ideal para proyectos escalables e industriales).

**1. Arquitectura con Switch-Case**
Es la implementación más común y fácil de depurar. Utiliza un tipo enumerado (`enum`) para definir los estados y una estructura `switch` dentro de un bucle para evaluar las transiciones según los eventos recibidos:

```c
typedef enum {
    ESTADO_INICIO,
    ESTADO_ESPERA,
    ESTADO_PROCESANDO,
    ESTADO_ERROR
} Estado;

void actualizar_fsm(Estado *estado_actual, int evento) {
    switch (*estado_actual) {
        case ESTADO_INICIO:
            if (evento == EV_START) *estado_actual = ESTADO_ESPERA;
            break;
        case ESTADO_ESPERA:
            if (evento == EV_DATA) *estado_actual = ESTADO_PROCESANDO;
            break;
        case ESTADO_PROCESANDO:
            if (evento == EV_OK) *estado_actual = ESTADO_INICIO;
            else if (evento == EV_ERR) *estado_actual = ESTADO_ERROR;
            break;
        case ESTADO_ERROR:
            // Lógica de recuperación
            break;
    }
}

```

**2. Arquitectura con Tabla de Estados y Punteros a Funciones**
Para diagramas más densos (máquinas tipo Mealy o Moore), conviene asociar a cada estado una función específica y una tabla de salto. Esto evita un `switch` gigante y desacopla la lógica:

```c
typedef enum { ESTADO_A, ESTADO_B, NUM_ESTADOS } EstadoId;

typedef struct {
    void (*accion_entrada)(void);
    EstadoId (*siguiente_estado)(int evento);
} EstadoHandler;

EstadoId trans_A(int ev) { return (ev == 1) ? ESTADO_B : ESTADO_A; }
void ejecutar_A(void) { /* Lógica de estado A */ }

const EstadoHandler tabla_fsm[NUM_ESTADOS] = {
    [ESTADO_A] = { ejecutar_A, trans_A },
    // ...
};

```

**Pasos Clave para Desarrollar el Trabajo Práctico**

* **Identificar Nodos y Arcos:** Enumera todos los estados posibles del diagrama y define con claridad los eventos que desencadenan cada transición.
* **Clasificar el Tipo de Máquina:** Determina si las salidas dependen únicamente del estado actual (Moore) o también de las entradas en el momento de la transición (Mealy).
* **Prevenir Estados Indefinidos:** Agrega siempre una cláusula `default` en el `switch` para manejar eventos no contemplados o caídas de flujo.
* **Separar Entradas y Control:** Diseña una función encargada de leer/simular los eventos (teclado, sensores, archivos) y otra dedicada únicamente a ejecutar la lógica de la FSM.

¿Podrías compartir el diagrama de estados específico que debes implementar o describir qué sistema necesita simular tu trabajo práctico?
