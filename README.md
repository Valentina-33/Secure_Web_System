# Secure Web System as an Enterprise Risk Control

Sistema web pequeño que se protege con controles de riesgo empresarial:

- **Frontend React** (Vite) con autenticación de usuarios mediante **Auth0**.
- **Backend Spring Boot** protegido como **OAuth 2.0 Resource Server**: valida el JWT emitido por Auth0, su emisor (`issuer`) y su audiencia (`audience`).
- **Despliegue público por HTTPS** con **Duck DNS**, **Apache HTTP Server** como proxy inverso y un certificado **Let's Encrypt**.

## URL desplegada

**https://paula-risk.duckdns.org**

## Arquitectura

```
Browser
  │ login and access token
  ▼
Auth0 ───────────────► React frontend ── Bearer token ──► Spring Boot API
                                                        │
Internet ── Duck DNS ── Apache + TLS ──────────────────┘
```

El navegador inicia sesión en Auth0 y obtiene un access token con audiencia `https://enterprise-risk-api`. El frontend envía ese token en la cabecera `Authorization: Bearer <token>` a `GET /api/risk-status`. En producción, Apache sirve el frontend y reenvía `/api/` a `127.0.0.1:8080`, de modo que el puerto 8080 nunca queda expuesto a internet.

## Estructura del repositorio

| Ruta | Contenido |
|------|-----------|
| `backend/` | API Spring Boot (Java 21, Maven): `SecurityConfig`, `RiskController` y `application.properties` |
| `frontend/` | Cliente React (Vite) con `@auth0/auth0-react` |
| `deploy/enterprise-risk-api.service` | Unidad systemd del backend (Parte 5) |
| `deploy/enterprise-risk.conf` | Virtual host de Apache (Parte 6) |
| `docs/evidence/` | Capturas de evidencia |

## Configuración de Auth0

| Elemento | Valor |
|----------|-------|
| API | `Enterprise Risk Lab API` |
| Identifier (audience) | `https://enterprise-risk-api` |
| Signing algorithm | `RS256` |
| Permission | `read:risk`: Read the protected risk status |
| SPA | `Enterprise Risk Lab Frontend` con **User-Delegated Access** a `read:risk` |
| Callback / Logout URLs / Web Origins | `http://localhost:5173,https://paula-risk.duckdns.org` |

## Ejecución local

**Backend**

```bash
cd backend
./mvnw spring-boot:run
```

`http://localhost:8080/api/risk-status` responde `401 Unauthorized` sin token, como se espera.

**Frontend**

Crea `frontend/.env.local` a partir de `frontend/.env.example`. Ese archivo no se versiona.

```
VITE_AUTH0_DOMAIN=YOUR_TENANT.us.auth0.com
VITE_AUTH0_CLIENT_ID=YOUR_CLIENT_ID
VITE_AUTH0_AUDIENCE=https://enterprise-risk-api
VITE_API_URL=http://localhost:8080
```

```bash
cd frontend
npm install
npm run dev
```

## Despliegue

1. **Servidor:** Ubuntu con un usuario de despliegue que no sea root, firewall activo y solo los puertos TCP 80 y 443 abiertos.
2. **Duck DNS:** el subdominio apunta a la IP pública del servidor.
3. **Paquetes:** `sudo apt install -y apache2 certbot python3-certbot-apache openjdk-21-jre`.
4. **Backend:** el JAR va en `/opt/enterprise-risk-api/app.jar` y corre con el usuario de sistema `riskapi` mediante `deploy/enterprise-risk-api.service`, que lo hace escuchar solo en `127.0.0.1:8080` (`SERVER_ADDRESS=127.0.0.1`).
5. **Frontend:** `dist/` se copia a `/var/www/enterprise-risk-frontend`, compilado con `VITE_API_URL=https://paula-risk.duckdns.org`.
6. **Apache y TLS:** se activa `deploy/enterprise-risk.conf` con `a2enmod ssl proxy proxy_http rewrite headers` y luego se ejecuta `sudo certbot --apache -d paula-risk.duckdns.org`. Certbot crea `enterprise-risk-le-ssl.conf` para el puerto 443 y agrega la redirección HTTP → HTTPS al VirtualHost del puerto 80. En ese VirtualHost hubo que añadir `RewriteEngine On` a nivel de VirtualHost: sin esa línea, la redirección no se aplicaba, porque el motor de rewrite solo estaba activo dentro de `<Directory>`.
7. **Renovación:** `sudo certbot renew --dry-run` termina sin errores.

