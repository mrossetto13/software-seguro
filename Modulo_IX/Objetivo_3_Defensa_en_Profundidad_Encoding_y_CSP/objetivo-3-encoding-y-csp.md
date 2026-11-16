# Objetivo 3: Defensa en Profundidad (Encoding y CSP)

## Escenario

¿Cómo puede un arquitecto de seguridad mitigar definitivamente el riesgo de XSS? Con **dos capas de defensa** que atacan el problema desde niveles distintos:

1. **Output Encoding** (codificación de salida) a nivel de código — evita que el dato se convierta en código.
2. **Content Security Policy (CSP)** a nivel de cabeceras HTTP — limita lo que puede ejecutarse aunque algo se cuele.

---

## Capa 1: Output Encoding (Codificación de Salida)

### El principio: separar datos de código en el momento exacto de la salida

El XSS existe porque un dato del usuario termina siendo interpretado como código HTML/JS. La codificación de salida ataca la raíz: **antes de escribir el dato en el documento, se transforma para que el navegador lo interprete siempre como texto, nunca como estructura**.

### La regla de oro

> No se filtra la *entrada* (el atacante puede enviar lo que quiera); se **codifica la salida** en el momento exacto en que el dato se escribe en el HTML.

### Por qué "contextual": cada contexto tiene su codificación

El mismo dato es inofensivo o letal según *dónde* se inserte. Por eso la codificación debe ser **consciente del contexto**:

| Contexto de salida | Codificación | Ejemplo |
|--------------------|--------------|---------|
| **Cuerpo HTML** (texto) | Entidades HTML: `<` → `&lt;`, `>` → `&gt;`, `&` → `&amp;` | `<div>` + dato + `</div>` |
| **Atributo HTML** | Entidades + comillas obligatorias: `"` → `&quot;`, `'` → `&#39;` | `<img alt="DATO">` |
| **JavaScript** | Codificación JS (\xHH, \uXXXX) — nunca concatenar a `innerHTML` | `<script>var x = 'DATO';</script>` |
| **URL** | URL-encoding (`%3C` para `<`) + validar esquema | `<a href="DATO">` |
| **CSS** | Escape CSS (raro; mejor no permitir CSS dinámico) | `style="..."` |

### Ejemplo del problema y la solución

**Sin codificar (vulnerable):**

```js
// El comentario del usuario: <img src=x onerror=robarCookies()>
element.innerHTML = "Comentario: " + userComment;
// El navegador interpreta el HTML del usuario → XSS
```

**Con codificación de salida:**

```js
// El dato se convierte en texto inerte:
element.textContent = "Hola, " + userComment;
// El navegador muestra literalmente: Hola, <img src=x onerror=...>
// Se VE el código, pero no se EJECUTA.
```

**Con codificación HTML de entidades (servidor):**

```js
// Entrada:  <img src=x onerror=alert(1)>
// Salida:   &lt;img src=x onerror=alert(1)&gt;
// El navegador lo muestra como texto: "<img src=x onerror=alert(1)>"
// Los caracteres < y > ya no pueden abrir etiquetas: son entidades.
```

### Cómo se implementa en la práctica

- **Frameworks con auto-escaping** (la forma recomendada): React, Angular, Vue, Django Templates, Jinja2, Blade... codifican automáticamente toda variable interpolada. El desarrollador solo debe cuidarse de los métodos "de escape" (`dangerouslySetInnerHTML`, `|safe`, `v-html`).
- **Codificadores manuales** cuando se construye HTML a mano: librerías como OWASP Java Encoder, `encodeForHTML()`, `encodeForJavaScript()`, etc.
- **Sanitización** (no codificación) solo si el usuario debe enviar HTML rico: DOMPurify elimina lo peligroso y conserva lo benigno.

### Por qué es la capa "definitiva" a nivel de código

La codificación de salida no adivina ataques (como las listas negras): **garantiza estructuralmente que el dato no puede convertirse en código**, igual que los prepared statements separan datos de SQL. El input del atacante puede llegar tal cual a la página, pero el navegador lo interpretará siempre como texto visible, nunca como marcado ejecutable.

---

## Capa 2: Content Security Policy (CSP)

### Qué es

