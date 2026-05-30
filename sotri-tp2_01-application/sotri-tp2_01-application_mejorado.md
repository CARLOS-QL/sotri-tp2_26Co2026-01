# Análisis del Proyecto STM32 con FreeRTOS

## Paso 6 Explicación de los archivos main.c, stm32f4xx_it.c, freertos.c, FreeRTOSConfig.h y startup_stm32f446re:

### 1. Evolución de las variables SysTick y SystemCoreClock desde el inicio hasta el while(1) de `main.c`La evolución temporal y los cambios de valor de estas variables clave del hardware y del software CMSIS siguen la siguiente secuencia de eventos:
#### Fase de Reset (`Reset_Handler`)
Al energizar el microcontrolador, se ejecuta el vector de Reset en el archivo ensamblador.

- **SystemCoreClock:** Por defecto, al arrancar, el microcontrolador opera con su oscilador interno básico de baja velocidad HSI (High Speed Internal). El valor inicial asignado en las librerías CMSIS antes de la reconfiguración suele ser el valor de fábrica (usualmente 16 MHz en la serie F4).

- **SysTick:** El periférico de hardware SysTick se encuentra completamente deshabilitado (registros de control en 0), por lo que no genera interrupciones ni incrementa variables de conteo.

#### Llamada a `SystemInit()`
Dentro de Reset_Handler, se ejecuta bl SystemInit. Esta función reestablece el mapa de registros del reloj (RCC) a un estado inicial seguro. El valor de SystemCoreClock sigue reflejando la frecuencia del HSI. El SysTick se mantiene deshabilitado.

#### Ejecución de `main()` y llamada a ``HAL_Init()``
El programa salta a la función main() e invoca inmediatamente `HAL_Init()`.

Por defecto, en la capa HAL de ST, `HAL_Init()` configura un temporizador para la base de tiempo interna de la HAL (funciones HAL_Delay y timeouts). En este proyecto específico, Timer 1 (TIM1) ha sido seleccionado como la base de tiempo de la HAL en lugar de SysTick.

- **SysTick:** Sigue deshabilitado en hardware, ya que la base de tiempo de la HAL utiliza TIM1.

- **SystemCoreClock:** Sigue manteniendo la frecuencia inicial del HSI de fábrica.

#### Llamada a ``SystemClock_Config()``
Se ejecuta esta función en `main.c` para encender el PLL configurando el HSI como fuente.

De acuerdo a las ecuaciones del PLL (PLLM = 16, PLLN = 336, PLLP = DIV4), el reloj del sistema (SYSCLK) se eleva a 84 MHz.

Hacia el final de esta inicialización de los relojes, la función interna de ST (HAL_RCC_ClockConfig) calcula y actualiza la variable global interna:$$\mathbf{SystemCoreClock = 84000000 \text{ Hz (84 MHz)}}$$
- **SysTick:** Aún permanece deshabilitado.

#### Llamada a ``osKernelStart()`` (Lanzamiento de FreeRTOS)
Antes del lazo while(1), el código inicializa periféricos (incluido el Timer 2) y crea las tareas iniciales. Finalmente invoca `osKernelStart()`.

Dentro del núcleo de FreeRTOS, al arrancar el planificador (vTaskStartScheduler), el RTOS se encarga de configurar el hardware del SysTick utilizando la macro configCPU_CLOCK_HZ (asociada directamente a SystemCoreClock, es decir, 84 MHz) y configTICK_RATE_HZ (definida en 1000 Hz).

- **SysTick:** En este preciso instante, el periférico SysTick se activa en hardware, cargándose con el valor de conteo para desbordarse cada 1 milisegundo (84,000 ciclos de reloj). A partir de aquí, la variable interna del kernel de FreeRTOS (xTickCount) comienza a evolucionar incrementándose en 1 con cada interrupción del SysTick.

