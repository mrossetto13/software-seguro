# Objetivo 3: Peligros del "Directory Listing"

## 1. ¿Qué es el Directory Listing?

Es una función de los servidores web (Apache, Nginx, IIS, etc.) que genera automáticamente una página HTML con el contenido de un directorio cuando la URL solicitada apunta a una carpeta y no a un archivo. Está pensada para compartir archivos de forma simple, pero en un sitio en producción casi siempre es un **error de configuración** (CWE-548: *Exposure of Information Through Directory Listing*, dentro de la categoría *Security Misconfiguration* del OWASP Top 10).

| Servidor | Directiva que lo habilita | Valor por defecto |
|---|---|---|
| Apache | `Options Indexes` (módulo `mod_autoindex`) | Varía según la distribución; en varias configuraciones de Debian/Ubuntu aparece activo para `/var/www/` |
| Nginx | `autoindex on;` | Desactivado |
| IIS | *Directory Browsing* | Desactivado |

## 2. ¿Qué ocurre técnicamente al navegar a un directorio sin índice?

1. El atacante solicita, por ejemplo, `GET /uploads/ HTTP/1.1`.
2. El servidor traduce la URL a una ruta del sistema de archivos (por ejemplo `/var/www/html/uploads/`) y detecta que es un directorio.
3. Busca un archivo índice según la directiva `DirectoryIndex` (Apache) o `index` (Nginx): `index.html`, `index.php`, etc.
4. Si **no lo encuentra** y el listado está activo, el servidor lee el contenido del directorio y construye dinámicamente una página HTML ("Index of /uploads/") con un enlace por cada entrada, junto con tamaño y fecha de modificación. Responde `200 OK`.
5. Si el listado está desactivado, responde `403 Forbidden` (en el log de Apache aparece *"Directory index forbidden by Options directive"*; en Nginx, *"directory index of ... is forbidden"*).

Ejemplo simplificado de lo que ve el atacante:

```
Index of /uploads/

Name                    Last modified        Size
Parent Directory                              -
backup_2025-03.sql.gz   2025-03-14 02:00     48M
config.php.bak          2025-01-22 17:41     2.1K
factura_0042.pdf        2025-06-02 11:09     310K
```

## 3. ¿Por qué es peligroso?

- **Reconocimiento sin esfuerzo:** el atacante no necesita adivinar nombres de archivos ni hacer fuerza bruta de rutas; el servidor se los entrega.
- **Expone archivos no enlazados:** recursos que nunca aparecen en el sitio (backups, pruebas, exportaciones) quedan visibles igualmente, porque el listado muestra todo lo que hay en la carpeta, no solo lo que la aplicación enlaza.
- **Revela la estructura interna:** nombres de carpetas, convenciones de nombres, tecnologías y versiones, lo que facilita ataques posteriores.
- **Descarga directa:** cada archivo listado se puede bajar con un clic, sin autenticación.
- **Es fácil de encontrar a escala:** buscadores y escáneres indexan estas páginas (por ejemplo, con búsquedas del tipo `intitle:"index of"`), por lo que no hace falta que el atacante conozca el sitio de antemano.

## 4. Archivos sensibles que pueden quedar expuestos

| Tipo | Ejemplos | Impacto |
|---|---|---|
| Backups y volcados | `backup.zip`, `site.tar.gz`, `dump.sql`, `db.sql.gz` | Acceso completo a datos de usuarios y a la base de datos |
| Configuración y secretos | `.env`, `config.php`, `wp-config.php.bak`, `web.config`, `settings.py.old`, `.htpasswd` | Credenciales de base de datos, claves de API, secretos de sesión |
| Control de versiones | carpeta `.git/`, `.svn/` | Reconstrucción del código fuente completo y del historial (a veces con credenciales antiguas) |
| Logs | `access.log`, `error.log`, `debug.log` | Rutas internas, errores con trazas, tokens o datos personales en URLs |
| Claves y certificados | `id_rsa`, `*.pem`, `*.key`, `*.pfx` | Acceso a servidores o suplantación de identidad |
| Código fuente y temporales | `*.php~`, `*.swp`, `*.bak`, `*.old`, `*.orig` | El código puede descargarse como texto plano en lugar de ejecutarse, revelando lógica y credenciales |
| Archivos de usuarios | carpeta `uploads/` con facturas, DNI, contratos | Fuga de datos personales (con implicancias legales, como la Ley 25.326 de Protección de Datos Personales en Argentina) |
| Archivos de diagnóstico | `phpinfo.php`, `test.php`, `info.php` | Versiones, módulos y rutas del servidor |
| Documentación interna | manuales, planillas, notas de despliegue | Información de arquitectura y de personal |

