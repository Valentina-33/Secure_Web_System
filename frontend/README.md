# Frontend

React app (built with Vite) that logs you in with Auth0 and calls the protected API.

## How to run it

1. Copy `.env.example` to `.env.local` and fill in the Domain and Client ID of your Auth0 app.
2. Install the dependencies and start the dev server:

```bash
npm install
npm run dev
```

3. Open `http://localhost:5173`.

For the production build, create `.env.production.local` with `VITE_API_URL=https://paula-risk.duckdns.org` and run `npm run build`. The output goes to `dist/`.

The `.env.local` and `.env.production.local` files are never pushed to the repo.

More details in the [main README](../README.md).
