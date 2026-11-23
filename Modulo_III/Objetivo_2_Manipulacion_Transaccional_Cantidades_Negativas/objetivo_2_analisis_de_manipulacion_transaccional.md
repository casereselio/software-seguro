# Objetivo 2: Análisis de Manipulación Transaccional (Cantidades Negativas)

Las fallas de lógica de negocio en flujos transaccionales —como aceptar cantidades negativas en un carrito de compras— son errores críticos que comprometen directamente la **integridad financiera** de una plataforma de comercio electrónico (**CWE-20: Improper Input Validation** y **CWE-840: Business Logic Errors**).

---

## 1. ¿Por qué ocurre técnicamente esta falla?

El problema nace de una combinación de tres errores clásicos de diseño e implementación:

### a) Confianza ciega en la validación del lado del cliente (Frontend)
Muchos desarrolladores restringen el selector de cantidad únicamente en la interfaz web mediante atributos HTML como `<input type="number" min="1" max="10">` o validaciones en JavaScript. Sin embargo, los controles en el navegador son meramente de usabilidad (UX), no de seguridad: un usuario puede interceptar la petición HTTP `POST /api/cart/add` con un proxy (como Burp Suite) o enviar la solicitud directamente por `curl`, modificando el cuerpo JSON de `{"product_id": 42, "quantity": 1}` a `{"product_id": 42, "quantity": -5}`.

### b) Uso de tipos de datos con signo sin validación de rango (Dominio)
En el backend, el campo `quantity` suele recibirse como un entero estándar con signo (`int` / `INTEGER`), que acepta tanto valores positivos como negativos y cero. Si el código solo verifica que el dato sea numérico (o ni siquiera valida su rango), `-5` supera el parseo sin lanzar excepciones.

### c) Operación aritmética directa sin invariantes de negocio
Al calcular el subtotal y el total del pedido:

```python
# Ejemplo vulnerable
subtotal = product.price * request.json["quantity"]   # 100 * (-5) = -500
cart.total += subtotal                                # Total negativo (-500)
user.balance -= cart.total                            # balance - (-500) = balance + 500
```

Cuando el flujo de cobro o billetera interna resta el total del carrito al saldo del usuario (o procesa un monto negativo como reembolso/crédito), la regla de signos convierte un débito en una **acreditación indebida de saldo** (o reduce el costo de otros productos legítimos agregados en el mismo carrito).

---

## 2. Cómo debe validar y mitigar estrictamente el desarrollador del lado del servidor

La única frontera de confianza es el **servidor (Backend)**. La mitigación debe aplicarse en múltiples capas:

### 1. Validación estricta de tipo y rango (Allow-list / Schema Validation)
Antes de ejecutar cualquier cálculo, validar el esquema de entrada (por ejemplo, con Pydantic, Zod, Joi o Bean Validation):
* Debe ser estrictamente un **número entero** (rechazar flotantes como `0.5`, cadenas, `null`, `NaN` o notación científica).
* Debe cumplir un **rango acotado de negocio**: `1 <= quantity <= MAX_ALLOWED_PER_ORDER` (y nunca superior al stock disponible real).
* Rechazar cualquier valor fuera de rango con `400 Bad Request` o `422 Unprocessable Entity` (nunca corregir silenciosamente con `abs()` sin auditar, ya que enmascara intentos de abuso).

```python
# Ejemplo seguro (Python + Pydantic)
from pydantic import BaseModel, Field

class AddToCartRequest(BaseModel):
    product_id: int = Field(..., gt=0)
    quantity: int = Field(..., ge=1, le=50)  # Entero estricto entre 1 y 50
```

### 2. Verificación de invariantes de negocio en el cálculo y el checkout
Además de validar cada parámetro individual, el backend debe verificar invariantes globales antes de confirmar el pago:
* **Nunca confiar en precios ni totales enviados por el cliente:** consultar siempre el precio unitario vigente desde la base de datos.
* **Validar el total final:** comprobar explícitamente que `subtotal > 0` para cada ítem y que `order.total > 0` (o `>= 0` únicamente si existe un cupón legítimo verificado del lado del servidor con tope en `0`, nunca negativo).
* **Separar operaciones de débito y crédito:** el módulo de pagos jamás debe aceptar montos negativos; las devoluciones/reembolsos deben tramitarse por un flujo independiente con control de autorización específico.

### 3. Restricciones a nivel de Base de Datos (Defensa en Profundidad)
Incorporar restricciones `CHECK` en el esquema relacional para que, incluso ante un error en la capa de aplicación, la base de datos rechace registros inválidos:

```sql
ALTER TABLE cart_items
  ADD CONSTRAINT chk_quantity_positive CHECK (quantity > 0);

ALTER TABLE orders
  ADD CONSTRAINT chk_total_non_negative CHECK (total_amount >= 0);
```

### 4. Pruebas unitarias y monitoreo
Incluir casos de prueba automatizados con valores límite y maliciosos (`-1`, `-99999`, `0`, `1.5`, `9999999999`, *integer overflow*) y registrar alertas de seguridad cuando se reciban peticiones con cantidades negativas que no podrían originarse en la interfaz legítima.
