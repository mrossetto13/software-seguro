# Objetivo 1: El Arte de la Evasión de Filtros (WAF Bypass)

## Escenario

Un desarrollador intenta bloquear ataques XSS creando un filtro que **elimina automáticamente las etiquetas `<script>` y `</script>`** de los comentarios de los usuarios. Este documento explica técnicamente por qué esa defensa es inútil y presenta payloads alternativos que ejecutan código **sin usar la etiqueta `<script>`**.

---

## 1. Por qué esta defensa es inútil

### a) La etiqueta `<script>` no es la única forma de ejecutar JavaScript

El filtro ataca **una** de las decenas de formas de ejecutar código en un navegador. HTML ofrece múltiples vectores de ejecución:

- **Atributos de eventos** (`onerror`, `onload`, `onmouseover`, `onfocus`...): ejecutan JS sin ninguna etiqueta `<script>`.
- **Pseudo-protocolos** (`javascript:` en `href`, `src`, `action`, `formaction`...).
- **Etiquetas con comportamiento activo**: `<img>`, `<svg>`, `<iframe>`, `<body>`, `<input>`, `<video>`, `<details>`, `<math>`...
- **CSS y SVG** con capacidades de scripting en contextos específicos.

Eliminar `<script>` es como cerrar una puerta de un edificio con 50: el atacante simplemente usa otra.

### b) El filtro de eliminación crea un vector nuevo: la "recomposición"

Un filtro que **elimina** el string `<script>` (en vez de rechazar/rechazar la petición) es evadible por anidamiento:

```
Input del usuario:   <scr<script>ipt>alert(1)</scr</script>ipt>
Después del filtro:  <script>alert(1)</script>
```

El filtro elimina el `<script>` **interno**, y los fragmentos restantes se **recomponen** formando la etiqueta que se quería bloquear. Es el fallo clásico de sanitizar con una sola pasada de reemplazo.

### c) No hay contexto de salida (output encoding)

El error de fondo es el mismo que en SQLi: el desarrollador intenta **limpiar el dato** en vez de **separar datos de código**. La defensa real contra XSS es la **codificación contextual al escribir en el HTML** (HTML entity encoding en texto, atributos, JS, URLs) y una **Content Security Policy** — no filtrar patrones en la entrada.

---

## 2. Payloads que ejecutan código sin `<script>`

### Payload 1: Atributo de evento en una imagen

```html
<img src=x onerror=alert(document.cookie)>
```

**Cómo funciona:**

- `<img src=x>` intenta cargar una imagen desde la ruta relativa `x`, que **no existe**.
- El navegador dispara el evento `error` de la imagen.
- El atributo `onerror` ejecuta el JavaScript: `alert(document.cookie)`.
- **No hay etiqueta `<script>` en ninguna parte** — el vector completo es una etiqueta de imagen con un manejador de eventos.

Si el filtro bloqueara también `onerror`, existen decenas de eventos equivalentes: `onload`, `onfocus` (con `autofocus`), `onmouseover`, `ontoggle` (en `<details>`), `onanimationstart`...

Variantes del mismo vector:

```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<video src=x onerror=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
```

### Payload 2: Pseudo-protocolo `javascript:`

```html
<a href="javascript:alert(document.domain)">Ganaste un premio</a>
```

**Cómo funciona:**

- El atributo `href` acepta el pseudo-protocolo `javascript:`, que ejecuta el código al hacer clic.
- No hay etiqueta `<script>` ni atributo de evento: es un enlace completamente válido para el navegador.
- Variante sin interacción del usuario (auto-ejecución):

```html
<iframe src="javascript:alert(1)">
<form action="javascript:alert(1)"><button>Enviar</button></iframe>
<object data="javascript:alert(1)">
```

Y con ofuscación para evadir filtros que busquen el literal `javascript:`:

```html
<a href="jav&#x09;ascript:alert(1)">   <!-- entidad HTML + tab -->
<a href="&#106;avascript:alert(1)">    <!-- la 'j' como entidad numérica -->
```

El navegador **decodifica las entidades HTML antes de interpretar la URL**, por lo que `&#106;avascript:` se convierte en `javascript:` en el momento de la ejecución — después de que el filtro ya pasó.

### Payload 3 (bonus): SVG con script embebido

```html
<svg onload=alert(1)>
<svg><animate onbegin=alert(1) attributeName=x dur=1s>
```

SVG es un documento XML completo dentro del HTML: trae sus propios eventos (`onbegin`, `ontoggle`) y elementos animables que disparan eventos sin interacción.

---

## 3. Por qué el enfoque "filtrar patrones" está condenado

| Problema | Explicación |
|----------|-------------|
| **Superficie de ataque enorme** | Cientos de etiquetas y atributos de evento; el filtro cubre 1-2 |
| **Recomposición** | Eliminar en una pasada permite `<scr<script>ipt>` → `<script>` |
| **Codificación del navegador** | El navegador decodifica entidades (`&#106;`) *después* del filtro del servidor |
| **Contexto de salida ignorado** | El mismo dato es inofensivo en un atributo y letal en otro; filtrar "en la entrada" no puede saberlo |
| **Mantenimiento infinito** | Cada nuevo vector (nuevas etiquetas HTML, nuevos eventos) requiere actualizar el filtro |

Es exactamente el mismo patrón de las listas negras en SQLi (Objetivo 3 de la serie anterior): **intentar adivinar todos los ataques es una carrera perdida**.

---

## 4. Cómo se defiende correctamente

1. **Codificación contextual en la salida** (la defensa principal): convertir `<`, `>`, `&`, `"`, `'` a entidades HTML *según el contexto* donde se imprime el dato (texto, atributo, JS, URL). Los frameworks modernos (React, Angular, Django templates, Jinja) lo hacen por defecto.
2. **Content Security Policy (CSP)**: cabecera como `Content-Security-Policy: default-src 'self'; script-src 'self'` que bloquea la ejecución de scripts inline y de pseudo-protocolos, mitigando el impacto aunque algo se cuele.
3. **Listas blancas de HTML** si se permite formato en comentarios: usar una librería de sanitización probada (DOMPurify en el cliente, sanitizadores del framework en el servidor) — nunca un filtro casero con `replace()`.
4. **Atributos `HttpOnly` en cookies** de sesión: aunque haya XSS, el JavaScript no puede leer la cookie.
5. **Validación de entrada como capa extra** (no como única): restringir longitud, formato y caracteres esperados.

---

## 4. Resumen

> **Conclusión:** el filtro que elimina `<script>` es inútil por tres razones: (1) existen decenas de vectores de ejecución sin esa etiqueta (atributos de evento, pseudo-protocolos, SVG); (2) un filtro de eliminación por reemplazo es recomponible (`<scr<script>ipt>`); y (3) el navegador decodifica y ejecuta *después* del filtro, con reglas que el filtro no modela. La defensa correcta no es filtrar la entrada con patrones, sino **codificar la salida contextualmente** y aplicar CSP — el mismo principio de toda la serie: los filtros adivinan, la arquitectura correcta elimina el problema.
