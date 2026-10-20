# Secure Web System as an Enterprise Risk Control

This is a small web app built to practice security controls. It has three parts:

- A **React frontend** (built with Vite) where you log in with **Auth0**.
- A **Spring Boot backend** that works as an **OAuth 2.0 Resource Server**. It only answers if the request has a valid JWT from Auth0, and it checks who issued the token (`issuer`) and which API it was made for (`audience`).
- A **public HTTPS deployment**, using **Duck DNS** for the domain, **Apache** as a reverse proxy and a free **Let's Encrypt** certificate.

## Where it's deployed

**https://paula-risk.duckdns.org**

If the link doesn't load, the EC2 instance is probably stopped. We shut it down when it's not in use to save AWS lab credits. All the proof that it worked is in `docs/evidence/`.

## How it works

1. You open the frontend and click **Log in**.
2. Auth0 asks for your email and password. If they're right, it gives the browser an access token for the `https://enterprise-risk-api` API.
3. When you click **Call protected API**, the frontend calls `GET /api/risk-status` and sends the token in the `Authorization: Bearer <token>` header.
4. The backend checks the token. If it's valid, it answers with a JSON. If there's no token or it's not valid, it answers `401`.
5. On the server, everything comes in through Apache on ports 80 and 443. Apache serves the frontend and passes `/api/` requests to the backend, which only listens inside the server on `127.0.0.1:8080`. That way port 8080 is never open to the internet.

## What's in the repo

| Folder or file | What's inside |
|---|---|
| `backend/` | The Spring Boot API with Java 21 and Maven. This is where `SecurityConfig`, `RiskController` and `application.properties` live |
| `frontend/` | The React app with `@auth0/auth0-react` |
| `deploy/enterprise-risk-api.service` | The systemd service that keeps the backend running on the server (Part 5) |
| `deploy/enterprise-risk.conf` | The Apache site config (Part 6) |
| `docs/evidence/` | Screenshots and command output used as proof |

## Auth0 setup

| What | Value |
|---|---|
| API | `Enterprise Risk Lab API` |
| Identifier (audience) | `https://enterprise-risk-api` |
| Signing algorithm | `RS256` |
| Permission | `read:risk`, used to read the protected risk status |
| Browser app (SPA) | `Enterprise Risk Lab Frontend`, with **User-Delegated Access** to `read:risk` |
| Callback URLs, Logout URLs and Web Origins | `http://localhost:5173,https://paula-risk.duckdns.org` |

The SPA needs **User-Delegated Access** because it acts on behalf of the person who logged in. *Client Access* is for machine to machine apps and doesn't work for a browser login.

## Running it locally

**Backend**

```bash
cd backend
./mvnw spring-boot:run
```

If you open `http://localhost:8080/api/risk-status` without a token, you get `401 Unauthorized`. That's expected.

**Frontend**

Copy `frontend/.env.example` to `frontend/.env.local` and fill in the Domain and Client ID of the Auth0 app. This file is never pushed to the repo.

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

Then open `http://localhost:5173`, log in and click **Call protected API**.

## How it was deployed

1. **Server:** an AWS EC2 instance with Ubuntu and an Elastic IP (`98.82.207.254`). We created a `deploy` user that isn't root and turned on the `ufw` firewall. From the internet you can only get in through ports 80 and 443 for the web, and port 22 to manage the server over SSH. Port 8080 isn't open.
2. **Duck DNS:** the `paula-risk.duckdns.org` subdomain points to the Elastic IP.
3. **Packages:** installed with `sudo apt install -y apache2 certbot python3-certbot-apache openjdk-21-jre`.
4. **Build:** the frontend and backend were built on my computer with `npm run build` and `./mvnw clean package`, then copied to the server with `scp`. The production build uses `VITE_API_URL=https://paula-risk.duckdns.org`.
5. **Backend:** the JAR lives in `/opt/enterprise-risk-api/app.jar` and runs as the `riskapi` system user through `deploy/enterprise-risk-api.service`. I added `SERVER_ADDRESS=127.0.0.1` to that service so it only listens inside the server. Without that line, Spring Boot listens on every network interface and port 8080 would only be protected by the firewall.
6. **Frontend:** the contents of `dist/` went to `/var/www/enterprise-risk-frontend`.
7. **Apache:** we turned on the modules with `sudo a2enmod ssl proxy proxy_http rewrite headers`, enabled the `deploy/enterprise-risk.conf` site and disabled the default site.
8. **HTTPS:** the certificate came from `sudo certbot --apache -d paula-risk.duckdns.org`. Certbot created the port 443 config and added the redirect from HTTP to HTTPS. That redirect didn't work at first because `RewriteEngine On` was only inside the `<Directory>` block, so we also added it to the port 80 VirtualHost.
9. **Renewal:** `sudo certbot renew --dry-run` finished with no errors. Certbot also sets up automatic renewal.
10. **Auth0:** we added `https://paula-risk.duckdns.org` to Callback URLs, Logout URLs and Web Origins.

