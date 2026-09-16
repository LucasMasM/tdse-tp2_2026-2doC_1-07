A continuación se presenta el análisis del funcionamiento de los archivos adjuntos, los cuales en conjunto implementan la estructura base de un **Sistema Disparado por Eventos (Bare Metal - Event-Triggered System)** con capacidad para medir el tiempo de ejecución (profiling) de sus tareas.

### 1. Análisis del Funcionamiento General

* **`dwt.h`**: Proporciona macros y funciones en línea para utilizar el hardware DWT (*Data Watchpoint and Trace*) del procesador ARM Cortex. Permite contar los ciclos exactos de reloj del procesador y convertirlos a microsegundos, lo que funciona como un cronómetro de muy alta precisión.


* **`logger.h` / `logger.c`**: Implementan un sistema de registro de mensajes (logs). Destaca el uso de macros (`LOGGER_INFO`, `LOGGER_LOG`) que deshabilitan las interrupciones del microcontrolador antes de preparar la cadena con `snprintf` y enviarla mediante Semihosting (si está habilitado), para luego volver a habilitarlas.


* **`systick.c`**: Implementa una función de retardo bloqueante en microsegundos (`systick_delay_us`) leyendo directamente los registros hardware del temporizador SysTick.


* **`app_it.c`**: Maneja las interrupciones de la aplicación. En particular, la función `HAL_SYSTICK_Callback` incrementa el contador global `g_app_tick_cnt` en cada interrupción del SysTick (típicamente cada 1 milisegundo).


* **`app.c`**: Es el núcleo de la aplicación. Posee un arreglo de tareas (`task_cfg_list`) compuesto por el sensor, el sistema y el actuador. Coordina la inicialización de las tareas y, dentro del bucle principal, controla cuándo ejecutarlas y mide su rendimiento en el tiempo.



---

### 2. Evolución de las Variables

A continuación se detalla cómo evolucionan las variables desde la llamada a `app_init()` y durante el bucle de `app_update()`.

**Variables de Control Globales:**

* **`g_app_tick_cnt`**:
* **Unidad:** Adimensional (Ticks de SysTick, típicamente equivalentes a milisegundos).
* **Evolución:** Se inicializa en `0` a través de `app_it_init()`. En segundo plano, incrementa su valor de 1 en 1 de forma asíncrona mediante la interrupción del SysTick (`HAL_SYSTICK_Callback()`). En el loop principal (`app_update()`), si esta variable es mayor a 0, se decrementa en 1 y habilita una nueva ejecución del bucle de tareas.




* **`g_app_runtime_us`**:
* **Unidad:** Microsegundos ($\mu s$).


* **Evolución:** En cada nuevo ciclo de ejecución disparado en `app_update()`, se reinicia a `0`. Mientras el iterador recorre y ejecuta las tareas, acumula el tiempo de ejecución (`LET`) de todas ellas. Representa el tiempo total de procesamiento consumido por las tareas en el ciclo actual.




* **`index`**:
* **Unidad:** Adimensional (índice de iteración).
* **Evolución:** Es una variable local empleada en los bucles `for` tanto en `app_init()` como en `app_update()`. Su valor itera progresivamente de `0` a `2` (es decir, avanza hasta ser menor a `TASK_QTY`, que equivale a 3) para apuntar a cada tarea (sensor, system, actuator).





Estructura de Datos de las Tareas (`task_dta_list[index]`):

* **`.NOE` (Number Of Executions - Número de ejecuciones)**:
* **Unidad:** Adimensional (cantidad).
* **Evolución:** Inicia en `0` (`TASK_X_NOE_INI`). En `app_update()`, se incrementa en `+1` cada vez que la tarea finaliza su rutina `task_update()`. Su valor crecerá indefinidamente indicando cuántas veces se ejecutó la tarea en la vida del programa.




* **`.LET` (Last Execution Time - Último tiempo de ejecución)**:
* **Unidad:** Microsegundos ($\mu s$).


* **Evolución:** Inicia en `0` (`TASK_X_LET_INI`). En `app_update()`, se actualiza en cada iteración capturando el valor exacto del contador de hardware DWT al finalizar la tarea (`cycle_counter_get_time_us()`). Su valor fluctúa constantemente reflejando lo que demoró la tarea en ese instante preciso.




* **`.BCET` (Best-Case Execution Time - Mejor tiempo de ejecución)**:
* **Unidad:** Microsegundos ($\mu s$).


* **Evolución:** Inicia deliberadamente alto en `1000` (`TASK_X_BCET_INI`). Durante `app_update()`, si el último tiempo de ejecución (`LET`) resulta ser menor que el `BCET` actual, se sobrescribe con ese nuevo valor. Su evolución es siempre descendente, quedando "congelado" en el registro del tiempo de ejecución más rápido.




* **`.WCET` (Worst-Case Execution Time - Peor tiempo de ejecución)**:
* **Unidad:** Microsegundos ($\mu s$).


* **Evolución:** Inicia en `0` (`TASK_X_WCET_INI`). Durante `app_update()`, si el tiempo medido actual (`LET`) es mayor que el `WCET` almacenado, este último asume el nuevo valor. Su evolución es siempre ascendente, reteniendo como una "marca de agua alta" el tiempo máximo histórico que tardó la tarea en ejecutarse.





---

### 3. Impacto de usar `LOGGER_INFO()`

Si se coloca una llamada a `LOGGER_INFO()` dentro del código de alguna de las tareas, el impacto sobre el sistema será severo debido a su arquitectura subyacente:

1. **Mecanismo Semihosting**: El logger está configurado por defecto para usar Semihosting (`LOGGER_CONFIG_USE_SEMIHOSTING = 1`) y ejecuta la función bloqueante `printf()` para enviar caracteres al depurador por el cable JTAG/SWD. Esta acción requiere pausar el procesador miles de ciclos de reloj, lo que suele tomar varios milisegundos.


2. **Desactivación de interrupciones**: La macro encapsula su comportamiento entre un `__asm("CPSID i")` y un `__asm("CPSIE i")` para inhabilitar interrupciones globalmente mientras imprime.



Por lo tanto, **el impacto en las variables evaluadas será el siguiente**:

* **Impacto en `task_dta_list[index].WCET`**: Dado que la función bloquea el flujo del programa por un tiempo prolongado mientras el contador de ciclos DWT sigue su marcha, el `LET` de esa iteración en particular se disparará (marcará miles de microsegundos extra). Por consiguiente, **la variable `WCET` atrapará y guardará inmediatamente este valor inflado de manera permanente**, arruinando las métricas del "Peor caso de ejecución" de la tarea al registrar el retardo introducido por la consola de depuración en lugar de la lógica pura del programa.


* **Impacto en `g_app_runtime_us`**: En el ciclo específico en el que se imprima el log, el inmenso valor temporal aportado por esa tarea se sumará a `g_app_runtime_us`. Esto provocará que ese ciclo de medición evidencie una falsa sobrecarga masiva de microsegundos sobre la CPU. Además, al deshabilitar interrupciones por tanto tiempo, el SysTick podría perder incrementos de tiempo en hardware, rompiendo la cadencia predecible del sistema.