### 2. Comportamiento del programa desde Reset_Handler hasta antes del while(1) de `main.c`
El flujo de control secuencial del programa realiza las siguientes acciones críticas de inicialización de bajo nivel y del sistema operativo:

Inicialización de memoria (Reset_Handler):

El procesador extrae el puntero de pila inicial (_estack) y salta a la dirección del Reset_Handler.

Configura el puntero de pila (sp).

Llama a SystemInit para inicializar la interfaz de memoria flash y los registros de control de reloj por defecto.

Realiza la copia del segmento de datos inicializados (.data) desde la memoria Flash hacia la memoria RAM.

Limpia y llena con ceros el segmento de variables no inicializadas (.bss) en la RAM.

Ejecuta los constructores estáticos llamando a __libc_init_array y finalmente salta a la función main().

Configuración de la Capa de Abstracción de Hardware (main -> HAL_Init):

En main(), se invoca `HAL_Init()`. Esto configura la latencia de la memoria Flash, habilita la caché de instrucciones/datos si aplica, establece la prioridad del grupo de interrupciones de la CPU y arranca el temporizador interno de la HAL (TIM1) para el conteo de milisegundos de la plataforma.

Configuración Avanzada del Reloj (SystemClock_Config):

Modifica los multiplexores de reloj para activar el oscilador HSI, enganchar el circuito multiplicador PLL a 84 MHz y configurar los divisores de los buses periféricos AHB, APB1 y APB2.

Inicialización de Periféricos de la Aplicación:

Se ejecutan las funciones autogeneradas por CubeMX: MX_GPIO_Init(), MX_USART2_UART_Init() y MX_TIM2_Init() para configurar los pines de entrada/salida, los parámetros de la comunicación serial UART y las propiedades del Timer 2.

Se inicia explícitamente el Timer 2 en modo interrupción con HAL_TIM_Base_Start_IT(&htim2).

Se ejecuta la inicialización propia de la aplicación mediante la rutina app_init().

Creación de Objetos del Sistema Operativo (FreeRTOS):

El programa define y crea estática o dinámicamente las estructuras del RTOS mediante la capa CMSIS-RTOS V1. Se define el hilo de ejecución por defecto defaultTask apuntando a la función StartDefaultTask con prioridad normal y un tamaño de stack de 128 palabras. El hilo es instanciado con osThreadCreate().

Lanzamiento del Planificador (osKernelStart):

Se llama a `osKernelStart()`. En este momento, el código principal de main() "pierde" el control y la CPU pasa a ser gobernada por el planificador de FreeRTOS. El planificador realiza un cambio de contexto forzado hacia la tarea de mayor prioridad lista para ejecutarse (en este caso, defaultTask).

Nota Crítica: Debido a esto, el flujo de ejecución nunca llega al lazo while(1) infinito que se encuentra al final de main(), a menos que ocurra una falla crítica por falta de memoria RAM al intentar alojar el Kernel.

### 3. Interacción del SysTick y el Timer 1 con FreeRTOS
En esta implementación, el diseño separa de forma limpia el temporizador del sistema operativo del temporizador de las funciones de soporte de STMicroelectronics (HAL):

Interacción del SysTick con FreeRTOS (¿Cómo y para qué?):

- **¿Cómo?:** En el archivo `FreeRTOSConfig.h`, el manejador de interrupción nativo de FreeRTOS xPortSysTickHandler se mapea directamente mediante una directiva #define al nombre estándar de la tabla de vectores del ARM Cortex-M: SysTick_Handler.

- **¿Para qué?:** El SysTick actúa como el corazón rítmico (Heartbeat) de FreeRTOS. Cada 1 ms (configurado a 1000 Hz), el hardware del SysTick interrumpe a la CPU. La rutina de FreeRTOS atiende la interrupción, incrementa el contador de tiempo del sistema operativo (xTickCount), verifica si el tiempo de bloqueo de alguna tarea ha expirado, evalúa si es necesario realizar un cambio de contexto (Preemption) y ejecuta las funciones periódicas del usuario registradas en el hook del Tick (vApplicationTickHook).

