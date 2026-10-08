# Objetivo 2: Análisis de la Tríada CIA

## 1. Los tres pilares, explicados en palabras simples

### Confidencialidad (Confidentiality)

Que la información solo la vea quien tiene derecho a verla. Un dato confidencial filtrado puede generar un daño es permanente. Se protege con autenticación, control de acceso y cifrado.

### Integridad (Integrity)

Que la información sea correcta, completa y no haya sido alterada sin autorización, ya sea por un atacante o por un error. Para esto, no importa solo que el dato exista, sino que sea confiable. Se protege con controles de acceso de escritura, validación de entradas, firmas digitales, hashes y registros de auditoría.

### Disponibilidad (Availability)

Que los sistemas y los datos estén accesibles cuando los usuarios legítimos los necesitan. Un sistema seguro pero caído no sirve. Se protege con redundancia, balanceo de carga, límites de tasa (rate limiting), backups y planes de recuperación.

## 2. Un ataque por cada pilar

Una misma vulnerabilidad puede afectar varios pilares (una inyección SQL, por ejemplo, puede leer, modificar y borrar datos). Por eso elegí un ejemplo cuyo impacto **principal y directo** recae sobre un solo pilar.

### 2.1 Confidencialidad: IDOR (Insecure Direct Object Reference)

**Qué es:** una falla de control de acceso (CWE-639) en la que la aplicación usa un identificador controlable por el usuario para acceder a un objeto y no verifica que ese objeto le pertenezca.

**Cómo ocurre:** en una aplicación de turnos médicos, el usuario autenticado consulta su turno con:

```
GET /api/appointments/1042
```

Si el servidor solo comprueba que hay una sesión válida, pero no que el turno 1042 pertenece a ese usuario, el atacante cambia el número (`1043`, `1044`, ...) y lee los turnos y datos de salud de otras personas. Es fácil de automatizar con una herramienta como Burp Intruder.

**Pilar afectado:** confidencialidad. Se **lee** información ajena sin autorización; no se modifica nada ni se tumba el servicio.

**Mitigación:** verificar en el servidor, en cada petición, que el recurso pertenece al usuario autenticado; usar identificadores no predecibles (UUID) como defensa adicional, no como único control; registrar y alertar sobre accesos anómalos.

### 2.2 Integridad: CSRF (Cross-Site Request Forgery)

**Qué es:** un ataque (CWE-352) que engaña al navegador de una víctima autenticada para que envíe una petición no deseada a un sitio en el que tiene sesión iniciada. El navegador adjunta automáticamente las cookies de sesión, de modo que el servidor cree que la acción es legítima.

**Cómo ocurre:** la víctima inicia sesión en su banco y luego abre una página maliciosa que contiene un formulario oculto como este:

```html
<form action="https://banco.ejemplo/transferir" method="POST" id="f">
  <input type="hidden" name="destino" value="CUENTA_DEL_ATACANTE">
  <input type="hidden" name="monto" value="50000">
</form>
<script>document.getElementById('f').submit();</script>
```

Si el banco no valida un token anti-CSRF, ejecuta la transferencia con la sesión de la víctima.

**Pilar afectado:** integridad. El ataque **modifica el estado** del sistema (saldo, contraseña, correo, datos de perfil) sin la voluntad del usuario. El atacante normalmente no ve la respuesta, así que no hay una fuga directa de información.

**Mitigación:** tokens anti-CSRF por sesión o formulario, cookies con atributo `SameSite=Lax` o `Strict`, verificación de los encabezados `Origin`/`Referer`, y reautenticación para acciones críticas.

### 2.3 Disponibilidad: DoS/DDoS por inundación HTTP (HTTP flood)

**Qué es:** un ataque de denegación de servicio (distribuido, cuando proviene de muchos orígenes) que satura la capacidad del servidor, la aplicación o el ancho de banda con un volumen enorme de peticiones aparentemente legítimas, por ejemplo miles de `GET /buscar?q=...` por segundo contra un endpoint costoso. Variantes de capa de aplicación como Slowloris mantienen conexiones abiertas a propósito para agotar los recursos del servidor.

**Cómo ocurre:** una red de dispositivos comprometidos (botnet) envía tráfico simultáneo. El servidor agota CPU, memoria, conexiones o ancho de banda, y los usuarios reales reciben errores `503` o tiempos de espera.

**Pilar afectado:** disponibilidad. No se roba ni se altera ningún dato: el objetivo es que el servicio **deje de responder**.

**Mitigación:** servicios anti-DDoS y CDN/WAF, limitación de tasa (rate limiting), timeouts y límites de conexiones por cliente, caché de respuestas costosas, escalado automático y redundancia geográfica.

## 3. Resumen comparativo

| Pilar | Pregunta que responde | Ataque de ejemplo | Qué le pasa a la información | Defensa clave |
|---|---|---|---|---|
| **Confidencialidad** | ¿Quién puede ver el dato? | IDOR | Se **lee** sin permiso | Control de acceso por objeto |
| **Integridad** | ¿El dato es fiable y no fue alterado? | CSRF | Se **modifica** sin consentimiento | Tokens anti-CSRF, `SameSite` |
| **Disponibilidad** | ¿Puedo acceder cuando lo necesito? | DDoS / HTTP flood | Se vuelve **inaccesible** | Rate limiting, WAF/CDN, redundancia |

