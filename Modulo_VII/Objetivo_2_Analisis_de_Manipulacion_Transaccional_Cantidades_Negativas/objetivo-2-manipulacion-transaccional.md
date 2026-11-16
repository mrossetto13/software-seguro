# Objetivo 2: Análisis de Manipulación Transaccional (Cantidades Negativas)

## Escenario teórico

Un e-commerce permite agregar productos al carrito. Al interceptar la petición (por ejemplo, con Burp Suite o las DevTools del navegador), un atacante modifica la cantidad del producto de `1` a `-5`. El servidor procesa el valor sin validar y el total del carrito se vuelve negativo, lo que termina **sumando saldo a la cuenta** del atacante en lugar de cobrarle.

---

## 1. ¿Por qué ocurre esta falla?

### Causa raíz: confianza en el cliente

La vulnerabilidad existe porque el servidor **confía en los datos que envía el cliente** sin validarlos. Es una violación directa del principio fundamental de seguridad:

> **"Never trust the client"** — Todo dato que llega del cliente debe tratarse como potencialmente malicioso, sin importar lo que la interfaz de usuario permita.

### Cadena técnica de la falla

1. **El frontend valida, el backend no.** La interfaz probablemente limita la cantidad a valores positivos (ej. `min="1"` en un input HTML), pero esa restricción **solo existe en el cliente**. Cualquier petición puede manipularse fuera del navegador.

2. **Falta de validación de entrada (input validation).** El endpoint recibe el campo `quantity` y lo procesa directamente:
   ```js
   // Código vulnerable
   const { productId, quantity } = req.body;
   const product = await Product.findById(productId);
   const total = product.price * quantity;  // 100 * -5 = -500
   cart.total += total;                     // el total baja en lugar de subir
   ```

3. **Tipado numérico permisivo.** En muchos lenguajes y frameworks, un entero negativo pasa sin problemas por la lógica de negocio: no hay una aserción de que `quantity > 0`.

4. **Lógica de negocio mal diseñada para valores negativos.** El cálculo `precio × cantidad` es matemáticamente válido con negativos, pero **semánticamente inválido** en el dominio del problema. El servidor no distingue entre "matemáticamente correcto" y "válido para el negocio".

5. **Efecto colateral en el saldo.** Si el sistema reembolsa o ajusta saldo cuando el total es negativo (por ejemplo, para manejar devoluciones), un total negativo inducido por el atacante dispara el mismo camino de código: **credita dinero en lugar de debitarlo**.

### Por qué es grave

- Impacto financiero directo: el atacante puede generar saldo ilimitado.
- Suele ser trivial de explotar: solo requiere interceptar y modificar un campo.
- Es una clase de bug muy común: **mass assignment / manipulación de parámetros** sobre campos numéricos (`quantity`, `price`, `discount`, `amount`).

---

## 2. ¿Cómo debería validar el desarrollador este campo del lado del servidor?

### a) Validación estricta de entrada (primera línea de defensa)

Rechazar toda petición cuyo campo no cumpla las reglas del dominio:

```js
// Validación explícita antes de tocar la lógica de negocio
function validateQuantity(quantity) {
  if (!Number.isInteger(quantity)) {
    throw new ValidationError("La cantidad debe ser un número entero");
  }
  if (quantity <= 0) {
    throw new ValidationError("La cantidad debe ser mayor que cero");
  }
  if (quantity > MAX_QUANTITY) {  // ej. 100
    throw new ValidationError("La cantidad excede el máximo permitido");
  }
}
```

Reglas mínimas para un campo de cantidad:

| Regla | Motivo |
|-------|--------|
| **Entero** (`Number.isInteger`) | Evita decimales inyectados (`0.0001`) |
| **Positivo estricto** (`> 0`) | Bloquea negativos y cero |
| **Límite máximo** | Evita overflow y compras absurdas (`999999999`) |
| **Rango coherente con el stock** | No pedir más unidades de las disponibles |

Idealmente, usar el **esquema de validación del framework** (Joi, Zod, class-validator, Pydantic, etc.):

```js
// Ejemplo con Zod
const schema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().positive().max(100),
});
schema.parse(req.body); // lanza error si no cumple
```

### b) Validación en el modelo / base de datos (defensa en profundidad)

No depender solo del código del endpoint:

```sql
-- Constraint a nivel de base de datos
ALTER TABLE cart_items
  ADD CONSTRAINT chk_quantity_positive CHECK (quantity > 0);
```

### c) Nunca confiar en precios ni totales del cliente

El servidor debe **recalcular todo** con sus propios datos:

```js
// El precio SIEMPRE sale de la base de datos, nunca del request
const product = await Product.findById(productId);
const total = product.price * validatedQuantity;
```

Si el cliente envía `price` o `total`, esos campos se **ignoran por completo**.

### d) Validar invariantes de negocio en las transacciones

- Verificar que el **total del carrito nunca sea negativo** antes de procesar el pago:
  ```js
  if (cart.total < 0) {
    throw new BusinessRuleError("El total del carrito no puede ser negativo");
  }
  ```
- Usar **transacciones atómicas** en la base de datos para que el débito del saldo y la creación de la orden sean consistentes.
- Registrar el saldo como valor no negativo con constraints (`CHECK (balance >= 0)` cuando aplique).

### e) Otras medidas complementarias

- **Listas blancas de campos**: solo aceptar los campos esperados en el body (evita que el atacante inyecte `price`, `role`, etc.).
- **Rate limiting y monitoreo**: alertar ante patrones anómalos (totales negativos, cantidades extremas).
- **Tests automatizados** con entradas maliciosas: cantidades negativas, cero, decimales, strings, números enormes.
- **Logging de intentos fallidos**: una cantidad negativa en una petición es casi con seguridad un ataque; vale la pena registrarlo.

---

## 3. Resumen

| Aspecto | Situación vulnerable | Solución |
|---------|---------------------|----------|
| Confianza | El servidor confía en el `quantity` del cliente | Validar todo input del servidor |
| Validación | No existe, o solo está en el frontend | Validación estricta: entero, `> 0`, máximo, en el backend |
| Cálculo | Usa valores del cliente | Recalcular precios/totales desde la base de datos |
| Negocio | Acepta totales negativos | Invariantes: total ≥ 0, constraints en BD, transacciones atómicas |
| Detección | Silencioso | Logging, alertas y tests con entradas maliciosas |

> **Conclusión:** la falla no está en la matemática sino en la falta de validación del dominio. El desarrollador debe tratar cada campo numérico del cliente como hostil y aplicar validación estricta en el servidor, respaldada por constraints en la base de datos e invariantes de negocio.
