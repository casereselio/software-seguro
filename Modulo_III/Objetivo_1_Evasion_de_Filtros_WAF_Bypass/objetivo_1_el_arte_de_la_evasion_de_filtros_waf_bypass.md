# Objetivo 1: El Arte de la Evasión de Filtros (WAF Bypass)

Intentar prevenir vulnerabilidades de **Cross-Site Scripting (XSS, CWE-79)** mediante un filtro de **lista negra (*blacklist*)** que elimina únicamente las etiquetas `<script>` y `</script>` es una defensa ineficaz y fácilmente evadible.

---

## 1. ¿Por qué esta defensa es técnicamente inútil?

El estándar HTML5 y los motores modernos de renderizado de navegadores ofrecen docenas de formas legítimas de ejecutar JavaScript que **no requieren la etiqueta `<script>`**. Un filtro ingenuo que busca y borra `<script>` falla por múltiples razones:

1. **Manejadores de eventos HTML (*Event Handlers*):** Casi cualquier etiqueta HTML (`<img>`, `<svg>`, `<body>`, `<details>`, `<iframe>`, `<a>`) acepta atributos de eventos como `onerror`, `onload`, `onmouseover`, `ontoggle` o `onfocus` que evalúan código JavaScript directamente.
2. **Filtros no recursivos (*Nested Tags*):** Si el filtro realiza una sola pasada eliminando `<script>`, una cadena anidada como `<scr<script>ipt>` se recompone a `<script>` después de borrar la coincidencia interna.
3. **Sensibilidad a mayúsculas/minúsculas y variaciones sintácticas:** Si la expresión regular es sensible a mayúsculas o rígida con los espacios, variantes como `<ScRiPt>` o `<script\t>` evaden la coincidencia exacta.
4. **Esquemas de URL ejecutables:** Atributos como `href` o `src` admiten el esquema `javascript:` en etiquetas como `<a>` o `<iframe>`, ejecutando código sin necesidad de `<script>`.

---

## 2. Ejemplos de payloads conceptuales en HTML/JS sin `<script>`

En una auditoría de seguridad autorizada (por ejemplo, para demostrar el riesgo con una prueba inofensiva como `alert()` o `console.log()`), se pueden emplear etiquetas alternativas con manejadores de eventos:

### Ejemplo 1: Etiqueta `<img>` con evento `onerror`
```html
<img src="x" onerror="alert('XSS')">
```
* **Cómo funciona:** El navegador intenta cargar la imagen desde la ruta inexistente `"x"`. Al fallar la carga inmediatamente, se dispara automáticamente el evento `onerror` y el navegador ejecuta el código JavaScript contenido en el atributo, sin ninguna interacción del usuario y sin usar `<script>`.

### Ejemplo 2: Etiqueta `<svg>` con evento `onload`
```html
<svg onload="alert('XSS')">
```
* **Cómo funciona:** El navegador analiza el elemento vectorial `<svg>` como parte del DOM; al finalizar su carga, ejecuta inmediatamente el manejador `onload`.

### Ejemplo 3: Etiqueta `<details>` con atributo `open` y evento `ontoggle`
```html
<details open ontoggle="alert('XSS')">
```
* **Cómo funciona:** El atributo `open` fuerza la apertura automática del elemento desplegable al renderizarse la página, disparando de forma inmediata el evento `ontoggle`.

---

## 3. Cómo mitigarlo correctamente

En lugar de listas negras frágiles, la prevención robusta de XSS exige:
* **Codificación de salida dependiente del contexto (*Context-Aware Output Encoding*):** Transformar caracteres especiales (`<`, `>`, `"`, `'`, `&`) en entidades HTML seguras antes de renderizar cualquier dato del usuario.
* **Sanitización basada en lista blanca (*Allow-list*):** Si la aplicación realmente necesita admitir HTML enriquecido en comentarios, utilizar librerías auditadas como **DOMPurify**, que permiten únicamente un conjunto mínimo de etiquetas y atributos seguros.
* **Content Security Policy (CSP):** Implementar cabeceras CSP estrictas que bloqueen la ejecución de JavaScript *inline* y manejadores de eventos.
