# Objetivo 4: Manejo Inseguro de Errores (Stack Traces)

El manejo de excepciones es un pilar clave de la seguridad aplicativa. Cuando ocurre un error inesperado en producción, la diferencia entre mostrar un mensaje controlado o volcar el estado interno de la aplicación marca la frontera entre un fallo contenido y una fuga crítica de información (**CWE-209: Generation of Error Message Containing Sensitive Information**).

---

## 1. Mensaje de Error Genérico vs. Stack Trace en Producción

| Característica | Mensaje de Error Genérico (Seguro) | Stack Trace Expuesto (Inseguro) |
| :--- | :--- | :--- |
| **Qué ve el usuario** | Un aviso neutro y breve (ej. *"Ocurrió un error interno. ID de referencia: `ERR-94F2A1`"*). | La pila completa de llamadas, archivo exacto, número de línea, tipo de excepción y, a menudo, fragmentos de código o consultas SQL. |
| **Código HTTP** | Código semántico correcto (`500 Internal Server Error` o `400 Bad Request`) sin detalles internos. | `500` (o incluso `200`) acompañado de una página de depuración (*debug page*) completa. |
| **Dónde queda el detalle técnico** | Exclusivamente en los logs internos del servidor / SIEM, asociado al ID de referencia. | Visible públicamente en el navegador o en el cuerpo JSON/HTML de la respuesta HTTP. |
| **Público objetivo** | Usuario final en entorno de **producción**. | Desarrollador en entorno **local/desarrollo** únicamente. |

### Ejemplo comparativo

**Respuesta segura (Producción):**
```json
{
  "error": "No fue posible procesar la solicitud. Contacte a soporte indicando el código ERR-8F31C."
}
```

**Stack Trace expuesto (Inseguro):**
```text
psycopg2.errors.SyntaxError: syntax error at or near "'"
LINE 1: SELECT id, email FROM users WHERE username = 'admin'' AND active = true
  File "/opt/app/backend/services/auth_service.py", line 84, in authenticate_user
    cursor.execute(query)
  File "/usr/local/lib/python3.11/site-packages/django/db/backends/utils.py", line 67, in execute
Django 4.2.1 | Python 3.11.4 | PostgreSQL 15.2 | DEBUG=True
```

---

## 2. ¿Por qué un Stack Trace facilita enormemente la fase de explotación?

Revelar una traza de pila detallada convierte una prueba de **caja negra** (donde el atacante opera a ciegas) en un escenario casi de **caja blanca**, acelerando y refinando los ataques de varias formas:

1. **Enumeración exacta de tecnología y versiones:** Expone el lenguaje, motor de base de datos, servidor web, framework y librerías de terceros con sus versiones precisas (ej. `Django 4.2.1`, `Spring Boot`, `PostgreSQL 15.2`). Esto permite al atacante consultar bases de **CVEs** y buscar exploits públicos listos para usar.
2. **Revelación de rutas absolutas del sistema de archivos:** Mostrar rutas como `/opt/app/backend/...` o `C:\inetpub\wwwroot\...` delata el sistema operativo, la estructura del proyecto y la ubicación exacta de archivos, facilitando ataques de **Path Traversal / Local File Inclusion (LFI)** o escritura de webshells.
3. **Oráculo directo para inyecciones (SQLi, SSTI, Deserialización):** Al ver errores literales de sintaxis SQL (`syntax error at or near...`), nombres de tablas y columnas, o excepciones de deserialización, el atacante sabe inmediatamente si su payload rompió la lógica esperada y puede ajustarlo con precisión quirúrgica (convirtiendo una posible inyección ciega en una *Error-Based SQLi*).
4. **Exposición de lógica interna y datos sensibles:** Frameworks en modo `DEBUG=True` (como Django, Laravel Ignition o Werkzeug) suelen volcar variables locales, argumentos de funciones, cabeceras internas e incluso **variables de entorno** (API keys, contraseñas de BD o `SECRET_KEY`).
5. **Mapeo de dependencias y controles de seguridad:** Los nombres de clases y middlewares en la traza revelan cómo se validan permisos, qué ORM se utiliza o qué sanitizadores están presentes, permitiendo diseñar evasiones específicas.

---

## 3. Mitigación recomendada

* **Desactivar el modo de depuración en producción:** Garantizar `DEBUG=False`, `display_errors = Off` (PHP), `APP_DEBUG=false` (Laravel) o `NODE_ENV=production` en todos los entornos expuestos.
* **Implementar un manejador global de excepciones (Global Error Handler):** Capturar todas las excepciones no controladas en un único punto para devolver siempre una respuesta genérica con un **Correlation ID**.
* **Registrar los detalles únicamente del lado del servidor:** Enviar el Stack Trace completo a un sistema de logs centralizado (ELK, Datadog, SIEM) con acceso restringido al equipo técnico.
* **Configurar páginas de error personalizadas:** Definir vistas genéricas para códigos `400`, `403`, `404` y `500` tanto en la aplicación como en el proxy inverso / servidor web.
