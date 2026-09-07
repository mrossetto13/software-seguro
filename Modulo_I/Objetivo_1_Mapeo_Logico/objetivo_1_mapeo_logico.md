# Objetivo 1 - Mapeo Lógico

## Descripción
<img width="590" height="666" alt="Diagrama de Flujo de draw io" src="https://github.com/user-attachments/assets/1cfe44fc-1af5-4e38-980a-9eac07dc460b" />

Cuando el usuario hace click, completa un formulario o interactua con el front-end, el sistema captura ese evento y arma una petición, con los datos en JSON, define que metodo usar (get, post, etc) y la URL. 
El Back-end es el que traduce lo que le envia el front-ed, resuelve la consulta y consulta si hace falta con la base de datos. 
El tramo protegido por el protocolo HTTPS es el que se encuentra entre el Front-end y el Back-end. Lo que sucee es que cuando el Front-end envia la petición, esta se cifra usando el TLS, este cifrado ocurre cuando el Front-end envia o cuando recibe la informacion del Back-end.
Luego de que el Back-end envie la información, el Front-end la "traduce", la procesa y la plasma en la web 
