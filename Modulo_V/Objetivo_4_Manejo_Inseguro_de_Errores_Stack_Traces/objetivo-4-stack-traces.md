# Objetivo 4: Manejo Inseguro de Errores (Stack Traces)

**Materia:** Ciberseguridad — Tarea de investigación
**Tema:** Exposición de Stack Traces en entornos de producción vs. mensajes de error genéricos

---

## 1. La diferencia fundamental

### 1.1 Mensaje de error genérico (configuración segura)

Cuando la aplicación falla, el usuario recibe una respuesta **controlada y sin información técnica**:

```
Lo sentimos, ha ocurrido un error inesperado. 
Por favor, intente nuevamente más tarde. (Ref: #A7X-2931)
```

Características:

- **No revela** tecnología, rutas internas, versiones ni estructura del código.
- El detalle real del error se registra **solo en el lado del servidor** (logs a los que el usuario no accede).
- El código de referencia (`Ref: #A7X-2931`) permite al soporte técnico buscar el error en los logs **sin exponer nada al cliente**.
- La respuesta HTTP puede ser un `500 Internal Server Error` con cuerpo neutro.

### 1.2 Stack Trace expuesto (configuración insegura)

Cuando `debug` está activado en producción (o el framework no está configurado), la misma falla devuelve al navegador la **traza completa de la pila de ejecución**:

```
Fatal error: Uncaught PDOException: SQLSTATE[42000]: Syntax error or access 
violation: 1064 You have an error in your SQL syntax near 'WHERE id = '' at line 1 
in /var/www/html/app/models/UserModel.php:47 
Stack trace: 
#0 /var/www/html/app/models/UserModel.php(47): PDO->query('SELECT * FROM u...')
#1 /var/www/html/app/controllers/LoginController.php(23): UserModel->find(NULL)
#2 /var/www/html/public/index.php(12): LoginController->authenticate()
#3 {main} thrown in /var/www/html/app/models/UserModel.php on line 47
```

Es decir: el atacante recibe, **gratis y sin autenticarse**, un diagnóstico interno de la aplicación.

---

## 2. ¿Qué filtra exactamente un Stack Trace?

| Información expuesta | Valor para el atacante |
| --- | --- |
| **Rutas absolutas del servidor** (`/var/www/html/app/...`, `C:\inetpub\...`) | Confirma el sistema operativo, estructura de directorios y ubicación de archivos para ataques de path traversal o LFI |
| **Tecnologías y versiones** (`PDOException`, `Spring Framework 5.3.2`, `.NET Framework 4.7`) | Permite buscar **CVEs conocidas** de esas versiones exactas |
| **Fragmentos de consultas SQL** (`SELECT * FROM u... WHERE id = ''`) | Revela nombres de tablas, columnas y a veces el punto exacto donde la entrada del usuario se concatena → **confirma inyección SQL** |
| **Nombres de clases, métodos y líneas** (`UserModel.php:47`) | Mapa de la lógica interna; facilita encontrar otros parámetros inyectables |
| **Valores de variables y parámetros** en algunos frameworks (páginas de debug tipo Django/Flask/Rails) | Pueden incluir **claves de API, credenciales de BD y variables de entorno** en el propio traceback interactivo |
| **Cookies, tokens o session IDs** que el framework imprima en el error | Posible **secuestro de sesión** |

---

## 3. Por qué acelera la fase de explotación

Un ataque tiene fases: **reconocimiento → enumeración → explotación → post-explotación**. Un stack trace expuesto le ahorra al atacante las dos primeras:

1. **Elimina la fase de reconocimiento a ciegas.** Sin stack trace, el atacante debe *adivinar* el stack tecnológico (fingerprinting con herramientas, prueba y error). Con stack trace, se lo entregan en bandeja: lenguaje, framework, versiones, librerías.

2. **Convierte la explotación en "ensayo y error con feedback".** Es la ventaja más crítica: el atacante modifica un parámetro (`id=1'`), provoca un error, y **el stack trace le responde si su payload funcionó**. Con una inyección SQL, cada mensaje de error valida el avance ("la sintaxis falló cerca de..."), permitiendo reconstruir la consulta a ciegas (*error-based SQL injection*) hasta extraer la base de datos completa.

3. **Reduce el tiempo de desarrollo del exploit.** Conocer el archivo, la línea y la función exacta donde falla el código permite apuntar directamente a la debilidad en lugar de descubrirla. Herramientas automatizadas (sqlmap, por ejemplo) dependen de mensajes de error diferenciados para confirmar la inyectabilidad.

4. **Habilita ataques encadenados.** La ruta absoluta filtrada sirve para explotar otra vulnerabilidad (ej.: un LFI que ahora sabe qué archivo incluir; un upload donde sabe dónde caerá el archivo).

> **En síntesis:** un stack trace no es la vulnerabilidad en sí, sino un **multiplicador de información** que convierte un ataque a ciegas en un ataque guiado, reduciendo horas de reconocimiento a segundos.

---

## 4. Cómo se mitiga

1. **Mensajes genéricos al cliente en producción** — desactivar el modo debug (`APP_DEBUG=false`, `DEBUG=False` en Django, `production` en el entorno de Rails).
2. **Registrar el detalle en el servidor, no en la respuesta** — el stack trace completo va al log (con rotación y restricción de acceso); al usuario solo un ID de correlación.
3. **Manejo de excepciones centralizado** — un handler global que capture cualquier excepción no controlada y devuelva la página de error neutra (evita que un flujo no previsto filtre la traza).
4. **Páginas de error personalizadas** para 404 y 500, sin dependencias del framework por defecto (las páginas por defecto de Tomcat, ASP.NET o Django ya filtran información).
5. **No exponer detalles en APIs** — respuestas JSON con código de error genérico; el detalle solo vía logs y monitoreo.
6. **Verificación en auditoría** — provocar errores deliberadamente (parámetros malformados, rutas inexistentes) y confirmar que la respuesta no filtra trazas.
