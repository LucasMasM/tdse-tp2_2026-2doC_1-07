A continuación se presenta el análisis detallado del funcionamiento del código fuente proporcionado en los tres archivos y la explicación paso a paso de la evolución de las variables/registros relacionados con **`SystemCoreClock`** y **`SysTick`**.

---

### 1. Análisis y Funcionamiento del Código Fuente

#### **A. `startup_stm32f103rbtx.s` (Archivo de Ensamblador de Arranque)**

Es el código de bajo nivel que se ejecuta inmediatamente tras un Reset del microcontrolador (Cortex-M3). Sus funciones principales son:

1. **Tabla de Vectores de Interrupción (`g_pfnVectors`)**: Define la ubicación de la pila en RAM (`_estack`), la dirección del punto de entrada (`Reset_Handler`) y las direcciones de todas las ISR (*Interrupt Service Routines*) de excepciones y periféricos.
2. **Rutina `Reset_Handler**`:
* Llama a `SystemInit()` para configurar el estado inicial del árbol de relojes del sistema.
* **Copia de la sección `.data**`: Copia las variables globales/estáticas inicializadas desde la memoria Flash (`_sidata`) hacia la memoria RAM (`_sdata` a `_edata`).
* **Llenado a cero de la sección `.bss**`: Limpia e inicializa en `0` las variables globales/estáticas no inicializadas en la RAM (`_sbss` a `_ebss`).
* Llama a `__libc_init_array` (inicializadores de la biblioteca estándar de C/C++).
* Salta finalmente a la función `main()` de C.



---

#### **B. `main.c` (Punto de Entrada de la Aplicación)**

Contiene el flujo principal de configuración e inicialización de periféricos del microcontrolador:

1. **Semihosting (Opcional)**: Si `LOGGER_CONFIG_USE_SEMIHOSTING` está activo, inicializa la comunicación de depuración mediante `initialise_monitor_handles()`.
2. **`HAL_Init()`**: Inicializa la biblioteca HAL de ST, configura la prioridad de las interrupciones del NVIC y arranca el temporizador de sistema **SysTick** para que genere una interrupción cada 1 ms.
3. **`SystemClock_Config()`**: Configura el oscilador y el multiplicador PLL:
* Usa el oscilador interno **HSI** (8 MHz).
* Aplica un divisor por 2 (`HSI_DIV2` = 4 MHz) que entra al PLL.
* Multiplica por 16 (`PLLMUL = 16`): $4 \text{ MHz} \times 16 = 64 \text{ MHz}$.
* Selecciona el PLL como fuente de reloj del sistema (**SYSCLK = 64 MHz**).
* Configura los divisores de bus: AHB = 1 (HCLK = 64 MHz), APB1 = 2 (PCLK1 = 32 MHz), APB2 = 1 (PCLK2 = 64 MHz).
* Ajusta la latencia de la memoria Flash a 2 estados de espera (`FLASH_LATENCY_2`).


4. **`MX_GPIO_Init()`**: Habilita los relojes de los puertos GPIO (A, B, C, D), configura el pin `LD2` como salida digital push-pull y el pin `B1` (botón del usuario) como entrada de interrupción por flanco ascendente (`EXTI15_10_IRQn`).
5. **`MX_USART2_UART_Init()`**: Configura el puerto serie USART2 a 115200 baudios, 8 bits de datos, 1 bit de parada y sin paridad.
6. **`app_init()`**: Ejecuta la inicialización propia de la aplicación definida por el usuario.
7. **Bucle `while (1)**`: Ejecuta de forma cíclica e indefinida la función `app_update()`.

---

#### **C. `stm32f1xx_it.c` (Manejadores de Interrupción)**

Contiene las funciones ISR que responden a las interrupciones de hardware del procesador y periféricos:

* **Excepciones de Núcleo**: `NMI_Handler`, `HardFault_Handler`, `MemManage_Handler`, `BusFault_Handler`, `UsageFault_Handler`, etc., que capturan fallos y entran en bucles infinitos de protección.
* **`SysTick_Handler()`**: Atiende la interrupción del temporizador SysTick (disparada cada 1 ms). Ejecuta `HAL_IncTick()`, que incrementa la variable interna `uwTick` empleada por las funciones de retardo/temporización de la HAL (`HAL_Delay()`, `HAL_GetTick()`).
* **`EXTI15_10_IRQHandler()`**: Atiende la interrupción externa asociada al botón de usuario `B1` invocando a `HAL_GPIO_EXTI_IRQHandler(B1_Pin)`.

---

