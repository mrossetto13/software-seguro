# Fuerza bruta vs. ataque de diccionario en formularios de login

## 1. Fuerza bruta tradicional

El atacante prueba combinaciones de caracteres de forma **sistemática y exhaustiva**, sin importar si tienen sentido o no:

```
a, b, c, ..., z, aa, ab, ac, ..., zz, aaa, ...
```

- **Cobertura:** total. En teoría siempre encuentra la contraseña.
- **Costo:** crece de forma exponencial con la longitud y la variedad de caracteres (minúsculas, mayúsculas, números, símbolos).
- **Limitación práctica:** contra un login web, que responde lento, limita intentos y puede bloquear, se vuelve inviable frente a contraseñas largas y aleatorias.

## 2. Ataque de diccionario

En lugar de recorrer todo el espacio de posibilidades, el atacante usa una **lista acotada de candidatos probables** según el comportamiento humano:

- Contraseñas filtradas en brechas anteriores.
- Palabras comunes, nombres, fechas, equipos de fútbol.
- Variaciones típicas: `Contraseña2024!`, `m4rtin123`, `Password1`.

Se apoya en que las personas eligen claves **predecibles** y las **reutilizan**.

- **Cobertura:** parcial; falla si la clave es realmente aleatoria o no está en la lista.
- **Costo:** bajo; muy rápido y eficiente.

## 3. Diferencia fundamental

| Aspecto | Fuerza bruta | Diccionario |
|---|---|---|
| Estrategia | Explora **todas** las combinaciones posibles | Explora solo las **más probables** |
| Cobertura | Total (en teoría) | Parcial |
| Tiempo / costo | Muy alto, exponencial | Bajo |
| Depende de | Poder de cómputo y cantidad de intentos permitidos | Calidad de la lista y de la predictibilidad del usuario |
| Efectividad en login web | Baja (pocos intentos disponibles) | Alta (se "gastan" bien los intentos) |

## 4. Mecanismos de defensa

Un desarrollador puede combinar varias capas:

### Rate limiting y bloqueo temporal
Limitar los intentos fallidos por usuario y por IP, con bloqueos progresivos o retrasos crecientes (*backoff* exponencial).

### CAPTCHA o desafíos adaptativos
Se activan tras varios intentos fallidos o ante comportamiento sospechoso, para frenar la automatización.

### Autenticación multifactor (MFA)
Aunque la contraseña sea descubierta, el atacante no puede completar el acceso sin el segundo factor.

### Políticas de contraseñas sensatas
- Exigir una longitud mínima.
- Rechazar claves comunes o ya filtradas (por ejemplo, validando contra listas como *Have I Been Pwned*), lo que neutraliza el ataque de diccionario en origen.

### Monitoreo y alertas
Detectar patrones como:
- Muchos fallos desde una misma IP.
- Muchos usuarios distintos probados con la misma contraseña (*password spraying*).

### Mensajes de error genéricos
Responder siempre algo como *"Usuario o contraseña incorrectos"*, para no revelar qué usuarios existen (evita la enumeración de usuarios).

### Almacenamiento seguro de contraseñas
Usar algoritmos de hash lentos y con sal (bcrypt, scrypt, Argon2) para que, si se filtra la base, el ataque offline sea costoso.
