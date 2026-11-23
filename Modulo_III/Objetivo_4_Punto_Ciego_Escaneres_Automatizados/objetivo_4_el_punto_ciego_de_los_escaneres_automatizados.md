# Objetivo 4: El Punto Ciego de los Escáneres Automatizados

Los escáneres de vulnerabilidades automatizados (como **Nessus**, **Acunetix**, **Qualys** o el escáner activo de **Burp Suite**) son herramientas indispensables para cubrir grandes superficies de ataque con rapidez. Sin embargo, tienen una limitación estructural insalvable: **son ciegos ante las fallas de lógica de negocio (CWE-840)**.

---

## 1. ¿Qué detectan bien los escáneres automatizados?

Las herramientas automatizadas trabajan comparando patrones conocidos (*firmas*, heurísticas y respuestas anómalas previsibles):

* **Configuraciones inseguras:** Cabeceras de seguridad faltantes, puertos expuestos, certificados TLS débiles, *Directory Listing* o versiones de software con CVEs publicados.
* **Fallas técnicas sintácticas:** Inyecciones (SQLi, XSS reflejado, Command Injection) donde introducir caracteres especiales provoca una señal reconocible en la respuesta (un error de sintaxis, un payload reflejado sin codificar o un retraso medible).

En todos estos casos existe una regla universal independiente de qué hace la empresa: un `<script>` reflejado sin escapar o un servicio desactualizado siempre constituye un hallazgo.

---

## 2. ¿Por qué la lógica de negocio es su mayor "punto ciego"?

Una falla de lógica de negocio ocurre cuando la aplicación **funciona técnicamente sin errores** (devuelve `HTTP 200 OK`, no lanza excepciones ni rompe sintaxis), pero permite realizar acciones que violan las **reglas comerciales o de diseño** de esa organización específica.

Un escáner automatizado fracasa en este terreno por cuatro razones fundamentales:

1. **Falta de comprensión semántica y de contexto:** Un escáner no sabe qué es un "carrito de compras", qué significa que un turno médico pertenezca a otro paciente, cuál es el precio lógico de un producto ni que un usuario estándar no debería aprobar su propio reembolso. Para la herramienta, cambiar `cantidad=1` por `cantidad=-5` o `rol="admin"` y recibir `200 OK` parece una respuesta completamente normal.
2. **Incapacidad de razonar flujos multipaso (Máquinas de Estado):** Muchas fallas lógicas requieren completar pasos en un orden específico o saltarse una etapa intermedia (por ejemplo, pasar del Paso 1 *"Seleccionar producto"* directo al Paso 4 *"Confirmar orden"* omitiendo el Paso 3 *"Pasarela de pago"*). Los escáneres prueban endpoints de forma aislada, sin entender la secuencia esperada del negocio.
3. **Ausencia de "reglas universales":** Lo que es crítico en una plataforma puede ser legítimo en otra. Un escáner no puede saber si un cupón es acumulable o de un solo uso, si una transferencia puede hacerse hacia la misma cuenta origen, o si un empleado de sucursal A puede consultar facturas de la sucursal B.
4. **Dificultad para detectar impactos indirectos y temporales:** Condiciones de carrera (*Race Conditions*), abusos de redondeo financiero o manipulaciones de autorización horizontal (IDOR/BOLA) requieren comparar dos cuentas distintas e interpretar si el dato devuelto realmente pertenece o no al usuario autenticado.

---

## 3. ¿Por qué la auditoría manual es indispensable?

El pentester humano aporta aquello que la automatización no posee: **comprensión del dominio, pensamiento adversario y creatividad**.

Durante una auditoría manual de lógica de negocio, el especialista:
* Estudia el modelo de negocio, los roles de usuario y los activos críticos de la aplicación.
* Analiza el tráfico real paso a paso y formula preguntas de abuso (*"¿Qué pasa si altero el orden de los pasos?"*, *"¿Qué ocurre si uso el ID del usuario B con la sesión del usuario A?"*, *"¿Qué pasa si envío cero, negativos o peticiones simultáneas?"*).
* Encadena hallazgos menores que un escáner clasificaría como informativos para demostrar un impacto real de negocio.

En conclusión, la **automatización optimiza la cobertura técnica**, pero solo la **auditoría manual profunda** permite descubrir las vulnerabilidades lógicas más críticas y costosas para una organización.
