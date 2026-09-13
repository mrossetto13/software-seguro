# Objetivo 3 - Handshake TLS

_**TLS Handshake: qué es y para qué sirve**_

El TLS Handshake es el proceso previo a cualquier comunicación HTTPS en el que el cliente y el servidor. Para esto, se ponen de acuerdo sobre cómo van a cifrar la conversación. Ocurre antes de que se envie la informacion, ya que su objetivo es establecer un canal seguro: autenticar al servidor, acordando un algoritmo de cifrado, y generando las claves con las que se van a cifrar los datos reales.

_**Cómo funciona**_

1) Client Hello: el navegador le dice al servidor qué versiones de TLS soporta, qué cipher suites (algoritmos de cifrado) puede usar y envía un número aleatorio que representa la clave de sesion.

2) Server Hello: el servidor elige la versión y cual cipher suite va a usar, envía su propia clave de sesion (que tambien es un número aleatorio), y le manda al cliente su certificado digital (que incluye su clave pública).

3) Verificación del certificado: el navegador valida que el certificado sea legítimo, comprobando que este cumpla con los siguientes:  si es emitido por una entidad de confianza, la fecha de vencimiento, coincidencia del dominio, el estado de revocación, los permisos del certificado (para qué está autorizado) y la integridad de la clave pública.

4) Intercambio de claves: cliente y servidor usan criptografía asimétrica para acordar de forma segura un secreto compartido.

5) Generación de claves de sesión: con ese secreto compartido, ambos lados derivan las mismas claves simétricas.

6) Finished: ambos confirman que todo coincide, y a partir de ahí toda la comunicación (incluida la primera petición HTTP) se cifra con esas claves simétricas.

_**El rol de los certificados digitales**_

El certificado cumple una función de autenticación, no de cifrado en sí mismo. Contiene la clave pública del servidor, el dominio al que pertenece, y está firmado digitalmente por una Autoridad Certificadora (CA) de confianza. El navegador verifica esa firma contra una lista de CAs confiables que trae preinstalada, para asegurarse de que efectivamente está hablando con el servidor real y no con un atacante haciendo un ataque man-in-the-middle. Sin el certificado, cualquiera podría hacerse pasar por el servidor y el cifrado no serviría de nada, porque estarías cifrando datos hacia un impostor.

_**¿Por qué se combinan cifrado asimétrico y simétrico?**_

Se usan los dos porque cada uno resuelve un problema distinto:

* **Asimétrico:** (clave pública/privada): permite que dos partes que nunca se comunicaron antes puedan intercambiar información de forma segura sin haber compartido un secreto previamente. Es ideal para el problema de "¿cómo nos ponemos de acuerdo en un secreto sin que alguien escuchando lo pueda robar?". Pero es computacionalmente costoso y lento si se usara para cifrar todo el tráfico.
* **Simétrico:** (misma clave para cifrar y descifrar): es mucho más rápido y eficiente para cifrar grandes volúmenes de datos, pero requiere que ambas partes ya tengan la misma clave, lo cual es un problema si no hay forma segura de compartirla de antemano.

La solución para este problema es usar lo mejor de cada uno: el cifrado asimétrico se usa solo durante el handshake, para autenticar al servidor y acordar de forma segura una clave simétrica compartida. Una vez que ambos lados tienen esa clave, se cambia a cifrado simétrico para el resto de la sesión, que es rápido y eficiente para el volumen de tráfico real (HTTP, imágenes, etc.).
