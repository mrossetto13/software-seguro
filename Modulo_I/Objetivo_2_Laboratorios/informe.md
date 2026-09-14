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


# Informe — Laboratorio Turnero

## Proceso

Primero analicé el enunciado e ingresé a la web con las credenciales brindadas, dejando la consola abierta en la pestaña "Network" con el filtro en XHR.

Inspeccioné el endpoint `GET api/1/appointments/` y así identifiqué los atributos de cada turno.

Hice clic en el botón "Cancelar" y noté que dispara el evento `DELETE /api/appointments/{id}`, el cual borra el turno.

Con esto en cuenta, intenté encontrar al usuario "xdalvik" incrementando el ID en el endpoint `GET`, pero llegué hasta el ID 30 sin encontrarlo.

Luego intenté ubicarlo a través del endpoint `DELETE`, pero tampoco funcionó.

Entonces utilicé Burp Suite para recorrer los IDs del `GET` del 0 al 150. Al analizar los resultados, encontré al usuario buscado en el ID 101.

Con ese dato, edité el evento `GET /api/{id}/appointments/` (usando el ID 101) para ver qué turnos tenía ocupados, obteniendo así el ID de cada uno.

Finalmente, con el ID de cada turno, ejecuté el evento de borrado (`DELETE /api/appointments/{id}`) para cada uno de los turnos deseados.

Luego de borrar todos los turnos del usuario xdalvik, obtuve el código indicando que había completado el laboratorio.
