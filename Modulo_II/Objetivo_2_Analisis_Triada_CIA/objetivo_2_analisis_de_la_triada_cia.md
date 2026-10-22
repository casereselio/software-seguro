# Objetivo 2: Análisis de la Triada CIA

La Triada CIA (Confidencialidad, Integridad y Disponibilidad) es el modelo base sobre el cual se construyen las políticas de seguridad de la información. A continuación, se detalla cada pilar junto con un ejemplo de cómo puede verse comprometido en un entorno web.

## 1. Confidencialidad (Confidentiality)
**¿Qué es?** 
Se trata de garantizar que la información sensible solo sea accesible para aquellas personas, sistemas o procesos que tienen la autorización explícita para verla. Es decir, prevenir que los datos caigan en manos equivocadas.

**Ejemplo de ataque/vulnerabilidad:** 
* **IDOR (Insecure Direct Object Reference / Referencia Insegura a Objetos Directos):** Un atacante cambia el número de ID de usuario en la URL de una aplicación bancaria (ej. de `id=1001` a `id=1002`) y logra ver el estado de cuenta y los datos personales de otro cliente porque el sistema no verificó correctamente si el atacante tenía permisos para leer esa información específica.

## 2. Integridad (Integrity)
**¿Qué es?** 
Consiste en asegurar que la información es precisa, completa y no ha sido alterada o modificada de forma no autorizada, ya sea de manera maliciosa o accidental. Los datos deben ser confiables.

**Ejemplo de ataque/vulnerabilidad:** 
* **Manipulación de Parámetros (Parameter Tampering) en E-commerce:** Un atacante intercepta la petición web que se envía al comprar un producto y modifica el parámetro del "precio" antes de que llegue al servidor (cambiando el valor de $1000 a $1). Si el servidor no valida esta información, el atacante altera la integridad de la transacción, comprando el artículo a un precio manipulado.

## 3. Disponibilidad (Availability)
**¿Qué es?** 
Asegura que los sistemas, aplicaciones y datos estén operativos y accesibles para los usuarios autorizados en el momento en que los necesiten. Sin disponibilidad, incluso un sistema íntegro y confidencial se vuelve inútil.

**Ejemplo de ataque/vulnerabilidad:** 
* **Ataque DDoS (Denegación de Servicio Distribuido):** Una red de computadoras infectadas (botnet) envía simultáneamente millones de peticiones falsas a un servidor web. El servidor colapsa al intentar procesar tanto tráfico, impidiendo que los usuarios legítimos (como clientes que intentan acceder a su correo o hacer una compra) puedan ingresar a la página.