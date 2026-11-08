# Plantilla de Ticket — Auditoría de Seguridad (Caja Negra) con OWASP ASVS

**Documento de referencia:** estructura, propósito de cada campo y valor generado para el lector final de un ticket de hallazgo de seguridad.

---

## 1. ¿Qué es esta plantilla?

Es una **plantilla estandarizada para reportar hallazgos técnicos** identificados durante evaluaciones externas de penetración (*pentesting*) realizadas **sin acceso previo al código fuente ni a la arquitectura interna** del sistema auditado — es decir, auditorías de **caja negra**.

Su objetivo es que **cada hallazgo se documente siempre de la misma manera**, con los mismos campos, las mismas clasificaciones y el mismo nivel de detalle, sin importar qué auditor lo redacte.

---

## 2. Estructura de la plantilla

La plantilla organiza los campos en **5 bloques lógicos**, que siguen el ciclo natural de lectura de un hallazgo: *qué es → dónde está → cómo se clasifica → qué significa → cómo se corrige*.

### Bloque 1 — Identificación del hallazgo

| Campo | Tipo | Propósito |
| --- | --- | --- |
| **Título de la Vulnerabilidad** | Texto | Nombre claro y accionable del hallazgo. Debe poder leerse en una lista de tickets y entenderse de qué se trata sin abrir el detalle. |
| **Nivel de Criticidad** | Selección (Crítica / Alta / Media / Baja) | Priorización inmediata. Es el primer filtro que usa el equipo para decidir qué atender primero. |

### Bloque 2 — Contexto del hallazgo

| Campo | Tipo | Propósito |
| --- | --- | --- |
| **Endpoint o URL Afectada** | Selección (lista "Endpoints API") | Ubicación exacta del problema dentro del sistema. Se selecciona de una lista predefinida del proyecto, lo que garantiza consistencia y permite trazabilidad automática. |
| **Caso de Uso Asociado** | Selección (lista "Casos de Uso") | Vincula el hallazgo con la funcionalidad de negocio afectada. Conecta el mundo técnico con el mundo del negocio. |

### Bloque 3 — Estandarización y clasificación

| Campo | Tipo | Propósito |
| --- | --- | --- |
| **Verificación OWASP ASVS v4.0.3** | Campo ASVS | Identifica qué controles del estándar ASVS se ven involucrados o violados por el hallazgo. Ancla el hallazgo a un marco de requisitos medibles y comprobables. |
| **Puntuación CVSS v3.1** | Campo CVSS | Cuantifica la severidad técnica del hallazgo con el estándar de la industria, de forma objetiva y reproducible. |

### Bloque 4 — Desarrollo técnico

| Campo | Tipo | Propósito |
| --- | --- | --- |
| **Resumen Ejecutivo** | Texto largo | El hallazgo en lenguaje de negocio, sin tecnicismos: qué pasó, qué riesgo implica y qué se recomienda. Es lo que lee un gerente o C-level. |
| **Descripción Detallada del Hallazgo** | Texto largo | El análisis técnico completo: causa raíz, comportamiento observado, condiciones de explotación y limitaciones. Es lo que lee el equipo técnico. |

### Bloque 5 — Verificación y corrección

| Campo | Tipo | Propósito |
| --- | --- | --- |
| **Prueba de Concepto (PoC) y Pasos de Reproducción** | Texto largo | Evidencia y pasos numerados que permiten a **cualquier persona** reproducir el hallazgo de forma independiente. Sin esto, un hallazgo no es verificable. |
| **Recomendación de Remediación & Mitigación** | Texto largo | Cómo corregir el problema: medidas concretas, priorizadas, con ejemplos de código o configuración cuando aplica. |

---

## 3. Flujo de lectura: cómo se usa la plantilla

