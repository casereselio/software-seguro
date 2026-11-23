# Objetivo 2: Anatomía del DOM XSS

El **DOM-Based XSS (*Cross-Site Scripting basado en el DOM*, CWE-79)** ocurre íntegramente del **lado del cliente**, cuando el código JavaScript de una página web lee datos controlados por el usuario desde una fuente insegura (*Source*) y los escribe en el Documento (DOM) a través de una función peligrosa (*Sink*) sin la debida sanitización ni codificación.

---

## 1. ¿Por qué es invisible para muchos WAF e IPS?

La diferencia fundamental respecto al **XSS Reflejado** y el **XSS Almacenado** radica en dónde se procesa el payload:

* En el **XSS Reflejado** y **Almacenado**, la entrada maliciosa **viaja dentro de la petición HTTP hacia el servidor**, donde un WAF (Web Application Firewall) o un IPS (Intrusion Prevention System) inspecciona parámetros `GET`, cuerpos `POST` o cabeceras y puede bloquear patrones sospechosos antes de que lleguen a la aplicación.
* En el **DOM-Based XSS**, el vector suele transmitirse dentro del **fragmento de la URL** (todo lo que sigue al símbolo `#`, conocido como *hash* o *fragment identifier*), por ejemplo:

```text
https://ejemplo.com/perfil#<payload_de_prueba>
```

### El papel clave del fragmento `#` en el protocolo HTTP
Por especificación estándar (RFC 3986), **los navegadores web nunca envían el fragmento `#...` en las peticiones HTTP hacia el servidor**.

1. Cuando la víctima abre el enlace `https://ejemplo.com/perfil#<datos>`, el navegador envía al servidor **únicamente**:
   ```http
   GET /perfil HTTP/2
   Host: ejemplo.com
   ```
2. El WAF, el IPS y los logs del servidor web ven una petición completamente limpia e inocente (`GET /perfil`), por lo que **no pueden inspeccionar ni bloquear** lo que viene después del `#`.
3. El servidor devuelve el HTML y JavaScript legítimo y estático de la página.
4. Recién en el navegador de la víctima, el JavaScript del frontend accede localmente al fragmento y lo inyecta en el DOM, disparando la ejecución.

---

## 2. Anatomía técnica: Sources y Sinks

Todo DOM XSS se compone de dos piezas en el código JavaScript cliente:

* **Source (Fuente de entrada no confiable):** Propiedades que el atacante puede controlar, como `location.hash`, `location.search`, `document.URL`, `document.referrer` o mensajes recibidos por `window.postMessage`.
* **Sink (Sumidero peligroso):** APIs del navegador que interpretan cadenas de texto como HTML o código ejecutable, como `element.innerHTML`, `document.write()`, `outerHTML`, `eval()`, `setTimeout(string)` o `location.href`.

### Ejemplo de código vulnerable en el navegador
```javascript
// Source: el navegador lee el fragmento tras '#' (nunca pasó por el servidor)
const seccion = decodeURIComponent(window.location.hash.substring(1));

// Sink inseguro: interpreta el texto como HTML en el DOM
document.getElementById("bienvenida").innerHTML = "Sección actual: " + seccion;
```

Si la cadena contenida en `location.hash` incluye marcado HTML con eventos, `innerHTML` lo analiza como nodos reales del DOM y ejecuta JavaScript en el origen de la aplicación vulnerable.

---

## 3. Cómo prevenir el DOM XSS

* **Usar sinks seguros de solo texto:** Reemplazar `innerHTML` o `document.write()` por **`textContent`** o **`innerText`**, que insertan el contenido estrictamente como texto literal y jamás lo interpretan como HTML:
  ```javascript
  document.getElementById("bienvenida").textContent = "Sección actual: " + seccion;
  ```
* **Sanitizar cuando sea obligatorio renderizar HTML:** Pasar cualquier cadena dinámica por **DOMPurify** antes de inyectarla en el DOM.
* **Validar URLs y esquemas:** Si se escribe en `location.href` o `a.href`, verificar estrictamente que el protocolo sea `https:` o `http:` (rechazando `javascript:` o `data:`).
* **Adoptar Trusted Types y CSP estricta:** Utilizar la API de **Trusted Types** junto con `Content-Security-Policy` para impedir a nivel de navegador el uso accidental de sinks peligrosos con cadenas sin validar.
