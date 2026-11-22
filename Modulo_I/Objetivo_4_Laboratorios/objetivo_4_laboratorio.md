# Objetivo 4: Ejecución de Laboratorios

A continuación detallo la resolución de los laboratorios prácticos avanzados de la plataforma Software Seguro, documentando mi metodología y las vulnerabilidades explotadas en cada desafío.

## 1. Laboratorio: Apagar la IA

**Vulnerabilidad:** Enumeración de Directorios Insegura (IDOR con Hashing Predictivo) y Fuzzing.
**Objetivo:** Descubrir un código de 16 dígitos oculto en una URL impredecible y generar su hash MD5.

* **Análisis inicial:** Al analizar los dos archivos HTML provistos en el enunciado, noté que la URL seguía el patrón `/codes/<hash_md5>`. Al tomar los hashes de las URLs de prueba y revertirlos, descubrí que correspondían a números secuenciales cortos: `9912` y `9995`.

* **Hipótesis:** Deduje que el código de 16 dígitos necesario para apagar la IA se encontraba en una ruta similar, cuyo ID numérico en texto plano (antes de ser hasheado) probablemente se encontraba en un rango cercano a los 10000.

* **Explotación (Automatización):** Para no probar miles de hashes manualmente, desarrollé un script en Python. El script iteró identificadores numéricos, los convirtió al vuelo en MD5 y realizó peticiones GET a cada URL resultante. Para extraer el código exacto de la respuesta HTML, utilicé la expresión regular `match = re.search(r'\b\d{16}\b', response.text)`.

* **Resultado:** Tras ejecutar el script, detectó exitosamente una respuesta HTTP 200 en el ID `11520` (hash `cdf49f5251e7b3eb4f009483121e9b64`) y la expresión regular extrajo el código de apagado: `5524663362514956`. Finalmente, convertí ese número a MD5 obteniendo la flag: `a8e0e8ff02dde0f62fdf4de5142d7de0`.

## 2. Laboratorio: El mejor secreto

**Vulnerabilidad:** Cifrado débil (Side-Channel visual y fuerza bruta basada en patrones).
**Objetivo:** Descifrar el archivo `secreto.zip` a partir de un video analizando el patrón de pulsaciones en un teclado modificado.

* **Análisis inicial (OSINT Visual):** Al observar detalladamente el video, me guié por las luces que se encendían al presionar cada tecla. Como desconocía el dígito real, le asigné un número arbitrario a cada tecla única en el orden en que fueron presionadas. Esto me permitió deducir un patrón estructural exacto de 12 posiciones utilizando 5 teclas únicas: `123341444453`.

* **Generación de Diccionario (Scripting):** Para explotar esta vulnerabilidad, programé un script en Python que iteró sobre los dígitos del 0 al 9 y generó todas las combinaciones posibles que respetaban ese patrón específico, creando un diccionario personalizado (`wordlist_patron.txt`) con 100.000 combinaciones posibles.

* **Extracción del Hash:** Migré mi entorno a una máquina virtual con Kali Linux. Utilicé la herramienta `zip2john` para extraer el hash de la contraseña que protegía el archivo y redirigí la salida hacia el archivo `hash.txt` (`zip2john secreto.zip > hash.txt`).

* **Cracking (Fuerza Bruta):** Con el hash aislado y mi diccionario a medida, ejecuté John the Ripper mediante el comando `john --wordlist=wordlist_patron.txt hash.txt`. La herramienta rompió el hash rápidamente y reveló que la contraseña real era `547795999937`, coincidiendo a la perfección con la estructura descubierta.

* **Resultado:** Con la contraseña en texto plano, procedí a desbloquear y descomprimir el archivo `secreto.zip`, obteniendo acceso directo al documento `secreto.txt` que contenía la flag final.

## 3. Laboratorio: Votación

**Vulnerabilidad:** Bypass de control de estado (Client-side state tracking bypass) y falta de validación en el servidor.
**Objetivo:** Obtener la flag logrando que la cantidad de votos de la UTN supere a los de Harvard.

* **Análisis inicial:** Al ingresar al laboratorio, analicé el tráfico HTTP al momento de emitir un voto legítimo. Observé que la petición `POST` al endpoint `/src/ctl/votacion.ctl.php` respondía con un `200 OK` y asignaba una nueva cookie mediante el encabezado `Set-Cookie: voto=...`.

* **Identificación del fallo lógico:** Al intentar enviar un segundo voto, el servidor me devolvía un error. Sin embargo, noté que la aplicación dependía exclusivamente de la presencia de la cookie `voto` en mi navegador para saber si yo ya había participado.

* **Prueba de concepto (PoC):** Envié la petición bloqueada al **Repeater** de Burp Suite y eliminé manualmente la cookie `voto` de las cabeceras, manteniendo mi sesión. Al enviarla, el servidor la procesó como válida y el contador de la UTN sumó un voto más, confirmando la vulnerabilidad.

* **Explotación (Automatización):** Dado que necesitaba enviar más de 3200 votos, desarrollé un script en Python utilizando la librería `requests`. En el script configuré un bucle `while` para enviar la petición `POST` reiteradas veces, inyectando los headers exactos de mi sesión pero omitiendo deliberadamente la cookie restrictiva.

* **Resultado:** Dejé corriendo el script hasta que el contador de peticiones exitosas superó la diferencia necesaria. Al recargar la página principal, la UTN había superado a Harvard y el sistema me reveló la flag final.

## 4. Laboratorio: Votación - Nueva versión

**Vulnerabilidad:** Bypass de restricción de IP (IP Spoofing) mediante la cabecera `X-Forwarded-For`.
**Objetivo:** Superar los votos de Harvard evadiendo la nueva restricción que impide votar más de una vez desde la misma dirección IP.

* **Análisis inicial:** Al igual que en la versión anterior, el objetivo era sumar votos automatizados para la UTN. Sin embargo, al aplicar el bypass de la cookie `voto` que usé en la versión 1, el servidor me bloqueó con el mensaje `{"error":"No se puede votar más de una vez desde la misma ip"}`.

* **Identificación del fallo:** Comprendí que el backend ahora estaba registrando la dirección IP de cada votante para evitar el fraude. No obstante, las aplicaciones web detrás de proxies o balanceadores de carga (como Cloudflare) suelen confiar en la cabecera HTTP `X-Forwarded-For` para identificar la IP original del cliente. Si el backend no valida correctamente de dónde proviene esa cabecera, es posible falsificarla.

* **Explotación (Automatización):** Modifiqué mi script de Python para inyectar dinámicamente la cabecera `X-Forwarded-For` en cada petición. Utilicé la librería `random` para generar una dirección IP distinta en cada iteración del bucle y enviarla junto con la petición `POST`, engañando al servidor para que creyera que cada voto provenía de una computadora diferente en el mundo.

* **Resultado:** Ejecuté el script modificado, logrando bypassear la restricción de IP de manera exitosa. Una vez que el contador de votos inyectados superó al de Harvard, recargué la página y el sistema validó la victoria, entregándome la flag correspondiente.