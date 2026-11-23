# Objetivo 1: El Arte de la Inyección Ciega (Blind SQLi)

En una **Inyección SQL Ciega (*Blind SQL Injection*, CWE-89)**, la aplicación es vulnerable a la manipulación de consultas SQL, pero **no devuelve en la respuesta HTTP ni los datos consultados ni mensajes de error de la base de datos**. En lugar de leer resultados directamente (como en un *Error-Based* o *Union-Based SQLi*), el auditor debe inferir la información bit a bit o carácter por carácter formulando preguntas de "sí o no" a la base de datos.

---

## 1. Diferencia técnica: Boolean-Based Blind vs. Time-Based Blind

| Criterio | Boolean-Based Blind (Basada en Booleanos) | Time-Based Blind (Basada en Tiempo) |
| :--- | :--- | :--- |
| **Señal observable (Oráculo)** | Cambio observable en la **respuesta HTTP** (contenido HTML diferente, longitud de bytes distinta, código `200` vs `404`, presencia o ausencia de un elemento). | Cambio observable en el **tiempo de respuesta** del servidor (respuesta inmediata vs. retraso intencional de *N* segundos). |
| **Cuándo se utiliza** | Cuando la página refleja al menos un estado binario (por ejemplo, muestra un producto si la condición es verdadera o "No encontrado" si es falsa). | Cuando la respuesta es **100% idéntica** sin importar si la consulta tuvo éxito o falló (por ejemplo, una pantalla genérica de *"Cargando..."* o *"Solicitud recibida"*). |
| **Mecanismo interno en BD** | Evalúa una condición lógica añadida a la cláusula `WHERE` (verdadera o falsa). | Combina una condición lógica con una función de pausa del motor SQL (como `SLEEP(n)` en MySQL, `pg_sleep(n)` en PostgreSQL o `WAITFOR DELAY` en SQL Server). |
| **Velocidad y confiabilidad** | Más rápida y menos sensible al ruido de red. | Más lenta y sensible a la latencia o congestión de red. |

---

## 2. ¿Cómo se extrae el nombre de la base de datos letra por letra ante una pantalla genérica de "Cargando..."?

Cuando la aplicación responde siempre exactamente con la misma pantalla genérica (sin cambios en contenido ni código de estado), el canal booleano no existe y el único canal lateral disponible es el **tiempo de respuesta (*Time-Based Blind SQLi*)**.

El proceso lógico de inferencia carácter por carácter funciona de la siguiente manera:

### Paso 1: Determinar la longitud del nombre
Se consulta primero cuántos caracteres tiene el nombre de la base de datos actual (`database()` / `current_database()`), evaluando condiciones sobre `LENGTH(database())`:
* *"Si la longitud es igual a 1, esperar 5 segundos; de lo contrario, responder de inmediato."*
* Si la respuesta llega en `0.2s`, la condición es **falsa**. Si tarda `~5.2s`, la condición es **verdadera** y se conoce el largo exacto (por ejemplo, 6 caracteres).

### Paso 2: Extraer cada posición carácter por carácter
Conocida la longitud, se toma cada posición `i` (de `1` hasta `N`) mediante funciones de subcadena (`SUBSTRING` / `SUBSTR`) y su valor numérico (`ASCII`):
* *"Si `ASCII(SUBSTRING(database(), 1, 1))` es igual al carácter `'a'` (97), pausar 5 segundos; si no, responder sin demora."*
* Para reducir drásticamente la cantidad de peticiones necesarias, en lugar de probar linealmente letra por letra (`a, b, c...`), se utiliza **búsqueda binaria** sobre el rango ASCII imprimible (`32` a `126`):
  1. ¿El código ASCII del primer carácter es `> 79`? (Si tarda 5s → sí; si responde rápido → no).
  2. Se divide el rango a la mitad en cada pregunta, identificando cualquier carácter en un máximo de **7 peticiones por letra** (`log2(95) ≈ 7`).

### Paso 3: Reconstrucción del dato
Al repetir el ciclo para las posiciones `1, 2, 3, ..., N`, cada retraso confirmado revela una letra exacta hasta reconstruir el nombre completo de la base de datos (y, por el mismo principio, nombres de tablas o columnas en una auditoría autorizada).

---

## 3. Impacto y prevención

El gran peligro de las inyecciones ciegas es la falsa creencia de que *"ocultar los errores SQL resuelve el SQL Injection"*. Suprimir mensajes de error es una buena práctica complementaria, pero **no elimina la vulnerabilidad subyacente**. La mitigación definitiva consiste siempre en separar datos de instrucciones mediante **Consultas Parametrizadas (*Prepared Statements*)**, reforzar el principio de mínimo privilegio en el usuario de base de datos y monitorear consultas anormalmente lentas o repetitivas en el WAF y logs de BD.