Una **cabecera HTTP** que le dice al navegador qué recursos tiene permitido cargar y ejecutar la página. Es un control **del lado del navegador**, impuesto por el servidor:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'
```

### Qué hace exactamente

Con esa política, el navegador **rechaza**:

- Scripts inline (`<script>alert(1)</script>` y atributos `onclick=...`): no están en la lista de orígenes permitidos.
- `javascript:` pseudo-protocolo en enlaces e iframes.
- Carga de scripts desde dominios externos (CDNs maliciosos, servidores del atacante).
- `eval()` y similares (sin `unsafe-eval`).
- Objetos `<object>`/`<embed>` (vector clásico de plugins).

### El valor estratégico: mitigación de segundo nivel

La relación entre las dos capas es la clave de la defensa en profundidad:

> **Output encoding** evita que el dato del usuario se convierta en código.
> **CSP** garantiza que, si algo se cuela igual (un bug, un sink olvidado, una librería comprometida), **no pueda ejecutarse**.

Son independientes: si el desarrollador olvida codificar un campo, la CSP bloquea el script inline del atacante. Si el CSP tiene un hueco, la codificación ya neutralizó el payload. **Para que el XSS funcione, el atacante necesita vencer ambas capas a la vez.**

### Directivas clave de una CSP robusta

```
Content-Security-Policy:
  default-src 'self';        # todo desde el propio origen por defecto
  script-src 'self';         # sin inline, sin eval, sin dominios externos
  object-src 'none';         # sin <object>/<embed> (Flash/plugins)
  base-uri 'self';           # evita <base> inyectado
  frame-ancestors 'none';    # anti-clickjacking
  form-action 'self';        # a dónde pueden enviarse formularios
```

### Los errores que rompen el CSP

Un CSP mal configurado vale tanto como no tenerlo:

- **`script-src 'unsafe-inline'`**: permite exactamente el vector que se quería bloquear (scripts inline y atributos de evento). Anula la protección contra XSS.
- **`unsafe-eval`**: permite `eval()`, reabriendo sinks peligrosos.
- **Dominios externos permisivos** (`script-src 'self' cdn-cualquiera.com`): si el atacante puede cargar JS de un dominio externo que controle (o un JSONP en él), el CSP no sirve.
- **Nonces mal usados**: si el nonce se filtra o se reutiliza, la protección cae.

### Nonces y hashes: CSP con scripts inline legítimos

Cuando la app necesita scripts inline, en vez de `unsafe-inline` se usan:

```
script-src 'self' 'nonce-r4nd0m-por-respuesta'
```

```html
<script nonce="r4nd0m-por-respuesta">...</script>
```

El nonce se genera **por respuesta** (impredecible para el atacante) y solo ese bloque inline puede ejecutarse. Un payload inyectado no conoce el nonce → el navegador lo bloquea.

---

## 3. Cómo trabajan juntas las dos capas

Escenario: un comentario malicioso se cuela por un descuido del desarrollador.

```html
<!-- El atacante logró inyectar: -->
<img src=x onerror=robarCookies()>
```

**Sin defensas:** el navegador lo interpreta → XSS.

**Con Output Encoding:** el dato se imprime como texto inerte:
```html
&lt;img src=x onerror=robarCookies()&gt;
<!-- Se ve como texto, no se ejecuta nada -->
```

**Si además un sink del cliente (DOM XSS) logra insertar HTML crudo:**
- La CSP `script-src 'self'` bloquea la ejecución del handler inline (`onerror` es inline).
- El intento queda registrado en la consola y en los reportes (`report-uri`), dando visibilidad al equipo.

| Capa | Nivel | Qué hace | Qué detiene |
|------|-------|----------|-------------|
| **Output Encoding** | Código (servidor/cliente) | Convierte datos en texto inerte según contexto | El 99% de los XSS: el payload nunca se convierte en marcado activo |
| **CSP** | Cabecera HTTP (navegador) | Restringe qué scripts puede ejecutar la página | Lo que se cuele: inline, externos, pseudo-protocolos; y reporta intentos |

### El principio de defensa en profundidad

Ninguna capa es "infalible" por sí sola, pero juntas hacen que el ataque requiera **fallas simultáneas e independientes**:

- El atacante tendría que encontrar un punto donde el framework *no* codifique (un `dangerouslySetInnerHTML` mal usado) **y** que la CSP permita ejecutarlo (nonce filtrado, dominio externo permitido, `unsafe-inline`).

Eso convierte un XSS trivial en un hallazgo excepcional — que es exactamente lo que "mitigar definitivamente" significa en seguridad: **no confiar en una sola barrera**.
