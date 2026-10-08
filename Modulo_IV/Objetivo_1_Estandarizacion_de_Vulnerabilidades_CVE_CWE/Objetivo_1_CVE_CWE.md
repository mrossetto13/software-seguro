# Objetivo 1: Estandarización de vulnerabilidades (CVE y CWE)

## 1. ¿Qué es un CVE?

**CVE (Common Vulnerabilities and Exposures)** es un sistema de identificadores públicos y únicos para vulnerabilidades *específicas* en productos *específicos*. Su formato es `CVE-AÑO-NÚMERO`, por ejemplo `CVE-2021-44228` (Log4Shell, en Apache Log4j).

Cada **CVE Record** contiene, básicamente: el identificador, una descripción breve, los productos y versiones afectados y referencias (avisos del fabricante, parches, reportes técnicos).

Puntos a tener en cuenta:

- Un CVE **no indica por sí solo** la severidad, si existe un exploit ni la prioridad de remediación. Eso se evalúa con otros datos (puntaje CVSS, catálogo KEV de CISA, contexto de la organización).
- El programa CVE es mantenido por MITRE con respaldo del gobierno de EE. UU. (CISA) y gobernado por una junta (CVE Board) con participación internacional.

## 2. ¿Qué es el catálogo CWE?

**CWE (Common Weakness Enumeration)** es un catálogo comunitario de **tipos de debilidades** de software y hardware, es decir, las causas raíz que originan las vulnerabilidades. Cada entrada tiene el formato `CWE-NÚMERO`:

| CWE | Debilidad |
|---|---|
| CWE-79 | Neutralización incorrecta de entradas en páginas web (XSS) |
| CWE-89 | Inyección SQL |
| CWE-352 | Cross-Site Request Forgery (CSRF) |
| CWE-862 | Falta de autorización |

El catálogo lo mantiene MITRE y se actualiza varias veces por año (el sitio oficial anuncia la versión 4.20 como la más reciente al momento de esta consulta). Cada año se publica el **CWE Top 25**, que ordena las debilidades más peligrosas a partir de los CVE publicados. En la edición 2025 (publicada en diciembre de 2025) lideran XSS (CWE-79), inyección SQL (CWE-89) y CSRF (CWE-352), y la falta de autorización (CWE-862) subió al cuarto puesto.

## 3. Diferencia entre CVE y CWE

| | **CVE** | **CWE** |
|---|---|---|
| Qué describe | Una vulnerabilidad concreta | Un tipo de debilidad (causa raíz) |
| Nivel | Instancia / caso | Categoría / clase |
| Ejemplo | CVE-2021-44228 (Log4Shell) | CWE-502 (deserialización de datos no confiables), CWE-917 (inyección de lenguaje de expresiones) |
| Responde a | "¿Qué producto y versión están afectados?" | "¿Qué error de diseño o programación lo causó?" |
| Relación | Un CVE se asocia a uno o más CWE | Un CWE puede estar detrás de miles de CVE |

## 4. Autoridades que asignan CVE: los CNAs

Los **CNAs (CVE Numbering Authorities)** son organizaciones autorizadas por el programa CVE para asignar identificadores y publicar CVE Records dentro de un alcance acordado (por ejemplo, solo sus propios productos). Pueden ser fabricantes (Microsoft, Red Hat, Google), proyectos de código abierto, CERTs nacionales, empresas de seguridad e investigadores.

Estructura del programa:

- **Top-Level Roots:** MITRE (general) y CISA ICS (sistemas de control industrial y dispositivos médicos).
- **Roots:** organizaciones que supervisan a un grupo de CNAs, como JPCERT/CC o ENISA, y desde 2026 también Thales.
- **CNA-LR (Last Resort):** asignan CVE cuando no existe un CNA con alcance sobre el producto afectado.

El número de CNAs crece de forma sostenida: el sitio oficial reportaba 450 en 2025 y las cifras publicadas en 2026 hablan de unos 500 participantes (las fuentes difieren según la fecha de corte).

**Ciclo básico:** el investigador reporta al CNA responsable → el CNA reserva un ID → coordina con el fabricante → publica el CVE Record junto con el aviso público.

## 5. ¿Dónde se consultan públicamente?

- **[cve.org](https://www.cve.org):** sitio oficial del programa, con el registro autoritativo de cada CVE.
- **[NVD (nvd.nist.gov)](https://nvd.nist.gov):** base del NIST que enriquece cada CVE con puntaje CVSS, CWE asociado y productos afectados (CPE).
- **[CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog):** catálogo de vulnerabilidades con explotación activa confirmada.
- **[cwe.mitre.org](https://cwe.mitre.org):** catálogo CWE y el Top 25.
- **Avisos de los fabricantes** (por ejemplo, Microsoft Security Response Center, Red Hat, Ubuntu Security Notices).

## 6. Herramientas automatizadas que detectan CVEs

| Herramienta | Tipo | Cómo usa los CVE |
|---|---|---|
| **Nessus (Tenable)** | Escáner de vulnerabilidades de red y sistemas (comercial) | Compara versiones y configuraciones detectadas contra una base de plugins vinculados a CVE y reporta cada hallazgo con su ID. |
| **OpenVAS / Greenbone** | Escáner de vulnerabilidades (código abierto) | Ejecuta pruebas de red (NVTs) asociadas a CVE y genera informes con severidad CVSS. |
| **Qualys VMDR** | Gestión de vulnerabilidades en la nube (comercial) | Inventaría activos y los correlaciona con CVE, priorizando según explotación conocida. |
| **Trivy (Aqua Security)** | Escáner de contenedores, repositorios e IaC (código abierto) | Revisa imágenes y dependencias contra bases de datos de CVE; fácil de integrar en CI/CD. |
| **OWASP Dependency-Check** | Análisis de composición de software (SCA, código abierto) | Identifica librerías de terceros con CVE conocidos en un proyecto. |
