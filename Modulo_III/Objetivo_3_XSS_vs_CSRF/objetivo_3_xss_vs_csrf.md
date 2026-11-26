# Objetivo 3: XSS vs. CSRF

Tanto **Cross-Site Scripting (XSS)** como **Cross-Site Request Forgery (CSRF)** son vulnerabilidades que involucran al navegador de una víctima autenticada para comprometer su cuenta. Sin embargo, sus **mecánicas, origen de la confianza explotada y capacidades reales** son esencialmente opuestas.

---

## 1. Diferencia fundamental en una frase

* **XSS (CWE-79) explota la confianza que el *usuario/navegador* deposita en un sitio web legítimo:** el atacante logra inyectar y ejecutar JavaScript arbitrario **dentro del propio origen de la aplicación vulnerable**.
* **CSRF (CWE-352) explota la confianza que el *sitio web* deposita en el navegador del usuario:** el atacante **no** ejecuta código dentro del sitio vulnerable, sino que desde un **sitio externo** fuerza al navegador de la víctima a enviar una petición HTTP autenticada con sus cookies hacia la aplicación vulnerable.

---

## 2. Comparación técnica directa

| Dimensión | Cross-Site Scripting (XSS) | Cross-Site Request Forgery (CSRF) |
| :--- | :--- | :--- |
| **Confianza abusada** | Confianza del **navegador** en el código que proviene del sitio legítimo. | Confianza del **servidor** en las cookies enviadas automáticamente por el navegador del usuario. |
| **Dónde se ejecuta el código** | **Dentro del origen** de la aplicación vulnerable (burla la *Same-Origin Policy* porque corre en el mismo dominio). | **En un dominio externo** controlado por el atacante (sujeto a las restricciones de la *Same-Origin Policy*). |
| **Dirección de la comunicación** | **Bidireccional (Lectura + Escritura):** el script puede enviar peticiones *y leer* el DOM, las respuestas HTTP, tokens y datos privados de la pantalla. | **Unidireccional (Ciego / Solo Escritura):** el atacante dispara una acción (ej. cambiar email o transferir), pero **no puede leer** la respuesta del servidor ni el contenido de la cuenta. |
| **Alcance del impacto** | Muy alto: robo de sesiones/tokens, lectura de datos sensibles, modificación de la interfaz (*phishing* interno), keylogging y **evasión total de protecciones anti-CSRF**. | Limitado a ejecutar acciones específicas de cambio de estado cuyos parámetros el atacante pueda predecir o forzar. |
| **Requiere que el sitio ejecute código inyectado** | **Sí.** Debe existir un punto de inyección HTML/JS sin codificación adecuada. | **No.** El sitio vulnerable puede estar completamente libre de XSS y aun así ser vulnerable a CSRF. |
| **Mitigación principal** | *Context-Aware Output Encoding*, sanitización por lista blanca (ej. DOMPurify) y **Content Security Policy (CSP)**. | **Tokens Anti-CSRF** aleatorios por sesión/petición, atributo de cookie **`SameSite=Lax/Strict`** y validación de cabecera `Origin`. |

---

## 3. Relación crítica entre ambos ataques

Un principio clave en arquitectura de seguridad web es que **un XSS rompe las defensas contra CSRF, pero no al revés**:

* Si una aplicación tiene una excelente protección contra **CSRF** (tokens anti-CSRF aleatorios y `SameSite=Strict`), un ataque CSRF externo será bloqueado.
* Sin embargo, si esa misma aplicación posee una vulnerabilidad de **XSS**, el JavaScript inyectado se ejecuta dentro del mismo origen legítimo: puede **leer directamente el token anti-CSRF desde el DOM** y enviar la petición maliciosa con el token válido y las cookies `SameSite`, anulando por completo la defensa anti-CSRF.

Por esta razón, ambas vulnerabilidades deben mitigarse de manera independiente pero coordinada dentro de una estrategia de defensa en profundidad.
