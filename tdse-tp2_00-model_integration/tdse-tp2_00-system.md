### 1. Análisis del Funcionamiento General

El conjunto de estos archivos implementa la **lógica central (Sistema)** y la **comunicación hacia la salida (Actuador)** de una arquitectura disparada por eventos:

* **`task_system.c` y `task_system_attribute.h`**: Conforman la tarea principal del sistema. Esta tarea actúa como el "cerebro", consumiendo los eventos generados por otras tareas (como el sensor) y decidiendo qué estado debe asumir el sistema en consecuencia.


* **`task_system_interface.c`**: Proporciona el mecanismo de cola circular (FIFO) para que el sistema reciba eventos de forma segura y asíncrona.


* **`task_actuator_interface.c` y `task_actuator_attribute.h`**: Definen la vía de comunicación hacia el hardware de salida. Cuando el sistema decide que hay que hacer algo (ej. encender un LED), utiliza esta interfaz para enviarle un evento directo a la tarea del actuador.



---

### 2. Evolución de las Variables de la Tarea del Sistema

A continuación se detalla la evolución desde la inicialización en `task_system_init()` y durante el bucle `task_system_update()`.

* **`index`**:
* En `task_system_init()`, es una variable local utilizada en el bucle `for`. Como `SYSTEM_DTA_QTY` equivale a `MODE_QTY` (que vale 1), `index` toma el valor **0** para inicializar el modo `NORMAL`, y luego el bucle finaliza.


* En `task_system_update()`, esta variable no existe. En su lugar, se utiliza directamente el índice definido por `g_task_system_mode` (que vale `NORMAL` o 0).




* **`task_system_dta_list[index].tick`**:
* **Unidad:** Ticks del sistema (milisegundos).
* **Evolución:** No se inicializa explícitamente en `task_system_init()` (asume el valor por defecto en memoria). En `task_system_update()`, **no se modifica** en el flujo normal. Solo tomaría el valor `DEL_SYS_MIN` (0) si la máquina de estados cayera accidentalmente en el caso `default` del `switch`.




* **`task_system_dta_list[index].state`**:
* **Evolución:** Se inicializa en **`ST_SYS_IDLE`** durante el inicio. En el loop principal, si se extrae de la cola el evento de activación, transita al estado **`ST_SYS_ACTIVE`**. Si posteriormente se recibe un evento de inactividad, retorna a **`ST_SYS_IDLE`**.




* **`task_system_dta_list[index].event`**:
* **Evolución:** Inicia forzado en **`EV_SYS_IDLE`**. Durante las actualizaciones, su valor es sobrescrito constantemente con lo que devuelve `get_event_task_system()`, adoptando **`EV_SYS_ACTIVE`** o **`EV_SYS_IDLE`** según lo que haya en la cola.




* **`task_system_dta_list[index].flag`**:
* **Evolución:** Inicia en **`false`**. Se activa a **`true`** en `task_system_normal_statechart()` cada vez que se detecta un nuevo evento en la cola. Inmediatamente después, si este evento produce un cambio en la máquina de estados, se consume y vuelve a **`false`**.





---

### 3. Comportamiento de `task_system_normal_statechart(void)`

*(Nota: Aunque la consulta hace referencia a `task_system_statechart(uint32_t index)`, el código provisto implementa `task_system_normal_statechart(void)`. Se explica esta última).*

El comportamiento en cada iteración es el siguiente:

1. **Consulta de la Cola:** Pregunta a la interfaz si hay algún evento pendiente (`any_event_task_system()`).


2. **Lectura de Evento:** Si hay un evento, lo extrae (`get_event_task_system()`), lo guarda en su variable `.event` y levanta su `.flag` (pone a `true`).


3. **Máquina de Estados (`switch`):**
* Si está en **`ST_SYS_IDLE`** y tiene el `.flag` en `true` con el evento **`EV_SYS_ACTIVE`**: Baja el flag (`false`), envía un evento **`EV_LED_ACTIVE`** al actuador **`ID_LED_A`** y cambia su estado a **`ST_SYS_ACTIVE`**.


* Si está en **`ST_SYS_ACTIVE`** y tiene el `.flag` en `true` con el evento **`EV_SYS_IDLE`**: Baja el flag, envía un evento **`EV_LED_IDLE`** al actuador, y retorna a **`ST_SYS_IDLE`**.


* **`default`:** Restablece las variables al estado de reposo.





---

### 4. Evolución de las Variables de la Cola (`event_task_system_queue`)

Estas variables gestionan la extracción de eventos (consumo) por parte de la tarea del sistema.

* **`i`**: Es una variable de iteración que solo existe durante la función inicializadora `init_event_task_system()`. Evoluciona de **0** a **15** (`QUEUE_LENGTH - 1`) para barrer el arreglo completo.


* **`event_task_system_queue.head`**: Se inicializa en **0** y **no evoluciona** por la ejecución de la tarea del sistema. Solo avanza cuando *otras* tareas (como el sensor) le insertan eventos.


* **`event_task_system_queue.tail`**:
* Se inicializa en **0**.


* Durante el loop principal, cada vez que el sistema detecta eventos y llama a `get_event_task_system()`, esta variable avanza en **+1**. Si llega al tope del arreglo (16), se reinicia a **0**.




* **`event_task_system_queue.count`**:
* Inicia en **0**.


* Cada vez que la tarea del sistema consume un evento con `get_event_task_system()`, esta variable disminuye en **-1**.




* **`event_task_system_queue.queue[i]`**:
* En el inicio, todas las posiciones (0 a 15) se llenan con el valor **`EMPTY`** (255).