Interacción del Timer 1 (TIM1) con FreeRTOS:

- **¿Cómo?:** En el archivo stm32f4xx_it.c, se observa la rutina TIM1_UP_TIM10_IRQHandler que invoca el manejador de interrupción genérico de la capa HAL: HAL_TIM_IRQHandler(&htim1).

- **¿Para qué?:** Dado que FreeRTOS toma posesión absoluta del SysTick, la capa HAL de ST no puede utilizarlo de forma segura para sus funciones de retardo (HAL_Delay) sin generar conflictos de prioridades o problemas de concurrencia. Por lo tanto, TIM1 se configura para dispararse de manera independiente y exclusiva cada 1 ms. En su rutina de interrupción periódica (HAL_TIM_PeriodElapsedCallback), se invoca HAL_IncTick(), lo que incrementa la variable interna de tiempo real de la HAL (uwTick). Esto asegura que las llamadas de temporización propias de la HAL sigan funcionando correctamente en un entorno multihilo.

### 4. Interacción del Timer 2 (TIM2) con la HAL del proyecto STM32
El Timer 2 (TIM2) tiene una interacción especializada con la HAL orientada a la monitorización y diagnóstico del rendimiento del firmware (Run-Time Statistics):

¿Cómo interactúa?:

En `main.c`, se inicializa el hardware del Timer 2 mediante MX_TIM2_Init() y se habilita su interrupción periódica en la línea HAL_TIM_Base_Start_IT(&htim2).

En el archivo de interrupciones stm32f4xx_it.c, la función del vector físico TIM2_IRQHandler captura el evento del temporizador y delega el procesamiento a la rutina unificada HAL_TIM_IRQHandler(&htim2).

La arquitectura de la HAL procesa el evento, limpia las banderas de hardware y realiza una llamada de retorno (Callback) hacia la función HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) ubicada en `main.c`.

Dentro de este callback, el código evalúa explícitamente si la interrupción proviene del Timer 2 (if (htim->Instance == TIM2)) y, si es verdadero, incrementa la variable global de alta frecuencia denominada ulHighFrequencyTimerTicks++.

¿Para qué interactúa?:

El propósito fundamental del TIM2 en este diseño es servir como un contador de tiempo de alta resolución y alta frecuencia independiente de la base de tiempo estándar de 1 ms.

Esto se puede constatar en `FreeRTOSConfig.h`, donde se declaran las funciones externas configureTimerForRunTimeStats y getRunTimeCounterValue, las cuales se vinculan internamente con el acumulador ulHighFrequencyTimerTicks.

Gracias a esta interacción guiada por la HAL, FreeRTOS adquiere la capacidad de medir con precisión milimétrica cuántos ciclos de procesamiento consume cada tarea individual en la CPU. Esto permite generar estadísticas detalladas de carga de trabajo en tiempo de ejecución para optimizar el rendimiento del sistema embebido.

## Paso 8: Análisis de los archivos app.c, app_it.c, task_btn.c, task_led.c, task_led_interface.c y freertos.c

A continuación se presenta un análisis y explicación detallada del funcionamiento del código fuente provisto en la segunda tanda de archivos adjuntos (`app.c`, `app_it.c`, `task_btn.c`, `task_led.c`, `task_led_interface.c` y `freertos.c`).

Este conjunto de archivos implementa una aplicación embebida multi-tarea basada en FreeRTOS empleando un diseño de arquitectura orientada a eventos, ocultamiento de información y modelado formal mediante Máquinas de Estado Finitas (MEF / Statecharts) del tipo Run-to-Completion (Ejecución completa antes del próximo evento).

### 1. Arquitectura General y Flujo de Diseño
La aplicación tiene como objetivo primordial desacoplar la lectura física de un botón (con debounce/antirrebote por software) de la acción física de parpadear un diodo LED. En lugar de interconectar los componentes mediante variables globales desprotegidas, se utiliza un módulo de interfaz para el paso de eventos asíncronos.

