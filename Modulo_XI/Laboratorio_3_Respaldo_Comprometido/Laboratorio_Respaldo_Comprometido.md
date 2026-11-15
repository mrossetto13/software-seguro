# Laboratorio: Respaldo Comprometido — NómiNet (panel de RRHH)

## Objetivo

Obtener acceso a la cuenta `admin` del panel de administración de NómiNet, sin conocer su contraseña, y conseguir el código de resolución del laboratorio.

## Metodología

### 1. Reconocimiento de endpoints

Se abrió la consola de desarrollador del navegador y se refrescó la página principal, de forma que quedaran registrados en la pestaña de red (Network) todos los endpoints consultados por el frontend al cargar.

### 2. Comentario revelador en el código fuente

Al inspeccionar el código fuente de la página, se encontró el siguiente comentario dejado por el equipo de desarrollo:

```html
<!-- TODO (dev): sacar el respaldo de /backup/usuarios.sql antes de pasar a prod -->
```

Este comentario, pensado como una nota interna para el equipo de desarrollo, quedó expuesto en el HTML servido al público — una filtración de información (CWE-200: *Exposure of Sensitive Information*) que reveló la existencia de un archivo de backup en una ruta predecible.

### 3. Acceso al backup de la base de datos

Se accedió directamente a `/backup/usuarios.sql`, el cual estaba públicamente disponible sin ningún control de acceso. El archivo resultó ser un volcado SQL completo de la base de datos `nominet`, con la siguiente advertencia en su propio encabezado:

```sql
-- ============================================================================
-- Respaldo automático de la base "nominet" — Panel RRHH NómiNet
-- Generado por el script de backup nocturno (cron). NO DEBE QUEDAR PÚBLICO.
-- Contiene la tabla de usuarios con sus hashes de contraseña.
-- ============================================================================

CREATE TABLE usuarios (
  id            INTEGER PRIMARY KEY,
  usuario       TEXT NOT NULL UNIQUE,
  nombre        TEXT,
  rol           TEXT,
  password_hash TEXT NOT NULL
);

-- Las contraseñas se guardan con sha512crypt ($6$), el mismo esquema que
-- /etc/shadow en Linux. "Hasheadas = seguras", decía el que las eligió...
INSERT INTO usuarios (id, usuario, nombre, rol, password_hash) VALUES
  (1, 'admin', 'Soporte TI', 'admin', '$6$lab64adm$Guw66KEaFrRz0Itikk4AskR7aODQ/ZQ.nVlE.Wls5KIuyTu5abF.7I6oJENqCgtnvB1EsdUUDrGzqiP9zq0FU0'),
  ...
```

El volcado contenía además 49 registros de usuarios adicionales, que no fueron relevantes para este objetivo: el foco se puso únicamente en la fila del usuario `admin`.

### 4. Identificación del esquema de hash

El propio comentario del script indicaba que las contraseñas se almacenan con **sha512crypt** (prefijo `$6$`), el mismo esquema que usa `/etc/shadow` en Linux. Que las contraseñas estén hasheadas no las hace invulnerables: un hash sin sal única por sitio, sin *pepper*, y sobre todo generado a partir de una contraseña débil, sigue siendo crackeable fuera de línea con suficiente poder de cómputo y un buen diccionario.

### 5. Cracking del hash con Hashcat

Se utilizó **Hashcat** (ya incluido en Kali Linux) para intentar recuperar la contraseña en texto plano a partir del hash, mediante un ataque de diccionario:

1. Se guardó el hash del usuario `admin` en un archivo de texto (`Sha512.txt`):
   ```
   $6$lab64adm$Guw66KEaFrRz0Itikk4AskR7aODQ/ZQ.nVlE.Wls5KIuyTu5abF.7I6oJENqCgtnvB1EsdUUDrGzqiP9zq0FU0
   ```
2. Se ejecutó Hashcat con el modo correspondiente a sha512crypt (`-m 1800`), ataque de diccionario (`-a 0`), usando la wordlist `rockyou.txt` incluida en Kali:
   ```bash
   hashcat -m 1800 -a 0 Sha512.txt /usr/share/wordlists/rockyou.txt
   ```
3. Hashcat logró crackear el hash, recuperando la contraseña en texto plano correspondiente al usuario `admin`.

### 6. Login como administrador

Con las credenciales `admin` / *(contraseña crackeada)*, se inició sesión en el panel de administración de NómiNet, obteniendo acceso con el rol de administrador y el código de resolución del laboratorio.

## Resultado

Se obtuvo acceso completo a la cuenta `admin` del panel de RRHH de NómiNet, explotando una cadena de dos fallas independientes: exposición de un backup de base de datos por un comentario de desarrollo olvidado, y una contraseña débil detrás de un hash crackeable por diccionario.

> **Contraseña obtenida:** _(completar con la contraseña recuperada por Hashcat)_
> **Código de resolución:** _(completar con el código obtenido tras el login)_

## Causa raíz

- **Exposición de información sensible en comentarios de código (CWE-200 / CWE-615):** un comentario de desarrollo (`TODO`) destinado al equipo quedó visible en el HTML servido en producción, revelando la existencia y ubicación exacta de un archivo de backup.
- **Backup público sin control de acceso:** el archivo `/backup/usuarios.sql` era accesible directamente por URL, sin autenticación ni restricción alguna, pese a la advertencia explícita en su propio contenido ("NO DEBE QUEDAR PÚBLICO").
- **Contraseña débil (CWE-521):** aunque el hash en sí (sha512crypt) es un esquema razonable y lento de calcular, una contraseña presente en diccionarios comunes como `rockyou.txt` es crackeable en tiempos prácticos sin importar el algoritmo de hash utilizado.

## Recomendaciones de remediación

1. **Eliminar backups y archivos temporales del document root** antes de pasar a producción, o directamente generarlos fuera de la carpeta servida públicamente por el servidor web.
2. **No dejar comentarios de desarrollo (`TODO`, rutas internas, credenciales, etc.) en el HTML/JS servido al cliente**; revisar esto como parte del checklist de pase a producción, idealmente de forma automatizada (linters, pipelines de CI/CD que fallen ante patrones como `TODO`, `FIXME`, `password`, etc.).
3. **Restringir el acceso a cualquier ruta de backup** mediante autenticación, o mejor aún, no exponerla en ningún servidor accesible desde internet.
4. **Exigir contraseñas robustas** para cuentas administrativas, en particular políticas que impidan contraseñas presentes en diccionarios conocidos (verificación contra listas como *Have I Been Pwned* o `rockyou.txt` al momento de definir la contraseña).
5. **Agregar autenticación multifactor (MFA)** para cuentas con rol de administrador, de forma que una contraseña comprometida no sea suficiente para tomar control de la cuenta.
6. **Rotar inmediatamente** la contraseña del usuario `admin` y, dado que el backup completo quedó expuesto, las de los 49 usuarios restantes también.

## Herramientas utilizadas

- Herramientas de desarrollador del navegador (consola, pestaña Network, Ver código fuente).
- Hashcat (modo `1800` — sha512crypt).
- Wordlist `rockyou.txt` (incluida en Kali Linux).
