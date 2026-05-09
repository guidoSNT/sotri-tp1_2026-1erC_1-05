# Análisis del Código Fuente - sotri-tp1_01-application

## 1. Análisis y Explicación del Funcionamiento de Cada Archivo

### 1.1. `startup_stm32f446retx.s` (Archivo de Arranque)

Este archivo en ensamblador ARM (sintaxis unificada, Thumb) es el primer código que se ejecuta tras un reset del microcontrolador STM32F446RE (Cortex-M4). Su función es:

1. **Definición de la tabla de vectores de interrupción (`g_pfnVectors`):** Ubicada en la sección `.isr_vector`, contiene las direcciones de todos los handlers de excepción e interrupción. La primera entrada es el valor inicial del Stack Pointer (`_estack`) y la segunda es la dirección de `Reset_Handler`. Le siguen los handlers de excepciones del sistema (NMI, HardFault, MemManage, BusFault, UsageFault, SVC, PendSV, SysTick) y las interrupciones externas de periféricos (TIM1, TIM2, USART, SPI, DMA, etc.).

2. **`Reset_Handler`:** Es el punto de entrada tras un reset. Realiza las siguientes operaciones:
   - Carga el Stack Pointer con `_estack` (tope de la RAM).
   - Llama a `SystemInit` para inicialización básica del sistema de reloj.
   - Copia la sección `.data` (variables inicializadas) desde Flash a SRAM.
   - Llena con ceros la sección `.bss` (variables no inicializadas).
   - Llama a `__libc_init_array` (constructores estáticos de C/C++).
   - Llama a `main()`.

3. **`Default_Handler`:** Un loop infinito que captura cualquier interrupción no manejada explícitamente.

4. **Aliases débiles (`.weak`):** Todos los handlers de interrupción se declaran como alias débiles hacia `Default_Handler`. Esto permite que si el programador define un handler con el mismo nombre en C, este reemplazará automáticamente al default.

---

### 1.2. `main.c` (Programa Principal)

Es el archivo principal de la aplicación. Su función es:

1. **Inicialización del hardware y periféricos:**
   - `HAL_Init()`: Inicializa la HAL (Hardware Abstraction Layer), configura el SysTick para generar interrupciones cada 1 ms (como base de tiempo), y habilita la caché de instrucciones.
   - `SystemClock_Config()`: Configura el PLL usando HSI (16 MHz) como fuente, con PLLM=16, PLLN=336, PLLP=4, resultando en un SYSCLK de **84 MHz**. APB1 se divide por 2 (42 MHz) y APB2 no se divide (84 MHz).
   - `MX_GPIO_Init()`: Configura GPIO para el LED (LD2, salida push-pull) y botón (B1, interrupción por flanco de bajada).
   - `MX_USART2_UART_Init()`: Configura USART2 a 115200 baudios, 8 bits, sin paridad.
   - `MX_TIM2_Init()`: Configura TIM2 con prescaler=1 (divide por 2) y periodo=83999, generando interrupciones a una frecuencia de 500 Hz (cada 2 ms) — calculado como: 84 MHz / 2 / 84000 = 500 Hz.

2. **Inicio del timer TIM2** con `HAL_TIM_Base_Start_IT(&htim2)` para habilitar la interrupción de update.

3. **Inicialización de la aplicación** mediante `app_init()`.

4. **Creación de tareas de FreeRTOS:** Se define `defaultTask` (si `_defaultTask_` está definido) con prioridad normal y stack de 128 words.

5. **Inicio del scheduler de FreeRTOS** con `osKernelStart()`. A partir de este punto, el control lo toma FreeRTOS y el `while(1)` nunca debería ejecutarse.

6. **`HAL_TIM_PeriodElapsedCallback`:** Callback que se invoca cuando un timer genera interrupción de overflow:
   - Si es **TIM1**: llama a `HAL_IncTick()` para incrementar `uwTick` (base de tiempo de la HAL).
   - Si es **TIM2**: incrementa `ulHighFrequencyTimerTicks` (usado para estadísticas de tiempo de ejecución de FreeRTOS).