```
┌─────────────────────────────────────────────────────┐
│  Título + Criticidad                                │  ← ¿Qué es y qué tan grave? (5 segundos)
├─────────────────────────────────────────────────────┤
│  Resumen Ejecutivo                                  │  ← ¿Qué significa para el negocio? (2 minutos)
├─────────────────────────────────────────────────────┤
│  Endpoint + Caso de Uso                             │  ← ¿Dónde está y qué negocio afecta? (30 segundos)
├─────────────────────────────────────────────────────┤
│  ASVS + CVSS                                        │  ← ¿Cómo se clasifica formalmente? (1 minuto)
│  Descripción Detallada                              │  ← ¿Cuál es el problema técnico? (5 minutos)
├─────────────────────────────────────────────────────┤
│  PoC + Reproducción                                 │  ← ¿Cómo lo verifico yo mismo?
├─────────────────────────────────────────────────────┤
│  Remediación                                        │  ← ¿Qué hago para corregirlo?
└─────────────────────────────────────────────────────┘
```

Cada lector entra por el nivel de profundidad que necesita: un gerente lee hasta el Resumen Ejecutivo; un desarrollador llega hasta la Remediación; un auditor externo valida con la PoC.

---

## 4. Valor generado para el usuario final

### 4.1 Para quien recibe el ticket (equipo de desarrollo y negocio)

- **Priorización objetiva:** la combinación Criticidad + CVSS elimina la subjetividad. No hay discusión sobre "qué tan grave es": el número y su vector lo definen.
- **Trazabilidad completa:** saber exactamente qué endpoint y qué caso de uso de negocio están afectados permite planificar la corrección dentro del ciclo de desarrollo, no como un parche aislado.
- **Verificabilidad:** la PoC con pasos de reproducción permite al desarrollador **confirmar el hallazgo por sí mismo** antes y después de corregirlo. El ticket se cierra con evidencia, no con promesas.
- **Instrucciones accionables:** la remediación no dice "mejorar la seguridad", sino qué control específico implementar, con ejemplos concretos. Reduce el tiempo de corrección y la fricción entre auditoría y desarrollo.

### 4.2 Para quien gestiona la seguridad (líderes de proyecto, compliance)

- **Lenguaje común con la industria:** ASVS y CVSS son estándares reconocidos internacionalmente. Los reportes son comparables entre auditorías, proveedores y períodos, y son aceptados en procesos de compliance y certificación.
- **Métricas agregables:** al tener campos tipados y consistentes, se pueden generar métricas automáticas: distribución de severidades, endpoints más afectados, controles ASVS que más fallan, evolución temporal.
- **Auditoría de la auditoría:** la estructura uniforme permite revisar la calidad del trabajo de los auditores: ¿todos los hallazgos tienen PoC? ¿todos tienen clasificación CVSS justificada?

### 4.3 Para la organización

- **Conocimiento institucional:** el hallazgo deja de vivir en el mail de un consultor y pasa a ser un activo documentado, consultable y reutilizable.
- **Reducción de riesgo recurrente:** al mapear cada hallazgo a controles ASVS, la organización identifica patrones (ej.: "siempre fallamos en validación de entrada") y puede corregir la causa, no solo el síntoma.
- **Defensa ante terceros:** ante un incidente o una certificación, la organización puede demostrar con documentación formal cómo se identificó, clasificó y remedió cada vulnerabilidad.

---

## 5. Por qué caja negra define la plantilla

Al no tener acceso al código fuente, el auditor **no puede** señalar "la línea 42 de `auth.py` está mal". Todo lo que reporta proviene de **lo observable desde afuera**: peticiones, respuestas, comportamientos, errores.

Por eso la plantilla exige:

1. **Endpoint afectado** — la única "ubicación" que el auditor puede conocer con certeza.
2. **PoC con pasos de reproducción** — la única prueba válida en caja negra es la reproducible desde el exterior.
3. **ASVS como referencia** — el estándar permite evaluar si un control *existe y funciona* observando el comportamiento, sin leer el código.
