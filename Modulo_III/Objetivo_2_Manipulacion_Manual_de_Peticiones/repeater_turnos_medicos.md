# Objetivo 2: Manipulación manual de peticiones (Repeater) — Turnos Médicos

## Objetivo

Tomar una petición interceptada en el Objetivo 1, enviarla al módulo Repeater de Burp Suite, modificar manualmente un parámetro y observar la respuesta del servidor.

## Entorno

- **Herramienta:** Burp Suite Community Edition v2026.8
- **Aplicación:** Turnos Médicos (laboratorio de Software Seguro)
- **Petición base:** `GET /api/1/appointments/` (enviada a Repeater con "Send to Repeater" desde HTTP history)

## Prueba A — Alterar la cookie `session`

Se tomó la petición `GET /api/1/appointments/` con una sesión válida y se modificó manualmente el valor de la cookie `session`, agregando el sufijo `1233` al final del token.

![Repeater - cookie session alterada](img/Repeater_Cookie.png)

**Request modificado:**
```
GET /api/1/appointments/ HTTP/2
Host: chl-e4b38160-7774-46e4-9b53-b23acc5fa5e7-turnero.softwareseguro.com.ar
Cookie: session=<token original>...1233
```

**Respuesta del servidor:**
```
HTTP/2 302 Found
Location: /login?next=%2Fapi%2F1%2Fappointments%2F
Set-Cookie: session=<nuevo token vacío/reseteado>; HttpOnly; Path=/
```


## Prueba B — Modificar el User-Agent

Sobre la misma petición (con la cookie `session` original, válida), se modificó el header `User-Agent`, cambiándolo a:

```
User-Agent: Mozilla/5.0 (Prueba QA) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
```

![Repeater - User-Agent alterado](img/Repeater_UserAgent.png)

**Respuesta del servidor:**
```
HTTP/2 403 Forbidden
Cache-Control: private, max-age=0, no-store, no-cache, must-revalidate
X-Frame-Options: SAMEORIGIN
Report-To: {"group":"cf-nel", ...}
Nel: {"report_to":"cf-nel", ...}
```