7. **Funciones para estadísticas de runtime de FreeRTOS:**
   - `configureTimerForRunTimeStats()`: Resetea el contador `ulHighFrequencyTimerTicks` a 0.
   - `getRunTimeCounterValue()`: Retorna el valor actual de `ulHighFrequencyTimerTicks`.

---

### 1.3. `stm32f4xx_it.c` (Handlers de Interrupción)

Contiene las rutinas de servicio de interrupción (ISR) del sistema:

1. **Excepciones del procesador Cortex-M4:**
   - `NMI_Handler`: Loop infinito ante interrupción no enmascarable.
   - `HardFault_Handler`: Loop infinito ante fallo grave.
   - `MemManage_Handler`: Loop infinito ante fallo de gestión de memoria.
   - `BusFault_Handler`: Loop infinito ante fallo de bus.
   - `UsageFault_Handler`: Loop infinito ante instrucción inválida.
   - `DebugMon_Handler`: Handler del monitor de debug (vacío).

2. **Interrupciones de periféricos:**
   - `TIM1_UP_TIM10_IRQHandler`: ISR de TIM1 update. Llama a `HAL_TIM_IRQHandler(&htim1)`, que eventualmente invoca `HAL_TIM_PeriodElapsedCallback` con TIM1, lo cual ejecuta `HAL_IncTick()`.
   - `TIM2_IRQHandler`: ISR de TIM2. Llama a `HAL_TIM_IRQHandler(&htim2)`, que eventualmente invoca `HAL_TIM_PeriodElapsedCallback` con TIM2, incrementando `ulHighFrequencyTimerTicks`.

**Nota importante:** Los handlers `SVC_Handler`, `PendSV_Handler` y `SysTick_Handler` NO están definidos en este archivo porque están mapeados directamente a las funciones del port de FreeRTOS mediante `#define` en `FreeRTOSConfig.h`.

---

### 1.4. `FreeRTOSConfig.h` (Configuración de FreeRTOS)

Define los parámetros de configuración del kernel FreeRTOS:

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `configUSE_PREEMPTION` | 1 | Scheduler preemptivo habilitado |
| `configCPU_CLOCK_HZ` | `SystemCoreClock` | Frecuencia del CPU (84 MHz en runtime) |
| `configTICK_RATE_HZ` | 1000 | Tick del OS cada 1 ms |
| `configMAX_PRIORITIES` | 7 | 7 niveles de prioridad de tareas |
| `configMINIMAL_STACK_SIZE` | 128 | Stack mínimo (128 words = 512 bytes) |
| `configTOTAL_HEAP_SIZE` | 15360 | Heap de FreeRTOS (15 KB) |
| `configGENERATE_RUN_TIME_STATS` | 1 | Estadísticas de runtime habilitadas |
| `configUSE_IDLE_HOOK` | 1 | Hook de tarea idle habilitado |
| `configUSE_TICK_HOOK` | 1 | Hook de tick habilitado |
| `configCHECK_FOR_STACK_OVERFLOW` | 1 | Detección de stack overflow habilitada |
| `configSUPPORT_STATIC_ALLOCATION` | 1 | Permite asignación estática de tareas |
| `configSUPPORT_DYNAMIC_ALLOCATION` | 1 | Permite asignación dinámica de tareas |

**Mapeo de handlers del port FreeRTOS a nombres CMSIS:**
- `vPortSVCHandler` → `SVC_Handler`
- `xPortPendSVHandler` → `PendSV_Handler`
- `xPortSysTickHandler` → `SysTick_Handler`

**Configuración de prioridades de interrupción:**
- `configPRIO_BITS` = 4 (16 niveles de prioridad en Cortex-M4)
- `configLIBRARY_LOWEST_INTERRUPT_PRIORITY` = 15 (prioridad más baja)
- `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` = 5 (máxima prioridad desde la que se puede llamar a API de FreeRTOS)
- `configKERNEL_INTERRUPT_PRIORITY` = 15 << 4 = 240 (0xF0)
- `configMAX_SYSCALL_INTERRUPT_PRIORITY` = 5 << 4 = 80 (0x50)

