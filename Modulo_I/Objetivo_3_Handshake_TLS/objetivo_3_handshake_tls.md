# Objetivo 3: Profundización Técnica (El "Handshake" TLS)


## ¿Qué es?

__El handshake TLS es el proceso de negociacion mediante el cual un cliente y un servidor establecen una **conexión segura y encriptada** antes de intercambiar datos de aplicación.__  
__Este protocolo verifica la identidad del servidor, negocia los algoritmos de encriptación y genera claves de sesión para la comunicación futura__


## El Rol de los Certificados Digitales
El certificado digital funciona exactamente como el documento de identidad del servidor. 
* Durante el handshake, el servidor le envía este certificado al cliente.
* El navegador verifica que el certificado haya sido validado y firmado por una Autoridad Certificadora (CA) de confianza, y que corresponda legítimamente al dominio que solicitamos.
* Esto es el pilar de la autenticación. Nos garantiza que estamos enviando nuestras credenciales al servidor real y evita que caigamos en un ataque de *Man-in-the-Middle* (MitM) donde un tercero suplante la identidad del destino.


## ¿Por qué dos tipos de cifrado?
__Cada familia resuelve un problema que la otra no puede:__


| | Asimétrico | Simétrico |
|---|---|---|
| **Velocidad** | Muy lento (100–1000× más costoso) | Muy rápido, apto para grandes volúmenes |
| **Problema que resuelve** | Compartir un secreto entre partes que nunca se han conocido, sin transmitirlo en claro | Cifrar el tráfico real de forma eficiente |
| **Uso en TLS** | Autenticación + acuerdo de la clave de sesión | Cifrado de todos los datos de la conexión |  
  
__Esta combinación, conocida como cifrado híbrido, aprovecha la seguridad del intercambio de claves asimétrico y la velocidad del cifrado simétrico.__