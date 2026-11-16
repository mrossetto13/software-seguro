# Objetivo 3: La Única Mitigación Definitiva (Prepared Statements)

## Escenario

Muchos desarrolladores intentan parchear las inyecciones SQL con **listas negras** (bloquear `SELECT`, `UNION`...) o **escapando comillas**. Ambas técnicas son ineficientes y evadibles. Este documento explica técnicamente cómo funciona una **Consulta Parametrizada (Prepared Statement)** a nivel de arquitectura, y por qué neutraliza por completo cualquier intento de inyección, sin importar qué caracteres envíe el usuario.

---

## 1. Por qué las defensas "de parche" fracasan

### Listas negras: una carrera perdida de antemano

Bloquear palabras es un juego imposible de ganar:

- **Sinónimos funcionales**: si bloqueás `SLEEP`, existe `BENCHMARK`; si bloqueás `UNION SELECT`, existe `union/**/select`; si bloqueás `OR`, existe `||`.
- **Ofuscación infinita**: comentarios (`SEL/**/ECT`), codificación URL (`%53ELECT`), doble codificación, case mixing (`sElEcT`), Unicode...
- **El atacante no necesita keywords**: `' OR '1'='1` no usa `UNION` ni `SELECT`. Y con `'; --` basta para romper muchas lógicas.

El problema de fondo: la lista negra intenta **adivinar todos los ataques posibles**, pero el espacio de payloads es ilimitado y el del desarrollador, finito.

### Escapar comillas: frágil por diseño

Escapar `'` como `\'` depende de:

- **El charset de la conexión**: con GBK o Big5, un byte como `0xbf` seguido de `0x27` forma una secuencia multibyte válida donde el `0x27` (comilla) queda "absorbido" — el escape clásico se evita (el famoso *SQL truncation / multibyte escape attack*).
- **Que TODOS los desarrolladores escapen en TODOS los lugares, siempre**: un solo punto olvidado (un ORDER BY dinámico, un LIKE, un reporte viejo) reabre la puerta.
- **Doble escape / doble decodificación**: si el valor pasa por dos capas (URL decode + escape), el escape se rompe.

> **Conclusión parcial:** escapar y filtrar son **mitigaciones de superficie**. Actúan sobre el *contenido* del dato, cuando el problema real es **estructural**: que el dato pueda llegar a interpretarse como código.

---

## 2. Prepared Statements: cómo funcionan a nivel de arquitectura

### El cambio fundamental: separar código de datos

La inyección SQL existe por una sola razón: **el código y los datos viajan en el mismo canal**. Cuando la aplicación concatena strings:

```js
"SELECT * FROM users WHERE id = " + userInput
```

la base de datos recibe **un único string** y no tiene forma de saber qué parte es programa y qué parte es dato. Si el "dato" contiene SQL, se ejecuta como SQL.

Los **Prepared Statements** eliminan esa mezcla con una arquitectura de **dos fases**:

### Fase 1: Preparación (parse + plan, sin datos)

La aplicación envía al servidor de base de datos la **plantilla de la consulta con placeholders**:

```sql
SELECT * FROM users WHERE id = ?     -- MySQL/PostgreSQL/SQLite
SELECT * FROM users WHERE id = :id   -- estilo nombrado (Oracle, PDO)
SELECT * FROM users WHERE id = $1    -- PostgreSQL nativo
```

El servidor SQL entonces:

1. **Parsea** la consulta: verifica sintaxis y semántica.
2. **Compila un plan de ejecución** (qué índices usar, orden de joins).
3. **Fija la estructura**: a partir de aquí, la consulta es un *programa ya compilado* cuyo esqueleto **no puede cambiar**.

Crucial: en este momento **el valor del usuario todavía no existe** para el servidor. El `?` es un hueco tipado, no texto a interpolar.

### Fase 2: Ejecución (solo datos)

La aplicación envía los valores **por un canal separado** del protocolo de base de datos:

```js
// Node.js
db.query("SELECT * FROM users WHERE id = ?", [userInput]);

// Java (JDBC)
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setInt(1, userId);

// Python
cursor.execute("SELECT * FROM users WHERE id = %s", (userInput,));

// PHP (PDO)
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$userInput]);
```

El servidor toma el valor recibido y lo **inserta como dato tipado** en el placeholder del plan ya compilado. El valor se trata **íntegramente como literal** — nunca como parte de la gramática SQL.

### El punto arquitectónico clave

> **El parseo ocurre ANTES de que exista el dato del usuario.**

