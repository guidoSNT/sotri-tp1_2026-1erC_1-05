# Actividad 1 - paso 6

¡Hola! Es un excelente ejercicio analizar la secuencia de arranque y la integración de un RTOS en un microcontrolador STM32.

Antes de comenzar, quiero hacer una pequeña aclaración: he notado que en tu solicitud mencionas algunos archivos con nombres ligeramente distintos (como startup_stm32f103rbtx.s y stm32f1xx_it.c), pero basaré mi análisis estrictamente en los archivos que adjuntaste, que corresponden a un STM32F4 (startup_stm32f446retx.s y stm32f4xx_it.c). Además, preguntas por la interacción del Timer 4 (TIM4), pero en el código provisto no hay configuración para este periférico; en su lugar, te explicaré el rol crucial de TIM1 y TIM2, que son los que realmente rigen el sistema en este proyecto.

Aquí tienes el análisis detallado de la arquitectura de tu firmware.

1. Función de los archivos fuente
startup_stm32f446retx.s: Es el código en ensamblador que se ejecuta inmediatamente después de un reinicio del microcontrolador. Se encarga de inicializar el entorno de ejecución en C, configurando el puntero de pila y cargando la tabla de vectores de interrupción.

main.c: Contiene la rutina principal (main) de la aplicación. Aquí se inicializan los periféricos, los relojes del sistema, la capa HAL de ST, y se configuran y lanzan las tareas y el planificador (scheduler) de FreeRTOS.

stm32f4xx_it.c: Aloja las Rutinas de Servicio de Interrupción (ISRs) del sistema, tanto del núcleo ARM Cortex-M4 (como HardFault_Handler) como de los periféricos específicos del STM32 (como TIM1_UP_TIM10_IRQHandler y TIM2_IRQHandler).

FreeRTOSConfig.h: Contiene las definiciones de configuración estática para adaptar el núcleo de FreeRTOS a este hardware específico (como tamaño del heap, tasa del tick, y mapeo de interrupciones ARM a funciones de FreeRTOS).

freertos.c: Es un archivo de soporte que define "hooks" (funciones callback) del sistema operativo, como el control de desbordamiento de pila y las funciones necesarias para generar estadísticas de tiempo de ejecución de las tareas.

2. Evolución de las variables SysTick y SystemCoreClock
SystemCoreClock: Aunque no se declara explícitamente en el C que adjuntaste, esta variable global del estándar CMSIS representa la frecuencia del reloj central. Comienza con el valor por defecto del oscilador interno (generalmente 16 MHz en la familia F4) cuando se ejecuta la llamada a SystemInit desde el archivo de arranque. Posteriormente, al ejecutarse la función SystemClock_Config() dentro de main.c, el sistema configura el oscilador HSI y el PLL (Multiplicador a 336, Divisores a 16, 4, 2, etc.) para acelerar la CPU a su frecuencia de trabajo objetivo.

SysTick: Es el temporizador del sistema del núcleo Cortex-M. Al principio del programa, la HAL de ST suele usarlo para retardos, pero en FreeRTOSConfig.h se encuentra la macro #define xPortSysTickHandler SysTick_Handler. Esto significa que, una vez que FreeRTOS inicia (osKernelStart()), toma control absoluto del SysTick para generar una interrupción a 1000 Hz (configTICK_RATE_HZ) que actúa como el "latido" del sistema operativo, decidiendo qué tarea debe ejecutarse.

3. Comportamiento del programa desde Reset_Handler hasta el while(1)
La secuencia cronológica exacta de ejecución es la siguiente:

Inicio en Ensamblador (Reset_Handler): El microcontrolador inicia la ejecución aquí, carga el puntero de pila (_estack) y llama a SystemInit para la configuración básica del hardware.

Inicialización de Memoria: El código copia los datos con valores iniciales desde la memoria Flash hacia la SRAM (_sdata a _edata) y llena con ceros la sección de variables no inicializadas (_sbss a _ebss).

Llamada a constructores estáticos: Se invoca a __libc_init_array y finalmente el flujo salta a la función main del entorno C.

