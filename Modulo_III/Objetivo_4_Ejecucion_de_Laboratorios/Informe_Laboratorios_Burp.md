# Informe de Laboratorios — Pruebas de Seguridad con Burp Suite

## Índice

1. Laboratorio "Apagar la IA"
2. Laboratorio "Votación"
3. Laboratorio "Votación (nueva versión)"
4. Laboratorio "El mejor secreto"

---

## 1. Laboratorio "Apagar la IA"

### 1.1 Objetivo

Encontrar el código de aprobación necesario para completar el laboratorio, a partir de un valor hash mostrado al inicio de la consigna.

### 1.2 Análisis previo

Al leer el enunciado se observó que el hash mostrado al comienzo no es un valor arbitrario, sino que corresponde a la codificación de un número. Al decodificarlo se pudo identificar aproximadamente el rango numérico dentro del cual debía encontrarse el valor correcto buscado, lo que redujo considerablemente el espacio de búsqueda frente a un ataque de fuerza bruta sin acotar.

### 1.3 Metodología

1. Se decodificó el hash inicial para obtener una referencia numérica y así delimitar el rango de valores candidatos.
2. Con ese rango, se armó un diccionario de valores hash candidatos a probar.
3. Se configuró Burp Intruder sobre la petición correspondiente, usando el diccionario armado como payload, con la condición de que el proceso recorriera los valores del diccionario hasta encontrar un valor de 16 dígitos (que era el formato esperado por la aplicación).
4. Una vez localizado el valor de 16 dígitos válido, se lo convirtió a MD5.
5. El resultado de esa conversión fue el código de aprobación que permitió completar el laboratorio.

### 1.4 Resultado

Se obtuvo el código de aprobación correcto y se completó el laboratorio.

---

## 2. Laboratorio "Votación"

### 2.1 Objetivo

Lograr que la cantidad de votos de la UTN supere a la de Harvard mediante votaciones repetidas.

### 2.2 Análisis previo

Tras analizar el enunciado, se realizó una primera votación manual y se observó el resultado en pantalla. Al intentar votar una segunda vez, la aplicación bloqueaba el intento mediante el valor de la cookie de sesión, impidiendo repetir la acción desde la misma sesión.

### 2.3 Metodología

1. Se interceptó la petición de voto con Burp Proxy y se envió a Repeater para analizar su estructura.
2. Se identificó que el bloqueo se basaba en el valor almacenado en la cookie.
3. Se modificó manualmente el request agregando un valor incremental en el campo relevante, para evitar que el servidor reconociera cada intento como repetido. Esta modificación funcionó como prueba de concepto.
4. Se configuró Burp Intruder para automatizar el envío de la petición, incrementando ese valor de forma secuencial en cada iteración.
5. Se dejó correr el ataque hasta que la cantidad de votos de la UTN superó a la de Harvard.

### 2.4 Resultado

Se logró superar en votos a Harvard y se completó el laboratorio.

---

## 3. Laboratorio "Votación (nueva versión)"

### 3.1 Objetivo

Repetir el objetivo del laboratorio anterior (que los votos de la UTN superen a los de Harvard, sobre una versión del sistema con una protección adicional.

### 3.2 Análisis previo

Se realizó una primera votación y se analizó el resultado. Dado que el enunciado era similar al laboratorio anterior, se intentó repetir la misma técnica, pero esta vez el sistema devolvía un mensaje de error indicando que no se podía votar más de una vez desde la misma dirección IP, lo que mostraba que la protección había cambiado de la cookie a la IP de origen.

### 3.3 Metodología

1. Se interceptó la petición de voto con Burp y se identificó que el control ahora se basaba en la IP de origen informada al servidor.
2. Se probó agregar el encabezado `X-Forwarded-For` con un valor de IP distinto al de origen real, y el sistema aceptó el voto como si proviniera de esa IP.
3. Se configuró Burp Intruder para incrementar de forma automática ciertos octetos de la IP indicada en el encabezado `X-Forwarded-For` en cada envío.
4. Se dejó correr el ataque hasta que la cantidad de votos de la UTN superó a la de Harvard.

### 3.4 Resultado

Se logró superar en votos a Harvard y se completó el laboratorio.

---

## 4. Laboratorio "El mejor secreto"

### 4.1 Objetivo

Obtener el código/patrón necesario para pasar el laboratorio a partir de información mostrada en un video.

### 4.2 Análisis previo

El enunciado incluía un video en el que se mostraba una secuencia de botones presionados. Observando el video con atención se identificó el patrón completo de botones utilizado.

### 4.3 Metodología

1. Se revisó el video cuadro por cuadro hasta reconstruir el patrón completo de botones presionados.
2. Con ese patrón identificado, se generó un diccionario de valores (combinaciones de dígitos del 0 al 9) que coincidieran con la secuencia observada.
3. Se cargó ese diccionario en Burp Intruder como lista de payloads.
4. Se ejecutó un ataque de fuerza bruta sobre el campo correspondiente, probando cada combinación del diccionario generado.
5. Al encontrar la combinación correcta, se completó el laboratorio.

### 4.4 Resultado

Se identificó la combinación correcta mediante fuerza bruta acotada y se completó el laboratorio.
