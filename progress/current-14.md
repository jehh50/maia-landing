# SESION ACTUAL — Agente 14

Feature en curso: 14 — Creación de interfaz de administración

Plan:
- Backend: extender `ensureSchema()` con tabla `users` (sin role), añadir `src/users.js` (bcryptjs hash+verify), `src/auth.js` (login/logout/me + requireAuth con JWT en cookie httpOnly), montar router en `app.js` con `cookie-parser`. Script CLI `scripts/create-user.js`.
- Tests backend: `tests/auth.test.js` con esquema temporal — 401 sin creds, 200 cookie set, /me con/sin cookie, logout, duplicado.
- Frontend: instalar `react-router-dom`, añadir router en `main.tsx` (`/`, `/admin/login`, `/admin/*`). `client/src/admin/{Login,AdminGuard,AdminLayout,AdminHome}.tsx`. Helpers `login/logout/getMe` en `lib/api.ts` con `credentials:'include'`.
- Tests RTL: `client/src/admin/__tests__/Login.test.tsx` (render, error vacíos, submit válido con mock fetch).
- Verificación: `npm test` server, `npx tsc -b && npx vitest run && npx vite build` client.
