# Objetivo 1: Anatomía de un Ataque CSRF (Cross-Site Request Forgery)

## Escenario

Este documento explica el flujo exacto de un ataque CSRF exitoso, por qué ocurre el fallo (con énfasis en el comportamiento automático de los navegadores con las cookies), y cómo el atributo de cookie **SameSite** (`Lax` y `Strict`) mitiga radicalmente este vector.

---

## 1. La idea central del ataque

**CSRF (Cross-Site Request Forgery)** = "falsificación de petición entre sitios". El atacante **no roba** la sesión de la víctima: la **usa**. Engaña al navegador de la víctima para que envíe una petición al sitio donde ella ya está autenticada, y el servidor la procesa como si fuera una acción legítima de la víctima.

La víctima no hace clic en "transferir dinero". Hace clic en un enlace inofensivo (o ni siquiera eso) y su propio navegador, **sin que ella sepa**, envía la transferencia.

---

## 2. La causa raíz: el comportamiento automático de las cookies

### Cómo funciona la autenticación web

Cuando un usuario inicia sesión en `banco.com`, el servidor le entrega una **cookie de sesión**. A partir de ese momento, el navegador la adjunta **automáticamente** a **cada petición** que hace hacia `banco.com` — sin que el usuario ni la página hagan nada.

Ese comportamiento es lo que hace que la sesión "funcione": no tenés que loguearte en cada clic. Pero tiene una consecuencia crítica:

> **El navegador no distingue de dónde proviene la petición.** Si una petición hacia `banco.com` se origina en `banco.com` o en `malo.com`, el navegador la envía igual: con la cookie de sesión incluida.

### El modelo de seguridad del navegador: Same-Origin Policy (y su límite)

La **Same-Origin Policy (SOP)** impide que JavaScript de `malo.com` **lea** la respuesta de `banco.com` (protege la *lectura*). Pero **no impide que se *envíe* la petición**. El navegador permite:

- Cargar recursos cross-origin: `<img src="https://banco.com/...">`, `<form action="https://banco.com/...">`, `<iframe>`, `<script>`, fetch con `no-cors`...
- Y en todos esos casos, **adjunta las cookies** de `banco.com` automáticamente.

Es decir: la SOP protege que el atacante *vea* la respuesta, pero el ataque CSRF no necesita verla — solo necesita que la petición **se ejecute** en el servidor.

### La fórmula del CSRF

```
Autenticación por cookie automática + petición cross-origin permitida
+ servidor que no verifica el origen = CSRF
```

El servidor recibe una petición con credenciales válidas y **asume que la inició el usuario conscientemente**. Esa asunción es la vulnerabilidad.

---

## 3. El flujo exacto de un ataque CSRF exitoso

### Paso a paso

1. **La víctima está autenticada.** Inició sesión en `https://banco.com`; su navegador guarda la cookie de sesión (`sessionid=abc123`).

2. **La víctima visita el sitio del atacante.** Por phishing, un comentario con enlace, un anuncio, o una pestaña abierta en segundo plano, navega a `https://malo.com/premio.html`.

3. **La página del atacante contiene una petición oculta hacia el banco.** Por ejemplo:

   ```html
   <!-- Opción A: formulario auto-enviado -->
   <form id="f" action="https://banco.com/transferir" method="POST">
     <input type="hidden" name="destino" value="atacante-cuenta">
     <input type="hidden" name="monto" value="10000">
   </form>
   <script>document.getElementById("f").submit();</script>

   <!-- Opción B: para endpoints GET mal diseñados, ni siquiera hace falta JS -->
   <img src="https://banco.com/transferir?destino=atacante&monto=10000" width="0" height="0">
   ```

4. **El navegador de la víctima envía la petición.** Al procesar el formulario (o la imagen), el navegador hace la petición hacia `banco.com` y — comportamiento automático — **adjunta la cookie de sesión de la víctima**.

5. **El servidor procesa la petición como legítima.** Ve `sessionid=abc123` → sesión válida → ejecuta la transferencia de $10.000 a la cuenta del atacante.

6. **La víctima no nota nada.** La respuesta (si el atacante puede leerla, no; pero no importa) va al contexto de `malo.com` o se descarta. El banco muestra la transferencia en el historial; la víctima nunca la autorizó.

### Diagrama del flujo

```
Víctima                    Navegador                     malo.com              banco.com
   │  (1) ya logueada          │                             │                     │
   │  cookie: sessionid=abc123 │                             │                     │
   │                           │                             │                     │
   │──(2) visita "premio"──────┼────────────────────────────>│                     │
   │                           │<──(3) HTML con form oculto──│                     │
   │                           │                             │                     │
   │                           │──(4) POST /transferir ──────────────────────────>│
   │                           │   Cookie: sessionid=abc123  │                     │
   │                           │   (¡adjuntada automáticamente!)                   │
   │                           │                             │                     │
   │                           │<──(5) 200 OK: transferencia ejecutada ───────────│
   │                           │                             │                     │
```

### Requisitos para que funcione

