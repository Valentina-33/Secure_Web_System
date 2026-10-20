# Frontend

App en React (hecha con Vite) que inicia sesión con Auth0 y llama a la API protegida.

## Cómo correrla

1. Copia `.env.example` como `.env.local` y pon el Domain y el Client ID de tu aplicación en Auth0.
2. Instala las dependencias y arranca el servidor de desarrollo:

```bash
npm install
npm run dev
```

3. Abre `http://localhost:5173`.

Para el build de producción, crea `.env.production.local` con `VITE_API_URL=https://paula-risk.duckdns.org` y corre `npm run build`. El resultado queda en `dist/`.

Los archivos `.env.local` y `.env.production.local` no se suben al repositorio.

Más detalles en el [README principal](../README.md).
