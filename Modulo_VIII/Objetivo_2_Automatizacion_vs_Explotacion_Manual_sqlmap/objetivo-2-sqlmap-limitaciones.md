# Objetivo 2: Automatización vs Explotación Manual (sqlmap)

## Escenario

sqlmap es la herramienta de explotación de inyecciones SQL más potente del mercado, pero depender exclusivamente de ella es un error de novatos. Este documento enumera y explica escenarios técnicos del mundo real donde sqlmap **falla estrepitosamente**, pero un atacante manual (o un pentester manipulando la petición en Burp Suite) **sí logra explotar** la vulnerabilidad.

---

## 1. Por qué sqlmap falla: la naturaleza de la automatización

sqlmap es un motor de **patrones y respuestas esperadas**. Envía payloads predefinidos, mide la respuesta y decide si hay inyección según firmas conocidas: errores de SQL visibles, diferencias booleanas medibles, retardos temporales consistentes.

Eso significa que falla siempre que la vulnerabilidad **no encaje en sus patrones**, aunque la inyección exista y sea explotable a mano. Los escenarios siguientes son casos reales y frecuentes.

---

## 2. Escenario 1: Inyección en segundo plano (Second-Order SQLi)

### El caso

La aplicación guarda el input del usuario **sin consultarlo en ese momento**, y lo usa más tarde en otra consulta SQL, en otra parte del sistema.

Ejemplo clásico:

1. El formulario de registro guarda el nombre de usuario tal cual:
   ```
   username: admin'--
   ```
   La petición de registro hace un simple `INSERT` parametrizado → **no hay error, no hay señal**.
2. Más tarde, otro módulo (login, panel de administración, job nocturno) construye:
   ```sql
   SELECT * FROM users WHERE username = 'admin'--'
   ```
   → la consulta se rompe o se explota **en un lugar que sqlmap nunca visitó**.

### Por qué sqlmap falla

- sqlmap prueba el endpoint de registro y ve respuestas normales: no hay error, no hay diferencia booleana, no hay retardo. **Concluye "no inyectable"**.
- La inyección no se manifiesta en la respuesta de la petición que la inyecta, sino en **otra petición, otro endpoint, otro momento**. sqlmap no modela ese flujo de dos etapas: su lógica es *petición → respuesta inmediata*.
- Incluso si el atacante apuntara sqlmap al segundo endpoint, el valor malicioso no está en el parámetro de esa petición sino **almacenado en la base de datos**, fuera del alcance del fuzzing.

### Cómo lo explota el pentester manual

1. **Mapea el flujo completo** de la aplicación: dónde se guarda el input y dónde se *reutiliza*.
2. Inyecta el payload en el primer punto (registro, perfil, dirección).
3. **Navega manualmente** hasta el punto donde ese dato se usa en una consulta: login, búsqueda de usuarios, exportación, panel admin.
4. Observa el comportamiento: un error genérico, un login que falla de forma extraña, un listado que no carga.
5. Ajusta el payload a mano (comillas, comentarios, `UNION`) hasta confirmar la ejecución SQL en el segundo punto.

La clave es **entender el flujo de datos de la aplicación**, algo que ninguna herramienta automática hace bien porque requiere comprender la lógica de negocio (el mismo punto ciego del Objetivo 4 de la serie).

---

## 3. Escenario 2: WAF, filtros y ofuscación no estándar

### El caso

La aplicación está detrás de un WAF o tiene filtros de entrada que bloquean los payloads clásicos de sqlmap: `UNION SELECT`, `' OR 1=1`, `SLEEP(`, espacios, comentarios, etc.

Los filtros típicos:

- Bloquean keywords: `UNION`, `SELECT`, `SLEEP`, `INFORMATION_SCHEMA`.
- Bloquean caracteres: espacios, comillas, `--`, `/*`.
- Firmas conocidas de sqlmap: el User-Agent por defecto (`sqlmap/1.x`) o los payloads canónicos están en las listas del WAF.

### Por qué sqlmap falla

- sqlmap tiene **tamper scripts** (`space2comment`, `charencode`, `randomcase`...) pero son transformaciones **genéricas y conocidas**. Un WAF bien configurado las reconoce, porque las firmas de sqlmap son públicas y los fabricantes de WAF las prueban.
- sqlmap no "entiende" el filtro: no analiza *qué* está bloqueando ni *por qué*. Solo prueba combinaciones de tamper scripts y, si ninguna funciona, abandona.
- Con `--random-agent` y tamper scripts se evaden WAFs viejos, pero un WAF moderno con reglas específicas contra sqlmap (que existen y son comunes) lo detecta por su **estructura de peticiones**: cientos de peticiones con patrones de fuzzing en segundos.