**Macros para estadísticas de runtime:**
- `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS` → `configureTimerForRunTimeStats`
- `portGET_RUN_TIME_COUNTER_VALUE` → `getRunTimeCounterValue`

---

### 1.5. `freertos.c` (Soporte de FreeRTOS)

Proporciona implementaciones necesarias para el funcionamiento de FreeRTOS:

1. **`configureTimerForRunTimeStats()`** (weak): Versión débil que se sobreescribe en `main.c`. Resetea el contador de estadísticas.

2. **`getRunTimeCounterValue()`** (weak): Versión débil que se sobreescribe en `main.c`. Retorna el valor del contador de alta frecuencia.

3. **`vApplicationIdleHook()`**: Se ejecuta en cada iteración de la tarea idle. Está vacía pero disponible para código de usuario (ahorro de energía, por ejemplo).

4. **`vApplicationTickHook()`**: Se ejecuta en cada tick del OS (cada 1 ms). Está vacía pero disponible para código de usuario ejecutable desde contexto de interrupción.

5. **`vApplicationStackOverflowHook()`**: Se invoca si se detecta un desbordamiento de stack en alguna tarea.

6. **`vApplicationGetIdleTaskMemory()`**: Proporciona la memoria estática para la tarea idle (TCB y stack), requerida porque `configSUPPORT_STATIC_ALLOCATION = 1`.

---

## 2. Evolución de las Variables `SysTick` y `SystemCoreClock`

### 2.1. Variable `SystemCoreClock`

| Momento de ejecución | Valor de `SystemCoreClock` | Razón |
|---------------------|---------------------------|-------|
| Tras `Reset_Handler` → `SystemInit` | **16,000,000** (16 MHz) | `SystemInit()` configura el sistema al estado por defecto usando HSI (oscilador interno de 16 MHz). `SystemCoreClock` se inicializa con el valor de HSI. |
| Tras `SystemClock_Config()` en `main()` | **84,000,000** (84 MHz) | La función configura el PLL: HSI(16MHz) / PLLM(16) * PLLN(336) / PLLP(4) = 84 MHz. Internamente `HAL_RCC_ClockConfig()` llama a `SystemCoreClockUpdate()` que actualiza la variable a 84 MHz. |
| Hasta el `while(1)` | **84,000,000** (84 MHz) | No cambia después de la configuración del reloj. |

### 2.2. Timer SysTick (Registro de Hardware)

El SysTick es un timer decremental de 24 bits integrado en el Cortex-M4.

| Momento de ejecución | Estado del SysTick | Configuración |
|---------------------|-------------------|---------------|
| Tras `Reset_Handler` → `SystemInit` | **Deshabilitado** | `SystemInit()` no configura el SysTick. |
| Tras `HAL_Init()` | **Habilitado, periodo = 16,000 ciclos** | `HAL_Init()` llama a `HAL_InitTick(TICK_INT_PRIORITY)` que configura SysTick con `SystemCoreClock / 1000 = 16000` (para 1 ms a 16 MHz). Sin embargo, dado que se usa FreeRTOS con TIM1 como timebase de la HAL, es posible que `HAL_Init()` configure TIM1 en su lugar y no el SysTick directamente. |
| Tras `SystemClock_Config()` | **Reconfigurado, periodo = 84,000 ciclos** | `HAL_RCC_ClockConfig()` internamente llama a `HAL_InitTick()` para reconfigurar la base de tiempo con la nueva frecuencia. Con TIM1 como timebase de HAL, TIM1 se reconfigura. Si se usa SysTick provisional, se reconfigura con `84000000/1000 = 84000`. |
| Tras `osKernelStart()` (inicio del scheduler FreeRTOS) | **Habilitado por FreeRTOS, periodo = 84,000 ciclos** | FreeRTOS toma control del SysTick. Lo configura con: `SystemCoreClock / configTICK_RATE_HZ = 84000000 / 1000 = 84000 - 1 = 83999` como valor de recarga. SysTick genera interrupciones cada 1 ms. El handler `SysTick_Handler` es `xPortSysTickHandler` de FreeRTOS. |

