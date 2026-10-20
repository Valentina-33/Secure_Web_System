# Secure Web System as an Enterprise Risk Control

Este proyecto es un sistema web pequeño hecho para practicar controles de seguridad. Tiene tres partes:

- Un **frontend en React** (hecho con Vite) donde la persona inicia sesión con **Auth0**.
- Un **backend en Spring Boot** que funciona como **OAuth 2.0 Resource Server**. Solo responde si la petición trae un token JWT válido de Auth0, y revisa quién lo emitió (`issuer`) y para qué API es (`audience`).
- Un **despliegue público con HTTPS**, usando **Duck DNS** para el dominio, **Apache** como proxy inverso y un certificado gratuito de **Let's Encrypt**.

## Dónde está desplegado

**https://paula-risk.duckdns.org**

## Cómo funciona

1. La persona entra al frontend y le da clic a **Log in**.
2. Auth0 le pide su correo y contraseña. Si son correctos, le entrega al navegador un access token para la API `https://enterprise-risk-api`.
3. Cuando la persona le da clic a **Call protected API**, el frontend llama a `GET /api/risk-status` y manda el token en la cabecera `Authorization: Bearer <token>`.
4. El backend revisa el token. Si es válido, responde con un JSON. Si no hay token o no sirve, responde `401`.
5. En el servidor, todo entra por Apache en los puertos 80 y 443. Apache entrega el frontend y le pasa las peticiones de `/api/` al backend, que solo escucha dentro del servidor en `127.0.0.1:8080`. Así el puerto 8080 nunca queda abierto a internet.

## Qué hay en el repositorio

| Carpeta o archivo | Qué contiene |
|---|---|
| `backend/` | La API en Spring Boot con Java 21 y Maven. Ahí están `SecurityConfig`, `RiskController` y `application.properties` |
| `frontend/` | La app en React con `@auth0/auth0-react` |
| `deploy/enterprise-risk-api.service` | El servicio de systemd que deja corriendo el backend en el servidor (Parte 5) |
| `deploy/enterprise-risk.conf` | La configuración del sitio en Apache (Parte 6) |
| `docs/evidence/` | Las capturas y las salidas de comandos que sirven de evidencia |

## Configuración en Auth0

| Qué | Valor |
|---|---|
| API | `Enterprise Risk Lab API` |
| Identifier (audience) | `https://enterprise-risk-api` |
| Algoritmo de firma | `RS256` |
| Permiso | `read:risk`, que sirve para leer el estado de riesgo protegido |
| Aplicación del navegador (SPA) | `Enterprise Risk Lab Frontend`, con **User-Delegated Access** a `read:risk` |
| Callback URLs, Logout URLs y Web Origins | `http://localhost:5173,https://paula-risk.duckdns.org` |

La SPA necesita **User-Delegated Access** porque actúa en nombre de la persona que inició sesión. El acceso tipo *Client Access* es para comunicación entre máquinas y no sirve para un login desde el navegador.

## Cómo correrlo en local

**Backend**

```bash
cd backend
./mvnw spring-boot:run
```

Si abres `http://localhost:8080/api/risk-status` sin token, responde `401 Unauthorized`. Eso es lo esperado.

**Frontend**

Copia `frontend/.env.example` como `frontend/.env.local` y llénalo con el Domain y el Client ID de la aplicación en Auth0. Este archivo no se sube al repositorio.

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

Luego abres `http://localhost:5173`, inicias sesión y le das clic a **Call protected API**.

## Cómo se desplegó

1. **Servidor:** una instancia EC2 de AWS con Ubuntu y una Elastic IP (`98.82.207.254`). Se creó el usuario `deploy`, que no es root, y se activó el firewall `ufw`. Desde internet solo se puede entrar por los puertos 80 y 443 para la web, y por el 22 para administrar el servidor por SSH. El 8080 no está abierto.
2. **Duck DNS:** el subdominio `paula-risk.duckdns.org` apunta a la Elastic IP.
3. **Paquetes:** se instalaron con `sudo apt install -y apache2 certbot python3-certbot-apache openjdk-21-jre`.
4. **Compilación:** el frontend y el backend se compilaron en mi computador con `npm run build` y `./mvnw clean package`, y se copiaron al servidor con `scp`. Para el build de producción se usó `VITE_API_URL=https://paula-risk.duckdns.org`.
5. **Backend:** el JAR quedó en `/opt/enterprise-risk-api/app.jar` y corre con el usuario de sistema `riskapi` usando `deploy/enterprise-risk-api.service`. A ese servicio le agregué `SERVER_ADDRESS=127.0.0.1` para que escuche solo dentro del servidor. Sin esa línea, Spring Boot escucha en todas las interfaces y el puerto 8080 solo quedaría protegido por el firewall.
6. **Frontend:** el contenido de `dist/` quedó en `/var/www/enterprise-risk-frontend`.
7. **Apache:** se activaron los módulos con `sudo a2enmod ssl proxy proxy_http rewrite headers`, se habilitó el sitio `deploy/enterprise-risk.conf` y se deshabilitó el sitio por defecto.
8. **HTTPS:** el certificado se sacó con `sudo certbot --apache -d paula-risk.duckdns.org`. Certbot creó la configuración del puerto 443 y agregó la redirección de HTTP a HTTPS. Esa redirección no funcionaba porque `RewriteEngine On` solo estaba dentro del bloque `<Directory>`, así que se agregó también a nivel del VirtualHost del puerto 80.
9. **Renovación:** `sudo certbot renew --dry-run` terminó sin errores. Certbot deja programada la renovación automática.
10. **Auth0:** se agregó `https://paula-risk.duckdns.org` en Callback URLs, Logout URLs y Web Origins.

