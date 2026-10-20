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

1. **Server:** an AWS EC2 instance with Ubuntu and an Elastic IP (`98.82.207.254`). We created a `deploy` user that isn't root and turned on the `ufw` firewall. In the AWS security group, ports 80 and 443 are open to everyone for the web, and port 22 (SSH) is only open to my own IP. Port 8080 isn't open.
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

All the images are in `docs/evidence/`.

### What the workshop asks for

**Successful API call while logged in, on the deployed site.** The address bar shows `paula-risk.duckdns.org` and the API answers with `"status": "protected"` and the user's `subject`.

![Successful authenticated API call on the deployed site](docs/evidence/authenticated-api-call.png)

**Valid certificate.** Output of `openssl` and `certbot`: the certificate is for `paula-risk.duckdns.org`, it was issued by Let's Encrypt, the check returns `Verify return code: 0 (ok)` and the renewal test passes. The plain text version is in [certificate.txt](docs/evidence/certificate.txt).

![Certificate check output](docs/evidence/certificate.png)

The same certificate seen in Chrome's certificate viewer, opened from the icon next to the address bar. It's issued to `paula-risk.duckdns.org` by Let's Encrypt (YE1). The browser is set to Spanish, so the labels are in Spanish, and the dates are in Colombia time (UTC-5).

![Certificate in the browser](docs/evidence/browser-certificate.png)

**Part 7 checks.** Output of the commands used for the checklist: 401 without a token, 301 from HTTP to HTTPS, port 8080 closed from the internet, the backend listening only on `127.0.0.1`, the firewall rules, the services running and the service logs. The plain text version is in [verification.txt](docs/evidence/verification.txt).

![Part 7 checks output](docs/evidence/verification.png)

### Part 1: Auth0 setup

The `Enterprise Risk Lab API` with its identifier `https://enterprise-risk-api` and the `read:risk` permission ("Read the protected risk status").

![Auth0 API with the read:risk permission](docs/evidence/00c-auth0-api-permissions.png)

The API was created with the RS256 signing algorithm. You can also tell from the backend: Spring Boot only accepts RS256 tokens by default (`spring.security.oauth2.resourceserver.jwt.jws-algorithms` defaults to `RS256`), and we didn't change that, so the successful API call means the token was signed with RS256.

The SPA created in Auth0, with its Domain and Client ID. The Client Secret stays hidden and isn't used anywhere.

![Auth0 SPA settings](docs/evidence/00a-auth0-spa-settings.png)

User-Delegated Access from the SPA to the API, with the `read:risk` permission granted.

![User-Delegated Access with read:risk](docs/evidence/00b-auth0-user-delegated-access.png)

### Parts 2 and 3: testing it locally

The Auth0 login screen for the app, and logging in with the test user.

![Auth0 login screen](docs/evidence/01-auth0-login-page.png)

![Logging in with the test user](docs/evidence/02-auth0-login-test-user.png)

Back in the app at `localhost:5173` after logging in, and the API call working locally.

![Logged in locally](docs/evidence/03-local-signed-in.png)

![Authenticated API call locally](docs/evidence/04-local-authenticated-api-call.png)

### Part 4: server and Duck DNS

The Elastic IP `98.82.207.254` linked to the instance, and the EC2 instance running.

![Elastic IP linked to the instance](docs/evidence/05-aws-elastic-ip.png)

![EC2 instance running](docs/evidence/06-aws-ec2-instance.png)

The `paula-risk` subdomain created in Duck DNS, its IP set to the Elastic IP, and `nslookup` showing it resolves to that IP.

![Subdomain added in Duck DNS](docs/evidence/07-duckdns-domain-added.png)

![Duck DNS IP updated](docs/evidence/08-duckdns-ip-updated.png)

![nslookup for the subdomain](docs/evidence/09-nslookup-duckdns.png)

Logging in to the server over SSH and installing the packages.

![SSH into the server](docs/evidence/10-ssh-server.png)

![Installing the packages](docs/evidence/11-apt-install.png)

