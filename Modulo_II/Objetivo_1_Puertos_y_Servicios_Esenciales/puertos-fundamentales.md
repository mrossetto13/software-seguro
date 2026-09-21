# Puertos fundamentales y sus servicios por defecto

## Lista de puertos

### Puerto 20: FTP (datos)
- **Protocolo:** TCP
- **Servicio:** File Transfer Protocol, canal de datos.
- **Qué hace:** transporta los archivos y listados de directorios entre cliente y servidor FTP (en modo activo).
- **Seguridad:** el tráfico va en texto plano. Preferir SFTP o FTPS.

### Puerto 21: FTP (control)
- **Protocolo:** TCP
- **Servicio:** File Transfer Protocol, canal de control.
- **Qué hace:** gestiona la sesión FTP: autenticación y comandos (`LIST`, `GET`, `PUT`, etc.).
- **Seguridad:** las credenciales viajan sin cifrar. Es un objetivo frecuente de fuerza bruta y de acceso anónimo mal configurado.

### Puerto 22: SSH
- **Protocolo:** TCP
- **Servicio:** Secure Shell.
- **Qué hace:** acceso remoto cifrado a una terminal. También soporta transferencia de archivos (SFTP/SCP) y túneles.
- **Seguridad:** usar autenticación por clave, deshabilitar login de root y considerar cambiar el puerto o limitar accesos.

### Puerto 23: Telnet
- **Protocolo:** TCP
- **Servicio:** Telnet.
- **Qué hace:** acceso remoto a una terminal, sin cifrado.
- **Seguridad:** obsoleto e inseguro (usuario y contraseña en texto plano). Reemplazado por SSH. Si aparece abierto, es un hallazgo crítico.

### Puerto 25: SMTP
- **Protocolo:** TCP
- **Servicio:** Simple Mail Transfer Protocol.
- **Qué hace:** envío de correo y transferencia entre servidores de correo.
- **Seguridad:** mal configurado puede funcionar como *open relay* (envío de spam). Muchos ISP lo bloquean para usuarios finales. Para clientes se usan 587 (submission) o 465 (SMTPS).

### Puerto 53: DNS
- **Protocolo:** UDP y TCP
- **Servicio:** Domain Name System.
- **Qué hace:** traduce nombres de dominio a direcciones IP. Usa UDP para consultas normales y TCP para respuestas grandes y transferencias de zona.
- **Seguridad:** vigilar las transferencias de zona (`AXFR`) abiertas y los ataques de amplificación DDoS.

### Puerto 80: HTTP
- **Protocolo:** TCP
- **Servicio:** Hypertext Transfer Protocol.
- **Qué hace:** tráfico web sin cifrar.
- **Seguridad:** todo viaja en texto plano. Lo habitual es redirigir a HTTPS.

### Puerto 110: POP3
- **Protocolo:** TCP
- **Servicio:** Post Office Protocol v3.
- **Qué hace:** descarga correo desde el servidor al cliente (normalmente lo elimina del servidor).
- **Seguridad:** sin cifrado por defecto. Existe la variante segura POP3S en el puerto 995.

### Puerto 443: HTTPS
- **Protocolo:** TCP (y UDP para HTTP/3 con QUIC)
- **Servicio:** HTTP sobre TLS/SSL.
- **Qué hace:** tráfico web cifrado.
- **Seguridad:** el estándar actual para la web. Verificar certificados válidos y versiones de TLS modernas.

### Puerto 3306: MySQL / MariaDB
- **Protocolo:** TCP
- **Servicio:** base de datos MySQL y MariaDB.
- **Qué hace:** conexiones de clientes y aplicaciones al motor de base de datos.
- **Seguridad:** no debería estar expuesto a Internet. Restringir por firewall o red interna y evitar credenciales por defecto.

---

## Resumen

| Puerto | Protocolo | Servicio | Cifrado por defecto |
|--------|-----------|----------|---------------------|
| 20 | TCP | FTP (datos) | No |
| 21 | TCP | FTP (control) | No |
| 22 | TCP | SSH / SFTP | Sí |
| 23 | TCP | Telnet | No |
| 25 | TCP | SMTP | No (STARTTLS opcional) |
| 53 | UDP/TCP | DNS | No |
| 80 | TCP | HTTP | No |
| 110 | TCP | POP3 | No |
| 443 | TCP/UDP | HTTPS | Sí |
| 3306 | TCP | MySQL / MariaDB | Opcional (TLS) |

> **Nota:** Estos puertos y sus usos son *por defecto* según la convención de la IANA. Cualquier servicio puede configurarse para correr en otro puerto, así que un puerto abierto no garantiza qué servicio hay detrás.
