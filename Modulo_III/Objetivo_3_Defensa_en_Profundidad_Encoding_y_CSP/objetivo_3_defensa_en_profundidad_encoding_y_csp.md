# Objetivo 3: Defensa en Profundidad (Encoding y CSP)

Mitigar definitivamente el riesgo de **Cross-Site Scripting (XSS, CWE-79)** en una aplicación moderna requiere aplicar el principio de **Defensa en Profundidad (*Defense in Depth*)**: combinar una **primera capa preventiva en el código** (*Output Encoding*) con una **segunda capa restrictiva en el navegador** (*Content Security Policy*), de modo que si una falla, la otra contenga el impacto.

---

## 1. Primera capa: Output Encoding (Codificación de Salida) a nivel de código

El **Output Encoding** consiste en transformar los caracteres con significado sintáctico especial en su representación literal segura **justo antes de renderizar datos no confiables** en la interfaz del usuario. De esta forma, el navegador siempre interpreta los datos como **contenido visible** y nunca como **código ejecutable**.

### Regla fundamental: debe ser dependiente del contexto (*Context-Aware*)
No alcanza con aplicar la misma codificación en todas partes; la codificación debe corresponder al lugar exacto del documento donde se inserta el dato:

| Contexto de salida | Caracteres críticos | Estrategia segura |
| :--- | :--- | :--- |
| **Cuerpo HTML** (`<div>DATO</div>`) | `<`, `>`, `&`, `"`, `'` | Escapar a entidades HTML (`&lt;`, `&gt;`, `&amp;`, `&quot;`, `&#x27;`). |
| **Atributo HTML** (`<input value="DATO">`) | Comillas, espacios y caracteres no alfanuméricos | Usar siempre comillas alrededor del atributo y codificar para contexto de atributo HTML. |
| **Contexto JavaScript** | Comillas, `\`, `\n`, `</script>` | Serializar de forma segura (ej. `JSON.stringify` escapando `<`, `>`, `&`) o pasar datos mediante atributos `data-*` leídos por JS. |
| **Contexto URL** (`href="DATO"`) | Esquemas `javascript:`, `data:` | Aplicar `encodeURIComponent` en parámetros y validar con lista blanca que el protocolo comience con `https://` o `http://`. |

Los frameworks modernos (React, Angular, Vue, Jinja2, Razor, Thymeleaf) aplican *auto-escaping* por defecto en el contexto HTML siempre que el desarrollador **evite** funciones que desactivan esta protección (como `dangerouslySetInnerHTML`, `v-html`, `|safe` o `innerHTML`).

---

## 2. Segunda capa: Content Security Policy (CSP) a nivel de cabeceras HTTP

La **Content Security Policy (CSP)** es una cabecera de respuesta HTTP mediante la cual el servidor le declara al navegador **qué orígenes de scripts, estilos, imágenes y conexiones están explícitamente autorizados** a ejecutarse en la página.

### Cómo actúa como red de seguridad ante un XSS
Incluso si un desarrollador olvidara aplicar *Output Encoding* en un campo y un atacante lograra inyectar HTML o JavaScript en la página, una política CSP estricta bloquea su ejecución en el navegador:

1. **Bloqueo de scripts *inline* y manejadores de eventos:** Al no incluir `'unsafe-inline'`, el navegador rechaza automáticamente etiquetas `<script>...</script>` inyectadas y atributos como `onerror="..."` o `onload="..."`.
2. **Bloqueo de `eval()` y ejecución dinámica de strings:** Al omitir `'unsafe-eval'`, se neutralizan funciones como `eval()` y `new Function()`.
3. **Restricción de origen de scripts y exfiltración:** Mediante directivas como `script-src` y `connect-src`, impide cargar scripts desde servidores externos del atacante y limita el envío de datos a dominios no autorizados.
4. **Control mediante *Nonces* criptográficos o *Hashes*:** Permite ejecutar únicamente los scripts legítimos del sitio que porten un `nonce` aleatorio único generado por el servidor en cada respuesta HTTP.

### Ejemplo de cabecera CSP robusta
```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-r4nd0mUn1qV4lu3' 'strict-dynamic';
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
```

* `default-src 'self'`: por defecto, solo permite recursos del mismo origen.
* `script-src 'nonce-...' 'strict-dynamic'`: solo ejecuta scripts autorizados con el *nonce* de esa petición (sin `'unsafe-inline'` ni `'unsafe-eval'`).
* `object-src 'none'`: bloquea plugins heredados (`<object>`, `<embed>`).
* `base-uri 'self'` y `frame-ancestors 'none'`: previene secuestro de URLs relativas y ataques de *Clickjacking*.

---

## 3. Sinergia de ambas capas

* **Output Encoding** elimina la **causa raíz** del XSS evitando que los datos del usuario se interpreten como código.
* **CSP** actúa como **barrera de contención**, evitando que un error humano en el código derive en ejecución arbitraria de JavaScript y proporcionando telemetría temprana mediante reportes de violación (`report-to` / `Content-Security-Policy-Report-Only`).

Implementadas en conjunto, ambas capas reducen el riesgo de explotación de XSS a un nivel residual mínimo.
