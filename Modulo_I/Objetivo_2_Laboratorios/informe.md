# Informe — Laboratorio Presupuesto

## Proceso

Primero analicé el enunciado e ingresé a la web con las credenciales brindadas, dejando la consola abierta en la pestaña "Network" con el filtro en XHR.

Inspeccioné el endpoint `/api/gastos` y así identifiqué los atributos de cada gasto (id, título, categoría, monto, fecha, revisado).

Hice clic en el botón "Revisar" y noté que dispara una petición que actualiza el estado de la columna "Revisado".

Con esa información, calculé los nuevos montos necesarios para cumplir los requisitos del enunciado (promedio total, promedio por categoría, mínimo y máximo).

Intenté primero un `PUT`, editando el mismo objeto que había recibido en el `GET`, pero no funcionó.

Luego probé un `PATCH` con el mismo enfoque, y tampoco tuvo efecto.

Finalmente, analicé la petición que dispara el botón "Revisar" (un `POST`) y reemplacé su body con los montos calculados y el estado en `true` (ej: `{"monto": "5000.00", "revisado": true}`).

Repitiendo este último paso para actualizar cada gasto uno por uno con su valor correspondiente, y al refrescar la página, obtuve el código de aprobación.
