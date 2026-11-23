# Objetivo 5: El Espejismo del Cifrado en el Cliente (Frontend Encryption)

Implementar cifrado de parámetros en JavaScript dentro del navegador para "impedir" que un atacante manipule peticiones HTTP es un antipatrón arquitectónico conocido como **seguridad por oscuridad (CWE-602: Client-Side Enforcement of Server-Side Security)**. Aunque a primera vista el tráfico capturado en un proxy parezca ilegible, esta medida ofrece una **falsa sensación de seguridad**.

---

## 1. ¿Dónde reside el fallo fundamental de diseño?

El error de arquitectura parte de violar un principio básico de la seguridad web: **el cliente (navegador) es un entorno completamente no confiable bajo control absoluto del usuario**.

* **TLS/HTTPS ya protege el canal en tránsito:** El cifrado contra terceros en la red (ataques Man-in-the-Middle externos) ya lo resuelve correctamente TLS/HTTPS. Cifrar el cuerpo en JavaScript no añade una nueva capa de confianza frente al propio usuario que opera el navegador.
* **El navegador necesita el código y la clave para cifrar:** Para que el frontend pueda encriptar los datos antes de enviar el `POST`, el servidor inevitablemente debe entregarle al navegador el **algoritmo de cifrado** (en el código JavaScript) y la **clave o mecanismo de derivación de clave** (hardcodeada en el bundle, descargada por un endpoint o derivada de la sesión).
* **Quien controla el entorno de ejecución controla la lógica:** Dado que el atacante es dueño de la máquina y del navegador donde se ejecuta ese JavaScript, tiene visibilidad total sobre los datos **antes** de que se cifren y acceso directo a la función que los cifra.

---

## 2. Cómo un auditor/atacante evade esta protección (Análisis táctico)

Incluso si el JavaScript está minificado u ofuscado, la evasión sigue un flujo metódico y directo:

1. **Inspección y depuración del código cliente (DevTools):**
   * Usando las herramientas de desarrollador del navegador (*Sources / Debugger*), se analiza el archivo `.js` encargado de interceptar el envío del formulario o la llamada `fetch` / `XMLHttpRequest`.
   * Se busca la función de cifrado (por ejemplo, referencias a `CryptoJS`, `Web Crypto API` / `crypto.subtle.encrypt`, `AES`, `RSA` o `JSEncrypt`).
2. **Intercepción previa al cifrado (*Breakpoints* en memoria):**
   * Se coloca un *breakpoint* justo en la línea anterior a la invocación de la función de cifrado (donde el objeto JSON todavía está en **texto plano**).
   * Desde la consola de DevTools en tiempo de ejecución, se leen las variables en claro, se modifican los valores deseados (por ejemplo, monto, rol o destinatario) y se deja continuar la ejecución: **el propio código legítimo de la aplicación cifra el payload manipulado con el formato y clave válidos y lo envía al backend**.
3. **Extracción del algoritmo y clave para automatización en Proxy:**
   * Si se desea manipular peticiones repetidamente o automatizar pruebas desde **Burp Suite**, se extrae el algoritmo, modo de operación (ej. AES-CBC/GCM), IV y clave desde el JavaScript o mediante *hooking* en consola.
   * También es habitual replicar la función en un script externo (Python/Node.js) o usar una extensión de proxy (como un script de procesamiento de payloads) que descifre/recifre automáticamente el cuerpo HTTP al vuelo.

---

## 3. ¿Cuál es la arquitectura correcta?

En lugar de gastar esfuerzo en ofuscar o cifrar en el cliente, la seguridad real debe construirse en el servidor y en la capa de transporte:

* **Confiar en HTTPS (TLS 1.2 / 1.3)** con HSTS para proteger la confidencialidad e integridad de los datos en tránsito.
* **Validación y autorización estricta en el Backend:** Asumir que *toda* petición proveniente del cliente puede haber sido manipulada. Nunca confiar en precios, roles, permisos, saldos o identificadores de propietario enviados desde el navegador; siempre recalcularlos y verificarlos contra la sesión autenticada y la base de datos del servidor.
* **Controles anti-manipulación reales cuando apliquen:** Usar sesiones firmadas o tokens JWT verificados únicamente con claves privadas que **jamás abandonan el servidor**, junto con validaciones de esquema, control de acceso a nivel de objeto (BOLA/IDOR) y protección CSRF.