**Resumen de la evolución del registro SysTick LOAD:**

```
Reset             → SysTick LOAD = indefinido (SysTick deshabilitado)
HAL_Init()        → SysTick LOAD = 15999 (16 MHz / 1000 - 1) [temporalmente, si HAL usa SysTick]
                    O bien TIM1 se configura como timebase de HAL
SystemClock_Config → SysTick/TIM1 se reconfigura para 84 MHz
osKernelStart()   → SysTick LOAD = 83999 (84 MHz / 1000 - 1), controlado por FreeRTOS
```

---

## 3. Comportamiento del Programa desde `Reset_Handler` hasta antes del `while(1)`

### Secuencia de ejecución paso a paso:

1. **`Reset_Handler` (startup_stm32f446retx.s:60)**
   - Se carga el SP con `_estack` (tope de RAM).
   - Se llama a `SystemInit()`: configura el FPU (si aplica), resetea configuraciones de reloj al estado por defecto. `SystemCoreClock` = 16 MHz.

2. **Inicialización de memoria (startup_stm32f446retx.s:66-95)**
   - Se copia `.data` de Flash a SRAM (variables globales inicializadas).
   - Se llena `.bss` con ceros (variables globales no inicializadas, incluyendo `ulHighFrequencyTimerTicks = 0`).

3. **`__libc_init_array` (startup_stm32f446retx.s:98)**
   - Ejecuta constructores estáticos de C++/C.

4. **`main()` (main.c:83)**

5. **`HAL_Init()` (main.c:96)**
   - Habilita caché de instrucciones/datos.
   - Configura grupo de prioridades NVIC (4 bits de preemption).
   - Llama a `HAL_InitTick()` → configura **TIM1** como base de tiempo de la HAL (1 ms). El SysTick queda libre para FreeRTOS.
   - `uwTick = 0`, `uwTickFreq = HAL_TICK_FREQ_1KHZ`.

6. **`SystemClock_Config()` (main.c:103)**
   - Habilita reloj del regulador de voltaje (PWR).
   - Configura escalado de voltaje (Scale 3).
   - Habilita HSI (16 MHz) y configura PLL: 16/16*336/4 = **84 MHz**.
   - Configura buses: HCLK=84MHz, APB1=42MHz, APB2=84MHz.
   - `HAL_RCC_ClockConfig()` internamente llama a `SystemCoreClockUpdate()` → `SystemCoreClock = 84000000`.
   - Se reconfigura la base de tiempo (TIM1) para operar a 84 MHz.

7. **`MX_GPIO_Init()` (main.c:110)**
   - Habilita relojes de GPIOA, GPIOB, GPIOC, GPIOH.
   - Configura LD2 (PA5) como salida push-pull, nivel bajo inicial.
   - Configura B1 (PC13) como entrada con interrupción por flanco de bajada.

8. **`MX_USART2_UART_Init()` (main.c:111)**
   - Configura USART2: 115200 bps, 8N1, TX+RX, sin control de flujo.

9. **`MX_TIM2_Init()` (main.c:112)**
   - Configura TIM2: prescaler=1 (divide por 2), periodo=83999, modo ascendente.
   - Frecuencia de interrupción: APB1_timer_clk / (PSC+1) / (ARR+1).
   - APB1 timer clock = 42 MHz * 2 = 84 MHz (porque APB1 prescaler > 1, el timer recibe x2).
   - Frecuencia = 84 MHz / 2 / 84000 = **500 Hz** (cada 2 ms).

10. **`HAL_TIM_Base_Start_IT(&htim2)` (main.c:115)**
    - Habilita la interrupción de update de TIM2 y arranca el timer.
    - A partir de aquí, cada 2 ms se incrementa `ulHighFrequencyTimerTicks`.

