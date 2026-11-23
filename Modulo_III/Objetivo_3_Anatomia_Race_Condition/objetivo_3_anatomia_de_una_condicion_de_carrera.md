# Objetivo 3: Anatomía de una Condición de Carrera (Race Condition)

Una **Condición de Carrera (Race Condition)** —y en particular su variante **TOCTOU (Time-of-Check to Time-of-Use)**, clasificada como **CWE-362 / CWE-367**— ocurre cuando un sistema procesa múltiples peticiones concurrentes sobre un mismo recurso compartido sin sincronización adecuada, haciendo que el resultado final dependa del orden y la velocidad exacta con la que los hilos llegan a la base de datos.

---

## 1. ¿Cómo ocurre la ventana de vulnerabilidad (TOCTOU)?

Imaginemos un sistema de e-commerce que ofrece un cupón de descuento de **un solo uso** (`DESC50`). El flujo habitual en un backend no sincronizado consta de tres pasos separados:

1. **Verificación (*Time of Check*):** El servidor consulta la base de datos: *"¿El cupón `DESC50` todavía está disponible (`usado = false`)?"* → La BD responde que **sí**.
2. **Procesamiento de negocio:** El servidor aplica el descuento al carrito o acredita el beneficio al usuario (esto toma algunos milisegundos de cómputo o latencia de red).
3. **Actualización (*Time of Use*):** El servidor escribe en la base de datos: `UPDATE cupones SET usado = true WHERE codigo = 'DESC50'`.

El fallo reside en la **ventana de tiempo (Race Window)** que existe entre el paso 1 y el paso 3. Si el backend atiende múltiples hilos en paralelo sin bloquear el registro, todas las peticiones que lleguen dentro de esa fracción de segundo leerán `usado = false` antes de que la primera logre guardar `usado = true`.

---

## 2. Sincronización de peticiones simultáneas (ej. Turbo Intruder / HTTP/2)

Si se enviaran 50 peticiones de forma secuencial o mediante hilos simples sin sincronización, la latencia de red (*jitter*) haría que llegaran escalonadas y solo la primera tendría éxito. Para demostrar esta vulnerabilidad durante una auditoría de seguridad se utilizan técnicas de sincronización de paquetes (como las implementadas en **Burp Suite Repeater — Send group in parallel** o **Turbo Intruder**):

* **Técnica del último byte (*Last-Byte Sync* en HTTP/1.1) o *Single-Packet Attack* (en HTTP/2):**
  1. Se preparan 50 peticiones idénticas (`POST /api/cupones/canjear` con `{"codigo": "DESC50"}`).
  2. La herramienta abre las conexiones (o multiplexa los streams en una misma conexión HTTP/2) y envía casi todo el contenido de las 50 peticiones reteniendo el último byte o trama final en una barrera (*gate*).
  3. Una vez que todas están listas en espera, se libera el último byte simultáneamente en un único paquete TCP/IP.
* **Efecto en el servidor:** Las 50 peticiones llegan al backend en el mismo milisegundo. Varios *workers* ejecutan el `SELECT` de validación al mismo tiempo, todos ven `usado = false`, y varios de ellos aplican el descuento antes de que el primer `UPDATE` confirme el cambio en la base de datos. Como resultado, un cupón de un solo uso termina aplicándose múltiples veces sobre el mismo pedido o cuenta.

---

## 3. Cómo se mitiga correctamente

La única solución real consiste en garantizar la **atomicidad** y el **aislamiento** de la operación en la capa de datos:

* **Bloqueo pesimista a nivel de fila (*Row-Level Locking*):** Utilizar transacciones SQL con `SELECT ... FOR UPDATE` para bloquear el registro del cupón hasta que la transacción realice `COMMIT` o `ROLLBACK`. Las peticiones concurrentes quedan en cola y, al leer, ya encuentran `usado = true`.
* **Actualización condicional atómica (*Atomic Compare-and-Swap*):** Resolver la verificación y el cambio de estado en una sola sentencia SQL verificando la cantidad de filas afectadas (`affected_rows == 1`):

```sql
UPDATE cupones
SET usado = TRUE, usado_por = :user_id, usado_en = NOW()
WHERE codigo = :codigo AND usado = FALSE;
```

* **Restricciones de unicidad (`UNIQUE`):** Crear un índice único en la tabla de canjes (ej. `UNIQUE (cupon_id)` o `UNIQUE (cupon_id, usuario_id)`) para que el motor de base de datos rechace automáticamente cualquier inserción duplicada simultánea.
* **Claves de idempotencia y locks distribuidos:** En arquitecturas distribuidas (microservicios), emplear *idempotency keys* o bloqueos distribuidos (como Redis `SET NX` con TTL) junto con niveles de aislamiento transaccional estrictos (`SERIALIZABLE` o `REPEATABLE READ`).
