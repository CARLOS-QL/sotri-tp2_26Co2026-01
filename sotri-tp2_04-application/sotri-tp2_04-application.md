# TP2 - Actividad 04: Interrupciones y FreeRTOS

## ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción?

Dentro de una rutina de servicio de interrupción no se deben usar las funciones normales de FreeRTOS que puedan bloquear la ejecución. En su lugar, se deben utilizar las funciones especiales terminadas en `FromISR`.

Algunos ejemplos son:

```c
xQueueSendFromISR();
xQueueReceiveFromISR();
xSemaphoreGiveFromISR();
xSemaphoreTakeFromISR();
xTaskNotifyFromISR();
vTaskNotifyGiveFromISR();
xEventGroupSetBitsFromISR();
```

Estas funciones están diseñadas para ejecutarse desde una interrupción y permiten comunicarse con tareas de forma segura.

---

## ¿Métodos para delegar el procesamiento de interrupciones a una Tarea?

Una interrupción debe ejecutar la menor cantidad de código posible. Por eso, una técnica común es delegar el procesamiento pesado a una tarea.

Esto se puede hacer mediante:

- Semáforo binario
- Cola
- Notificación directa a tarea
- Event groups

Por ejemplo, una interrupción puede liberar un semáforo con:

```c
xSemaphoreGiveFromISR(h_btn_led_bin_sem, &xHigherPriorityTaskWoken);
```

Luego una tarea queda esperando con:

```c
xSemaphoreTake(h_btn_led_bin_sem, portMAX_DELAY);
```

De esta manera, la interrupción solo avisa que ocurrió un evento y la tarea realiza el procesamiento principal.

---

## ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?

Para enviar datos desde una interrupción hacia una tarea se usa:

```c
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
task_led_ev_t value_to_send = EV_LED_BLINK;

xQueueSendFromISR(h_btn_led_q, &value_to_send, &xHigherPriorityTaskWoken);

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

La tarea puede recibir el dato con:

```c
task_led_ev_t received_value;

if (xQueueReceive(h_btn_led_q, &received_value, portMAX_DELAY) == pdPASS)
{
    task_led_dta.event = received_value;
    task_led_dta.flag = true;
}
```

También es posible recibir datos desde una cola dentro de una interrupción usando:

```c
xQueueReceiveFromISR();
```

Sin embargo, lo más común es que la ISR envíe datos a una cola y que una tarea los procese.

---

## ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?

En algunas portaciones de FreeRTOS, como Cortex-M, existe un modelo de anidamiento de interrupciones basado en prioridades.

Esto significa que una interrupción de mayor prioridad puede interrumpir a otra de menor prioridad.

Sin embargo, no todas las interrupciones pueden llamar funciones de FreeRTOS. Solo pueden usar funciones `FromISR` aquellas interrupciones cuya prioridad sea igual o menor que la prioridad máxima permitida por FreeRTOS.

En Cortex-M esto se controla con:

```c
configMAX_SYSCALL_INTERRUPT_PRIORITY
```

Si una interrupción tiene una prioridad demasiado alta, no debe llamar funciones de FreeRTOS, porque podría afectar el funcionamiento del kernel.

En resumen, el modelo permite interrupciones anidadas, pero las llamadas al kernel desde ISR deben respetar las prioridades configuradas en FreeRTOS.



## Paso 03 - Atención del botón por interrupción

Se modificó `task_btn` para atender el botón mediante interrupción externa, reemplazando el mecanismo de consulta periódica del pulsador.

La interrupción del botón se atiende en `HAL_GPIO_EXTI_Callback()`. Dentro de esta rutina se libera un semáforo binario mediante `xSemaphoreGiveFromISR()`, permitiendo despertar a `task_btn` cuando ocurre un evento del pulsador.

```c
xSemaphoreGiveFromISR(h_btn_led_bin_sem, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

En `task_btn`, la tarea queda bloqueada esperando el semáforo mediante `xSemaphoreTake()`:

```c
if (xSemaphoreTake(h_btn_led_bin_sem, portMAX_DELAY) == pdTRUE)
{
    task_btn_statechart();
}
```

Durante la depuración se verificó que el botón ya no se atiende por polling cada 50 ms, sino mediante interrupción. La tarea `task_btn` permanece bloqueada hasta que la ISR libera el semáforo.

También se comprobó temporalmente el ingreso a la rutina de interrupción agregando un mensaje de depuración dentro de `HAL_GPIO_EXTI_Callback()`. Al presionar el botón, el comportamiento del sistema cambió, confirmando que la ISR sí se ejecutaba. Luego se retiró el mensaje porque no es recomendable usar funciones de impresión dentro de una interrupción.

Se configuró el GPIO del botón para detectar flancos de subida y bajada:

```c
GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING_FALLING;
```

Con esta configuración se detecta tanto la presión como la liberación del pulsador.

Finalmente, se observó que el sistema mantiene el comportamiento esperado:

* al presionar el botón, el LED comienza a parpadear;
* al liberar el botón, el LED se apaga.

Con esta modificación, la comunicación entre la rutina de interrupción y `task_btn` se realiza mediante un semáforo binario de FreeRTOS.

