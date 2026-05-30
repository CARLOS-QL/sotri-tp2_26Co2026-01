# TP2 - Aplicación con Colas en FreeRTOS

En el proyecto STM32 se trabaja con FreeRTOS y se declara una cola mediante un `QueueHandle_t`, por ejemplo `h_btn_led_q`, que permite comunicar tareas como `task_btn` y `task_led`.

Además, la aplicación se inicializa desde `app_init()` antes de iniciar el scheduler con `osKernelStart()`.

---

## ¿Cómo eliminar una Cola?

Una cola se elimina usando la función:

```c
vQueueDelete(h_btn_led_q);
```

Esta función libera los recursos asociados a la cola. Después de eliminarla, el handler ya no debe utilizarse.

---

## ¿Cómo crear una Cola?

Una cola se crea utilizando:

```c
h_btn_led_q = xQueueCreate(cantidad_elementos, tamanio_elemento);
```

Ejemplo:

```c
h_btn_led_q = xQueueCreate(10, sizeof(task_led_ev_t));
```

Esto crea una cola con capacidad para almacenar 10 elementos del tipo `task_led_ev_t`.

---

## ¿Cómo gestiona una Cola los datos que contiene?

Una cola administra los datos en memoria mediante una estructura FIFO (First In First Out), es decir, el primer dato que entra es el primero que sale.

Cada elemento enviado a la cola es copiado internamente por FreeRTOS, por lo que es necesario indicar el tamaño del dato al momento de crear la cola.

---

## ¿Cómo enviar datos a una Cola?

Para enviar un dato se utiliza:

```c
xQueueSend(h_btn_led_q, &evento, portMAX_DELAY);
```

También puede utilizarse:

```c
xQueueSendToBack(h_btn_led_q, &evento, tiempo_bloqueo);
```

El dato enviado es copiado dentro de la cola.

---

## ¿Cómo recibir datos de una Cola?

Para recibir datos se utiliza:

```c
xQueueReceive(h_btn_led_q, &evento_recibido, portMAX_DELAY);
```

Si la cola contiene datos, la tarea obtiene el elemento más antiguo almacenado.

Si la cola está vacía, la tarea puede quedar bloqueada esperando información.

---

## ¿Qué significa bloquearse en una Cola?

Bloquearse significa que una tarea queda suspendida temporalmente hasta que la cola esté disponible.

Por ejemplo:

```c
xQueueReceive(h_btn_led_q, &evento, portMAX_DELAY);
```

Si la cola está vacía, la tarea permanecerá esperando hasta que otra tarea escriba un dato en ella.

---

## ¿Cómo bloquearse en varias Cola?

Para esperar eventos provenientes de varias colas puede utilizarse un `QueueSet`.

Ejemplo general:

```c
QueueSetHandle_t xQueueSet;

xQueueSet = xQueueCreateSet(10);

xQueueAddToSet(cola1, xQueueSet);
xQueueAddToSet(cola2, xQueueSet);

QueueSetMemberHandle_t cola_activa;

cola_activa = xQueueSelectFromSet(xQueueSet, portMAX_DELAY);
```

De esta manera una tarea puede esperar datos provenientes de múltiples colas simultáneamente.

---

## ¿Cómo sobrescribir datos en una Cola?

Para sobrescribir datos se utiliza:

```c
xQueueOverwrite(h_btn_led_q, &evento);
```

Esta función se utiliza principalmente en colas de un solo elemento.

Si la cola ya contiene información, el dato anterior es reemplazado por el nuevo.

---

## ¿Cómo vaciar una Cola?

Para vaciar completamente una cola se utiliza:

```c
xQueueReset(h_btn_led_q);
```

Esto elimina todos los datos almacenados, pero la cola continúa existiendo y puede reutilizarse.

---

## ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?

Las prioridades determinan qué tarea se ejecuta primero cuando varias tareas están listas para ejecutarse.

Si una tarea de mayor prioridad se encuentra bloqueada esperando datos desde una cola, al llegar un nuevo dato FreeRTOS puede desbloquearla inmediatamente y darle el control del procesador antes que a tareas de menor prioridad.

Por ejemplo, si `task_led` posee mayor prioridad que `task_btn`, al enviarse un evento desde la cola, `task_led` podrá ejecutarse inmediatamente para procesarlo.

En conclusión, las colas permiten comunicar tareas de manera segura, mientras que las prioridades determinan el orden de ejecución de dichas tareas.


# Paso 03 - Comunicación entre task_btn y task_led mediante Cola

Se modificó el mecanismo de comunicación entre las tareas `task_btn` y `task_led`, reemplazando la comunicación directa mediante `put_event_task_led()` por una Cola de FreeRTOS.

Se declaró un handler global de tipo `QueueHandle_t` llamado `h_btn_led_q`, el cual fue creado en `app_init()` mediante `xQueueCreate()`. La cola fue configurada para almacenar eventos del tipo `task_led_ev_t`.

La tarea `task_btn` fue modificada para enviar eventos hacia la cola utilizando `xQueueSendToBack()`. Cuando el pulsador es presionado se envía el evento `EV_LED_BLINK`, y cuando el pulsador es liberado se envía el evento `EV_LED_OFF`.

La tarea `task_led` fue modificada para recibir eventos desde la cola mediante `xQueueReceive()`. Cuando recibe el evento `EV_LED_BLINK`, el LED comienza a parpadear; cuando recibe el evento `EV_LED_OFF`, el LED se apaga.

Durante la depuración se observó que el comportamiento funcional del sistema se mantuvo igual al de la implementación anterior:

* al presionar el botón, el LED comienza a parpadear,
* al liberar el botón, el LED se apaga.

Sin embargo, la comunicación entre tareas pasó a realizarse mediante mensajes almacenados en una cola FIFO de FreeRTOS.

También se observó en la consola de depuración que:

* `task_btn` genera y envía correctamente los eventos,
* la cola almacena temporalmente dichos eventos,
* `task_led` recibe los eventos y ejecuta las acciones correspondientes.

Se utilizó transmisión no bloqueante en `task_btn` mediante:

```c
xQueueSendToBack(h_btn_led_q, &value_to_send, BTN_TICK_DEL_ZERO);
```

Esto evita que la tarea del botón quede detenida si la cola estuviera llena.

Para la recepción en `task_led` se utilizó `xQueueReceive()`, permitiendo desacoplar ambas tareas y mejorar la organización del sistema.

Con esta modificación el sistema implementa un modelo productor-consumidor:

* `task_btn` actúa como productor de eventos,
* `task_led` actúa como consumidor,
* la cola funciona como mecanismo de sincronización y transferencia de datos entre tareas.