## Evidence

What the workshop asks for:

| Deliverable | File |
|---|---|
| Successful API call while logged in, on the deployed site | [authenticated-api-call.png](docs/evidence/authenticated-api-call.png) |
| Valid certificate (`openssl` and `certbot` output) | [certificate.png](docs/evidence/certificate.png) and [certificate.txt](docs/evidence/certificate.txt) |
| Part 7 checks (command output) | [verification.png](docs/evidence/verification.png) and [verification.txt](docs/evidence/verification.txt) |

Screenshots of each step, in order:

| Screenshots | What they show |
|---|---|
| 01 and 02 | The Auth0 login screen and logging in with the test user |
| 03 and 04 | The local test: logged in and calling the API from `localhost:5173` |
| 05 and 06 | The Elastic IP linked to the instance and the EC2 instance running |
| 07, 08 and 09 | The subdomain created in Duck DNS, its IP updated and the `nslookup` check |
| 10 to 14 | SSH access, installing packages, the `deploy` user and the firewall |
| 15 | Copying the files to the server with `scp` |
| 16 | The production URLs saved in Auth0 |
| 17 | The site running at `https://paula-risk.duckdns.org` |

## Checklist

| Item | Status | How we checked it |
|---|---|---|
| Without logging in, `/api/risk-status` returns 401 | Done | `verification.txt`: it returns 401 with no token and also with a fake token |
| A logged in user can call `/api/risk-status` from the frontend | Done | `authenticated-api-call.png` |
| The backend checks the JWT issuer and audience | Done | Both are set in `application.properties` and in the systemd service. A fake token gets 401 |
| The app is live at `https://paula-risk.duckdns.org` | Done | `verification.txt`: it returns 200 over HTTPS, and HTTP redirects to HTTPS with 301 |
| The certificate is valid and the browser shows a secure connection | Done | `certificate.txt`: issued by Let's Encrypt, `Verify return code: 0 (ok)`, expires on January 18, 2027 |
| Port 8080 can't be reached from the internet | Done | `verification.txt`: from outside the request times out, and on the server it only listens on `127.0.0.1` |
| `sudo certbot renew --dry-run` works | Done | `certificate.txt` |

## Risk controls

| Control | Risk it lowers | Evidence |
|---|---|---|
| Auth0 login | Someone pretending to be someone else, or passwords being handled badly | The Auth0 login and the user shown in the app |
| JWT check in the API | Someone using the API without permission | It returns 401 without a valid token |
| Audience check | A token made for another API being used on this one | The API only accepts tokens for `https://enterprise-risk-api` |
| HTTPS with a valid certificate | Someone seeing passwords or tokens while they travel over the network | The certificate details in `certificate.txt` |
| Apache as a reverse proxy | The app's port being exposed | Only ports 80 and 443 are open |
| Service logs | Taking too long to look into an incident | The `journalctl -u enterprise-risk-api` output in `verification.txt` |

**Authentication vs. authorization.** Authentication answers *who you are*. Here Auth0 does it when you log in. Authorization answers *what you're allowed to do*. Here the backend does it: it only lets requests into `/api/**` if they have a valid token, signed by the Auth0 tenant and made for this API. Every other route is blocked with `denyAll()`, except `/actuator/health`, which stays open so you can check the service is up.

## Security notes

- We never push `.env.local`, server keys, Auth0 secrets or the Duck DNS token to the repo. The screenshot that showed the Duck DNS token is left out with `.gitignore`.
- The SPA only uses the Auth0 Client ID. It never uses the Client Secret.
- CORS only accepts `http://localhost:5173` and `https://paula-risk.duckdns.org`, and only allows `GET` requests.
- Access tokens aren't shown on screen, aren't written to logs and aren't saved by hand.
- The backend runs as a system user with no extra privileges (`riskapi`), and the deployment uses a user that isn't root.