### Cómo lo explota el pentester manual

El humano analiza el filtro y diseña un payload a medida:

1. **Identificar qué bloquea el filtro**, probando pieza por pieza:
   - ¿Bloquea `UNION SELECT` pero no `union/**/select`?
   - ¿Bloquea espacios pero no `%09` (tab) ni paréntesis?
   - ¿Bloquea `SLEEP(` pero no `BENCHMARK(` o `GET_LOCK(`?
2. **Construir un payload equivalente** con la sintaxis que el filtro no contempla:
   ```sql
   ' UNION/**/SELECT/**/username,password/**/FROM/**/users-- -
   ```
   o usar sinónimos funcionales que el filtro no lista:
   ```sql
   ' OR (SELECT COUNT(*) FROM information_schema.tables)>0 AND '1'='1
   ```
3. **Usar Burp Intruder/Repeater** para probar variantes una a una, observando qué pasa exactamente con cada modificación.
4. Si el WAF bloquea por firma de sqlmap, basta con **hacer las peticiones a mano desde Repeater**: una petición humana con un payload ofuscado no coincide con ningún patrón automatizado.

La diferencia es que el humano **razona sobre el filtro** (es un problema de lógica, como los vistos en toda la serie), mientras que sqlmap solo **prueba combinaciones** de su arsenal.

---

## 4. Escenario 3 (bonus): Inyecciones con lógica de aplicación compleja

### El caso

El parámetro vulnerable requiere **condiciones previas de la aplicación** para ser procesado:

- Un token CSRF válido y fresco en cada petición.
- Una sesión con un estado concreto (carrito con items, flujo en el paso 3 de 5).
- Un nonce de un solo uso, o un HMAC del body calculado por JavaScript del cliente.
- Un captcha o un token anti-automatización.

### Por qué sqlmap falla

- sqlmap envía cientos de peticiones idénticas; si el token CSRF es de un solo uso, **todas fallan** después de la primera y la herramienta concluye que no hay inyección.
- Si el flujo exige estar en un estado particular, sqlmap no lo sabe: sus peticiones llegan "fuera de contexto" y la app las rechaza por validación de estado.
- Con HMAC de cliente (Objetivo 5 de la serie), sqlmap no puede generar el body válido porque no ejecuta el JavaScript de la aplicación.

### Cómo lo explota el pentester manual

- Usa **Burp** con macros o `Match & Replace` para renovar el token CSRF automáticamente en cada petición.
- Mantiene la sesión en el estado correcto y prueba la inyección **dentro del flujo real**, con Repeater.
- Si hay HMAC de cliente, extrae el algoritmo del JS y genera los bodies manipulados él mismo (como vimos en el Objetivo 5).

---

## 5. Tabla resumen

| Escenario | Por qué falla sqlmap | Por qué gana el humano |
|-----------|---------------------|------------------------|
| **Second-Order SQLi** | Solo ve petición→respuesta inmediata; el payload explota en otro endpoint, otro momento | Mapea el flujo de datos y conecta "dónde se guarda" con "dónde se usa" |
| **WAF / filtros custom** | Sus tamper scripts son públicos y genéricos; el WAF los firma | Analiza el filtro y diseña ofuscación a medida, petición a petición |
| **Lógica de aplicación** (CSRF, estado, HMAC) | No puede mantener contexto de flujo ni generar tokens dinámicos | Usa macros de Burp, mantiene el estado y replica el cifrado del cliente |

---

## 6. La lección de fondo

Los tres escenarios comparten la misma raíz, y es la misma de toda la serie:

> **Las herramientas automatizan patrones conocidos; las vulnerabilidades reales viven en los detalles particulares de cada aplicación.**

sqlmap es excelente para:

- Confirmar inyecciones *clásicas* y directas.
- Ahorrar tiempo en la explotación una vez identificada la inyección.
- Cubrir superficie amplia rápidamente.

Pero el pentester experimentado la usa como **acelerador**, no como sustituto del análisis: primero entiende la aplicación (flujo, filtros, contexto), y recién entonces decide si sqlmap sirve o si la explotación manual en Burp es el único camino.

> **Conclusión:** sqlmap falla estrepitosamente cuando la vulnerabilidad exige **contexto** (second-order), **creatividad** (evasión de filtros a medida) o **conciencia de flujo** (tokens, estado, HMAC) — las tres cosas que, por ahora, solo aporta el criterio humano. La automatización encuentra lo obvio; la explotación manual encuentra lo que importa.