* Durante las actualizaciones, la posición específica de donde se extrajo un dato (apuntada por la cola, `tail`) se vuelve a sobrescribir inmediatamente con el valor **`EMPTY`** tras la lectura, limpiando la celda.





---

### 5. Evolución de las Variables del Actuador a través de la Interfaz

Cuando la tarea del sistema desea encender o apagar el actuador, hace uso de la función `put_event_task_actuator()`. Sus variables internas se ven afectadas así:

* **`identifier`**: Es el argumento que pasa el sistema. En este caso específico, toma el valor fijo de **`ID_LED_A`** (que equivale a 0) en cada llamada. Esto le indica a la función sobre qué actuador del arreglo operar.


* **`task_actuator_dta_list[identifier].event`**: Modifica su valor directamente basándose en lo que ordenó el sistema. Evoluciona pasando a ser **`EV_LED_ACTIVE`** o **`EV_LED_IDLE`**.


* **`task_actuator_dta_list[identifier].flag`**: Cada vez que el sistema escribe un nuevo evento sobre el actuador, esta variable booleana se setea forzosamente a **`true`**.Aquí tienes el análisis del funcionamiento del código fuente proporcionado, estructurado según la evolución de las variables y el comportamiento de las funciones solicitadas.



### Evolución de las variables de la Tarea del Sistema

Al iniciarse el sistema mediante la función `task_system_init()` y durante el bucle principal en `task_system_update()`, las variables de `task_system_dta_list` evolucionan de la siguiente manera:

* **`index`**: Se utiliza en `task_system_init()` para iterar sobre la cantidad de sistemas (definido por `SYSTEM_DTA_QTY`, que equivale a 1). Inicia en 0 y finaliza al alcanzar el valor de la cantidad de sistemas.


* **`task_system_dta_list[index].tick`**: Aunque la estructura contempla esta variable, no se inicializa explícitamente en el bucle de `task_system_init()`. En caso de caer en el caso `default` de la máquina de estados, se asigna al valor de `DEL_SYS_MIN` (0). Según los registros (logs) del sistema, la unidad de medida del tiempo es en milisegundos (mS).


* **`task_system_dta_list[index].state`**: Al inicio, se inicializa en el estado de reposo `ST_SYS_IDLE`. Durante las ejecuciones de `task_system_update()`, alterna entre `ST_SYS_IDLE` y `ST_SYS_ACTIVE` según los eventos recibidos.


* **`task_system_dta_list[index].event`**: Comienza con el valor `EV_SYS_IDLE`. En el bucle principal, si hay eventos pendientes, toma el valor extraído de la cola de eventos.


* **`task_system_dta_list[index].flag`**: Se inicializa en `false`. Durante el bucle de actualización, cambia a `true` cuando se detecta un nuevo evento en la cola, y vuelve a `false` inmediatamente después de procesar la transición de estado correspondiente.



### Comportamiento de `task_system_normal_statechart()`

Cabe destacar que en el código proporcionado, la función solicitada lleva el nombre de `task_system_normal_statechart(void)`. Su comportamiento define una Máquina de Estados Finitos (FSM):

* Primero, verifica si hay algún evento en la cola; si lo hay, levanta el `flag` a `true` y actualiza el `event`.


* Si el estado actual es `ST_SYS_IDLE`, el `flag` es verdadero y el evento es `EV_SYS_ACTIVE`, envía una señal de activación al actuador (`EV_LED_ACTIVE` para `ID_LED_A`), baja el `flag` a `false` y transita al estado `ST_SYS_ACTIVE`.


* Si el estado es `ST_SYS_ACTIVE`, el `flag` es verdadero y el evento es `EV_SYS_IDLE`, envía una señal de reposo al actuador (`EV_LED_IDLE`), baja el `flag` y vuelve al estado `ST_SYS_IDLE`.



### Evolución de las variables de la Cola de Eventos

Estas variables pertenecen a la estructura `event_task_system_queue` y gestionan un buffer circular:

* **`i`**: Es una variable local utilizada en `init_event_task_system()` para iterar desde 0 hasta la longitud máxima de la cola (15, dado que `QUEUE_LENGTH` es 16).


* **`head` (cabeza)**: Comienza en 0 durante el inicio. Se incrementa cada vez que se agrega un evento con `put_event_task_system()`, reiniciándose a 0 si alcanza el límite de la cola.


* **`tail` (cola)**: Comienza en 0 al inicio. Avanza secuencialmente cada vez que `get_event_task_system()` lee un evento, volviendo a 0 al alcanzar el límite.


* **`count` (contador)**: Inicializado en 0. Aumenta en 1 al insertar un evento y disminuye en 1 al extraerlo, representando la cantidad actual de eventos pendientes.


* **`queue[i]`**: Durante la inicialización, todas las posiciones se llenan con el valor `EMPTY` (255). Al insertar eventos, la posición apuntada por `head` toma el valor del nuevo evento y, tras ser leído por `tail`, esa posición vuelve a marcarse como `EMPTY`.



### Evolución de las variables del Actuador

Estas variables se gestionan a través de la función `put_event_task_actuator()`:

* **`identifier`**: Representa el ID del actuador destino (como `ID_LED_A`) y se utiliza como índice para acceder al arreglo `task_actuator_dta_list`.


* **`task_actuator_dta_list[identifier].event`**: No se modifica durante la inicialización principal. Adopta un nuevo valor (como `EV_LED_ACTIVE` o `EV_LED_IDLE`) cuando la máquina de estados del sistema hace una transición y llama a la función de interfaz del actuador.


* **`task_actuator_dta_list[identifier].flag`**: Cambia a `true` simultáneamente con la actualización del evento para notificar a la tarea del actuador que tiene un comando pendiente de procesar.