El efecto en cadena es el problema real: una credencial encontrada en un `.env` o en un backup puede servir para acceder a la base de datos o a otros sistemas.

## 5. Cómo se detecta (en pruebas autorizadas)

- Navegar manualmente a directorios comunes: `/uploads/`, `/backup/`, `/images/`, `/logs/`, `/admin/`, `/.git/`.
- Buscar en la respuesta la cadena `Index of /`: `curl -s https://sitio/uploads/ | grep -i "Index of"`.
- Usar herramientas de descubrimiento de contenido (gobuster, dirb, ffuf) o escáneres como Nikto y Burp Suite, que reportan el listado de directorios como hallazgo.

Estas pruebas deben hacerse solo sobre sistemas propios o con autorización expresa, como los laboratorios del curso.

## 6. Mitigación

### 6.1 Desactivar el listado

**Apache** (en el `VirtualHost` o `<Directory>`, o en `.htaccess` si `AllowOverride` incluye `Options`):

```apache
<Directory /var/www/html>
    Options -Indexes +FollowSymLinks
</Directory>
```

**Nginx** (es el valor por defecto, pero conviene dejarlo explícito y revisar que ningún `location` lo reactive):

```nginx
location / {
    autoindex off;
}
```

**IIS:** desactivar *Directory Browsing* para el sitio o la aplicación.

### 6.2 Bloquear archivos y carpetas sensibles

```apache
# Apache: extensiones de respaldo, logs y configuración
<FilesMatch "\.(bak|old|orig|sql|log|env|ini|swp)$|~$">
    Require all denied
</FilesMatch>

# Apache: repositorios
<DirectoryMatch "/\.(git|svn)">
    Require all denied
</DirectoryMatch>
```

```nginx
# Nginx: archivos ocultos (excepto .well-known) y extensiones sensibles
location ~ /\.(?!well-known) { deny all; }
location ~* \.(bak|old|orig|sql|log|env|ini|swp)$ { deny all; }
```

### 6.3 Buenas prácticas de fondo

- **No guardar archivos sensibles dentro del document root** (backups, `.env`, logs, claves): ubicarlos fuera de la carpeta publicada.
- **Incluir un archivo índice** en los directorios públicos, como medida secundaria (no sustituye a desactivar el listado).
- **Principio de mínimo privilegio** en permisos del sistema de archivos y de la cuenta del servidor web.
- **Limpiar los despliegues:** no publicar archivos temporales, copias `.bak` ni carpetas `.git`; automatizarlo en el pipeline de CI/CD.
- **Guardar los archivos subidos por usuarios** fuera del webroot o en almacenamiento con control de acceso, y servirlos mediante la aplicación.
- **Revisar periódicamente** la configuración con escáneres y guías de hardening (por ejemplo, los CIS Benchmarks para Apache y Nginx).
- **Si ya hubo exposición:** retirar el archivo, rotar todas las credenciales y claves que contenía, y revisar los logs para determinar si hubo descargas.

### 6.4 Verificación

Después de aplicar los cambios, recargar el servidor (`apachectl graceful` o `nginx -s reload`) y comprobar que un directorio sin índice devuelve `403 Forbidden` (o `404`) y no el listado:

```bash
curl -I https://sitio/uploads/
```

## 7. Conclusión

El Directory Listing no es una vulnerabilidad de código, sino una **mala configuración** muy sencilla de corregir y de gran impacto: convierte el servidor web en un explorador de archivos público. La defensa combina desactivar la directiva (`Options -Indexes` / `autoindex off`), mantener los archivos sensibles fuera del directorio publicado y verificar de forma periódica que ningún directorio quede expuesto.

## Referencias

- [Apache HTTP Server, documentación de mod_autoindex](https://httpd.apache.org/docs/2.4/mod/mod_autoindex.html)
- [Nginx, módulo ngx_http_autoindex_module](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html)
- [CWE-548: Exposure of Information Through Directory Listing](https://cwe.mitre.org/data/definitions/548.html)