## Evidencias

Las evidencias que pide el taller:

| Entregable | Archivo |
|---|---|
| Llamada exitosa a la API con sesión iniciada, en el sitio desplegado | [authenticated-api-call.png](docs/evidence/authenticated-api-call.png) |
| Certificado válido (salida de `openssl` y `certbot`) | [certificate.png](docs/evidence/certificate.png) y [certificate.txt](docs/evidence/certificate.txt) |
| Revisión de la Parte 7 (salida de los comandos) | [verification.png](docs/evidence/verification.png) y [verification.txt](docs/evidence/verification.txt) |

Capturas de cada paso, en orden:

| Capturas | Qué muestran |
|---|---|
| 01 y 02 | La pantalla de login de Auth0 y el inicio de sesión con el usuario de prueba |
| 03 y 04 | La prueba en local: sesión iniciada y llamada a la API desde `localhost:5173` |
| 05 y 06 | La Elastic IP asociada y la instancia EC2 corriendo |
| 07, 08 y 09 | El subdominio creado en Duck DNS, su IP actualizada y la prueba con `nslookup` |
| 10 a 14 | La conexión por SSH, la instalación de paquetes, el usuario `deploy` y el firewall |
| 15 | La copia de los archivos al servidor con `scp` |
| 16 | Las URLs de producción guardadas en Auth0 |
| 17 | El sitio funcionando en `https://paula-risk.duckdns.org` |

## Lista de verificación

| Punto | Estado | Cómo se comprobó |
|---|---|---|
| Sin login, `/api/risk-status` responde 401 | Listo | `verification.txt`: responde 401 sin token y también con un token falso |
| Con sesión iniciada se puede llamar a `/api/risk-status` desde el frontend | Listo | `authenticated-api-call.png` |
| El backend revisa el issuer y la audiencia del JWT | Listo | `application.properties` y el servicio de systemd configuran los dos. Un token falso recibe 401 |
| La app está en `https://paula-risk.duckdns.org` | Listo | `verification.txt`: responde 200 por HTTPS, y HTTP redirige a HTTPS con 301 |
| El certificado es válido y el navegador muestra la conexión segura | Listo | `certificate.txt`: emitido por Let's Encrypt, `Verify return code: 0 (ok)`, vence el 18 de enero de 2027 |
| El puerto 8080 no se puede usar desde internet | Listo | `verification.txt`: desde afuera se agota el tiempo de espera, y en el servidor escucha solo en `127.0.0.1` |
| `sudo certbot renew --dry-run` funciona | Listo | `certificate.txt` |

## Controles de riesgo

| Control | Qué riesgo baja | Evidencia |
|---|---|---|
| Login con Auth0 | Que alguien se haga pasar por otra persona o que las contraseñas se manejen mal | El login de Auth0 y el usuario que aparece en la app |
| Validación del JWT en la API | Que alguien use la API sin permiso | Responde 401 si no hay un token válido |
| Validación de la audiencia | Que un token hecho para otra API se use en esta | La API solo acepta tokens para `https://enterprise-risk-api` |
| HTTPS con certificado válido | Que alguien vea contraseñas o tokens mientras viajan por la red | Los datos del certificado en `certificate.txt` |
| Apache como proxy inverso | Que el puerto de la aplicación quede expuesto | Solo los puertos 80 y 443 están abiertos |
| Logs del servicio | Tardar mucho en investigar un incidente | La salida de `journalctl -u enterprise-risk-api` en `verification.txt` |

**Diferencia entre autenticación y autorización.** La autenticación responde *quién eres*. Aquí la hace Auth0 cuando la persona inicia sesión. La autorización responde *qué puedes hacer*. Aquí la hace el backend: solo deja pasar a `/api/**` las peticiones con un token válido, firmado por el tenant de Auth0 y hecho para esta API. Cualquier otra ruta se bloquea con `denyAll()`, menos `/actuator/health`, que queda abierta para revisar que el servicio está vivo.

## Notas de seguridad

- Nunca se suben al repositorio `.env.local`, las llaves del servidor, secretos de Auth0 ni el token de Duck DNS. La captura donde se veía el token de Duck DNS se dejó por fuera con `.gitignore`.
- La SPA usa solo el Client ID de Auth0. Nunca usa el Client Secret.
- CORS solo acepta `http://localhost:5173` y `https://paula-risk.duckdns.org`, y solo permite peticiones `GET`.
- Los access tokens no se muestran en pantalla, no se escriben en los logs y no se guardan a mano.
- El backend corre con un usuario de sistema sin privilegios (`riskapi`) y el despliegue se hace con un usuario que no es root.
