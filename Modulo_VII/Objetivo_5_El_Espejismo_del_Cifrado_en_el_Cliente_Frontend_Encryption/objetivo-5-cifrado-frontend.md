# Objetivo 5: El Espejismo del Cifrado en el Cliente (Frontend Encryption)

## Escenario

Una aplicación financiera cifra todo el body de las peticiones POST y los parámetros sensibles **mediante JavaScript en el navegador** antes de enviarlos al backend. Los arquitectos afirman que, gracias a esto, es imposible que un atacante intercepte y manipule los valores con un proxy como Burp Suite, porque "todo viaja encriptado".

**Esa afirmación es una falsa sensación de seguridad.** Este documento analiza por qué.

---

## 1. El fallo fundamental de diseño

### El principio violado: el cliente nunca es confiable

> **El atacante controla por completo el entorno del cliente.**

El cifrado se ejecuta en el navegador de la víctima... pero también en el navegador del atacante. El JavaScript que cifra, las claves que usa y los datos en claro **antes** del cifrado están todos presentes en la máquina del atacante cuando él mismo usa la aplicación.

La afirmación de los arquitectos confunde dos amenazas distintas:

| Amenaza | ¿Qué protege el cifrado en el cliente? |
|---------|----------------------------------------|
| **Observador pasivo en la red** (sniffer) | ✅ Sí lo protege (aunque TLS ya lo hace) |
| **El propio usuario / atacante con control del cliente** | ❌ No lo protege en absoluto |

El cifrado en el cliente protege los datos **en tránsito**, pero el atacante de este escenario no está en la red: **está en el extremo**. Es el dueño del navegador, y por lo tanto:

- Tiene el código JavaScript que cifra (descargado públicamente).
- Tiene las claves o material de cifrado que ese código usa.
- Tiene los datos **en claro** justo antes de cifrarse.
- Puede modificar el propio código JavaScript antes de que se ejecute.

### La analogía

Es como enviar una carta en una caja fuerte cuyo manual de instrucciones, la llave y la carta sin cerrar viajan dentro del mismo paquete. Cualquiera que reciba el paquete puede abrir la caja fuerte, leer la carta, cambiarla y volver a cerrar la caja.

### El error conceptual: cifrado ≠ integridad de la lógica

Aunque el cifrado fuera perfecto (AES-256, RSA, lo que sea), solo garantiza **confidencialidad en el canal**. No garantiza que:

- el remitente sea quien dice ser (más allá de la sesión),
- los datos cifrados sean los "legítimos",
- el flujo de la aplicación no pueda manipularse.

El servidor, al recibir un body cifrado, **no puede distinguir** entre uno generado por su propio JavaScript y uno generado por el atacante con el mismo algoritmo y las mismas claves.

---

## 2. Pasos tácticos del atacante para evadir la protección

### Paso 1: Obtener el código de cifrado

El JavaScript que cifra es un asset público: se descarga con la página (o en bundles `.js` referenciados por el HTML). El atacante simplemente lo lee:

- DevTools → pestaña **Sources** → buscar funciones como `encrypt(`, `AES`, `RSA`, `JSEncrypt`, `CryptoJS`.
- O descargar los bundles y analizarlos localmente.

### Paso 2: Encontrar las claves o el material de cifrado

Casos típicos (todos frecuentes en implementaciones reales):

- **Clave hardcodeada** en el JavaScript (la más común): visible en el bundle.
- **Clave derivada de datos públicos** (un salt fijo, un ID de sesión): reproducible.
- **Clave intercambiada con RSA pero con la privada "escondida"** en el cliente: también visible.
- **Clave recibida del servidor en una respuesta previa**: visible en el historial de red.

Si el esquema usa RSA para envolver una clave AES, ambas operaciones ocurren en el cliente y son replicables.

### Paso 3: Interceptar con Burp Suite

Con el proxy de Burp, el atacante ve las peticiones con el body cifrado:

```
POST /api/transfer HTTP/1.1
Content-Type: application/json

{"data":"U2FsdGVkX1+9z8kQ7..."}
```

No puede leerlo todavía, pero **eso no importa**: no necesita descifrarlo en tránsito.

### Paso 4: La evasión — tres caminos posibles

#### Camino A: Cifrar él mismo (el más directo)

