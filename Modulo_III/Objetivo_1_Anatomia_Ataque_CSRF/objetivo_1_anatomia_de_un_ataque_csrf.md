# Objetivo 1: Anatomía de un Ataque CSRF (Cross-Site Request Forgery)

El **Cross-Site Request Forgery (CSRF / Falsificación de Petición en Sitios Cruzados, CWE-352)** es una vulnerabilidad que permite a un sitio externo malicioso inducir al navegador de un usuario autenticado a ejecutar **acciones no deseadas** dentro de una aplicación web en la que la víctima ya tiene una sesión activa (por ejemplo, cambiar su correo electrónico, modificar su contraseña o iniciar una transferencia).

---

## 1. ¿Por qué ocurre este fallo de seguridad?

La causa raíz combina dos factores:

1. **Comportamiento automático de los navegadores con las cookies:** Por diseño del protocolo HTTP, cada vez que un navegador realiza una petición hacia un dominio (ej. `banco.ejemplo.com`), **adjunta automáticamente todas las cookies vigentes de ese dominio** —incluida la cookie de sesión (`sessionid`)—, sin importar desde qué sitio web se originó la solicitud.
2. **Ausencia de verificación de intención/origen en el servidor:** Si el backend confía *únicamente* en la presencia de la cookie de sesión para autorizar acciones sensibles (sin validar un token impredecible ni el origen de la petición), **no puede distinguir** si la solicitud la hizo conscientemente el usuario desde el formulario legítimo o si fue disparada en segundo plano desde una página externa.

---

## 2. Flujo exacto de un ataque CSRF exitoso

Supongamos que una aplicación vulnerable permite cambiar el email del perfil mediante `POST https://app.ejemplo.com/cuenta/cambiar-email` usando solo la cookie de sesión:

1. **Sesión activa:** La víctima inicia sesión normalmente en `https://app.ejemplo.com`. Su navegador almacena la cookie de sesión (`Set-Cookie: session=abc123...`).
2. **Visita a un sitio controlado por un tercero:** Sin cerrar sesión, la víctima abre en otra pestaña una página externa (`https://promocion-externa.test`) que contiene un formulario HTML oculto apuntando al endpoint vulnerable de `app.ejemplo.com`:
   ```html
   <form action="https://app.ejemplo.com/cuenta/cambiar-email" method="POST">
     <input type="hidden" name="nuevo_email" value="tercero@externo.test" />
   </form>
   <script>document.forms[0].submit();</script>
   ```
3. **Envío automático de credenciales:** Al enviarse el formulario hacia `app.ejemplo.com`, el navegador **adjunta automáticamente la cookie `session=abc123...`** perteneciente a `app.ejemplo.com`.
4. **Ejecución legítima en el servidor:** El backend recibe una petición HTTP válida con la cookie de sesión auténtica de la víctima, valida que el usuario está logueado y **ejecuta el cambio de correo electrónico** sin que la víctima lo haya autorizado.

---

## 3. Cómo mitiga este vector el atributo de cookie `SameSite`

El atributo **`SameSite`** en la directiva `Set-Cookie` le indica al navegador cuándo está permitido enviar una cookie en peticiones originadas desde sitios externos (*cross-site*), atacando directamente el mecanismo del CSRF:

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax
```

### Valores de `SameSite`

| Valor | Comportamiento del navegador | Nivel de protección frente a CSRF |
| :--- | :--- | :--- |
| **`Strict`** | El navegador **nunca** envía la cookie en ninguna petición *cross-site*, ni siquiera al hacer clic en un enlace externo hacia el sitio. | **Máxima protección.** Bloquea por completo ataques CSRF basados en cookies (tanto `POST` como `GET`), a costa de que si un usuario entra desde un enlace en su email deberá volver a autenticarse o recargar. |
| **`Lax`** *(estándar moderno por defecto)* | Bloquea el envío de la cookie en subpeticiones y envíos de formularios `POST` *cross-site*, pero **sí la envía** en navegaciones de nivel superior seguras (método `GET` al hacer clic en un enlace normal). | **Alta protección práctica.** Bloquea los ataques CSRF clásicos por formularios `POST` manteniendo buena experiencia de usuario, siempre que la aplicación **nunca cambie estado mediante peticiones `GET`**. |
| **`None`** | Permite enviar la cookie en cualquier contexto *cross-site* (requiere obligatoriamente el flag `Secure` sobre HTTPS). | **Sin protección contra CSRF** por sí solo; exige controles adicionales obligatorios. |

---

## 4. Defensa en profundidad complementaria

Aunque `SameSite=Lax` o `Strict` mitiga radicalmente la gran mayoría de los escenarios CSRF, la recomendación OWASP es combinarlo con:
* **Tokens Anti-CSRF (*Synchronizer Token Pattern*):** Un valor criptográficamente aleatorio vinculado a la sesión que se envía en un campo oculto o cabecera personalizada (`X-CSRF-Token`) y que un sitio externo no puede leer debido a la *Same-Origin Policy (SOP)*.
* **Validación de cabeceras `Origin` y `Referer`** en peticiones que modifican estado.
* **Re-autenticación o confirmación** para acciones críticas (cambio de contraseña, transferencias, baja de cuenta).
