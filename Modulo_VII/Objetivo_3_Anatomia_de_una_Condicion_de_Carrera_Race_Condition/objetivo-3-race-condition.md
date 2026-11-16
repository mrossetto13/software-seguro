# Objetivo 3: Anatomía de una Condición de Carrera (Race Condition / TOCTOU)

## Escenario teórico

Un e-commerce emite códigos de descuento de **un solo uso**. Un atacante intercepta la petición de canje y, con una herramienta como **Turbo Intruder** (extensión de Burp Suite), envía **50 peticiones simultáneas** con el mismo código. Varias de ellas pasan la verificación "¿este código ya fue usado?" **antes** de que el servidor marque el código como usado, logrando canjearlo múltiples veces.

---

## 1. ¿Qué es una condición de carrera?

Una **race condition** ocurre cuando el resultado de un proceso depende del **orden o la temporización** de eventos concurrentes, y el código no está protegido contra esa concurrencia.

La variante clásica en seguridad se conoce como **TOCTOU** (*Time-Of-Check To Time-Of-Use*): hay una ventana de tiempo entre el momento en que el sistema **verifica** una condición y el momento en que **actúa** sobre ella. Si otro proceso se cuela en esa ventana, la verificación queda obsoleta.

### La analogía clásica

Dos personas compran entradas para el mismo asiento en cines separados. Cada empleado verifica que el asiento esté libre (pantalla que aún lo muestra disponible) y lo vende. Ambas verificaciones son "correctas" individualmente, pero la ventana entre *verificar* y *reservar* permite vender el asiento dos veces.

---

## 2. Anatomía de la falla en el canje de cupones

### El código vulnerable

```python
# Endpoint: POST /api/redeem  { "code": "DESCUENTO50" }

def redeem_code(user, code):
    coupon = db.query("SELECT * FROM coupons WHERE code = ?", code)

    # 1. TIME-OF-CHECK: ¿ya se usó?
    if coupon.used:
        raise Error("Código ya utilizado")

    # ... (ventana de tiempo: milisegundos o más) ...

    # 2. TIME-OF-USE: aplicar y marcar como usado
    apply_discount(user, coupon.amount)
    db.execute("UPDATE coupons SET used = true WHERE code = ?", code)
```

El problema: entre el `SELECT` (paso 1) y el `UPDATE` (paso 2) existe una **ventana de tiempo**. Durante esa ventana, el código *sigue apareciendo como no usado* en la base de datos.

### Por qué 50 peticiones simultáneas lo explotan

Los servidores web modernos procesan peticiones **en paralelo** (múltiples workers, hilos, procesos o eventos asíncronos). Si llegan 50 peticiones casi al mismo tiempo:

1. Las 50 pasan por el `SELECT` **antes de que cualquiera ejecute el `UPDATE`**.
2. Las 50 leen `used = false` → las 50 pasan la validación.
3. Las 50 aplican el descuento.
4. El `UPDATE` final se ejecuta 50 veces, pero ya no importa: el daño está hecho.

El atacante no necesita que las 50 tengan éxito — con que **2 o 3** se cuelen en la ventana, ya canjeó el código varias veces.

---

## 3. Cómo lo haría el atacante con Turbo Intruder

Turbo Intruder está diseñado para enviar **altos volúmenes de peticiones con concurrencia precisa**, algo que el Intruder tradicional (por su arquitectura de cola) no logra tan bien.

### Paso a paso teórico

1. **Capturar la petición de canje** con el proxy de Burp: `POST /api/redeem` con el body `{"code":"DESCUENTO50"}` y la cookie/token de sesión.

2. **Enviarla a Turbo Intruder** y escribir un script de ataque en Python:

   ```python
   def queueRequests(target, wordlists):
       engine = RequestEngine(
           endpoint=target.endpoint,
           concurrentConnections=30,   # 30 conexiones en paralelo
           requestsPerConnection=100,
           pipeline=False
       )
       # Preparar la petición HTTP cruda con el mismo código 50 veces
       for i in range(50):
           engine.queue(target.req, gate="race")

       # "gate": las peticiones se retienen y se disparan TODAS A LA VEZ
       engine.openGate("race")

   def handleResponse(req, interesting):
       table.add(req)
   ```

3. **El detalle clave — el "gate" (compuerta):** las 50 peticiones se *preparan y retienen* en el servidor de Turbo Intruder. Al llamar `openGate`, se disparan **en el mismo instante**, minimizando la dispersión temporal (jitter) de la red. El objetivo es que lleguen al servidor dentro de la misma ventana de milisegundos.

