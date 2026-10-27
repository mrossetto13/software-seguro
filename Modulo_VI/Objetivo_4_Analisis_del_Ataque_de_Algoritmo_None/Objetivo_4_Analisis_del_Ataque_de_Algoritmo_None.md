# Objetivo 4: Análisis del Ataque de Algoritmo "None" en JWT

**Materia:** Ciberseguridad — Tarea de investigación
**Tema:** Falsificación de identidad mediante el algoritmo "none" en JWT

---

## 1. Fundamento: cómo funciona un JWT

Un JWT (JSON Web Token) tiene tres partes codificadas en Base64Url, separadas por puntos:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  .  eyJzdWIiOiIxMjM0NSIsInJvbGUiOiJ1c3VhcmlvIn0  .  SflKxwRJSMeKKF2QT4f...
        HEADER                                        PAYLOAD                                  SIGNATURE
```

- **Header** — declara el algoritmo de firma: `{"alg": "HS256", "typ": "JWT"}`
- **Payload** — los *claims* (datos): identidad, rol, expiración, etc.
- **Signature** — el servidor calcula `HMAC_SHA256(header.payload, clave_secreta)`. Es la **prueba matemática de que el payload no fue alterado**: solo quien posee la clave secreta puede producir una firma válida.

El flujo normal: el usuario se autentica → el servidor emite un JWT firmado → en cada petición el servidor **verifica la firma** antes de confiar en el payload. Si alguien modifica el payload (ej.: cambia `"role":"user"` por `"role":"admin"`), la firma ya no coincide y el token es rechazado.

---

## 2. La vulnerabilidad: el algoritmo "none"

El estándar JWT (RFC 7519) define un algoritmo especial: **`none`** — "sin firma". Su existencia es legítima en teoría: para tokens que ya viajan por un canal confiable y solo necesitan integridad estructural, no criptográfica.

Un token "none" se ve así:

```
Header:  {"alg": "none", "typ": "JWT"}
Payload: {"sub": "12345", "role": "admin"}
Signature: (vacía — el token termina en el segundo punto)
```

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiIxMjM0NSIsInJvbGUiOiJhZG1pbiJ9.
```

**El problema:** si el backend acepta `alg: none` (o peor, si su librería lo acepta por defecto), el servidor **omite la verificación de firma por completo** y confía ciegamente en el contenido del payload.

---

## 3. El ataque, paso a paso

1. **El atacante obtiene un token legítimo** (se registra o inicia sesión normalmente). Es público por diseño — viaja en cada petición.

2. **Lo decodifica** (Base64 no es cifrado, es solo codificación reversible):

   ```
   Header:  {"alg":"HS256","typ":"JWT"}
   Payload: {"sub":"12345","user":"martin","role":"user"}
   ```

3. **Forja un nuevo token:**
   - Cambia el header a `{"alg":"none","typ":"JWT"}`
   - Cambia el payload a `{"sub":"1","user":"admin","role":"admin"}`
   - **Deja la firma vacía** (el token termina en el segundo punto, a veces con un punto final: `...In0.`)

4. **Envía el token falsificado** en el header `Authorization: Bearer eyJhbGciOiJub25lIi...In0.`

5. **El backend vulnerable:**
   - Lee el header → `alg: none` → "este token no lleva firma"
   - **No verifica nada** (no hay firma que verificar)
   - Confía en el payload → `role: admin` → **el atacante es administrador**

---

## 4. Por qué no hace falta conocer la clave secreta

Aquí está el corazón técnico del análisis. La seguridad del JWT descansa en un supuesto: *modificar el payload invalida la firma, y solo el servidor puede firmar de nuevo*.

El ataque `none` **rompe el supuesto eliminando la firma del juego**:

- El atacante **nunca necesita romper el HMAC ni adivinar la clave** — simplemente le dice al servidor: *"este token no está firmado, verificalo como tal"*.
- El servidor vulnerable ejecuta la verificación **según lo que dice el propio token** (dato controlado por el atacante). Es una falla de diseño: **la decisión de verificar se toma basándose en información no confiable**.
- Es el equivalente a un control de acceso donde el visitante le dice al guardia "no hace falta que me identifiques" y el guardia le cree.

> **En síntesis:** la clave secreta solo es necesaria para *producir* una firma válida. Si el servidor acepta tokens sin firma, la clave deja de tener rol alguno — la autenticidad se declara, no se demuestra.

---

## 5. Variantes del ataque

- **`alg: None`, `none`, `NONE`, `nOnE`** — algunas librerías comparan el string de forma inconsistente; variar mayúsculas puede evadir el filtro.
- **Confusión de algoritmos (RS256 → HS256)** — ataque hermano: si el servidor firma con RSA (clave pública/privada) pero acepta `HS256`, el atacante firma el token con HMAC usando la **clave pública** (que es conocida) como secreto. Misma raíz: confiar en el `alg` del token.
- **Sin verificación de `alg` en absoluto** — librerías antiguas de varios lenguajes (versiones de `jwt-simple`, `pyjwt < 2.0` con ciertas configuraciones) aceptaban `none` por defecto.

---

## 6. Cómo se mitiga

1. **Rechazar explícitamente `alg: none`** — la defensa principal. La librería debe configurarse con una lista blanca de algoritmos permitidos:

   ```python
   # PyJWT — nunca usar algs=["none"] ni omitir el parámetro
   payload = jwt.decode(token, clave, algorithms=["HS256"])
   ```

2. **Fijar el algoritmo en el servidor, no leerlo del token** — el backend debe *exigir* su algoritmo configurado (ej.: siempre HS256 o siempre RS256) y rechazar cualquier header que difiera.

3. **Validar todos los claims** igualmente: `exp`, `iss`, `aud`, `sub` — la firma correcta no exime de validar el contenido.

4. **Mantener las librerías actualizadas** — las vulnerabilidades `none` conocidas (CVE-2015-9235 en `jwt-simple` de Node.js, entre otras) ya están parcheadas en versiones modernas.

5. **Verificación en auditoría:** tomar un token legítimo, forjarlo con `alg: none` y rol admin, enviarlo — si el servidor responde 200 con acceso privilegiado, la app es vulnerable.
