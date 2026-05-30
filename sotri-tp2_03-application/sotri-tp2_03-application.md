# TP2 - Aplicación 03: Semáforos en FreeRTOS

## ¿Cómo crear y usar semáforos binarios y semáforos contadores?

En FreeRTOS, los semáforos permiten sincronizar tareas o coordinar el acceso a ciertos eventos del sistema. No se utilizan principalmente para transferir datos, sino para indicar que ocurrió un evento o que un recurso está disponible.

---

## Semáforo binario

Un semáforo binario solo puede tener dos estados:

* disponible
* no disponible

Se puede entender como una señal de tipo encendido/apagado.

### Creación de un semáforo binario

```c
SemaphoreHandle_t h_btn_led_bin_sem;

h_btn_led_bin_sem = xSemaphoreCreateBinary();

configASSERT(h_btn_led_bin_sem != NULL);
```

### Entregar un semáforo binario

Cuando una tarea o interrupción quiere avisar que ocurrió un evento, puede liberar el semáforo con:

```c
xSemaphoreGive(h_btn_led_bin_sem);
```

Desde una interrupción se debe usar:

```c
BaseType_t xHigherPriorityTaskWoken = pdFALSE;

xSemaphoreGiveFromISR(h_btn_led_bin_sem, &xHigherPriorityTaskWoken);

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

### Tomar un semáforo binario

Una tarea puede esperar el evento usando:

```c
xSemaphoreTake(h_btn_led_bin_sem, portMAX_DELAY);
```

Si el semáforo no está disponible, la tarea puede quedar bloqueada hasta que otra tarea o interrupción lo libere.

---

## Semáforo contador

Un semáforo contador puede almacenar una cantidad mayor a uno. Se utiliza cuando pueden acumularse varios eventos o cuando existen varios recursos disponibles.

### Creación de un semáforo contador

```c
SemaphoreHandle_t h_counter_sem;

h_counter_sem = xSemaphoreCreateCounting(5, 0);

configASSERT(h_counter_sem != NULL);
```

En este ejemplo:

* `5` es el valor máximo del contador.
* `0` es el valor inicial.

### Entregar un semáforo contador

Cada vez que ocurre un evento, se incrementa el contador:

```c
xSemaphoreGive(h_counter_sem);
```

### Tomar un semáforo contador

Cada vez que una tarea atiende un evento, decrementa el contador:

```c
xSemaphoreTake(h_counter_sem, portMAX_DELAY);
```

Si el contador es cero, la tarea queda bloqueada hasta que se libere el semáforo.

---

## Ejemplo de uso en el proyecto STM32

En el proyecto STM32, una tarea puede detectar el estado del botón y liberar un semáforo para avisar a otra tarea. Por ejemplo, `task_btn` puede liberar un semáforo cuando detecta una pulsación, mientras que `task_led` puede esperar ese semáforo para ejecutar una acción sobre el LED.

Ejemplo general:

```c
/* En task_btn */
xSemaphoreGive(h_btn_led_bin_sem);

/* En task_led */
if (xSemaphoreTake(h_btn_led_bin_sem, portMAX_DELAY) == pdTRUE)
{
    HAL_GPIO_TogglePin(LED_A_PORT, LED_A_PIN);
}
```

Con este mecanismo, `task_led` no necesita consultar constantemente el estado del botón. Solo se ejecuta cuando recibe la señal mediante el semáforo.

---

## ¿Cuáles son las diferencias entre semáforos binarios y semáforos contadores?

La diferencia principal está en la cantidad de eventos o recursos que pueden representar.

Un semáforo binario solo puede indicar si ocurrió o no ocurrió un evento. Su valor práctico es 0 o 1. Si se libera varias veces antes de que una tarea lo tome, no acumula múltiples eventos.

Un semáforo contador, en cambio, sí puede acumular varios eventos hasta un valor máximo definido. Cada llamada a `xSemaphoreGive()` incrementa el contador y cada llamada a `xSemaphoreTake()` lo decrementa.

Por ejemplo, si se presiona un botón varias veces rápidamente, un semáforo binario solo indicará que existe una pulsación pendiente. En cambio, un semáforo contador puede registrar varias pulsaciones pendientes, siempre que no supere su valor máximo.

---


---

## Observación durante la depuración

# Paso 03 - Comunicación entre task_btn y task_led mediante Semáforo Binario

Se modificó el mecanismo de comunicación entre las tareas `task_btn` y `task_led`, reemplazando el mecanismo anterior por un semáforo binario de FreeRTOS.

Para ello, se declaró un handler de tipo `SemaphoreHandle_t` llamado `h_btn_led_bin_sem`, el cual fue creado en `app_init()` mediante la función:

```c
h_btn_led_bin_sem = xSemaphoreCreateBinary();
```

La tarea `task_btn` fue modificada para liberar el semáforo utilizando `xSemaphoreGive()` cuando se detecta un evento del pulsador.

Cuando el botón es presionado, la tarea genera el evento `EV_LED_BLINK` y libera el semáforo:

```c
task_led_dta.event = EV_LED_BLINK;
xSemaphoreGive(h_btn_led_bin_sem);
```

Cuando el botón es liberado, se genera el evento `EV_LED_OFF` y nuevamente se libera el semáforo:

```c
task_led_dta.event = EV_LED_OFF;
xSemaphoreGive(h_btn_led_bin_sem);
```

La tarea `task_led` fue modificada para tomar el semáforo mediante `xSemaphoreTake()`:

```c
if (xSemaphoreTake(h_btn_led_bin_sem, LED_TICK_DEL_ZERO) == pdTRUE)
{
    task_led_dta.flag = true;
}
```

Durante la depuración inicialmente se utilizó una espera bloqueante con `portMAX_DELAY`, lo que provocó que la tarea `task_led` permaneciera bloqueada esperando el semáforo y el LED dejara de parpadear, permaneciendo encendido mientras el botón estaba presionado.

Posteriormente se reemplazó por una recepción no bloqueante utilizando `LED_TICK_DEL_ZERO`, permitiendo que la tarea continuara ejecutando periódicamente la máquina de estados y que el LED volviera a parpadear correctamente.

Se observó que:

* `task_btn` libera correctamente el semáforo al detectar eventos del botón.
* `task_led` recibe correctamente la señal mediante `xSemaphoreTake()`.
* el LED comienza a parpadear al presionar el botón.
* el LED se apaga al liberar el botón.

A diferencia de la implementación con colas realizada en el TP2-02, el semáforo binario no transfiere datos entre tareas, sino que únicamente sincroniza la ejecución mediante señales de evento.

Con esta modificación el sistema implementa sincronización entre tareas utilizando semáforos binarios de FreeRTOS.