Configuración en main.c: Se ejecuta HAL_Init() para reiniciar periféricos e inicializar la interfaz Flash. Luego, SystemClock_Config() ajusta los relojes del sistema.

Inicialización de Periféricos: Se ejecutan las rutinas para configurar puertos GPIO (MX_GPIO_Init()), comunicaciones UART (MX_USART2_UART_Init()) y el temporizador TIM2 (MX_TIM2_Init()).

Arranque de TIM2: Se activa el temporizador 2 mediante interrupciones llamando a HAL_TIM_Base_Start_IT(&htim2).

Creación de Hilos (Threads): Se define y crea el hilo principal defaultTask mediante las APIs envolventes del CMSIS-RTOS (osThreadCreate).

Arranque del Sistema Operativo (osKernelStart()): Esta instrucción inicializa el planificador de FreeRTOS, enciende el SysTick para el RTOS y cede el control a las tareas.

El Bucle Infinito (Inalcanzable): Justo antes de llegar al while(1) principal de main.c, hay un comentario explícito: /* We should never get here as control is now taken by the scheduler */. El programa nunca llega a ejecutar el while(1) del main a menos que falle la asignación de memoria dinámica para inicializar el planificador. La ejecución normal se queda "atrapada" alternando entre las tareas de FreeRTOS, como StartDefaultTask que tiene su propio for(;;) con un osDelay(1).

4. Interacción de SysTick y los Timers (TIM1 y TIM2) con FreeRTOS
Dado que FreeRTOS "secuestra" el SysTick para administrar el tiempo entre tareas, la librería HAL de ST necesita una nueva base de tiempo para funciones como HAL_Delay().

TIM1 (Base de tiempo de HAL): Al no tener acceso a SysTick, la HAL utiliza las interrupciones del TIM1 (TIM1_UP_TIM10_IRQHandler que deriva a HAL_TIM_IRQHandler(&htim1)). Dentro de main.c, la función de callback HAL_TIM_PeriodElapsedCallback verifica si la interrupción proviene del TIM1 (htim->Instance == TIM1) y, de ser así, llama a HAL_IncTick() para incrementar el tiempo interno de la HAL.

TIM2 (Estadísticas de FreeRTOS): En el archivo FreeRTOSConfig.h, está activada la opción configGENERATE_RUN_TIME_STATS 1. Esto requiere un temporizador de alta frecuencia ajeno al SysTick. El TIM2 cumple este propósito generando interrupciones manejadas por TIM2_IRQHandler. En el callback de main.c, cada vez que desborda TIM2, se incrementa la variable ulHighFrequencyTimerTicks. Esta variable es proporcionada a FreeRTOS mediante la función getRunTimeCounterValue() en freertos.c para perfilar cuánto tiempo de CPU consume cada tarea.

TIM4: Como se mencionó antes, no existe declaración o inicialización de TIM4 (MX_TIM4_Init, o TIM4_IRQHandler) en este código fuente. Su función es nula en esta etapa del proyecto.


# Actividad 1 - paso 8

Este conjunto de archivos implementa una aplicación basada en el sistema operativo en tiempo real FreeRTOS. El propósito principal del código es leer el estado de un botón (con protección antirrebote) y controlar el parpadeo de un LED mediante máquinas de estados que se ejecutan en tareas (hilos) independientes y concurrentes.

A continuación, se detalla el funcionamiento de cada archivo y cómo interactúan entre sí:

1. app.c (Inicialización de la aplicación)
Este archivo es el punto de entrada para la configuración del RTOS y de la aplicación en sí.

Inicialización de variables: Configura en cero los contadores globales del sistema (g_app_tick_cnt, g_task_idle_cnt, g_app_stack_overflow_cnt).

Creación de tareas: Utiliza la API de FreeRTOS xTaskCreate para instanciar dos hilos de ejecución concurrentes: task_btn y task_led.

Prioridades: Ambas tareas son creadas con la misma prioridad (tskIDLE_PRIORITY + 1ul), lo que significa que el planificador de FreeRTOS alternará el tiempo de CPU entre ambas utilizando Time Slicing.

Validación: Se usa configASSERT(pdPASS == ret) para garantizar que el sistema se detenga si hay un error por falta de memoria al crear las tareas.

