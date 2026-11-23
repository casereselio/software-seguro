# Objetivo 3: La Única Mitigación Definitiva (Prepared Statements)

Intentar frenar inyecciones SQL (**CWE-89**) mediante **listas negras** (bloquear palabras como `SELECT`, `UNION`, `OR` o `--`) o reemplazando comillas simples manualmente es un enfoque frágil e insuficiente: existen múltiples variantes sintácticas, codificaciones alternativas, comentarios en línea y contextos numéricos (sin comillas) que permiten evadir esos filtros con facilidad, además de romper entradas legítimas (como apellidos con apóstrofe, ej. `O'Connor`).

La mitigación arquitectónica definitiva consiste en el uso de **Consultas Parametrizadas (*Prepared Statements*)**.

---

## 1. ¿Cómo funciona un Prepared Statement a nivel de arquitectura?

En una consulta vulnerable construida por concatenación de cadenas, el código de la aplicación arma un único *string* mezclando instrucciones SQL con la entrada del usuario y lo envía al motor de base de datos. El motor no tiene forma de saber qué parte escribió el programador y qué parte provino del usuario: todo se interpreta como código ejecutable.

Con un **Prepared Statement**, la comunicación entre el driver de la aplicación y el motor de base de datos se divide estrictamente en **dos fases separadas** a nivel de protocolo:

### Fase 1: Preparación y compilación de la plantilla (*Prepare*)
La aplicación envía al motor de base de datos **únicamente la estructura fija de la consulta**, utilizando marcadores de posición (*placeholders* como `?`, `$1` o `:usuario`):

```sql
SELECT id, email, rol FROM usuarios WHERE username = ? AND activo = 1;
```

El motor de base de datos realiza en este momento:
1. **Análisis léxico y sintáctico (*Parsing*):** construye el **Árbol de Sintaxis Abstracta (AST)** de la consulta.
2. **Validación semántica y plan de ejecución (*Query Plan*):** optimiza y fija de forma inmutable qué tablas, columnas, operadores y lógica ejecutará la consulta.

En esta etapa, **los datos del usuario todavía no existen en la consulta**, por lo que la estructura lógica queda sellada.

### Fase 2: Vinculación de parámetros y ejecución (*Bind & Execute*)
En un paso posterior (en paquetes/campos separados del protocolo binario de la BD), la aplicación envía los valores provistos por el usuario para rellenar cada *placeholder*:

```python
# Ejemplo seguro en Python (psycopg / sqlite3 / mysql-connector)
cursor.execute(
    "SELECT id, email, rol FROM usuarios WHERE username = %s AND activo = 1",
    (user_input,)
)
```

El motor toma el valor enviado y lo inserta **directamente en el nodo de dato literal** del plan de ejecución ya compilado, **sin volver a pasar nunca por el parser SQL**.

---

## 2. ¿Por qué neutraliza por completo cualquier intento de inyección?

Porque establece una **separación estricta e inquebrantable entre el plano de control (código SQL) y el plano de datos (parámetros)**.

Si un usuario envía una cadena con sintaxis SQL maliciosa (por ejemplo, comillas simples, operadores lógicos o comentarios SQL), el motor de base de datos no la interpreta como una instrucción que modifique la cláusula `WHERE`: simplemente busca en la columna `username` un usuario cuyo nombre literal contenga exactamente esos caracteres. Como la estructura AST ya estaba definida y compilada en la Fase 1, ningún carácter especial puede alterar el significado de la consulta.

---

## 3. Consideraciones importantes de implementación

* **Usar parametrización real (nativa del driver):** Asegurarse de que el driver no esté usando "emulación" insegura mal configurada y nunca concatenar variables dentro del string SQL ni siquiera usando un ORM (por ejemplo, evitar `raw()` con interpolación de strings).
* **Limitación conocida (identificadores dinámicos):** Los *Prepared Statements* parametrizan **valores**, no nombres de tablas, columnas ni direcciones de ordenamiento (`ORDER BY`). En esos casos puntuales, se debe validar la entrada contra una **lista blanca estricta (*Allow-list*)** definida en el código del servidor:

```python
COLUMNAS_PERMITIDAS = {"fecha": "created_at", "precio": "price"}
columna = COLUMNAS_PERMITIDAS.get(orden_usuario, "created_at")
```

* **Defensa en profundidad:** Complementar siempre las consultas parametrizadas con validación de tipos/esquemas, principio de mínimo privilegio en el usuario de base de datos y manejo seguro de errores.
