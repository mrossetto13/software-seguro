# Objetivo 4: El Punto Ciego de los Escáneres Automatizados

## Escenario

Herramientas como Nessus, Acunetix o el escáner activo de Burp Suite son excelentes encontrando fallas técnicas (XSS, configuraciones inseguras, versiones vulnerables), pero **fracasan rotundamente** al buscar fallas lógicas. ¿Por qué la lógica de negocio es el mayor punto ciego de la automatización, y por qué la auditoría manual es indispensable?

---

## 1. Qué hacen bien los escáneres automatizados

Los escáneres detectan fallas que comparten tres características:

1. **Son genéricas**: el XSS, el SQLi o un header de seguridad faltante se manifiestan igual en cualquier aplicación.
2. **Tienen un patrón detectable**: se puede enviar un payload (`<script>alert(1)</script>`, `' OR 1=1--`) y **observar una respuesta anómala** que confirma la vulnerabilidad.
3. **No requieren entender el propósito del sistema**: un XSS en un formulario de contacto es un XSS sin importar si la web vende zapatos o gestiona hospitales.

Por eso funcionan: comparan el comportamiento observado contra una **base de datos de firmas conocidas** y patrones de respuesta.

---

## 2. Por qué la lógica de negocio es invisible para la automatización

### a) No existe "firma" de una falla lógica

Una cantidad negativa en el carrito, un cupón canjeado 50 veces o un cambio de precio en la petición **no generan errores, stack traces ni respuestas anómalas**. El servidor responde `200 OK` en todos los casos. La falla no está en *cómo* responde el servidor sino en *qué significa* esa respuesta en el dominio del negocio.

> Un escáner ve: petición → 200 OK → "todo bien".
> Un auditor ve: 200 OK → "acaba de acreditar saldo sin pagar".

### b) Requiere entender la intención del sistema

Detectar que `quantity: -5` es un ataque exige saber que **una cantidad negativa no tiene sentido en un carrito de compras**. Ese conocimiento no está en ninguna base de datos de firmas: es conocimiento del dominio. El escáner no sabe qué es un "carrito", un "saldo" o un "cupón de un solo uso".

### c) Cada aplicación tiene reglas únicas

Las fallas técnicas son universales; las lógicas son **específicas de cada negocio**:

- ¿Puede un usuario cancelar un pedido ya enviado?
- ¿Un descuento del 100% es válido en este contexto?
- ¿Puede un estudiante ver las notas de otro estudiante si cambia el ID de la URL?
- ¿El flujo de pago puede saltearse un paso?

No existe un payload genérico para "violar la regla de negocio X" porque **el atacante (y el auditor) primero debe descubrir cuál es la regla**.

### d) Las fallas lógicas son secuencias, no peticiones aisladas

Un escáner prueba peticiones individuales. Las fallas lógicas suelen requerir **encadenar pasos en un orden específico**: crear un carrito → aplicar cupón → modificar cantidad → cancelar un ítem → observar el estado final del saldo. La explotación es un *proceso*, no un disparo único.

### e) El "éxito" del ataque parece tráfico legítimo

Un atacante que explota una falla lógica usa los mismos endpoints, los mismos métodos y respuestas válidas que un usuario normal. Desde el punto de vista de un sistema automatizado, **es indistinguible del tráfico legítimo**. No hay anomalía técnica que detectar: hay una anomalía *semántica*.

---

## 3. Por qué la auditoría manual es indispensable

### Lo que aporta el humano

| Capacidad | Ejemplo aplicado |
|-----------|------------------|
| **Entender el dominio** | Saber que un saldo nunca puede ser negativo |
| **Formular hipótesis** | "Si el cupón se valida antes de marcarlo, quizás haya una race condition" |
| **Encadenar pasos** | Combinar manipulación de parámetros + concurrencia + flujos alternativos |
| **Interpretar contexto** | Distinguir un `200 OK` benigno de uno que implica un estado inválido |
| **Creatividad** | Preguntar "¿qué pasaría si...?" fuera de los patrones esperados |

### El flujo de una auditoría manual de lógica

1. **Mapear la lógica de negocio**: qué reglas gobiernan el sistema (precios, roles, límites, flujos).
2. **Identificar supuestos del desarrollador**: "el cliente nunca enviará un precio", "el cupón se marca antes de responder", "el ID de la URL siempre pertenece al usuario".
3. **Atacar los supuestos**: violar cada supuesto deliberadamente (valores negativos, IDs ajenos, peticiones paralelas, saltarse pasos).
4. **Verificar el impacto en el estado**: no mirar la respuesta HTTP, sino el **estado resultante** (saldo, inventario, permisos).

### La división correcta del trabajo

No se trata de descartar los escáneres, sino de usar cada herramienta donde es fuerte:

- **Escáneres automatizados** → primera pasada: fallas técnicas, superficie de ataque, configuraciones, dependencias vulnerables. Rápidos, baratos, repetibles.
- **Auditoría manual** → segunda capa: lógica de negocio, autorización, flujos transaccionales, concurrencia. Requiere criterio, contexto y tiempo.