- La víctima tiene **sesión activa** en el sitio objetivo (cookie vigente).
- La acción sensible es **predecible**: endpoint y parámetros conocidos o adivinables (no requiere tokens únicos).
- El servidor **no verifica el origen** de la petición (no hay token anti-CSRF, no hay chequeo de `Origin`/`Referer`, o la cookie no tiene `SameSite`).
- La acción es ejecutable con una petición simple (GET o POST con parámetros simples).

---

## 4. La mitigación: el atributo de cookie `SameSite`

### Qué hace

`SameSite` es un atributo que el servidor puede fijar al emitir la cookie, que le ordena al navegador **cuándo adjuntarla en peticiones cross-site**:

```
Set-Cookie: sessionid=abc123; Secure; HttpOnly; SameSite=Lax
```

El navegador clasifica cada petición como **same-site** (mismo sitio que la página actual) o **cross-site** (sitio distinto), y aplica la regla:

| Valor | Peticiones same-site | Peticiones cross-site |
|-------|---------------------|----------------------|
| `SameSite=Strict` | ✅ Se envía la cookie | ❌ **Nunca** se envía la cookie |
| `SameSite=Lax` | ✅ Se envía la cookie | ⚠️ Solo en navegación de nivel superior **GET** (clic en enlace, redirección) |
| `SameSite=None` | ✅ Se envía | ✅ Se envía (requiere `Secure`) — el comportamiento clásico vulnerable |

### `SameSite=Strict`: máxima protección

La cookie **jamás** se envía en peticiones cross-site. Si la víctima viene desde `malo.com` (aunque sea con un simple clic en un enlace), el navegador no adjunta la cookie → el servidor ve una petición **no autenticada** → el ataque falla.

**Costo:** también rompe casos legítimos. Si un usuario recibe un enlace legítimo por email hacia `banco.com`, al llegar no estará logueado (la cookie no viajó) y deberá iniciar sesión de nuevo. Por eso Strict se usa en acciones críticas o cuando la UX lo tolera.

### `SameSite=Lax`: el equilibrio práctico

Permite la cookie solo en **navegación de nivel superior con método GET**: cuando el usuario *clickea un enlace* y el navegador navega a la página. En ese caso la cookie se envía (para que el enlace funcione y el usuario vea su sesión).

Pero bloquea todo lo demás cross-site:

- ❌ `POST` cross-site (el formulario del atacante) → **sin cookie** → CSRF bloqueado.
- ❌ Cargas de recursos cross-site (`<img>`, `<iframe>`, fetch/XHR) → **sin cookie**.
- ✅ Solo el GET de navegación por clic → con cookie (y un GET no debería tener efectos de todos modos).

**Por qué Lax "radicalmente" mitiga:** la gran mayoría de los ataques CSRF usan POST auto-enviado o GET embebido en recursos (`<img>`). Con Lax, ambos llegan sin credenciales. Y como los cambios de estado deberían ser POST/PUT/DELETE (nunca GET), el único caso que Lax permite (navegación GET) es por definición inofensivo si la app sigue la semántica HTTP correcta.

### El detalle que salva a Lax: la navegación top-level

Un `POST` desde `malo.com` no es "navegación de nivel superior" iniciada por el usuario: es una carga de recurso embebida. El navegador la clasifica como cross-site y **omite la cookie**. Solo un clic real del usuario en un enlace (que produce una navegación GET completa) recibe la cookie.

### Nota: el default de los navegadores modernos

Chrome y otros navegadores hoy aplican **`Lax` por defecto** a las cookies sin atributo `SameSite` ("Lax by default"). Esto mitigó masivamente el CSRF a nivel ecosistema. Pero no elimina la necesidad de defensas en la aplicación:

- Cookies legacy con `SameSite=None` explícito siguen siendo vulnerables.
- Existen variantes de bypass de Lax (ej. peticiones GET con efectos secundarios, métodos "seguros" con efectos, subdominios).
- `SameSite` es una capa; el token anti-CSRF y la verificación de origen siguen siendo necesarios (defensa en profundidad).

---

## 5. Las tres capas de defensa contra CSRF

| Capa | Mecanismo | Qué hace |
|------|-----------|----------|
| **SameSite (Lax/Strict)** | Atributo de cookie | El navegador no adjunta credenciales en peticiones cross-site |
| **Token anti-CSRF** | Código del servidor | El servidor solo acepta acciones si la petición lleva un token aleatorio que solo la página legítima conoce |
| **Verificación de Origin/Referer** | Servidor | Rechazar peticiones cuyo header `Origin` no sea el dominio legítimo |

El **token anti-CSRF** merece mención: el servidor incluye un valor aleatorio en cada formulario (`<input type="hidden" name="csrf" value="...">`) y lo valida al recibirlo. El sitio del atacante **no puede leerlo** (la SOP le impide leer respuestas de `banco.com`), así que su formulario oculto no puede incluirlo → el servidor rechaza la petición. Nota cómo la SOP, que era el "límite" que permitía el CSRF, aquí se convierte en el aliado: protege la *lectura* del token.
