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


# Informe — Laboratorio Gran Rifa 2019

## Proceso

Primero analicé el enunciado e ingresé a la web con las credenciales brindadas, dejando la consola abierta en la pestaña "Network" con el filtro en XHR.

Inspeccioné el endpoint `GET /api/numeros/` y así identifiqué los atributos de cada rifa.

Hice clic en el botón "Editar" y noté que dispara el evento `POST /api/numeros/{id}/editar/`, el cual edita el valor del campo "estado".

Con esto en cuenta, edité el evento `POST /api/numeros/{id}/editar/`, modificando el valor de `esta_pago` a `true`, y actualicé la página.

Al verificar que el cambio se había aplicado correctamente, repetí el proceso editando cada uno de los números vendidos, y así obtuve el código indicando que había completado el laboratorio.


# Informe — Laboratorio Ventas

## Proceso

Primero analicé el enunciado. Ingresé a la web y observé que el primer contacto con la página devolvía un "Not Found", por lo que analicé la consola. Actualicé la página para volver a capturar los eventos y confirmé que efectivamente obtenía un error 404 (Page Not Found).

Luego, teniendo en cuenta que el enunciado mencionaba que las ventas se encuentran en la sección `/ventas`, agregué `/ventas` al final de la URL. Ahí obtuve un error 301, lo que indicaba que a la URL le faltaba algo.

En lugar de solo `/ventas`, probé con `/ventas/?id=1` y obtuve un error 403 (Forbidden), lo que significa que la página existe pero no tengo acceso a ese recurso. Luego probé con `/ventas/?id=2` y obtuve un 404, lo cual indicaba que ese registro no existía. Al probar con varios números del 1 al 10, noté que había varios registros y que no eran correlativos.

Entonces utilicé Burp Suite para analizar los distintos registros, recorriendo los IDs del 1 al 9999, fijándome cuántas páginas devolvían un error 403 (indicando que el registro existía pero estaba restringido).

Al obtener el número del registro válido, lo convertí a MD5 y así obtuve el código de aprobación.