## Evidencias

| Entregable | Archivo |
|---|---|
| Llamada autenticada exitosa a la API desplegada | [`docs/evidence/authenticated-api-call.png`](docs/evidence/authenticated-api-call.png) |
| Certificado válido (salida de comandos) | [`docs/evidence/certificate.png`](docs/evidence/certificate.png) · [`certificate.txt`](docs/evidence/certificate.txt) |
| Verificación de la Parte 7 (salida de comandos) | [`docs/evidence/verification.png`](docs/evidence/verification.png) · [`verification.txt`](docs/evidence/verification.txt) |

Capturas del proceso, en orden:

| # | Paso |
|---|---|
| 01–02 | Login de Auth0 con el usuario de prueba |
| 03–04 | Flujo local: sesión iniciada y llamada autenticada desde `localhost:5173` |
| 05–06 | Instancia EC2 y Elastic IP |
| 07–09 | Subdominio de Duck DNS apuntando a la Elastic IP y resolución con `nslookup` |
| 10–14 | Acceso SSH, instalación de paquetes, usuario `deploy` y firewall `ufw` |
| 15 | Copia de los artefactos al servidor con `scp` |
| 16 | URLs de producción en Auth0 (Callback, Logout y Web Origins) |
| 17 | Aplicación servida en `https://paula-risk.duckdns.org` |

## Lista de verificación

- [x] Las peticiones sin autenticar a `/api/risk-status` devuelven 401.
- [x] Un usuario autenticado puede llamar a `/api/risk-status` desde el frontend.
- [x] El backend valida el emisor y la audiencia del JWT.
- [x] La aplicación está disponible en `https://paula-risk.duckdns.org`.
- [x] El navegador muestra un certificado válido y el candado de HTTPS.
- [x] El puerto 8080 no es accesible desde internet.
- [x] `sudo certbot renew --dry-run` termina sin errores.

## Reflexión sobre controles de riesgo

| Control | Riesgo que reduce | Evidencia |
|---------|-------------------|-----------|
| Autenticación con Auth0 | Suplantación de identidad y manejo débil de credenciales | Flujo de login de Auth0 e identidad del usuario |
| Validación del JWT en la API | Acceso no autorizado a la API | 401 sin un token válido |
| Validación de audiencia | Reutilización de un token contra otra API | La API solo acepta su audiencia (`https://enterprise-risk-api`) |
| HTTPS con certificado válido | Exposición de credenciales o tokens en tránsito | Detalles del certificado en el navegador |
| Proxy inverso Apache | Exposición directa del puerto de la aplicación | Solo los puertos 80 y 443 son públicos |
| Logs centralizados del servicio | Investigación tardía de incidentes | Salida de `journalctl -u enterprise-risk-api` |

**Autenticación y autorización.** La autenticación responde a *quién es* el usuario; aquí la realiza Auth0 con el login. La autorización responde a *qué puede hacer*: el backend solo deja pasar a `/api/**` las peticiones que traen un access token válido, firmado por el tenant de Auth0 y emitido para la audiencia de esta API. Cualquier otra ruta se rechaza con `denyAll()`.

## Notas de seguridad

- `.env.local`, las credenciales del servidor, los secretos de Auth0 y los tokens de Duck DNS nunca se suben al repositorio.
- La SPA usa solo el Client ID de Auth0, nunca un client secret.
- CORS está restringido a los orígenes exactos de desarrollo y producción, y solo permite `GET`.
- Los access tokens no se muestran, no se registran en logs y no se guardan manualmente.