11. **`app_init()` (main.c:118)**
    - Inicializa la lógica de aplicación del usuario.

12. **Creación de tarea (main.c:142-143)** (si `_defaultTask_` está definido)
    - Se crea `defaultTask` con `osThreadDef` y `osThreadCreate`.

13. **`osKernelStart()` (main.c:152)**
    - Internamente llama a `vTaskStartScheduler()`.
    - FreeRTOS llama a `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS` → `configureTimerForRunTimeStats()` → `ulHighFrequencyTimerTicks = 0`.
    - FreeRTOS configura el **SysTick** con periodo de 83999 (1 ms a 84 MHz) para su propio tick.
    - FreeRTOS configura las prioridades de PendSV y SysTick al nivel más bajo (0xF0).
    - El scheduler comienza a ejecutar tareas. **El control nunca retorna a `main()`**.
    - El `while(1)` de main.c:158 **nunca se alcanza** en ejecución normal.

---

## 4. Interacción de SysTick y TIM1 con FreeRTOS

### 4.1. SysTick → Tick de FreeRTOS

**Cómo interactúa:**

El SysTick es el timer que FreeRTOS utiliza como fuente del **tick del sistema operativo**. La configuración se realiza mediante:

- `FreeRTOSConfig.h` define `xPortSysTickHandler` como `SysTick_Handler` (línea 142). Esto mapea el handler del SysTick directamente a la función del port de FreeRTOS.
- `configTICK_RATE_HZ = 1000` define que el tick ocurre cada 1 ms.
- `configCPU_CLOCK_HZ = SystemCoreClock = 84 MHz` permite a FreeRTOS calcular el valor de recarga: 84000000/1000 - 1 = 83999.

**Para qué se usa:**

Cada vez que el SysTick genera una interrupción (cada 1 ms):
1. Se ejecuta `xPortSysTickHandler` (el handler del port de FreeRTOS).
2. Se incrementa el contador de ticks del kernel (`xTickCount`).
3. Se verifican timeouts de tareas bloqueadas (delays, semáforos con timeout, etc.).
4. Se ejecuta `vApplicationTickHook()` (definida en freertos.c).
5. Si una tarea de mayor prioridad se desbloquea, se activa el **context switch** mediante PendSV.

El SysTick es esencial para:
- **Temporización:** `osDelay()`, `vTaskDelay()`, timeouts.
- **Preemption por time-slicing:** Tareas de igual prioridad comparten el CPU.
- **Gestión de tareas bloqueadas:** Despertar tareas cuando expira su tiempo de espera.

### 4.2. TIM1 → Base de Tiempo de la HAL (NO directamente con FreeRTOS)

**Cómo interactúa indirectamente:**

TIM1 se configura como base de tiempo de la HAL (1 ms). Su interrupción (`TIM1_UP_TIM10_IRQHandler` en stm32f4xx_it.c:166) llama a `HAL_TIM_IRQHandler(&htim1)`, que a su vez invoca `HAL_TIM_PeriodElapsedCallback`. Dentro del callback (main.c:376-379):

```c
if (htim->Instance == TIM1) {
    HAL_IncTick();  // Incrementa uwTick
}
```

**Para qué se usa con FreeRTOS:**

TIM1 **no interactúa directamente con FreeRTOS**, sino con la HAL de STM32. Sin embargo, la HAL es utilizada por FreeRTOS indirectamente:
- La separación de TIM1 (para HAL) y SysTick (para FreeRTOS) es una **decisión de diseño crítica** de STM32CubeMX cuando se usa FreeRTOS.
- Esto evita conflictos: FreeRTOS necesita control total del SysTick para su scheduler, mientras la HAL necesita su propia base de tiempo (`uwTick`) para funciones como `HAL_Delay()` y timeouts de periféricos.
- TIM1 mantiene funcional la HAL (`HAL_GetTick()` retorna `uwTick`) sin interferir con el tick de FreeRTOS.

---