El flujo de control lógico e interconexión entre módulos sigue la siguiente estructura:

`app_it.c` (Capa de Interrupciones Externas): Captura el evento asíncrono de hardware del botón.

`task_btn.c` (Tarea de Procesamiento de Botón): Ejecuta una MEF para filtrar el ruido (debounce) y validar la pulsación.

`task_led_interface.c` (Capa de Mensajería/Abstracción): Provee una función segura para depositar el evento validado.

`task_led.c` (Tarea de Control del LED): Ejecuta otra MEF periódica que reacciona a los eventos depositados para encender, apagar o destellar el LED físico.

`app.c` (Inicializador del Subsistema): Crea las tareas e hilos bajo el planificador de FreeRTOS.

### 2. Análisis Detallado Archivo por Archivo
A. `app.c` (Inicialización y Alojamientos del RTOS)
Este archivo funciona como el punto de entrada lógico de la aplicación una vez que el kernel de FreeRTOS está listo. Su función principal es app_init():

Creación de Tareas: Invoca de manera nativa la API de FreeRTOS `xTaskCreate` para instanciar de forma dinámica dos hilos concurrentes:

task_btn: Tarea asignada al procesamiento del botón físico, configurada con un tamaño de stack equivalente a dos veces el mínimo (2 * configMINIMAL_STACK_SIZE) y una prioridad de tiempo real por encima de la tarea Idle (tskIDLE_PRIORITY + 1ul).

task_led: Tarea asignada a controlar el parpadeo del LED, con idénticas propiedades de stack y prioridad que la tarea del botón.

Diagnóstico de Memoria: Utiliza la función xPortGetFreeHeapSize() para verificar los bytes disponibles remanentes en el Heap (esquema heap_4.c), garantizando que la creación de hilos no haya comprometido el direccionamiento de memoria RAM.

B. `app_it.c` (Manejador de Interrupciones de Usuario)
Contiene la lógica de respuesta rápida para los eventos asíncronos de hardware externos (GPIO):

Provee la función sobreescrita `HAL_GPIO_EXTI_Callback`(uint16_t GPIO_Pin).

Al detectar un flanco (de bajada o subida) en el pin configurado para el botón (BTN_Pin), evalúa el estado físico del pin mediante HAL_GPIO_ReadPin.

Si el botón se encuentra presionado (asociado eléctricamente a BTN_ON), despacha de inmediato el evento primitivo de hardware EV_BTN_DOWN mediante la función interna put_event_task_btn(). De lo contrario, si se liberó, despacha EV_BTN_UP.

C. `task_btn.c` (Debounce y MEF del Botón)
Este archivo encapsula por completo el comportamiento del botón aplicando ocultamiento de información (Information Hiding) mediante la estructura estática privada task_btn_dta.

Función de Tarea (task_btn): Contiene un lazo infinito while(1). Adopta un enfoque periódico estricto usando `vTaskDelayUntil`(&last_wake_time, BTN_TICK_DEL_MAX). Esto garantiza que la tarea se despierte exactamente cada 40 milisegundos (BTN_TICK_DEL_MAX) para muestrear la MEF, aislando el comportamiento de las variaciones del planificador.

Máquina de Estados (task_btn_statechart): Implementa un Statechart Run-to-Completion de 4 estados para filtrar los rebotes mecánicos del pulsador:

ST_BTN_UP (Reposo): Espera el evento de hardware EV_BTN_DOWN. Al recibirlo, guarda la marca de tiempo tick actual (xTaskGetTickCount()) y transiciona a ST_BTN_FALLING.