1. Copia el algoritmo y la clave del JavaScript.
2. Construye su propio payload en claro: `{"amount": 999999, "to": "attacker"}`.
3. Lo cifra con el mismo esquema.
4. Lo envía al servidor desde Burp. El servidor lo descifra y procesa **como si fuera legítimo**.

El cifrado deja de ser una barrera: es solo un paso más del protocolo, y el atacante lo implementa.

#### Camino B: Manipular antes del cifrado (hooking del navegador)

Usar un proxy de interceptación de JavaScript o puntos de ruptura:

1. DevTools → **Sources** → poner un **breakpoint** en la función que cifra (o usar "XHR/fetch breakpoints").
2. Cuando la app se detiene justo antes de cifrar, **modificar la variable** con los datos en claro (ej. cambiar el monto).
3. Dejar continuar: la app misma cifra y envía el valor manipulado.

Herramientas: breakpoints en DevTools, extensiones como Requestly, o proxies con inyección de scripts (Mitmproxy con scripts de manipulación del DOM/JS).

#### Camino C: Desactivar el cifrado por completo

Modificar el código en el navegador:

1. En DevTools → Sources, localizar la función de cifrado.
2. **Editarla en vivo** ("Edit and save" con overrides locales) para que devuelva los datos en claro, o que use una clave propia.
3. O usar un proxy que reescriba las respuestas JS del servidor (Burp Match & Replace sobre los bundles) para servir una versión "neutra" del script.

En todos los casos, el atacante opera **dentro del navegador**, donde la aplicación no tiene ningún control.

### Paso 5: Enviar valores manipulados

Con cualquiera de los caminos, el atacante termina enviando al servidor un body **cifrado correctamente** pero con contenido malicioso. El backend lo descifra, confía en él y lo procesa. La protección no solo no detuvo el ataque: **dio a los arquitectos una falsa confianza** que probablemente hizo que omitieran validaciones del lado del servidor.

---

## 3. El daño colateral más grave

El peor efecto de este diseño no es técnico sino organizacional:

- El equipo cree que "los datos viajan encriptados" y **relaja las validaciones del servidor** ("el body siempre viene de nuestro JS").
- Cuando el cifrado del cliente se rompe (y se rompe siempre), el backend queda **desnudo**: sin validación de entrada, sin control de estado, sin límites de negocio.
- Es el mismo patrón de los Objetivos anteriores: **mover la confianza al cliente** — la zona que el atacante controla por completo.

---

## 4. Cómo debería diseñarse correctamente

1. **TLS (HTTPS)** para confidencialidad e integridad en tránsito — es su trabajo, y lo hace bien.
2. **Validación estricta en el servidor** de todo dato, venga cifrado o no: tipos, rangos, reglas de negocio (como en los Objetivos 2 y 3).
3. **El servidor nunca debe inferir seguridad del formato del mensaje.** Un body cifrado por el cliente no es más confiable que uno en claro: ambos los puede generar el atacante.
4. **Firmar operaciones críticas del lado del servidor** (nonce, HMAC de sesión) si se quiere protección extra contra replay — pero siempre con la lógica de validación en el backend.
5. El cifrado en el cliente solo tiene sentido para casos como **cifrado de contraseñas antes del hash del servidor** en contextos muy específicos, y aun así **nunca sustituye** TLS ni validación.

---

## 5. Resumen

| Creencia de los arquitectos | Realidad |
|------------------------------|----------|
| "Viaja encriptado, es imposible manipularlo" | El atacante tiene el código, las claves y los datos en claro en su propio navegador |
| El proxy no puede leer los datos | No necesita leerlos en tránsito: cifra él mismo o manipula antes del cifrado |
| El servidor puede confiar en el body cifrado | El servidor no puede distinguir cifrado legítimo de cifrado del atacante |
| La protección añade seguridad | Añade complejidad y, peor, **justifica omitir validaciones del backend** |

> **Conclusión:** el fallo fundamental es de arquitectura de confianza: se colocó el mecanismo de seguridad en el único entorno que el atacante controla por completo. El cifrado en el cliente puede ser, en el mejor de los casos, una capa de ofuscación; nunca un control de seguridad. La regla sigue siendo la misma de todos los objetivos anteriores: **la única zona confiable es el servidor, y toda validación crítica debe vivir ahí.**
