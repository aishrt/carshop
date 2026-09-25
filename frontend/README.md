# Rento Vroom: Frontend

App 1 of 2. The Rento Vroom website: React 19 + Vite + TypeScript.

- Deployed on its own as static files on AWS S3 + CloudFront (`www.<domain>`).
- Talks to the backend only through its REST API and Socket.IO (`VITE_API_URL`).
- Holds no secrets. Only public `VITE_` values are built into the bundle.
- Shares no code with `backend/`. API types are generated from `backend/openapi.json`.

The app is set up on Days 1–2. See [IMPLEMENTATION_PLAN.md](../IMPLEMENTATION_PLAN.md), sections 1.4 (SEO), 2 (structure and commands) and 13 (deployment).