### 2. Evolución de `SystemCoreClock` y `SysTick`

A continuación se detalla la evolución de la variable **`SystemCoreClock`** (frecuencia de la CPU en Hz) y del temporizador **`SysTick`** (tanto sus registros de hardware como la variable de software `uwTick` de la HAL) a lo largo del proceso de arranque:

```
Reset_Handler (Startup)
   │
   ├──> SystemInit()           ---> SystemCoreClock = 8,000,000 Hz | SysTick: Apagado
   ├──> Zero Fill (.bss)       ---> uwTick = 0
   │
   └──> main()
         ├──> HAL_Init()       ---> SystemCoreClock = 8,000,000 Hz | SysTick LOAD = 7,999 (1 ms) | uwTick empieza a contar
         ├──> SystemClock_Config()
         │      └──> HAL_RCC_ClockConfig()
         │                     ---> SystemCoreClock = 64,000,000 Hz | SysTick LOAD = 63,999 (reajustado a 1 ms)
         │
         └──> while (1)        ---> SystemCoreClock = 64,000,000 Hz | uwTick continúa incrementando en cada ms

```

---

#### **Paso 1: Inicio del `Reset_Handler` (startup_stm32f103rbtx.s)**

* **Estado de Hardware**: El microcontrolador inicia usando por defecto el oscilador interno **HSI** a **8 MHz**.
* **`SystemCoreClock`**: No está declarada ni asignada aún en RAM (su valor inicial por omisión en código C es 8,000,000 Hz).
* **`SysTick` (Hardware)**: Deshabilitado (`SysTick->CTRL` = 0).
* **`uwTick` (Software)**: Indeterminado (memoria RAM no inicializada).

#### **Paso 2: Ejecución de `SystemInit()` (dentro de `Reset_Handler`)**

* **`SystemCoreClock`**: Se establece en **8,000,000 Hz** (8 MHz), reflejando la frecuencia base del oscilador HSI.
* **`SysTick`**: Sigue deshabilitado.

#### **Paso 3: Limpieza de la sección `.bss` en RAM (dentro de `Reset_Handler`)**

* **`uwTick`**: Al ser una variable global inicializada implícitamente a 0, la rutina `FillZerobss` escribe el valor **0** en su dirección de memoria RAM.

#### **Paso 4: Ejecución de `HAL_Init()` en `main.c**`

* **`SystemCoreClock`**: Mantiene su valor de **8,000,000 Hz** (8 MHz).
* **`SysTick` (Hardware)**: `HAL_Init()` invoca internamente a `HAL_InitTick()`, configurando el SysTick para un intervalo de 1 ms:

$$\text{SysTick->LOAD} = \left(\frac{\text{SystemCoreClock}}{1000}\right) - 1 = \left(\frac{8,000,000}{1000}\right) - 1 = \mathbf{7,999}$$



Se activan las interrupciones y el contador de SysTick.
* **`uwTick` (Software)**: Comienza en **0**. Cada 1 ms, el hardware del SysTick dispara la interrupción `SysTick_Handler()`, ejecutando `HAL_IncTick()` e incrementando `uwTick` en +1 (**0, 1, 2, 3...**).

#### **Paso 5: Ejecución de `SystemClock_Config()` en `main.c**`

Al configurar el PLL ($8 \text{ MHz} / 2 \times 16 = 64 \text{ MHz}$) y llamar a `HAL_RCC_ClockConfig()`:

* **`SystemCoreClock`**: Se actualiza formalmente a **64,000,000 Hz** (64 MHz).
* **`SysTick` (Hardware)**: La HAL vuelve a llamar a `HAL_InitTick()` para recalibrar el temporizador a la nueva frecuencia de CPU, manteniendo el período de 1 ms:

$$\text{SysTick->LOAD} = \left(\frac{64,000,000}{1000}\right) - 1 = \mathbf{63,999}$$


* **`uwTick` (Software)**: Continúa incrementándose de manera transparente a razón de +1 por cada milisegundo real transcurrido.

#### **Paso 6: Llegada al bucle principal `while (1)` en `main.c**`

Al alcanzar el bucle de ejecución de la aplicación:

* **`SystemCoreClock`**: Permanece constante en **64,000,000 Hz** (64 MHz).
* **`SysTick->LOAD`**: Permanece constante en **63,999**.
* **`uwTick`**: Contendrá el número acumulado de milisegundos transcurridos durante las inicializaciones (por ejemplo, un valor entero $N \ge 0$) y **continuará incrementándose indefinidamente en segundo plano** por medio de las interrupciones `SysTick_Handler`.
