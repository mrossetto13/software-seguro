# Análisis de Token JWT

## Token analizado

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MzM5LCJyb2xlIjoiU1RVREVOVCIsInR2IjowLCJpYXQiOjE3OTQ4NTg5ODEsImV4cCI6MTc5NDg3Nzc4MX0.n5O1ad_efov9iFBFkWL_AVSGeXaPVcoZzen7G8bSe5s
```

---

## 1. Estructura técnica: Algoritmo de firma (Header)

El Header decodificado es:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- **Algoritmo de firma: `HS256` (HMAC-SHA256)**
  - Es un algoritmo de **clave simétrica**: la misma clave secreta se usa tanto para **firmar** el token en el servidor como para **verificar** la firma.
  - La firma se calcula sobre las partes `header.payload` y viaja en la tercera sección del token (Base64URL).
  - **Implicancia de seguridad:** el secreto nunca debe salir del servidor. Si se filtra, cualquier persona podría **falsificar tokens válidos** (por ejemplo, cambiar el rol de `STUDENT` a `ADMIN`).
  - A diferencia de algoritmos asimétricos (como `RS256`), aquí no es posible verificar la firma sin poseer el secreto completo.

---

## 2. Datos sensibles y roles en el Payload (texto plano)

El Payload decodificado es:

```json
{
  "id": 339,
  "role": "STUDENT",
  "tv": 0,
  "iat": 1794858981,
  "exp": 1794887781
}
```

| Campo | Valor | Descripción |
|-------|-------|-------------|
| `id` | `339` | Identificador numérico del usuario en la base de datos |
| `role` | `STUDENT` | **Rol del usuario enviado en texto plano** |
| `tv` | `0` | Probablemente "token version", contador para invalidar tokens antiguos |
| `iat` | `1794858981` | Timestamp de emisión (issued at) |
| `exp` | `1794887781` | Timestamp de expiración (≈ 8 horas después de la emisión) |

### Datos sensibles expuestos

- **`role: "STUDENT"`** — El rol viaja **en texto plano** dentro del payload. Cualquiera que capture o copie el token puede leerlo (Base64 **no es encriptación**, solo codificación).
- **`id: 339`** — Expone el identificador interno del usuario, información útil para enumerar usuarios o atacar endpoints por ID.

### Punto clave

> Un JWT **no está encriptado, solo está firmado**. La firma (HS256) garantiza **integridad y autenticidad** (que nadie lo modificó), pero **no confidencialidad**: el contenido del header y del payload es legible por cualquiera con acceso al token.

### Riesgo asociado

Si el backend **confiara ciegamente en el campo `role` del payload sin validar la firma**, un atacante podría modificar el token. Sin embargo, con la firma HS256 bien implementada, cualquier alteración del payload invalida el token, ya que el atacante no conoce el secreto para regenerar la firma válida.