![Packages installed](docs/evidence/12-apt-install-done.png)

Creating the `deploy` user (not root) and turning on the firewall with only SSH, 80 and 443 allowed.

![deploy user and firewall](docs/evidence/13-deploy-user-and-ufw.png)

![SSH into the server again](docs/evidence/14-ssh-server-again.png)

The inbound rules of the instance's security group in AWS: 80 and 443 open to everyone, 22 only from my IP, and nothing for 8080.

![Security group inbound rules](docs/evidence/14b-aws-security-group.png)

### Parts 5 and 6: deploying

Copying the frontend build, the backend JAR and the config files to the server with `scp`. This is the first copy. The first frontend build still had the example values from `.env.production.local`, so after fixing them we rebuilt it and copied it again the same way. The old file was removed from the server.

![Copying the files with scp](docs/evidence/15-scp-artifacts.png)

The production URL added to Callback URLs, Logout URLs and Web Origins in Auth0.

![Production URLs in Auth0](docs/evidence/16-auth0-production-urls.png)

The site running at `https://paula-risk.duckdns.org`, before logging in.

![Deployed site before logging in](docs/evidence/17-production-login.png)

## Checklist

| Item | Status | How we checked it |
|---|---|---|
| Without logging in, `/api/risk-status` returns 401 | Done | `verification.txt`: it returns 401 with no token and also with a fake token |
| A logged in user can call `/api/risk-status` from the frontend | Done | `authenticated-api-call.png` |
| The backend checks the JWT issuer and audience | Done | Both are set in `application.properties` and in the systemd service. A fake token gets 401 |
| The app is live at `https://paula-risk.duckdns.org` | Done | `verification.txt`: it returns 200 over HTTPS, and HTTP redirects to HTTPS with 301 |
| The certificate is valid and the browser shows a secure connection | Done | `browser-certificate.png` shows it in Chrome. `certificate.txt`: issued by Let's Encrypt, `Verify return code: 0 (ok)`, expires on January 18, 2027 (UTC) |
| Port 8080 can't be reached from the internet | Done | `14b-aws-security-group.png`: there's no rule for 8080. `verification.txt`: from outside the request times out, and on the server it only listens on `127.0.0.1` |
| `sudo certbot renew --dry-run` works | Done | `certificate.txt` |

## Risk controls

| Control | Risk it lowers | Evidence |
|---|---|---|
| Auth0 login | Someone pretending to be someone else, or passwords being handled badly | The Auth0 login and the user shown in the app |
| JWT check in the API | Someone using the API without permission | It returns 401 without a valid token |
| Audience check | A token made for another API being used on this one | The API only accepts tokens for `https://enterprise-risk-api` |
| HTTPS with a valid certificate | Someone seeing passwords or tokens while they travel over the network | The certificate in `browser-certificate.png` and `certificate.txt` |
| Apache as a reverse proxy | The app's port being exposed | Only ports 80 and 443 are open to the public (`14b-aws-security-group.png`) |
| Service logs | Taking too long to look into an incident | The `journalctl -u enterprise-risk-api` output in `verification.txt` |

**Authentication vs. authorization.** Authentication answers *who you are*. Here Auth0 does it when you log in. Authorization answers *what you're allowed to do*. Here the backend does it: it only lets requests into `/api/**` if they have a valid token, signed by the Auth0 tenant and made for this API. Every other route is blocked with `denyAll()`, except `/actuator/health`, which stays open so you can check the service is up.

## Security notes

- We never push `.env.local`, server keys, Auth0 secrets or the Duck DNS token to the repo. The screenshot that showed the Duck DNS token is left out with `.gitignore`.
- The SPA only uses the Auth0 Client ID. It never uses the Client Secret.
- CORS only accepts `http://localhost:5173` and `https://paula-risk.duckdns.org`, and only allows `GET` requests.
- Access tokens aren't shown on screen, aren't written to logs and aren't saved by hand.
- The backend runs as a system user with no extra privileges (`riskapi`), and the deployment uses a user that isn't root.
