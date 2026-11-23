# Objetivo 4: Análisis del Ataque de Algoritmo "None" (JWT)

El ataque del **algoritmo `none`** en **JSON Web Tokens (JWT)** —clasificado bajo **CWE-347: Improper Verification of Cryptographic Signature**— es una falla crítica de autenticación y autorización que permite a un atacante **forjar tokens arbitrarios y suplantar la identidad de cualquier usuario (incluido un administrador)** sin conocer jamás la clave secreta ni la clave privada del servidor.

---

## 1. ¿Por qué existe `"alg": "none"` en el estándar JWT?

La especificación **RFC 7519 (JWT)** y **RFC 7515 (JWS)** contempla el concepto de **Unsecured JWT** (tokens sin firma criptográfica), diseñado originalmente para escenarios internos donde la integridad del token ya fue verificada por otra capa de transporte o red (por ejemplo, comunicación mTLS entre servicios internos).

Para indicar que un token no lleva firma, el estándar permite declarar en el *Header*:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

El problema crítico surge cuando una librería de verificación JWT o el código del backend **confía ciegamente en el campo `alg` que viene dentro del propio token enviado por el cliente** para decidir qué algoritmo usar al verificarlo.

---

## 2. ¿Por qué un backend vulnerable permite falsificar cualquier identidad sin conocer la clave secreta?

Un JWT firmado legítimamente tiene tres partes: `Base64URL(Header).Base64URL(Payload).Base64URL(Signature)`.

En una implementación segura, el servidor utiliza su clave secreta (en `HS256`) o su clave pública (en `RS256`) para recalcular o verificar la tercera parte (`Signature`) sobre `Header.Payload`. Sin conocer el secreto del servidor, cualquier modificación del *Payload* rompe la firma y el token es rechazado.

En un **backend vulnerable al ataque `alg: none`**, ocurre una falla lógica de validación:

1. **Lectura del *Header* no confiable:** El servidor recibe el JWT y decodifica el *Header* **antes** de verificar la firma para saber qué algoritmo utilizar.
2. **Omisión de verificación criptográfica:** Al leer `"alg": "none"` (o variantes de mayúsculas/minúsculas como `"None"`, `"NONE"`, `"nOnE"` si el filtro es débil), la librería vulnerable interpreta que el token es un *Unsecured JWT* válido y **salta por completo el paso de verificación de firma** —sin consultar nunca la clave secreta del servidor—.
3. **Confianza directa en el *Payload* manipulado:** Al darse el token por "válido", el backend confía directamente en los *claims* del *Payload* (`sub`, `id`, `email`, `role`, `is_superadmin`) y otorga acceso con la identidad y privilegios declarados en él.

### Flujo conceptual de la falsificación
1. El usuario obtiene un JWT legítimo de su propia sesión (por ejemplo, `role: "user"`, `is_superadmin: false`, firmado con `HS256`).
2. Decodifica el *Header* y cambia `"alg": "HS256"` por `"alg": "none"`.
3. Decodifica el *Payload* y modifica los campos de identidad o privilegios (por ejemplo, `sub: "admin"` o `is_superadmin: true`, `role_id: 0`).
4. Codifica nuevamente el *Header* y el *Payload* en Base64URL y **elimina la tercera sección (la firma)**, conservando el punto separador final:
   ```text
   <Base64URL(Header_con_alg_none)>.<Base64URL(Payload_modificado)>.
   ```
5. Al enviar este token al backend vulnerable, el servidor acepta la identidad falsificada como auténtica **sin haber necesitado nunca la clave secreta**.

---

## 3. Impacto de seguridad

* **Compromiso total de autenticación (*Authentication Bypass*):** Suplantación de cualquier cuenta registrada en el sistema.
* **Escalación vertical de privilegios:** Obtención de acceso de administrador o superadministrador modificando *claims* como `role` o `is_superadmin`.
* **Evade controles basados en fortaleza del secreto:** Incluso si el servidor usa una clave privada RSA de 4096 bits o un secreto HMAC de alta entropía, el ataque los vuelve irrelevantes porque **la verificación criptográfica nunca se ejecuta**.

---

## 4. Cómo se mitiga correctamente

La mitigación definitiva exige no confiar nunca en el encabezado `alg` provisto por el cliente:

1. **Definir una lista blanca estricta de algoritmos en el servidor (*Explicit Algorithm Pinning*):** Al verificar un JWT, pasar explícitamente en el código el algoritmo esperado (por ejemplo, únicamente `HS256` o `RS256`), rechazando automáticamente `"none"` y cualquier otro algoritmo no autorizado:
   ```python
   # Ejemplo seguro en Python (PyJWT)
   payload = jwt.decode(
       token,
       key=SECRET_KEY,
       algorithms=["HS256"],  # Lista blanca explícita: rechaza "none" y algoritmos distintos
   )
   ```
   ```javascript
   // Ejemplo seguro en Node.js (jsonwebtoken)
   const payload = jwt.verify(token, process.env.JWT_SECRET, {
     algorithms: ["HS256"],
   });
   ```
2. **Rechazar tokens sin firma:** Configurar la librería para exigir siempre una firma no vacía y utilizar versiones actualizadas (las librerías modernas como `PyJWT >= 2.x`, `jsonwebtoken >= 9.x` o `jose` rechazan `alg: none` por defecto).
3. **Nunca usar `decode()` sin `verify()`:** Asegurarse de que la aplicación utilice siempre la función de verificación criptográfica (`jwt.verify` / `jwt.decode` con `verify_signature=True`) y no un simple decodificador Base64.
