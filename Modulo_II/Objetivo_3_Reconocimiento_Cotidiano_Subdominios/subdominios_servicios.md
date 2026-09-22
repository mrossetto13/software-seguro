# Servicios reales consumidos habitualmente bajo arquitectura de subdominios

La arquitectura de subdominios permite a una organización alojar múltiples aplicaciones o separar clientes (tenants) bajo un mismo dominio raíz, sin necesidad de registrar un dominio distinto para cada servicio. A continuación se listan cinco servicios de uso diario, destacando la porción de la URL correspondiente al subdominio.

## 1. Gmail

**URL completa:** `https://mail.google.com`

`**mail**.google.com`

El subdominio `mail` identifica el producto de correo electrónico dentro del dominio raíz `google.com`.

## 2. WhatsApp Web

**URL completa:** `https://web.whatsapp.com`

`**web**.whatsapp.com`

El subdominio `web` distingue la versión de escritorio/navegador de la aplicación, separándola del dominio raíz `whatsapp.com` (usado por la app móvil y el sitio institucional).

## 3. Google Docs

**URL completa:** `https://docs.google.com`

`**docs**.google.com`

El subdominio `docs` identifica el producto de edición de documentos dentro del ecosistema de `google.com`, siguiendo el mismo patrón que `mail` o `drive`.

## 4. Udemy Business (licencia Globant)

**URL completa:** `https://globant.udemy.com`

`**globant**.udemy.com`

En este caso el subdominio no identifica un producto, sino un **tenant/cliente** dentro de una plataforma multi-tenant: cada empresa que contrata Udemy Business obtiene su propio subdominio (`globant`, `empresa2`, etc.), lo que aísla el contenido y los usuarios de cada organización dentro del mismo dominio raíz `udemy.com`.

## 5. YouTube

**URL completa:** `https://www.youtube.com`

`**www**.youtube.com`

El subdominio `www` es la convención histórica de "world wide web" utilizada por la mayoría de los sitios como punto de entrada principal, a diferencia de los casos anteriores donde el subdominio identifica un producto o tenant específico.

## Resumen

| # | Servicio | Subdominio | Dominio raíz | Qué identifica |
|---|---|---|---|---|
| 1 | Gmail | `mail` | google.com | Producto |
| 2 | WhatsApp Web | `web` | whatsapp.com | Plataforma (web vs. app) |
| 3 | Google Docs | `docs` | google.com | Producto |
| 4 | Udemy Business (Globant) | `globant` | udemy.com | Tenant/cliente (multi-tenancy) |
| 5 | YouTube | `www` | youtube.com | Punto de entrada estándar |