## 5. Interacción del Timer 2 (TIM2) con la HAL del Proyecto STM32

### Cómo interactúa:

La cadena de interacción de TIM2 con la HAL es la siguiente:

1. **Configuración:** `MX_TIM2_Init()` usa la HAL (`HAL_TIM_Base_Init`, `HAL_TIM_ConfigClockSource`, `HAL_TIMEx_MasterConfigSynchronization`) para configurar los registros del timer TIM2.

2. **Inicio:** `HAL_TIM_Base_Start_IT(&htim2)` habilita la interrupción de update y arranca el timer.

3. **Interrupción:** Cuando TIM2 desborda (cada 2 ms), se genera la IRQ `TIM2_IRQHandler` (stm32f4xx_it.c:180). Esta función llama a `HAL_TIM_IRQHandler(&htim2)`.

4. **Despacho de la HAL:** `HAL_TIM_IRQHandler()` es una función genérica de la HAL que:
   - Verifica qué flag de interrupción del timer está activo.
   - Limpia el flag correspondiente.
   - Llama al callback apropiado: `HAL_TIM_PeriodElapsedCallback(htim)`.

5. **Callback:** En `HAL_TIM_PeriodElapsedCallback` (main.c:381-384):
   ```c
   if (htim->Instance == TIM2) {
       ulHighFrequencyTimerTicks++;
   }
   ```

### Para qué se usa:

TIM2 se usa como **contador de alta frecuencia para las estadísticas de tiempo de ejecución de FreeRTOS** (`configGENERATE_RUN_TIME_STATS = 1`).

**Propósito:**
- FreeRTOS necesita un timer con mayor resolución que su propio tick (1 ms) para medir con precisión cuánto tiempo CPU consume cada tarea.
- TIM2 genera interrupciones a **500 Hz** (cada 2 ms), incrementando `ulHighFrequencyTimerTicks`.
- Cuando FreeRTOS necesita calcular estadísticas, llama a `portGET_RUN_TIME_COUNTER_VALUE` → `getRunTimeCounterValue()` → retorna `ulHighFrequencyTimerTicks`.
- Con esta información, FreeRTOS puede reportar el porcentaje de CPU utilizado por cada tarea.

**Nota sobre la resolución:** Idealmente el timer de estadísticas debería tener una frecuencia 10-100x mayor que el tick del OS. En este caso es 500 Hz vs 1000 Hz del tick, lo cual resulta en una resolución de estadísticas de 2 ms (menor resolución que el tick). Esto puede deberse a una configuración específica del proyecto o a que el objetivo es solo una medición aproximada.

**Flujo resumido:**
```
TIM2 overflow (cada 2 ms)
  → NVIC genera IRQ
    → TIM2_IRQHandler() [stm32f4xx_it.c]
      → HAL_TIM_IRQHandler(&htim2) [HAL driver]
        → Limpia flag TIM_SR_UIF
        → HAL_TIM_PeriodElapsedCallback(&htim2) [main.c]
          → ulHighFrequencyTimerTicks++
```

---

## Resumen de la Arquitectura de Timers

| Timer | Frecuencia | Propósito | Controlado por |
|-------|-----------|-----------|----------------|
| **SysTick** | 1 kHz (1 ms) | Tick del scheduler FreeRTOS | FreeRTOS (port layer) |
| **TIM1** | 1 kHz (1 ms) | Base de tiempo de la HAL (`uwTick`) | HAL (`HAL_IncTick()`) |
| **TIM2** | 500 Hz (2 ms) | Estadísticas de runtime de FreeRTOS | Usuario (`ulHighFrequencyTimerTicks`) |

Esta separación de responsabilidades permite que:
- FreeRTOS tenga control exclusivo del SysTick para su scheduler preemptivo.
- La HAL mantenga su propia base de tiempo independiente (TIM1) para funciones como `HAL_Delay()` y timeouts de periféricos.
- Las estadísticas de runtime tengan su propio contador (TIM2) sin interferir con ninguno de los otros dos.
