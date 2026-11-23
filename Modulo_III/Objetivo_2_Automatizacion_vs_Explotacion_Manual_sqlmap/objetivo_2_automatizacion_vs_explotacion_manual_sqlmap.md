# Objetivo 2: Automatización vs. Explotación Manual (sqlmap)

**sqlmap** es la herramienta automatizada de referencia para detectar y validar inyecciones SQL. Sin embargo, depender exclusivamente de ella en una auditoría profesional conduce a falsos negativos y detecciones bloqueadas: un pentester experimentado comprende el protocolo y la lógica de la aplicación para resolver manualmente escenarios donde la automatización falla.

---

## Escenarios reales donde `sqlmap` falla y el análisis manual triunfa

### 1. Flujos con tokens dinámicos anti-CSRF, firmas HMAC o cifrado/codificación personalizada
* **Por qué falla `sqlmap`:** Muchas aplicaciones modernas exigen en cada petición un token anti-CSRF de un solo uso obtenido en un paso previo, una firma HMAC calculada sobre el cuerpo JSON con *timestamp/nonce*, o un parámetro codificado en un formato propietario (por ejemplo, JSON anidado dentro de Base64 con un *checksum*). Si `sqlmap` modifica un parámetro directamente sin recalcular la firma o renovar el flujo multipaso, el servidor rechaza todas las pruebas con `400 Bad Request` o `403 Forbidden` antes de llegar siquiera a la consulta SQL.
* **Cómo lo resuelve un pentester manual:** Analizando el flujo completo en **Burp Suite** y DevTools, entendiendo cómo se generan el token o la firma, probando payloads puntuales y quirúrgicos en *Repeater* (o escribiendo un pequeño script/extensión a medida que renueve el token y firme cada prueba).

### 2. Inyecciones SQL de Segundo Orden (*Second-Order SQL Injection*)
* **Por qué falla `sqlmap`:** En una inyección de segundo orden, el payload se envía y almacena de forma segura en un endpoint A (por ejemplo, al registrar un nombre de usuario o actualizar el perfil en `POST /api/profile`), pero recién se concatena de forma insegura en una consulta SQL cuando se ejecuta una acción distinta en un endpoint B (por ejemplo, `GET /api/reports/export` o al iniciar un cambio de contraseña). Como `sqlmap` evalúa el impacto en la misma petición donde inyectó el dato, concluye erróneamente que el parámetro "no es vulnerable".
* **Cómo lo resuelve un pentester manual:** Comprendiendo la arquitectura funcional de la aplicación, rastreando dónde se guarda un dato y en qué otras vistas o procesos en segundo plano se reutiliza ese valor almacenado, verificando la respuesta en el segundo endpoint.

### 3. Restricciones estrictas de WAF, Rate Limiting y bloqueos de cuenta
* **Por qué falla `sqlmap`:** Por defecto, `sqlmap` envía cientos o miles de peticiones con patrones y *User-Agents* ampliamente conocidos por los WAF (Cloudflare, AWS WAF, ModSecurity). Esto dispara rápidamente bloqueos por IP (`403` / `429`), CAPTCHAs o invalidación inmediata de la sesión de pruebas, generando ruido masivo en los logs defensivos.
* **Cómo lo resuelve un pentester manual:** Realizando pruebas de mínimo impacto (pocas peticiones cuidadosamente diseñadas en Burp Repeater), observando con precisión qué regla o palabra bloquea el filtro, respetando los tiempos de respuesta del objetivo y adaptando la sintaxis SQL válida para el motor específico sin provocar bloqueos ni saturar entornos de producción.

### 4. Sintaxis SQL no estándar o contextos complejos (JSONB, `ORDER BY`, cláusulas dinámicas)
* **Por qué falla `sqlmap`:** Cuando el parámetro del usuario se inserta dentro de estructuras complejas —como operadores JSON en PostgreSQL (`->>`), identificadores de columnas en `ORDER BY` / `GROUP BY`, procedimientos almacenados o consultas multi-sentencia con lógica de validación previa (ej. límite estricto de longitud de caracteres)— las plantillas genéricas de `sqlmap` suelen generar consultas sintácticamente inválidas y abandonar el vector.
* **Cómo lo resuelve un pentester manual:** Deduciendo la estructura exacta de la consulta subyacente mediante pruebas controladas paso a paso y construyendo una expresión adaptada exactamente a las restricciones del contexto y del motor de base de datos.

---

## Conclusión

Las herramientas automatizadas como `sqlmap` son útiles para ahorrar tiempo en vectores sencillos y repetitivos cuando se cuenta con autorización explícita, pero **nunca reemplazan el criterio técnico**. Entender SQL, HTTP y el flujo de negocio es lo que permite descubrir las vulnerabilidades complejas que realmente marcan la diferencia en una auditoría profesional.
