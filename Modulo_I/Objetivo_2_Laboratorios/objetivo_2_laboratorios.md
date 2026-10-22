# Objetivo 2: Ejecución de Laboratorios

A continuación detallo la resolución de los laboratorios prácticos de la plataforma Software Seguro, documentando mi metodología y las vulnerabilidades que exploté en cada uno de ellos.

## 1. Laboratorio: Turnero
**Vulnerabilidad:** Insecure Direct Object Reference (IDOR) y Manipulación de Métodos HTTP.
**Objetivo:** Eliminar únicamente los turnos médicos asociados al usuario malintencionado "xdalvik".

-   **Análisis inicial:** Al ingresar con las credenciales provistas (luis), intercepté la petición al intentar cancelar un turno propio. Identifiqué que la aplicación enviaba una petición `DELETE` al endpoint `/api/appointments/<ID>`.
-   **Enumeración:** Para encontrar los turnos de "xdalvik", utilicé la herramienta **Intruder** de Burp Suite, iterando sobre un rango de IDs en el endpoint de turnos.
    ![Enumeración con Intruder](/imagenes/lab1.1.png)
-   **Explotación:** Una vez que identifiqué los IDs que no pertenecían a mi usuario pero que el sistema me permitía manipular (IDOR), utilicé el módulo **Repeater** para enviar las peticiones `DELETE` a esos IDs específicos,logrando eliminar con éxito los turnos de "xdalvik" sin afectar al resto.  

    ![Eliminación de turno con Repeater](/imagenes/lab1.2.png)

---

## 2. Laboratorio: Ventas
**Vulnerabilidad:** Enumeración de IDs (Fuzzing) y Scripting para automatización.
**Objetivo:** Descubrir la cantidad de ventas de la competencia iterando IDs y generar un hash MD5 de esa cantidad.

-   **Análisis inicial:** Al acceder a la ruta `/ventas`, el servidor devolvía un error `400 Bad Request`. Interceptando y alterando la petición, descubrí que la ruta aceptaba una query string: `/ventas/?id=<numero>`.
-   **Descubrimiento:** Al probar manualmente distintos IDs, noté que el servidor respondía con `403 Forbidden` (indicando que la venta existe pero no tengo permisos para verla) o `404 Not Found` (indicando que la venta no existe). 
-   **Explotación (Automatización):** El rango a iterar era demasiado amplio y la versión Community de Burp Suite me limitaba la velocidad del Intruder. Para resolverlo de forma eficiente, desarrollé un script en Python. El script envía peticiones GET masivas, lleva un contador de cuántas respuestas `403` devuelve el servidor, y finalmente hashea ese contador en MD5.
    ![Script en Python](/imagenes/lab2.script.png)
-   **Resultado:** Al ejecutar el script en mi terminal, automaticé el descubrimiento y obtuve directamente la flag final.  
    ![Flag obtenida en consola](/imagenes/lab2-scriptfin.png)

---

## 3. Laboratorio: Presupuesto
**Vulnerabilidad:** Asignación Masiva (Mass Assignment / Parameter Tampering).
**Objetivo:** Alterar los montos de un presupuesto mediante la inyección de campos no autorizados en la petición de actualización.

-   **Análisis inicial:** Al visualizar el presupuesto, noté que la aplicación realiza una petición `GET` a `/api/gastos/` que devuelve un JSON con todos los ítems, sus categorías, montos y un flag booleano `revisado`. 
-   **Interceptación:** Al marcar un ítem como revisado desde la interfaz, observé que la aplicación enviaba una petición `POST` al endpoint `/api/gastos/<id>/editar/` con el payload `{"revisado": true}`.
-   **Explotación:** Utilizando el **Repeater** de Burp Suite, manipulé el JSON de la petición agregando un campo adicional que la interfaz gráfica no exponía: el `monto`. Al enviar el payload `{"monto": "0", "revisado": true}`, el servidor devolvió un `200 OK`, confirmándome que era vulnerable a Asignación Masiva.  
    ![Inyección de parámetro monto](/imagenes/lab3-3.png)
-   **Resultado:** Calculé matemáticamente los valores necesarios para cumplir con los promedios exigidos por el enunciado y envié sucesivas peticiones manipuladas para alterar la base de datos hasta que obtuve la flag.

---

## 4. Laboratorio: Gran Rifa 2019
**Vulnerabilidad:** Asignación Masiva (Mass Assignment / Parameter Tampering).
**Objetivo:** Modificar el estado de pago de una rifa ajena ("John Backus") alterando los parámetros de la petición.

-   **Análisis inicial:** Al cargar la página principal, el sistema realiza una petición `GET` a `/api/numeros` que devuelve un JSON detallado con el estado de todas las rifas. En este JSON descubrí un atributo clave que la interfaz no me permitía editar directamente: `"esta_pago": false`.  
    ![Atributo descubierto en JSON](/imagenes/lab4.3.png)
-   **Interceptación:** Al utilizar la función legítima de la aplicación para "Editar" el nombre del comprador, intercepté una petición `POST` a `/api/numeros/4/editar/` que enviaba como payload un JSON simple: `{"comprador": "John Backus"}`.  
    ![Petición original interceptada](/imagenes/lab4.5.png)
-   **Explotación:** Sabiendo que el backend confía ciegamente en el JSON entrante (como pude comprobar en el laboratorio de Presupuestos), utilicé el **Repeater** para inyectar el parámetro crítico que había descubierto en el primer paso. Modifiqué el payload agregando el estado de pago: `{"comprador": "John Backus", "esta_pago": true}`.  
    ![Payload modificado en Repeater](/imagenes/lab4.7.png)
-   **Resultado:** El servidor procesó mi petición inyectada respondiendo con `{"estado": "OK"}`. Al recargar la página en el navegador, la base de datos reflejó el pago y me reveló la flag exitosamente.  
    ![Flag final obtenida](/imagenes/lab4.flag.png)