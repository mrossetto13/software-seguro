# Laboratorio: PhotoVault — Exposición de fotos privadas vía metadata EXIF

## Objetivo

Obtener el código que se encuentra en la foto privada del administrador de PhotoVault, demostrando que una foto marcada como privada puede ser accedida por otro usuario.

## Análisis previo

Se creó una cuenta normal y se usó la plataforma como cualquier usuario: se subió una foto de prueba y se navegó la aplicación con Burp Suite interceptando el tráfico.

Al revisar el endpoint `GET /fotos/{id}`, se observó que recorriendo distintos IDs numéricos aparecían fotos ajenas que devolvían **403 Forbidden** — es decir, existen pero el acceso directo está bloqueado. Estos IDs fueron el primer indicio de que hay fotos privadas de otros usuarios (potencialmente del administrador) en ese rango.

## Intentos descartados

Antes de llegar al vector correcto, se probaron y descartaron los siguientes caminos:

- **Manipular la cookie `session`:** se identificó que la sesión es una cookie firmada de Flask (`itsdangerous`), con formato `payload.timestamp.firma`. Modificar el `uid` del payload sin poder re-firmar con la clave secreta del servidor invalida la firma, por lo que no es explotable sin conocer el `SECRET_KEY`.
- **Acceder a `/fotos/{id}/raw` directamente:** visto en el código fuente de `/mis-fotos` (`<img src="/fotos/{id}/raw">`), pero resultó tener el mismo control de acceso que `/fotos/{id}` (también devuelve 403 para fotos ajenas).

## Vector explotado: función "Buscar original"

La plataforma incluye una funcionalidad en `/herramientas/buscar-original`, descrita como: *"¿Tenés una copia recomprimida, recortada o de baja calidad? Subila y te mostramos la versión original en alta resolución de tu biblioteca. Las copias descargadas desde PhotoVault se identifican al instante. Para el resto hacemos una búsqueda visual."*

### Hallazgo 1 — la función no filtra por dueño ni privacidad

Se subió una imagen arbitraria, sin relación con las fotos propias, y el sistema respondió **"¡Encontramos el original!"**, devolviendo una foto privada (propia) completa embebida en base64, indicando **"Coincidencia: Búsqueda visual"**. Esto demostró que la función de búsqueda compara contra el catálogo completo de fotos (no solo las del usuario que consulta) y no aplica ningún control de acceso sobre el resultado.

### Hallazgo 2 — las fotos descargadas llevan un identificador en metadata EXIF

Se descargó una foto legítima desde la plataforma y se inspeccionó su metadata con `exiftool`:

```bash
exiftool -a -u -g1 oficina-nueva.jpg
```

Entre los campos EXIF estándar (dimensiones, fecha, software), apareció un campo no estándar:

```
Image Unique ID : PV-ASSET-000014
```

junto con otros campos relacionados (`Artist`, `Copyright: (c) Equipo PhotoVault - via PhotoVault`, `User Comment: downloaded-by:mrossetto`). Esto confirmó la hipótesis: cada descarga lleva incrustado un identificador interno (`PV-ASSET-XXXXXX`), y la mención de "se identifican al instante" sugiere que el backend usa ese campo para reconocer la foto por **coincidencia exacta de ID**, en vez de comparar el contenido de la imagen.

### Hallazgo 3 — el ID de metadata es confiable sin re-verificar permisos

Al editar ese campo con un ID distinto y volver a subir el archivo por "Buscar original", el sistema devolvía la foto correspondiente a **ese ID editado**, sin validar si el usuario autenticado tenía permiso para verla. El control de acceso que sí existe en `/fotos/{id}` **no se aplica** en este flujo alternativo.

## Metodología de explotación

1. Se editó el campo `ImageUniqueID` del archivo descargado con `exiftool`:

   ```bash
   exiftool -overwrite_original -ImageUniqueID="PV-ASSET-000079" oficina-nueva.jpg
   ```

2. Se verificó el cambio:

   ```bash
   exiftool -ImageUniqueID oficina-nueva.jpg
   ```

3. Se subió el archivo modificado a `/herramientas/buscar-original`.

4. Se repitió el proceso, probando distintos valores de ID — en particular, los mismos IDs numéricos que habían devuelto **403** al intentar accederlos directamente por `/fotos/{id}` — generando y subiendo un archivo editado por cada candidato.

5. Tras iterar sobre ese conjunto de IDs, se encontró una coincidencia que devolvía la foto privada del administrador con el código buscado visible en la imagen.

## Resultado

Se logró acceder al contenido de una foto privada ajena (la del administrador) explotando la falta de control de acceso en la función "Buscar original", combinada con la confianza indebida del backend en un identificador de metadata controlable por el cliente. Se obtuvo el código solicitado por el enunciado del laboratorio.

> **Código obtenido:** _(completar con el valor visible en la imagen)_

## Causa raíz

- **Falta de control de acceso (Broken Access Control / IDOR):** la ruta `/herramientas/buscar-original` no valida que el usuario tenga permiso sobre la foto que termina devolviendo, a diferencia de `/fotos/{id}`.
- **Confianza indebida en datos del cliente:** el identificador `ImageUniqueID`, incrustado en el archivo que el propio cliente sube, es editable por cualquiera con `exiftool` y el backend lo usa para resolver qué foto mostrar sin再verificar la propiedad del recurso.

## Recomendación de remediación

1. Aplicar la misma verificación de propiedad/autorización que usa `/fotos/{id}` también en el flujo de "Buscar original", antes de devolver cualquier imagen.
2. No confiar en metadata incrustada en un archivo subido por el usuario para determinar qué recurso del lado del servidor mostrar; si se necesita un identificador de "copia descargada", debe validarse contra una tabla interna junto con el `uid` de la sesión, no solo contra el ID en sí.
3. Si la detección "al instante" se basa en un hash de contenido en vez de metadata editable, usar un hash criptográfico del archivo completo (no un campo de texto plano modificable) y aun así revalidar permisos sobre el recurso encontrado antes de responder.

## Herramientas utilizadas

- Burp Suite (Proxy / Repeater) — interceptar y modificar peticiones HTTP.
- `exiftool` — inspeccionar y editar metadata EXIF de los archivos descargados.
