# Objetivo 2: SSRF y el Peligro en la Nube (AWS/GCP)

Una vulnerabilidad de **Server-Side Request Forgery (SSRF / Falsificación de Peticiones del Lado del Servidor, CWE-918)** ocurre cuando una aplicación web recibe una URL o destino controlado por el usuario (por ejemplo, para generar una vista previa de un enlace, importar una imagen desde URL o configurar un *webhook*) y el **propio servidor backend** ejecuta la petición HTTP hacia esa dirección sin validar adecuadamente el destino.

En infraestructuras Cloud (como **AWS**, **GCP** o **Azure**), el SSRF pasa de ser un problema de acceso a red interna a convertirse frecuentemente en un **compromiso total de la cuenta en la nube**.

---

## 1. ¿Qué es la dirección `169.254.169.254` en AWS?

La dirección IP **`169.254.169.254`** es una dirección *link-local* no enrutable desde Internet que los proveedores Cloud reservan para el **Instance Metadata Service (IMDS)** (en AWS EC2, GCP Compute Engine y Azure VM).

* Desde Internet, un atacante externo **no puede** conectarse directamente a `169.254.169.254`.
* Sin embargo, **desde dentro de una instancia EC2**, cualquier proceso local —incluida la aplicación web vulnerable a SSRF— puede realizar peticiones HTTP a `http://169.254.169.254/latest/meta-data/` para consultar información operativa y de seguridad de la máquina virtual.

Cuando el servidor vulnerable ejecuta un `GET` hacia esa IP en nombre del atacante (y le devuelve la respuesta), el atacante logra leer el servicio de metadatos internos de AWS.

---

## 2. ¿Qué información altamente sensible espera extraer un atacante?

El objetivo principal al consultar el IMDS de AWS (`http://169.254.169.254/latest/meta-data/`) es obtener **credenciales temporales de IAM**, además de datos clave de configuración:

1. **Credenciales temporales del Rol IAM asociado a la instancia (impacto crítico):**
   * Consultando `http://169.254.169.254/latest/meta-data/iam/security-credentials/` se obtiene el nombre del rol IAM asignado a la EC2 (por ejemplo, `app-prod-role`).
   * Luego, al consultar `.../iam/security-credentials/<NOMBRE_ROL>`, IMDSv1 devuelve un JSON con:
     * `AccessKeyId`
     * `SecretAccessKey`
     * `Token` (Session Token)
   * **Impacto:** El atacante puede cargar esas credenciales en su propia CLI de AWS y operar con **todos los permisos que tenga ese rol** sobre servicios de la nube: leer o descargar buckets **S3** completos, consultar secretos en **Secrets Manager / Parameter Store**, conectarse a bases de datos **RDS**, leer colas SQS o escalar privilegios en toda la cuenta AWS.
2. **Scripts de arranque (*User Data*):** En `http://169.254.169.254/latest/user-data`, muchas organizaciones incluyen scripts de inicialización que contienen contraseñas, tokens de repositorios, claves de API o variables de entorno sensibles.
3. **Topología de red y configuración de la instancia:** Direcciones IP privadas, *Security Groups*, identificador de VPC/subred, zona de disponibilidad, nombre de host y claves públicas SSH, facilitando el movimiento lateral hacia otros servicios internos.

*(En **Google Cloud Platform (GCP)** el concepto es equivalente usando `http://metadata.google.internal/computeMetadata/v1/`, aunque GCP exige la cabecera personalizada `Metadata-Flavor: Google`, lo que mitiga automáticamente SSRFs simples que no permitan inyectar cabeceras).*

---

## 3. Cómo mitigar el SSRF en entornos Cloud

### Mitigación a nivel de infraestructura AWS (la más efectiva contra robo de credenciales)
* **Forzar IMDSv2 (Deshabilitar IMDSv1):** IMDSv2 exige un flujo basado en sesión que requiere primero una petición **`PUT`** con la cabecera `X-aws-ec2-metadata-token-ttl-seconds` para obtener un token temporal y luego enviar ese token en una cabecera `X-aws-ec2-metadata-token` en el `GET`. Un SSRF clásico de tipo `GET` simple no puede completar ese intercambio.
* **Principio de Mínimo Privilegio en Roles IAM:** Limitar el rol asignado a la instancia EC2 únicamente a los recursos mínimos indispensables (nunca asignar permisos administrativos amplios como `AdministratorAccess` o `s3:*`).

### Mitigación a nivel de aplicación y red
* **Lista blanca estricta (*Allow-list*):** Si la aplicación debe descargar recursos externos, restringir los destinos únicamente a dominios y puertos aprobados y protocolos `https://`.
* **Bloqueo de rangos privados tras la resolución DNS:** Resolver el dominio antes de conectar y rechazar cualquier IP perteneciente a rangos privados o reservados (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, **`169.254.0.0/16`** y equivalentes IPv6), deshabilitando redirecciones HTTP automáticas que puedan evadir el filtro (protección contra *DNS Rebinding* y *Open Redirect*).
* **Aislamiento de salida (Egress Filtering):** Enrutar peticiones salientes de funciones de fetch/webhook a través de un proxy dedicado y aislado sin acceso a la red interna ni al servicio de metadatos.