2. task_btn.c (Lógica del Botón)
Este archivo contiene la rutina de la tarea encargada de leer el botón físico.

Bucle principal: La función task_btn se ejecuta en un bucle infinito (for (;;)) donde evalúa continuamente la función task_btn_statechart().

Lectura de eventos: Primero, lee el estado del pin del botón mediante HAL_GPIO_ReadPin. Si está presionado, asigna el evento EV_BTN_XX_DOWN; si no, asigna EV_BTN_XX_UP.

Máquina de estados (Antirrebote o Debounce):

ST_BTN_XX_UP: Es el estado de reposo. Si detecta el botón presionado (EV_BTN_XX_DOWN), guarda el instante de tiempo actual (tick) y pasa al estado ST_BTN_XX_FALLING.

ST_BTN_XX_FALLING: Espera a que transcurra un tiempo de guarda definido por DEL_BTN_XX_MAX (50 ticks). Si pasado ese tiempo el botón sigue presionado, se confirma como una pulsación real (filtrando rebotes mecánicos), se envía el evento de parpadeo al LED mediante put_event_task_led(EV_LED_XX_BLINK) y avanza al estado ST_BTN_XX_DOWN.

ST_BTN_XX_DOWN: El botón está sólidamente presionado. Si se detecta que se suelta (EV_BTN_XX_UP), guarda el tiempo actual y avanza a ST_BTN_XX_RISING.

ST_BTN_XX_RISING: Funciona igual que la caída, pero evalúa que el botón se haya soltado completamente tras el tiempo DEL_BTN_XX_MAX. Si es así, manda a apagar el LED con put_event_task_led(EV_LED_XX_OFF) y vuelve a ST_BTN_XX_UP.

3. task_led_interface.c (Comunicación entre tareas)
Este archivo actúa como un puente de comunicación (Interfaz) entre la tarea del botón y la tarea del LED.

Contiene la función put_event_task_led(task_led_ev_t event), la cual es invocada por el botón.

Su trabajo es modificar la estructura global task_led_dta, asignando el nuevo evento (event) y levantando una bandera indicadora de que hay una nueva orden (task_led_dta.flag = true).

4. task_led.c (Lógica del LED)
Este archivo contiene la rutina que maneja el actuador (LED).

Bucle principal: Al igual que el botón, la función task_led corre un bucle infinito que invoca periódicamente a su máquina de estados task_led_statechart().

Máquina de estados:

ST_LED_XX_OFF: El LED está apagado. Constantemente evalúa si la bandera flag es true y si el evento recibido es EV_LED_XX_BLINK. Si esto ocurre, baja la bandera (flag = false), enciende el LED físicamente mediante la capa HAL, y cambia su estado a ST_LED_XX_BLINK.

ST_LED_XX_BLINK: Si recibe un evento de apagado (EV_LED_XX_OFF), baja la bandera, apaga el pin físico y vuelve a ST_LED_XX_OFF. En caso de no recibir orden de apagado, utiliza la función HAL_GPIO_TogglePin para invertir el estado lógico del LED intermitentemente cada vez que transcurre el tiempo DEL_LED_XX_MAX (500 ticks), logrando así el efecto de parpadeo.

5. freertos.c (Hooks o Retrollamadas del SO)
Este archivo no controla la lógica del negocio, sino que ofrece funciones "Hook" que FreeRTOS llama automáticamente bajo ciertas condiciones para monitoreo y depuración:

vApplicationIdleHook(void): Se ejecuta automáticamente solo cuando ninguna de las tareas (Botón o LED) tiene trabajo que hacer. Incrementa la variable g_task_idle_cnt para medir qué tan inactivo está el procesador.

vApplicationTickHook(void): Se ejecuta con cada interrupción del "Tick" (latido) del sistema operativo. Incrementa la variable g_app_tick_cnt.

vApplicationStackOverflowHook: Es una medida de seguridad. Si el sistema operativo detecta que una tarea consumió más memoria de pila de la que se le asignó al crearla, entrará a esta función, donde incrementará g_app_stack_overflow_cnt y bloqueará la ejecución permanentemente con configASSERT( 0 ).