4. **Técnicas para estrechar aún más la ventana:**
   - **Single-packet attack (HTTP/2):** en servidores HTTP/2, las 50 peticiones se empaquetan en **un solo paquete TCP**, llegando literalmente juntas. Turbo Intruder lo implementa con `engine.queue(..., race=True)` o el modo "race single packet" de Burp Repeater.
   - **Last-byte sync (HTTP/1.1):** se envían todas las peticiones menos el último byte, y ese byte final se envía para todas a la vez, forzando el procesamiento simultáneo.
   - **Varias rondas:** si la primera tanda no logra duplicados, se repite el ataque (a veces la concurrencia del servidor varía).

5. **Leer los resultados:** el atacante busca respuestas `200 OK` múltiples para el mismo código. Si 3 de las 50 respondieron "código aplicado", el descuento se aplicó 3 veces.

### Por qué funciona aunque el código sea "de un solo uso"

El atributo "un solo uso" es una **regla de negocio**, pero su enforcement depende de una secuencia *leer → verificar → escribir* que **no es atómica**. La base de datos no sabe que esas 50 lecturas concurrentes se refieren al mismo recurso que no debe duplicarse.

---

## 4. Cómo prevenirlo del lado del servidor

### a) Atomicidad a nivel de base de datos (la solución principal)

Convertir la verificación y la escritura en **una sola operación atómica**:

```sql
-- El UPDATE verifica y marca en un solo paso
UPDATE coupons
SET used = true, used_by = :user_id, used_at = NOW()
WHERE code = :code AND used = false;
```

Si la consulta afecta **0 filas**, el código ya estaba usado → rechazar. Si afecta **1 fila**, el canje es legítimo. No existe ventana: la condición y la escritura son el mismo paso.

```js
// Equivalente en código de aplicación
const result = await db.query(
  "UPDATE coupons SET used = true WHERE code = ? AND used = false", [code]
);
if (result.affectedRows === 0) {
  throw new Error("Código ya utilizado");
}
applyDiscount(user, coupon.amount);
```

### b) Bloqueos (locks)

- **`SELECT ... FOR UPDATE`**: bloquea la fila desde la lectura hasta el commit de la transacción; las demás peticiones esperan.
- **Locks de aplicación** (Redis `SETNX`, mutex): garantizar exclusión mutua por código de cupón.

### c) Constraints únicos en la base de datos

```sql
-- Tabla de canjes: un cupón solo puede aparecer UNA vez
CREATE TABLE redemptions (
  coupon_id INT NOT NULL,
  user_id   INT NOT NULL,
  UNIQUE (coupon_id)          -- o UNIQUE(coupon_id, user_id) según la regla
);
```

Aunque 50 hilos intenten insertar, la base de datos rechazará 49 por la restricción de unicidad. Es la **última línea de defensa**: funciona incluso si el código de aplicación tiene bugs.

### d) Transacciones con aislamiento adecuado

Envolver *verificar + aplicar + marcar* en una transacción con nivel de aislamiento `SERIALIZABLE` (o usar bloqueo pesimista), de modo que transacciones concurrentes sobre la misma fila se serialicen.

### e) Medidas complementarias

- **Idempotencia:** claves de idempotencia por operación para que peticiones duplicadas devuelvan el mismo resultado sin re-ejecutarse.
- **Rate limiting por usuario/endpoint** (no lo resuelve, pero reduce el ruido).
- **Monitoreo y alertas:** detectar ráfagas de peticiones idénticas en milisegundos — patrón inequívoco de ataque de race condition.
- **Tests de concurrencia:** automatizar 50 peticiones paralelas en CI y verificar que el recurso se consume exactamente una vez.

---

## 5. Resumen

| Aspecto | Situación vulnerable | Solución |
|---------|---------------------|----------|
| Verificación | `SELECT` (check) y `UPDATE` (use) separados | Operación atómica: `UPDATE ... WHERE used = false` |
| Concurrencia | Workers paralelos procesan sin coordinación | Locks (`FOR UPDATE`, Redis) o single-packet defense |
| Integridad | Depende solo del código de aplicación | Constraints `UNIQUE` en la base de datos |
| Detección | Silencioso | Alertas por ráfagas de peticiones idénticas |

> **Conclusión:** la falla no es lógica sino *temporal*. Cada petición individual es "correcta"; el problema es que el intervalo entre verificar y actuar permite que múltiples peticiones pasen la misma verificación con el mismo estado obsoleto. La defensa es eliminar la ventana: hacer check y uso **atómicos**, respaldados por constraints únicos en la base de datos.
