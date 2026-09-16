### 1. Análisis del Funcionamiento General

El conjunto de estos cuatro archivos implementa el módulo de lectura de sensores y comunicación de eventos dentro de la arquitectura del sistema.

* **Archivos de Atributos (`task_sensor_attribute.h` y `task_system_attribute.h`)**: Definen las estructuras de datos, los estados posibles y los eventos tanto de la tarea del sensor como del sistema principal. Esto permite que el sistema se maneje mediante máquinas de estados finitos (FSM) estandarizadas.


* **`task_sensor.c`**: Contiene la lógica de inicialización y actualización de los sensores (en este caso, un botón físico). Implementa una máquina de estados no bloqueante que lee el pin GPIO y reacciona a los cambios.


* **`task_system_interface.c`**: Implementa una cola circular (FIFO) de comunicación. Sirve como puente para que la tarea del sensor pueda enviar señales (eventos) a la tarea del sistema de forma asíncrona, desacoplando ambas tareas.



---

### 2. Evolución de Variables de la Tarea Sensor

A continuación, se detalla la evolución de las variables durante la ejecución de `task_sensor_init()` y los sucesivos llamados a `task_sensor_update()`.

* **`index`**:
* **Evolución:** Es la variable local utilizada en los bucles `for`. Como `SENSOR_DTA_QTY` vale 1 (solo hay un botón configurado en el arreglo `task_sensor_cfg_list`), `index` simplemente toma el valor **0** en cada iteración y finaliza el bucle.




* **`task_sensor_dta_list[index].tick`**:
* **Unidad:** Ticks del sistema (milisegundos).
* **Evolución:** Durante `task_sensor_init()`, esta variable no se inicializa de forma explícita, por lo que asume el valor 0 de la sección `.bss`. En las sucesivas ejecuciones de `task_sensor_update()`, **no sufre modificaciones** dentro de la máquina de estados regular. Solo tomará el valor `DEL_BTN_MIN` (0) si la máquina de estados cayera accidentalmente en la condición `default` del bloque `switch`.




* **`task_sensor_dta_list[index].state`**:
* **Evolución:** En `task_sensor_init()`, se inicializa en **`ST_BTN_IDLE`**. Durante los sucesivos `task_sensor_update()`, alternará hacia **`ST_BTN_ACTIVE`** si el botón es presionado, y volverá a **`ST_BTN_IDLE`** cuando sea soltado.




* **`task_sensor_dta_list[index].event`**:
* **Evolución:** En `task_sensor_init()`, se fuerza a **`EV_BTN_UP`**. Durante los llamados a `task_sensor_update()`, se actualiza continuamente leyendo el estado real del pin físico. Si el pin coincide con el nivel lógico configurado como presionado, cambia a **`EV_BTN_DOWN`**; de lo contrario, se mantiene en **`EV_BTN_UP`**.





---

### 3. Comportamiento de la Función `task_sensor_statechart(uint32_t index)`

Esta función es el motor principal del sensor y se ejecuta en cada ciclo del sistema. Su comportamiento es el siguiente:

1. **Lectura del Hardware:** Comienza leyendo el estado del pin GPIO físico asociado al sensor apuntado por el `index`.


2. **Traducción a Evento Interno:** Compara la lectura con el nivel esperado de activación (`p_task_sensor_cfg->pressed`). Si coincide, dictamina que el evento actual es `EV_BTN_DOWN`; si no, es `EV_BTN_UP`.


3. **Máquina de Estados (`switch`):**
* Si el estado actual es **`ST_BTN_IDLE`**: Comprueba si ocurrió un `EV_BTN_DOWN`. De ser así, envía una señal de encendido a la cola del sistema (`put_event_task_system(signal_down)`) y transita al estado `ST_BTN_ACTIVE`.


* Si el estado actual es **`ST_BTN_ACTIVE`**: Comprueba si ocurrió un `EV_BTN_UP`. De ser así, envía una señal de apagado a la cola del sistema (`put_event_task_system(signal_up)`) y regresa al estado `ST_BTN_IDLE`.


* **`default`:** Si por algún error de memoria cae en un estado inválido, reinicia las variables a sus valores por defecto (Idle, Up y ticks al mínimo).





---

### 4. Evolución de las Variables de la Cola (`event_task_system_queue`)

Las variables asociadas a la cola de comunicación gestionan el flujo de eventos enviados por el sensor y leídos por el sistema principal. Se comportan de la siguiente manera:

* **Inicio (`init_event_task_system()`)**:


* `event_task_system_queue.head`: Se inicializa en **0**.


* `event_task_system_queue.tail`: Se inicializa en **0**.


* `event_task_system_queue.count`: Se inicializa en **0**.


* `event_task_system_queue.queue[i]`: Todo el arreglo (de 16 posiciones) se llena con el valor **`EMPTY`** (255).




* **Sucesivas ejecuciones (`task_sensor_update()`)**:
Cuando se presiona o suelta el botón, la máquina de estados invoca a `put_event_task_system()`. En este momento:


* **`queue[head]`**: La posición actual apuntada por la cabeza sobrescribe su valor `EMPTY` con el evento generado (por ejemplo, `EV_SYS_ACTIVE` o `EV_SYS_IDLE`).


* **`head`**: Se incrementa en **+1**. Si llega al límite físico de la cola (`QUEUE_LENGTH` = 16), vuelve al inicio (**0**), comportándose de forma circular.


* **`count`**: Se incrementa en **+1** indicando que hay un nuevo evento sin leer en la cola.


* **`tail`**: **No se modifica** durante las ejecuciones exclusivas del sensor. Solo avanzará (+1) y sobrescribirá su celda con `EMPTY` cuando la *otra* tarea (el sistema) llame a `get_event_task_system()` para consumir el evento y decrementar el `count`.