ST_BTN_FALLING (Validación de Bajada): Espera a que transcurra un tiempo de guarda mínimo de filtrado. Si al cumplirse el tiempo el evento sigue siendo EV_BTN_DOWN, se confirma la pulsación legítima. Realiza la acción de salida: notifica al LED invocando put_event_task_led(EV_LED_BLINK) y transiciona a ST_BTN_DOWN. Si fue un falso positivo (ruido), regresa a ST_BTN_UP.

ST_BTN_DOWN (Presionado Estable): Queda a la espera del evento de hardware EV_BTN_UP (liberación del botón) para transicionar a ST_BTN_RISING salvando el tiempo del tick.

ST_BTN_RISING (Validación de Subida): Aplica la misma lógica de retraso para el flanco de subida. Al confirmarse la liberación prolongada, transiciona de vuelta a su estado de reposo ST_BTN_UP.

D. `task_led_interface.c` (Capa de Abstracción de Datos / Interfaz)
Actúa como un puente seguro y un búfer intermedio de comunicación entre la tarea productora (task_btn) y la tarea consumidora (task_led).

Contiene la función pública put_event_task_led(task_led_ev_t event).

Cuando la MEF del botón confirma una pulsación válida, invoca esta función pasando la señal (por ejemplo, EV_LED_BLINK).

La función actualiza la estructura interna del LED modificando el miembro .event y colocando en true una bandera lógica de actualización (.flag = true). Esto emula el comportamiento de un buzón de mensajería básico mono-evento.

E. `task_led.c` (Control de Salida y MEF del LED)
Al igual que el módulo del botón, encapsula localmente el hardware del diodo LED (Puerto GPIO y Pin correspondientes) bajo variables estrictamente static.

Función de Tarea (task_led): Se ejecuta de forma periódica e independiente cada 50 milisegundos (LED_TICK_DEL_MAX) empleando la API `vTaskDelayUntil`. En cada ciclo de ejecución, invoca la rutina evaluadora de estados task_led_statechart().

Máquina de Estados (task_led_statechart): Controla las transiciones visuales del LED:

ST_LED_OFF (LED Apagado): Evalúa si existe una bandera activa (.flag == true) y si el evento recibido es EV_LED_BLINK. De cumplirse, limpia la bandera, almacena el tick de inicio, cambia al estado ST_LED_BLINK y energiza el pin físico del hardware con `HAL_GPIO_WritePin`(..., LED_ON).

ST_LED_BLINK (LED Destellando): Mientras se mantenga en este estado, evalúa de forma síncrona si ha expirado el tiempo de parpadeo configurado (LED_DEL_MAX). Si el tiempo expira, conmuta el pin de hardware (`HAL_GPIO_TogglePin`) para generar el efecto visual y resetea el contador de tiempo. Si de forma asíncrona la interfaz deposita un evento EV_LED_OFF, limpia la bandera, apaga el pin físico (LED_OFF) y regresa a ST_LED_OFF.

F. `freertos.c` (Hooks de Diagnóstico y Métricas)
Este archivo contiene las extensiones de instrumentación del kernel de FreeRTOS, las cuales han sido activadas mediante directivas en `FreeRTOSConfig.h`:

vApplicationIdleHook(void): Se ejecuta continuamente cuando el procesador no tiene tareas de usuario prioritarias que resolver (hilo Idle). Incrementa un contador global (g_task_idle_cnt++), útil para telemetría y cálculo del porcentaje de tiempo ocioso (estimación de uso de CPU).

vApplicationTickHook(void): Se ejecuta estrictamente dentro del contexto de la interrupción del SysTick (cada 1 ms). Incrementa la variable g_app_tick_cnt++. Es un gancho de alta velocidad para software de medición de tiempos reales.

vApplicationStackOverflowHook(...): Rutina de seguridad crítica. Si el puntero de pila de alguna de las tareas creadas (task_btn o task_led) desborda los límites asignados en el Heap, el kernel captura el fallo, extrae el nombre de la tarea ofensora (pcTaskName) y congela la ejecución para evitar la corrupción de datos en la memoria RAM.

