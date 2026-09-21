# Objetivo 1: Puertos y Servicios Esenciales
### A continuación se detalla la función de los puertos de red fundamentales y los servicios que operan en ellos por defecto:  


| Puerto | Protocolo | Servicio | Descripción |
|:------:|:---------:|:--------:|:------------|
| **20** | TCP | **FTP – Data** | Utilizado por el Protocolo de Transferencia de Archivos (FTP) exclusivamente para la transmisión de los datos y archivos entre el cliente y el servidor. |
| **21** | TCP | **FTP – Control** | Empleado por FTP para establecer la conexión inicial y enviar los comandos de control (como inicio de sesión, navegación de directorios) en un canal separado de los datos. |
| **22** | TCP | **SSH** | Secure Shell. Protocolo de administración remota que permite a los administradores acceder y ejecutar comandos en un servidor a través de un canal criptográficamente seguro |
| **23** | TCP | **Telnet** | Protocolo de acceso remoto **sin cifrar** (texto plano). Es legacy y está prácticamente reemplazado por SSH por razones de seguridad. |
| **25** | TCP | **SMTP** | Simple Mail Transfer Protocol. Es el estándar utilizado para el envío y enrutamiento de correos electrónicos entre servidores en Internet. |
| **53** | TCP/UDP | **DNS** | Domain Name System. Actúa como el directorio telefónico de Internet, traduciendo nombres de dominio legibles por humanos (ej. `softwareseguro.com.ar`) en direcciones IP. |
| **80** | TCP | **HTTP** | Hypertext Transfer Protocol.  Canal base para la navegación web tradicional, donde la información viaja en texto claro sin protección criptográfica. |
| **110** | TCP | **POP3** | Post Office Protocol versión 3. Utilizado por los clientes locales de correo para conectarse a un servidor, descargar los mensajes recibidos y, generalmente, borrarlos del servidor remoto. |
| **443** | TCP | **HTTPS** | HTTP Secure. Canal estándar para la web moderna. Transfiere el tráfico HTTP dentro de un túnel cifrado (TLS/SSL) para garantizar la privacidad y autenticidad de los datos. |
| **3306** | TCP | **MySQL / MariaDB** | Puerto por defecto del sistema de gestión de bases de datos relacionales MySQL (y su fork MariaDB). Los clientes y aplicaciones (web apps, ORM, etc.) se conectan por este puerto para ejecutar consultas SQL. No debería estar expuesto a Internet. |