Cuando llega el input, la consulta ya está parseada y compilada: su estructura (keywords, cláusulas, operadores) quedó congelada en el plan. El valor del usuario entra por el canal de *parámetros*, que el motor trata como datos opacos — sin importar si contiene `'`, `--`, `;`, `UNION SELECT` o cualquier otra cosa.

---

## 3. Por qué neutraliza CUALQUIER input: el ejemplo demoledor

Veamos qué pasa con el payload clásico:

```js
const userInput = "5 OR 1=1";
db.query("SELECT * FROM users WHERE id = ?", [userInput]);
```

**Con concatenación (vulnerable):**

```sql
SELECT * FROM users WHERE id = 5 OR 1=1
-- El motor ve dos condiciones: devuelve TODOS los usuarios
```

**Con Prepared Statement:**

```sql
-- Lo que el motor ejecuta, conceptualmente:
SELECT * FROM users WHERE id = '5 OR 1=1'
-- Busca un id cuyo valor LITERAL sea el string "5 OR 1=1"
-- No existe → devuelve 0 filas. Sin error, sin fuga.
```

El string completo `5 OR 1=1` se convierte en **el valor buscado**, no en una condición. Es literalmente como buscar un usuario cuyo id se llame "5 OR 1=1" — no lo encuentra y listo.

Lo mismo con payloads más agresivos:

```js
"'; DROP TABLE users; --"
// → se busca un id que sea literalmente ese string. Nada más.
```

El atacante puede enviar lo que quiera: **nunca sale del dominio de "dato"**. No hay puerta hacia la capa de código porque la estructura de la consulta ya se cerró antes de que el dato existiera.

### La analogía

Un prepared statement es como un formulario con casillas: el texto del formulario (las preguntas) ya está impreso e inmutable; el usuario solo rellena las casillas. Puede escribir cualquier barbaridad *dentro* de la casilla, pero nunca puede añadir una pregunta nueva al formulario. Con la concatenación, en cambio, el usuario co-escribe el formulario completo.

---

## 4. Matices técnicos importantes

### a) No es "escapado automático" — es separación estructural

Un error conceptual común: "los prepared statements escapan los caracteres". **Falso.** No hay escape, ni sanitización, ni transformación del input. El dato entra por un canal binario distinto del protocolo (ej. el protocolo binario de MySQL, o `PQexecParams` de PostgreSQL). El motor ni siquiera necesita "neutralizar" nada: el dato jamás entra en contacto con el parser SQL.

Esto explica por qué es **inmune a las evasiones** que rompen los escapes: el truco multibyte de GBK, la doble codificación, los comentarios... todos atacan el *escape*; ninguno puede atacar la **separación de canales**, porque no hay nada que evadir.

### b) Los placeholders solo cubren VALORES, no identificadores

Limitación importante: no se puede parametrizar un nombre de tabla, columna u orden:

```js
// ❌ NO funciona: el placeholder no puede ser un identificador
db.query("SELECT * FROM ? ORDER BY ?", [table, column]);
```

Para esos casos (dinámicos por diseño), la defensa es una **lista blanca** validada contra los valores permitidos:

```js
const allowedColumns = ["name", "price", "date"];
if (!allowedColumns.includes(column)) throw new Error("Columna inválida");
```

### c) Rendimiento: un beneficio extra

Como la consulta se parsea y planifica **una vez** y se ejecuta muchas veces con distintos parámetros, el servidor ahorra trabajo de compilación (y en algunos motores, `PREPARE` explícito reutiliza el plan). La seguridad aquí sale **gratis** y hasta mejora el rendimiento.

### d) ORM y query builders

ORMs (Prisma, Hibernate, SQLAlchemy, Eloquent) usan prepared statements por debajo, por lo que su uso habitual es seguro. **Pero** ojo con los métodos de "SQL crudo" (`$queryRaw`, `raw()`, `whereRaw()`): ahí se pierde la protección si se concatena input.

---

## 5. Comparación de defensas

| Técnica | Mecanismo | ¿Por qué falla / funciona? |
|---------|-----------|---------------------------|
| **Lista negra** | Bloquear keywords | Evadible con sinónimos, comentarios, codificación. Carrera infinita. |
| **Escapar comillas** | Transformar `'` → `\'` | Depende de charset, contexto y disciplina humana. Un solo olvido = vulnerable. |
| **Escapar + filtrar** | Combinación de ambos | Más capas de lo mismo: sigue habiendo código y datos en el mismo canal |
| **Prepared Statements** | Separación estructural código/dato en el protocolo | El dato **nunca** llega al parser SQL como código. No hay nada que evadir. |
| **Lista blanca** (para identificadores) | Validar contra valores permitidos | Complemento necesario donde los placeholders no aplican |
