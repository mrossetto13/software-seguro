# Objetivo 4 - Instalación Proxy

## Descripción
_Contenido pendiente._

# Burp:

## Intruder y ejecución del mismo
<img width="1919" height="999" alt="Burp - Intruder" src="https://github.com/user-attachments/assets/194a3e70-9bfc-49c6-bb8b-c30e585a4c7d" />

# Qué es un proxy y cuál es la diferencia con una VPN

Un proxy es un servidor intermediario entre tu dispositivo e Internet. En lugar de conectarte directo a un sitio, le pedís al proxy que haga la solicitud por vos. El sitio ve la IP del proxy y no la tuya.

Hay varios tipos:

Forward proxy: el que usás vos como cliente, por ejemplo para salir a Internet desde una red corporativa o cambiar tu IP aparente.
Reverse proxy: se ubica delante de un servidor para balancear carga, cachear o proteger el backend.

Un proxy suele funcionar a nivel de aplicación: solo redirige el tráfico de las apps configuradas para usarlo (un navegador, una herramienta como Burp Suite o curl). Normalmente no cifra el tráfico por sí mismo, salvo que la conexión ya use HTTPS.

Qué es una VPN

Una VPN (Virtual Private Network) crea un túnel cifrado entre tu dispositivo y un servidor VPN, y suele funcionar a nivel de sistema operativo o red. Todo el tráfico del dispositivo, de todas las apps, pasa por ese túnel. Quien está en el medio solo ve que te conectás a la VPN, no lo que hacés adentro.
