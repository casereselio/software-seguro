# Objetivo 2: Ingeniería Inversa a JSON Web Tokens (JWT)

En esta práctica se realizó un análisis pasivo del **JSON Web Token (JWT)** emitido durante el inicio de sesión en la plataforma del curso (`app.softwareseguro.com.ar`), almacenado en la cookie `auth`. El objetivo es examinar su estructura técnica, el algoritmo de firma utilizado y el tipo de información que viaja decodificable en texto plano dentro del *Payload*.

---

## 1. Estructura técnica del JWT analizado

Un JWT se compone de tres secciones separadas por puntos (`Header.Payload.Signature`), donde las dos primeras están simplemente **codificadas en Base64URL** (no cifradas) y la tercera corresponde a la firma criptográfica que garantiza su integridad.

### a) Header (Encabezado decodificado)
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

* **`alg: "HS256"` (HMAC-SHA256):** Se utiliza un algoritmo de firma **simétrico**. Esto significa que el servidor emplea **una única clave secreta compartida** tanto para generar la firma al iniciar sesión como para verificarla en cada petición posterior.
* **`typ: "JWT"`:** Indica que el tipo de token es un JSON Web Token estándar (RFC 7519).

### b) Payload (Cuerpo decodificado — datos sensibles redactados)
```json
{
  "id": "[ID_NUMERICO_REDACTADO]",
  "fullname": "Elio Ariel Caseres",
  "email": "[EMAIL_REDACTADO]",
  "role_id": 1,
  "plan_id": 2,
  "pagado": true,
  "fecha_pago": "2026-09-03T21:21:52.887Z",
  "plan_expires_at": null,
  "email_confirmed": true,
  "pending_payment": null,
  "confirmation_email_uuid": "[UUID_REDACTADO]",
  "created_at": "2026-09-20T23:56:27.277Z",
  "is_superadmin": false,
  "iat": 1795386667,
  "exp": 1795408267
}
```

### c) Firma (`Signature Verification Failed` en `jwt.io`)
Al decodificar el token en `jwt.io`, la interfaz muestra por defecto el *placeholder* `a-string-secret-at-least-256-bits-long` y el mensaje **`Signature Verification Failed`**.
* **Qué significa:** Es el comportamiento **esperado y correcto**. Como el cliente no conoce la clave secreta privada del backend, no puede validar ni falsificar la firma `HS256`.
* **Principio fundamental demostrado:** La firma de un JWT (JWS) protege únicamente la **integridad** (impide alterar el contenido sin que el servidor lo detecte), pero **no ofrece confidencialidad**: cualquier persona o script que acceda al token puede decodificar el *Header* y el *Payload* en Base64URL sin necesitar la clave secreta.

---

## 2. Análisis de los datos expuestos en el Payload

El *Payload* contiene una cantidad considerable de atributos de usuario, negocio y autorización en texto plano (Base64URL):

1. **Información Personal Identificable (PII):**
   * `id` (identificador numérico interno del usuario), `fullname`, `email` y `created_at`.
2. **Roles y flags de autorización interna:**
   * `role_id: 1` y `is_superadmin: false`.
   * *Análisis:* Aunque modificar `is_superadmin` a `true` en el cliente invalidaría la firma (siempre que el backend valide correctamente la firma `HS256` y rechace `alg: none`), exponer explícitamente el esquema de roles y privilegios en el cliente revela la estructura interna del modelo de autorización.
3. **Información comercial y de facturación:**
   * `plan_id: 2`, `pagado: true`, `fecha_pago`, `plan_expires_at` y `pending_payment`.
4. **Dato innecesario/sensible en el token (`confirmation_email_uuid`):**
   * Se incluye el UUID de confirmación de correo electrónico en cada sesión autenticada incluso cuando `email_confirmed` ya es `true`. Este dato no cumple ninguna función operativa recurrente dentro de un token de sesión y viola el **principio de minimización de datos**.
5. **Control de ciclo de vida del token (`iat` y `exp` — Buena práctica):**
   * `iat`: `1795386667` (`22/11/2026 22:31:07 UTC`)
   * `exp`: `1795408267` (`23/11/2026 04:31:07 UTC`)
   * **Duración (TTL):** `21.600 segundos` (**6 horas**). Contar con una expiración relativamente corta es una buena práctica para acotar la ventana de validez de un token en caso de compromiso.

---

## 3. Riesgos arquitectónicos y relación con el Objetivo 1

Al correlacionar este análisis con la auditoría de cookies del **Objetivo 1**, se destacan tres puntos clave de seguridad:

1. **JWT almacenado sin `HttpOnly`:** La cookie `auth` que transporta este JWT carece de la bandera `HttpOnly`. Si ocurriera un XSS, un script podría leer el token mediante `document.cookie`, decodificar toda la PII expuesta en el *Payload* y reutilizar el JWT para suplantar la sesión durante sus 6 horas de vigencia.
2. **Riesgo inherente de `HS256` con secretos débiles:** En algoritmos simétricos (`HS256`), la seguridad total depende de la entropía de la clave secreta del servidor (debe ser aleatoria y de al menos 256 bits) y de validar estrictamente el algoritmo esperado en el backend para impedir ataques de confusión de algoritmo o `alg: none`.
3. **Exceso de datos (*Over-exposure*) y desincronización de estado:** Incluir tantos campos de perfil y estado de pago dentro de un JWT sin estado (*stateless*) agranda el tamaño de cada petición HTTP (603 bytes) y dificulta revocar permisos en tiempo real si se revoca un rol o plan antes de que expire `exp`.

---

## 4. Recomendaciones de endurecimiento

* **Minimizar el Payload:** Incluir únicamente los *claims* estrictamente indispensables (`sub` con el identificador del usuario, `iat`, `exp` y `jti` para revocación), eliminando `confirmation_email_uuid`, datos de facturación y PII innecesaria.
* **Proteger el almacenamiento:** Activar la bandera **`HttpOnly`** en la cookie `auth` junto con `Secure` y `SameSite=Lax`.
* **Validación estricta en backend:** Verificar siempre la firma forzando explícitamente el algoritmo permitido (`algorithms=["HS256"]`) con un secreto criptográfico robusto (≥ 256 bits) —o evaluar algoritmos asimétricos como `RS256` / `EdDSA` si varios microservicios verifican tokens— y validar permisos críticos contra la base de datos del lado del servidor.
