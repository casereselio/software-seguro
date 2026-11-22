# Objetivo 3: Peligros del "Directory Listing"

El **Directory Listing** (listado de directorios, clasificado como **CWE-548: Exposure of Information Through Directory Listing**) es un error clásico de configuración en servidores web como Apache o Nginx que permite a cualquier usuario navegar y visualizar el contenido completo de una carpeta del servidor directamente desde el navegador.

---

## 1. ¿Qué ocurre técnicamente cuando está activo?

Cuando un cliente envía una petición HTTP `GET` hacia una ruta que apunta a un directorio (por ejemplo, `https://ejemplo.com/backups/`), el servidor web sigue un flujo de resolución predefinido:

1. **Búsqueda de archivo índice:** El servidor busca dentro de esa carpeta un archivo de entrada por defecto configurado en su directiva de índice (`DirectoryIndex` en Apache o `index` en Nginx), como `index.html`, `index.htm` o `index.php`.
2. **Ausencia de índice + Directory Listing habilitado:** Si ese archivo **no existe** y la directiva de auto-indexación está activa (`Options +Indexes` en Apache con `mod_autoindex`, o `autoindex on;` en Nginx), en lugar de devolver un error `403 Forbidden`, el servidor web lee el listado de archivos y subcarpetas del sistema de archivos y **genera dinámicamente una página HTML** (`Index of /backups/`) con enlaces directos de descarga, tamaños y fechas de última modificación para cada elemento.
3. **Recolección trivial (Reconocimiento / Google Dorking):** Un atacante no solo puede explorar manualmente subdirectorios descubiertos por fuerza bruta (con herramientas como `ffuf` o `gobuster`), sino que los propios buscadores indexan estas páginas (búsquedas clásicas como `intitle:"Index of /"`), exponiendo el inventario completo de archivos al público.

---

## 2. Archivos sensibles que pueden quedar expuestos

Cuando el listado de directorios está habilitado en rutas que no deberían ser navegables, un atacante puede encontrar y descargar archivos de alto impacto:

* **Copias de seguridad y volcados de base de datos:** `backup.zip`, `dump.sql`, `db_2026.tar.gz`, `site_old.rar` con tablas de usuarios, hashes de contraseñas o información confidencial de clientes.
* **Archivos de configuración y variables de entorno:** `.env`, `config.php.bak`, `web.config`, `settings.py~`, `wp-config.php.old` conteniendo credenciales de base de datos, API keys, secretos JWT o tokens de servicios Cloud.
* **Código fuente sin interpretar:** Copias antiguas con extensiones que el servidor no ejecuta (ej. `login.php.bak`, `app.jsp.old`), permitiendo descargar el código fuente del backend en texto plano para buscar nuevas vulnerabilidades (análisis de caja blanca).
* **Logs y trazas de aplicación:** `error.log`, `access.log`, `debug.log` que pueden filtrar tokens de sesión, rutas internas, direcciones IP y parámetros sensibles.
* **Llaves criptográficas y certificados privados:** Archivos `.pem`, `.key`, `.p12` o claves SSH dejadas accidentalmente en el *webroot*.
* **Documentos internos y archivos subidos por usuarios:** Facturas en PDF, reportes privados, DNI/PII o exportaciones en Excel/CSV almacenadas en carpetas como `/uploads/` o `/exports/`.
* **Metadatos de control de versiones:** Directorios `.git/` o `.svn/` que permiten reconstruir el historial completo del repositorio y secretos previamente eliminados.

---

## 3. Cómo se mitiga

La mitigación principal consiste en deshabilitar explícitamente la generación automática de índices en la configuración del servidor web y aplicar buenas prácticas de higiene en el directorio público.

### En Apache (`httpd.conf`, `apache2.conf` o `.htaccess`)
Eliminar la opción `Indexes` mediante la directiva `Options -Indexes`:

```apache
<Directory /var/www/html>
    Options -Indexes +FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

Con esta configuración, si un directorio no posee archivo índice, Apache devolverá `403 Forbidden`.

### En Nginx (`nginx.conf` o bloque `server` / `location`)
Asegurar que `autoindex` esté desactivado (es el valor por defecto, pero debe garantizarse explícitamente en cada bloque):

```nginx
server {
    listen 443 ssl;
    server_name ejemplo.com;
    root /var/www/html;

    location / {
        autoindex off;
        index index.html index.php;
    }
}
```

### Medidas complementarias (Defensa en Profundidad)
* **No almacenar archivos sensibles dentro del *webroot*:** Backups, archivos `.env`, logs y credenciales deben residir fuera de la carpeta pública del servidor web.
* **Bloquear acceso a archivos ocultos y respaldos:** Denegar explícitamente peticiones a patrones como `.git`, `.env`, `.bak`, `.sql` y `.old` a nivel de servidor web o WAF.
* **Hardening y auditoría continua:** Verificar periódicamente la configuración de despliegue (IaC / contenedores) para asegurar que el listado de directorios no se reactive en actualizaciones o nuevos entornos.
