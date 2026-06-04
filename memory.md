# Memory — MaIA Landing Page

## 2026-05-26 — Análisis inicial del proyecto

**Tarea ejecutada:** Análisis completo del repositorio `maia-landing`.

**Hallazgos principales:**
- El proyecto es una landing page estática de **1 880 líneas** contenida en un único `index.html` (~78 KB).
- No existe proceso de build con Vite a pesar de lo mencionado en `AGENT.md`; todo el CSS y JS está inline.
- Se creó el archivo `architecture.md` documentando la estructura completa: secciones HTML, tokens CSS, funciones JS, patrones de UI y deuda técnica.
- Los formularios de captura de leads (CTA final y ROI email) son simulados — no conectan con ningún backend.
- El logo utiliza el patrón tipográfico "ma**IA**" con "IA" en naranja (`#E8440A`).
- Breakpoints responsive en `640px` y `860px`.
- Links externos activos: Calendly para demos, WhatsApp para ventas enterprise.

**Archivos creados/modificados:**
- `architecture.md` — creado (nuevo)
- `memory.md` — actualizado (este archivo)

---

## 2026-05-26 — Feature id=2: Cambios de UI y contenido

**Tarea ejecutada:** Implementación del feature id=2 según lista de cambios entregada por el usuario.

**Cambios realizados en `index.html`:**
1. **Favicon** — Añadido `<link rel="icon">` apuntando a `docs/images/isotipo-maia.svg`.
2. **Logo SVG** — Reemplazado el logo tipográfico (HTML/CSS) por `<img src="docs/images/logo-maia.svg">` en navbar, footer y demo sidebar del hero.
3. **Navbar “Prueba gratis” → “Agenda un demo”** — Botón primario del navbar ahora apunta a Calendly.
4. **Menu móvil** — Mismo cambio de texto y destino que navbar.
5. **Hero CTA primario** — “Prueba gratis 14 días” → “Agenda un demo” (apunta a Calendly).
6. **Hero “Ver demo en vivo”** — URL actualizada a `https://www.youtube.com/watch?v=EngW7tLk6R8`.
7. **Sección Funciones** — Eliminada la feature card “Workflows visuales sin código” (quedan 5 cards).
8. **Sección Integraciones** — Eliminadas las pills: Instagram DM, Oracle, Monday (quedan 15 integraciones).
9. **Sección Precios** — Eliminados los botones “Probar gratis 14 días” en los planes Starter, Team y Growth.
10. **CTA Final** — Badge cambiado de “Sin riesgo · 14 días gratis” a “Sin riesgos”.

**Archivos creados/modificados:**
- `index.html` — modificado (10 cambios)
- `docs/architecture.md` — actualizado con historial de cambios y nueva estructura de archivos
- `feature_list.json` — feature id=2 marcado como `done`
- `memory.md` — actualizado (este archivo)

---

## 2026-05-26 — Feature id=3: Formulario de contacto + backend Node.js

**Tarea ejecutada:** Implementación del feature id=3. Se creó un backend Node.js (Express) en `/server` y un modal de formulario en la landing para capturar leads.

**Backend (`/server`):**
- Stack: Node.js 22 (ESM), Express 4, `better-sqlite3`, Nodemailer, dotenv. Tests con Vitest + Supertest.
- Endpoints:
  - `POST /api/contact` — valida email, persiste en `data/leads.db`, envía notificación SMTP (si `SMTP_HOST`). Responde `201 {ok,id}` / `422 {error}`.
  - `GET /api/health` — health check.
- Persistencia: tabla `leads (id, nombre, empresa, email, telefono, mensaje, tipo, created_at)`.
- Configuración por `.env` (ver `server/.env.example`): puerto, CORS, SMTP. Si `SMTP_HOST` está vacío el email se omite y solo se guarda el lead.
- Tests: 7 verdes (happy path, email inválido, email faltante, tipo desconocido normalizado a `demo`, tipo `contacto` preservado, health, método no permitido). Cada test usa `mkdtempSync` para una DB SQLite real por suite.

**Frontend (`index.html`):**
- Nuevo modal `#contactModal` con formulario: nombre*, empresa, email*, teléfono, mensaje, tipo (hidden).
- Cierre por: botón ✕, click en overlay, tecla `Escape`.
- Validación cliente: `EMAIL_RE` + campos requeridos, con estilos `.invalid` y mensaje `.form-status`.
- Envío con `fetch` JSON a `/api/contact`. URL configurable vía `window.MAIA_API_BASE`.
- Botones "Agenda un demo" del navbar, menú móvil, hero, pricing y CTA final ahora invocan `openContactModal('demo')` en lugar de Calendly.
- `startTrial()` (CTA final) y `sendROI()` (ROI) ahora hacen `POST /api/contact` con `tipo='email'`.
- Helper compartido `postLead(payload)`.

**Cambios estructurales:**
- Eliminado el endpoint PHP `api/contact.php` previo (no documentado, sustituido por la API Node).
- Reorganizada la estructura de archivos en `docs/architecture.md`.

**Verificación:**
- `npm test` en `/server` → 7/7 verdes.
- Smoke test E2E (curl): `POST` con datos válidos → 201; `POST` con email inválido → 422; `GET /api/health` → 200.
- Static serve de `index.html` desde `python3 -m http.server` → contiene `#contactModal` y `#contactForm`. Tamaño 87 458 bytes.

**Archivos creados:**
- `server/package.json`, `server/.env.example`, `server/README.md`
- `server/src/{app,server,db,email}.js`
- `server/tests/contact.test.js`

**Archivos modificados:**
- `index.html` — modal + estilos + JS reescrito
- `docs/architecture.md` — sección backend completa + estructura + historial
- `feature_list.json` — feature id=3 marcado como `done`
- `memory.md` — actualizado (este archivo)

**Archivos eliminados:**
- `api/contact.php` (y carpeta `api/` vacía)

---

## 2026-05-26 — Feature id=4: Migración SQLite → PostgreSQL externo

**Tarea ejecutada:** Reemplazar el motor de BBDD del backend de SQLite local a PostgreSQL externo, conservando el endpoint y los tests verdes.

**Cambios principales:**

1. **Dependencias:**
   - Eliminado `better-sqlite3`.
   - Añadido `pg ^8.13`.
   - `package.json` bumped a `1.1.0`.

2. **`server/src/db.js`:**
   - Exporta `createPool(config)` que acepta `DATABASE_URL` (o `host/user/database`) y opción `PGSSL`.
   - Exporta `ensureSchema(pool, { schema })` — idempotente; crea esquema, tabla `leads` con `BIGSERIAL` + `TIMESTAMPTZ` y dos índices (`email`, `created_at DESC`).
   - `insertLead(pool, lead, { schema })` usa `RETURNING id`.

3. **`server/src/app.js`:**
   - Recibe `pool` y `schema` por opciones (DI para tests).
   - `POST /api/contact` y `GET /api/health` ahora son `async`.
   - Health check ejecuta `SELECT 1` y responde `503` si la DB falla.

4. **`server/src/server.js`:**
   - Llama a `ensureSchema()` antes de `listen()`. Si falla, `process.exit(1)`.

5. **Configuración (`.env.example`):**
   - Añadidos `DATABASE_URL`, `DB_SCHEMA`, `PGSSL`.
   - Default local: `postgres:///maia-landing?host=/var/run/postgresql` (peer auth Unix socket).
   - Removido `DATA_DIR` (ya no se usa).

6. **Tests (`server/tests/contact.test.js`):**
   - 8 tests verdes contra PostgreSQL real (8.13).
   - Cada run crea un esquema único `maia_test_<ts>_<rand>` con `ensureSchema()` y lo borra con `DROP SCHEMA ... CASCADE` en `afterAll`.
   - Nuevo test: `created_at` es `TIMESTAMPTZ` reciente.

**Verificación ejecutada:**
- `npm test` → 8/8 verdes (171 ms tests + 1.03 s total).
- Smoke E2E: `POST /api/contact` con payload válido → `{ ok:true, id:1 }`; con email inválido → `422`; `GET /api/health` → `{ ok:true, db:true, mailer:false }`. Fila persistida verificable en `psql -d maia-landing -c "SELECT * FROM leads"`.

**Archivos modificados:**
- `server/package.json` (dep swap)
- `server/.env.example` (nuevas variables PG)
- `server/src/db.js` (reescrito)
- `server/src/app.js` (async + DI)
- `server/src/server.js` (ensureSchema)
- `server/tests/contact.test.js` (reescrito contra PG)
- `docs/architecture.md` (stack, schema SQL, env vars, historial)
- `feature_list.json` (id=4 → `done`)
- `memory.md` (este archivo)

**Hallazgo operacional (relevante para próximas tareas):**
- PostgreSQL local en este host usa peer-auth en el socket Unix (`/var/run/postgresql`). Para que `pg` (node) se conecte sin password en local hay que usar `postgres:///DB?host=/var/run/postgresql` en `DATABASE_URL`. Para proveedores externos (Supabase, Neon, RDS) usar la URL TCP normal con `PGSSL=true`.

---

## 2026-05-26 — Feature id=5: Integración SendGrid (reemplaza Nodemailer)

**Tarea ejecutada:** Reemplazar el transporte de correo SMTP/Nodemailer por SendGrid (`@sendgrid/mail`).

**Cambios:**

1. **Dependencias:**
   - `nodemailer` eliminado.
   - `@sendgrid/mail ^8.1.6` añadido.
   - `package.json` bumped a `1.2.0`.

2. **`server/src/email.js`** — reescrito:
   - `createMailer({ apiKey, from, to, sandbox?, client? })`.
   - Si `apiKey` vacía → no-op (`enabled:false`, retorna `{ skipped:true }`).
   - Llama a `sgMail.setApiKey(apiKey)` y `sgMail.send(msg)`.
   - Construye `text` (igual que antes) + `html` con tabla escapada anti-XSS.
   - Header `replyTo: lead.email` para responder directo al prospect.
   - Soporta `mailSettings.sandboxMode` cuando `SENDGRID_SANDBOX=true`.
   - Inyección de cliente (`config.client`) para tests sin red.

3. **`server/src/app.js`** — `createMailer()` se invoca sin args (lee de `process.env` directamente).

4. **`.env.example`** — variables `SMTP_*` eliminadas; añadidas `SENDGRID_API_KEY` y `SENDGRID_SANDBOX`. La API key real que el usuario había puesto en `.env.example` fue removida y sustituida por placeholder vacío (con advertencia explícita). `.env` actualizado al mismo formato (la API key real va aquí, sin commit).

5. **Tests:**
   - Nuevo archivo `tests/email.test.js` (4 tests): disabled cuando no hay key, envío con payload correcto, sandbox mode, escape HTML.
   - Suite total: **12/12 verdes** (8 contact + 4 email).

6. **DX:**
   - `server/dev-fullstack.js` añadido (script de conveniencia): arranca un solo Express que sirve los estáticos de la landing + el API en el mismo puerto. Útil para probar el modal sin levantar Apache. No se usa en producción.

**Verificación ejecutada:**
- `npm test` → 12/12 verdes (1.15 s).
- Smoke E2E con `dev-fullstack.js`: `GET /api/health` → `{ ok:true, db:true, mailer:false }`. `POST /api/contact` con key vacía → 201 + lead guardado (envío skipped). UI completa servida en `http://localhost:3001/`.

**Archivos modificados:**
- `server/package.json` (swap dep + bump 1.2.0)
- `server/src/email.js` (reescrito)
- `server/src/app.js` (limpieza llamada)
- `server/.env.example`, `server/.env`
- `server/tests/email.test.js` (nuevo)
- `server/dev-fullstack.js` (nuevo)
- `server/README.md`
- `docs/architecture.md` (stack + sección Email + env vars + historial)
- `feature_list.json` (id=5 → `done`)
- `memory.md` (este archivo)

**⚠️ Nota de seguridad:**
El usuario había pegado una API key real (`SG.LhfF-Ti-...`) en `.env.example`. Esa key debe considerarse comprometida si el archivo ya fue commiteado a git en algún punto. **Acción recomendada para el usuario:** revocarla en el dashboard de SendGrid (Settings → API Keys → Delete) y generar una nueva, que va únicamente en `.env` (ignorado por git).

---

## 2026-05-26 — Feature id=6: Migración a Vite + React 18 + TypeScript + Material UI

**Tarea ejecutada:** Migrar el frontend monolítico (`index.html` con CSS+JS inline) a una app moderna Vite + React + TypeScript con Material UI v6, en una carpeta `/client/` aislada del backend.

**Decisiones de arquitectura (validadas con el usuario antes de empezar):**
- Ubicación: `/client/` separada de `/server/` (no en raíz).
- UI lib: **Material UI** (no CSS Modules ni Tailwind).
- El `index.html` antiguo se conserva como `legacy.html` por una iteración (para referencia visual / fallback).

**Cambios principales:**

1. **Scaffold** (`/client/`):
   - `package.json` con scripts `dev / build / preview / test / typecheck`.
   - `vite.config.ts` con `@vitejs/plugin-react` + proxy `/api` → `http://localhost:3001` en dev.
   - `tsconfig.json` estricto (Vite preset + RTL types).
   - `index.html` mínimo (Vite inyecta bundle), favicon e Inter desde Google Fonts.
   - `public/logo-maia.svg`, `public/isotipo-maia.svg` (copiados desde `docs/images/`).

2. **Theme MUI** (`src/theme/theme.ts`):
   - Palette extendida con `brand.{orange,orangeHover,orangeLight,orangeXL,orangeXXL,green,greenLight}` y `surface.{main,soft,tint}` (declaración aumentada de tipos).
   - Tipografía Inter, `borderRadius: 12`, sombras custom.
   - Sobrescribe `MuiButton`, `MuiContainer`, `MuiTextField`.

3. **Estilos globales** (`src/styles/globals.css`):
   - Reset + variables CSS (`--orange`, `--bg-soft`, ...) para uso vía `sx={{ color: 'var(--orange)' }}` cuando el theme MUI no aplica.
   - Animaciones `fadeUp`, `blink`, `reveal` (scroll-triggered).

4. **Componentización** (`src/components/`):
   - `Navbar.tsx` — AppBar fija con scroll detection + menú móvil full-screen.
   - `Footer.tsx`, `WhatsAppFloat.tsx`.
   - `ContactModal.tsx` — `Dialog` MUI con formulario controlado, validación cliente (`EMAIL_RE`), submit a `/api/contact`, copy dinámico según `tipo` (`demo` | `contacto`).
   - `sections/Hero.tsx` — counters animados con `useCounter()` + demo window mockeada.
   - `sections/{Trust,Pain,Solution,Features,Integrations,Testimonials}.tsx` — grids con MUI `Grid`.
   - `sections/ROI.tsx` — sliders MUI + cálculo en `useMemo` + envío email a `/api/contact` (`tipo='email'`).
   - `sections/Pricing.tsx` — toggle anual/mensual (estado en `App`), 4 planes, link a modal para Enterprise.
   - `sections/FAQ.tsx` — `Accordion` MUI.
   - `sections/CTAFinal.tsx` — input email + botón con `tipo='email'`, link a modal.

5. **Lógica reutilizable**:
   - `lib/api.ts`: `postLead(payload)` tipado + `EMAIL_RE`. `VITE_API_BASE` opcional para apuntar a otro host.
   - `hooks/useReveal.ts`: `IntersectionObserver` para `.reveal` (se monta una vez en `App`).

6. **Tests** (`src/components/__tests__/ContactModal.test.tsx`):
   - Vitest + jsdom + RTL + `@testing-library/user-event`.
   - 6 tests: render demo, validación vacía, email inválido, submit success (mock `fetch`), submit error 422, copy contacto.
   - Mock global de `fetch` con `vi.spyOn(globalThis, 'fetch')`.

7. **Backend dev script** (`server/dev-fullstack.js`):
   - Ahora sirve `client/dist` si existe (con SPA-fallback para rutas no-`/api`).
   - Cae a `legacy.html` si la SPA no ha sido construida.

8. **Estructura final del repo**:
   ```
   maia-landing/
   ├── legacy.html              # ex-index.html monolítico (referencia)
   ├── client/                  # Vite + React + TS + MUI
   ├── server/                  # Express + pg + SendGrid
   ├── docs/, progress/, etc.
   ```

**Verificación ejecutada:**
- `npx tsc -b` → 0 errores.
- `npx vite build` → bundle prod `460 KB` (`141 KB` gzipped). 944 módulos transformados.
- `npx vitest run` (client) → **6/6 verdes** (~3.2 s tests).
- `npm test` (server) → **12/12 verdes** (8 contact + 4 email).
- Smoke E2E con Vite dev + backend: `GET http://localhost:5173/` sirve la SPA; `POST http://localhost:5173/api/contact` se proxea correctamente al backend en `:3001` y retorna `{ ok:true, id:4 }`. Lead persistido en Postgres.

**Archivos creados:**
- `client/{package.json, vite.config.ts, tsconfig.json, index.html, README.md}`
- `client/public/{logo,isotipo}-maia.svg`
- `client/src/{main.tsx, App.tsx, vite-env.d.ts}`
- `client/src/theme/theme.ts`
- `client/src/lib/api.ts`
- `client/src/hooks/useReveal.ts`
- `client/src/styles/globals.css`
- `client/src/components/{Navbar,Footer,WhatsAppFloat,ContactModal}.tsx`
- `client/src/components/sections/{Hero,Trust,Pain,Solution,Features,Integrations,ROI,Pricing,Testimonials,FAQ,CTAFinal}.tsx`
- `client/src/components/__tests__/ContactModal.test.tsx`
- `client/src/test/setup.ts`

**Archivos modificados:**
- `index.html` → renombrado a `legacy.html`.
- `server/dev-fullstack.js` (sirve SPA build con SPA-fallback).
- `docs/architecture.md` (stack frontend nuevo, estructura, historial).
- `feature_list.json` (id=6 → `done`).
- `memory.md` (este archivo).

**Notas de seguimiento (relevantes para próximas tareas):**
- `legacy.html` puede eliminarse cuando ya no haya dudas sobre la equivalencia visual.
- Bundle de 460 KB sin code-splitting. Si interesa optimizar: lazy-load secciones below-the-fold (`React.lazy`) o splittear MUI con `@mui/material/Box` direct imports.
- Tests del cliente cubren solo el ContactModal (componente crítico). Quedan sin test: ROI calc logic, Pricing toggle, useCounter — candidatos para próximas iteraciones.

---

## 2026-05-26 — Refactor de observability del mailer (post-feature 6)

**Tarea ejecutada:** Mejorar la trazabilidad del envío de correo sin filtrar detalles al frontend. Motivo: durante una prueba del modal, el lead se guardó pero el correo no llegó; el cliente vio `{ ok:true, id:N }` y no había forma de saber si el envío había fallado.

**Cambios:**
- `src/email.js`: `sendLead()` ya no lanza; siempre devuelve `{ status: 'sent' | 'skipped' | 'failed', ... }` con detalle del provider (statusCode, reason, body).
- `src/app.js`: loggea con prefijo `[mail]` (`status=sent code=...`, `status=skipped reason=...`, `status=failed code=... reason=...`). Nunca expone esa info al cliente — la respuesta sigue siendo `{ ok, id }`.
- `tests/email.test.js`: añadido test que cubre la rama `status='failed'` cuando el provider lanza 401 con cuerpo de error.
- Decisión del usuario: ese log estructurado debe quedarse server-side. El cliente no debe ver el detalle.

**Diagnóstico encontrado durante la verificación:** SendGrid respondía 401 con `"Maximum credits exceeded"`. La API key seguía válida pero la cuenta había agotado los créditos del plan free. Causa raíz operacional, no del código.

---

## 2026-05-26 — Feature id=8: Migración SendGrid → Nodemailer/SMTP

**Tarea ejecutada:** Reemplazar el transporte de correo SendGrid por Nodemailer/SMTP. Motivo del usuario: la cuenta SendGrid no era viable a corto plazo (créditos agotados, sin upgrade inmediato); se prefiere un SMTP estándar que pueda apuntar a cualquier proveedor.

**Cambios:**

1. **Dependencias:**
   - `@sendgrid/mail` eliminado.
   - `nodemailer ^6.10` reinstalado.
   - `package.json` bumped a `1.3.0`.

2. **`server/src/email.js`** — reescrito:
   - `createMailer({ host, port, secure, user, pass, from, to, transporter? })`.
   - Si `host` vacío → no-op (`enabled:false`, `{ status: 'skipped', reason: 'SMTP_HOST no configurada' }`).
   - `nodemailer.createTransport({ host, port, secure, auth: user ? {user,pass} : undefined })`.
   - `SMTP_SECURE` acepta booleano o string `"true"` (defensa contra el parsing de `.env`).
   - **Mismo contrato de respuesta que en feature 5+observability:** `{ status: 'sent', messageId, response, accepted, rejected }` / `{ status: 'failed', statusCode, reason, body }` / `{ status: 'skipped', reason }`.
   - Builders `buildSubject/buildText/buildHtml` intactos (HTML escapado anti-XSS).
   - Inyección de `transporter` para tests sin red (`transporter: fakeTransporter`).

3. **`server/src/app.js`** — pequeño ajuste en logs:
   - Cuando `status='sent'`, prefiere `messageId=<...>` (Nodemailer) sobre `code=<...>` (SendGrid legacy). Resto del flujo idéntico.
   - El `body` del fallo puede ser string (SMTP) u objeto (HTTP); el log lo serializa solo si es objeto.

4. **`.env.example`** — variables `SENDGRID_*` eliminadas; restauradas `SMTP_HOST/PORT/SECURE/USER/PASS`. Documenta ejemplos comentados de Gmail (App Password), Brevo, Mailgun, Amazon SES, Resend.

5. **`server/.env`** — variables actualizadas a SMTP (campos vacíos por defecto).

6. **Tests** (`server/tests/email.test.js`):
   - Reescritos: 5 tests (vs 4 previos para SendGrid). Cobertura:
     - skipped sin `SMTP_HOST`.
     - sent con `transporter` mockeado (verifica `from/to/replyTo/subject/text/html`).
     - escape HTML anti-XSS.
     - failed cuando `sendMail` lanza con `responseCode: 535` (auth fallido).
     - `SMTP_SECURE="true"` interpretado como booleano.
   - `tests/contact.test.js`: el `fakeMailer` ahora devuelve `{ status, messageId, response, accepted, rejected }` (consistente con Nodemailer).

**Verificación ejecutada:**
- `npm test` → **13/13 verdes** (5 email + 8 contact, 2.5 s).
- Smoke E2E: arranqué un server temporal con `SMTP_HOST` vacío. `POST /api/contact` → `{ ok:true, id:8 }`. Log server: `[mail] lead=8 to=nm-smoke@test.com status=skipped reason="SMTP_HOST no configurada"`. Cliente nunca ve el reason.
- `GET /api/health` → `{ ok:true, db:true, mailer:false }` (mailer:false porque SMTP_HOST está vacío).

**Archivos modificados:**
- `server/package.json` (swap dep + bump 1.3.0)
- `server/src/email.js` (reescrito con nodemailer)
- `server/src/app.js` (log adaptado)
- `server/.env.example`, `server/.env`
- `server/tests/email.test.js` (5 tests reescritos)
- `server/tests/contact.test.js` (fakeMailer alineado)
- `server/README.md`
- `docs/architecture.md` (stack + sección Email + env vars + 2 entradas en historial: refactor observability + feature 8)
- `feature_list.json` (id=8 → `done`)
- `memory.md` (este archivo)

**Notas operacionales:**
- Para envíos reales el usuario debe poblar `SMTP_HOST/PORT/USER/PASS` en `.env` y reiniciar el server.
- El antiguo `SENDGRID_API_KEY` que estuvo en `.env` debe revocarse en el dashboard de SendGrid (acción del usuario), independientemente de que ya no se use en código.
- Suite total del proyecto: 13 backend + 6 client = **19 verdes**.

---

## 2026-05-26 — design-system.md creado

**Tarea ejecutada:** Crear `docs/design-system.md` como fuente única de tokens visuales del proyecto. Extraído del theme MUI (`client/src/theme/theme.ts`), `globals.css` y `legacy.html`.

Contenido (12 secciones): Marca (orange, gradient, logos), Neutros y superficies, Estados, Tipografía (Inter + escala), Espaciado y layout, Radios, Sombras, Componentes (Button, Chip, Input, Card), Animaciones, **Subset para correos transaccionales** (tabla width 600, fonts con fallback, no box-shadow, plantilla wrapper HTML), Cómo extender, Historial.

Aplicación inmediata: la sección 10 (Tokens para correos) es la referencia para los templates HTML del feature 10 cuando se implemente.

---

## 2026-05-26 — Feature id=11: Actualización de campos del modal

**Tarea ejecutada:** Añadir industria y teléfono con prefijo de país al modal, marcar campos requeridos según el flujo.

**Decisiones del usuario:**
- Requeridos: Nombre, Email, **Teléfono**. Empresa e Industria opcionales.
- Lista de industrias: 12 + Otro (Retail, Finanzas, Salud, Educación, Manufactura, Logística, Telco, Servicios profesionales, Tecnología/SaaS, Gobierno, Hospitalidad, Inmobiliaria, Otro).

**Cambios backend (`/server`):**

1. **`src/db.js`:**
   - Schema actualizado: nueva columna `industria TEXT`.
   - `ensureSchema()` añade `ALTER TABLE leads ADD COLUMN IF NOT EXISTS industria TEXT` para migrar instancias existentes (idempotente).
   - `insertLead()` ahora persiste `industria`.

2. **`src/app.js`:**
   - Validación: `nombre` y `telefono` requeridos para `tipo='demo'\|'contacto'`. `tipo='email'` mantiene el flujo email-only sin requisitos extra (compatibilidad con CTA final / ROI).
   - `PHONE_RE = /^\+\d{7,15}$/` (E.164 estricto).
   - `normalizePhone()` quita espacios, `-`, `(`, `)` antes de validar y persistir.
   - Errores 422 ahora incluyen `field: 'email'\|'nombre'\|'telefono'` para que el frontend pueda enfocar el campo erróneo.

3. **`src/email.js`:** templates `buildText/buildHtml` incluyen línea de `Industria` y renombran `Mensaje → Comentarios`.

**Cambios frontend (`/client`):**

1. **Deps:** `mui-tel-input@^8.0.1` instalado (compatible con MUI v6, v9+ requeriría MUI v7).

2. **`src/lib/industries.ts`** (nuevo): array `INDUSTRIES` exportado con la lista cerrada + tipo `Industry`.

3. **`src/lib/api.ts`:** `LeadPayload` añade `industria?`; export de `PHONE_RE`.

4. **`src/components/ContactModal.tsx`:**
   - Reemplazado el `TextField` de teléfono por `MuiTelInput` con `defaultCountry='MX'` y `preferredCountries=['MX','CO','PE','CL','AR','VE','US','ES']`.
   - Añadido `TextField select` con la lista de industrias (opcional, primera opción "— sin especificar —").
   - Label "Mensaje" → "Comentarios".
   - Validación cliente: nombre/email/teléfono requeridos. Telefono validado con regex relajada (la rigurosa la hace el backend con feedback de `field`).
   - El payload normaliza el teléfono quitando espacios antes del POST.

5. **Tests:**
   - **Backend (`tests/contact.test.js`):** reescrito; **15 tests** incluyendo: payload completo persiste industria, normalización teléfono, 422 por nombre/teléfono faltantes, industria opcional, flujo email-only sin requisitos.
   - **Client (`__tests__/ContactModal.test.tsx`):** 7 tests; nuevo test "permite seleccionar industria y la envía en el payload". Selectors actualizados a `getByLabelText(/x/i, { selector: 'input' })` para evitar conflictos con MuiTelInput. Tipear con `user.keyboard()` en lugar de `paste()` (este último no propagaba el `onChange` del componente).

**Verificación ejecutada:**
- `npm test` server → **20/20 verdes** (15 contact + 5 email, 1 s).
- `npx vitest run` client → **7/7 verdes** (8 s).
- `npx tsc -b` → 0 errores.
- `npx vite build` → 629 KB / 189 KB gz (mui-tel-input agregó ~170 KB).
- Smoke E2E real: `POST` con todos los campos → `{ok:true, id:15}`; sin nombre → 422 `field:nombre`; sin teléfono → 422 `field:telefono`. Email enviado por SMTP: `[mail] lead=15 ... status=sent messageId=<...@maprotel.com>`. DB confirma `industria='Tecnología / SaaS'`.

**Archivos modificados:**
- `server/src/db.js` (schema + ALTER + insertLead)
- `server/src/app.js` (validación + normalize phone + field en errors)
- `server/src/email.js` (templates incluyen industria + label Comentarios)
- `server/tests/contact.test.js` (reescrito, 15 tests)
- `client/package.json` (+mui-tel-input)
- `client/src/lib/api.ts` (LeadPayload.industria + PHONE_RE)
- `client/src/lib/industries.ts` (nuevo)
- `client/src/components/ContactModal.tsx` (MuiTelInput + Select + Comentarios)
- `client/src/components/__tests__/ContactModal.test.tsx` (7 tests)
- `docs/architecture.md` (schema + payload + validación + historial)
- `feature_list.json` (id=11 → `done`)
- `memory.md` (este archivo)

**Notas operacionales:**
- Bundle bumped a 629 KB / 189 KB gz por mui-tel-input. Si interesa optimizar: usar `mui-tel-input/lite` o lazy-load del ContactModal (ahora se incluye en bundle principal).
- Para upgrade futuro a mui-tel-input v9+ se requiere bumpear MUI a v7.
- Suite total del proyecto: **20 backend + 7 client = 27 verdes**.

---

## 2026-05-26 — Feature id=12: detección de país desde prefijo del teléfono

**Tarea ejecutada:** Decodificar el país del lead desde el prefijo telefónico E.164 y persistirlo como texto en la BD para que el equipo de ventas pueda segmentar/filtrar.

**Decisiones técnicas:**
- **Librería:** `libphonenumber-js` (la versión estándar de la industria; ~150 KB en server, no afecta el bundle del cliente). Reusa metadata sin requerir nada del frontend.
- **Formato:** guardamos **dos** campos en BD — `pais` (nombre humano en español: "Venezuela", "México", "Estados Unidos") y `pais_iso` (código ISO 3166-1 alpha-2: "VE", "MX", "US"). El nombre humano es para el CRM/correos; el ISO es para queries/segmentación robustas.
- **Localización:** `Intl.DisplayNames(['es'], { type: 'region' })` (nativo Node ≥ 18, sin deps adicionales).
- **Robustez:** si el teléfono no es válido o no tiene prefijo, `detectCountry()` devuelve `{ iso: '', name: '' }` y la inserción procede normal (no se rompe el flujo).

**Cambios:**

1. **`server/src/phone.js`** (nuevo) — helper `detectCountry(phone)` con cache global del traductor.
2. **`server/src/db.js`** — schema añade `pais TEXT, pais_iso TEXT`. `ensureSchema()` ejecuta los `ALTER TABLE ADD COLUMN IF NOT EXISTS`. `insertLead()` pasa los nuevos campos.
3. **`server/src/app.js`** — antes del INSERT llama `detectCountry(telefono)` y agrega `pais`/`pais_iso` al objeto `lead`.
4. **`server/src/email.js`** — `buildText` y `buildHtml` ahora incluyen una fila "País" con `${pais} (${pais_iso})`.
5. **Tests:**
   - `tests/phone.test.js` (nuevo, **8 tests**): cubre VE/MX/US/ES/CO/PE, teléfonos inválidos y entradas nulas.
   - `tests/contact.test.js`: ampliado a 17 tests, añadidos casos "decodifica país +58 (Venezuela)" y "pais queda vacío cuando flow no exige teléfono".
   - Total: **30 backend verdes** (8 phone + 17 contact + 5 email).

**Verificación ejecutada:**
- `npm test` → 30/30 verdes (1.2 s).
- Smoke E2E: tres POST con prefijos distintos. DB confirma:
  ```
  id 17 — +584242848748 → Venezuela (VE)
  id 18 — +14155557890  → Estados Unidos (US)
  id 19 — +34611223344  → España (ES)
  ```
  Los 3 correos salieron por SMTP con `status=sent`.

**Archivos creados/modificados:**
- `server/package.json` (+libphonenumber-js)
- `server/src/phone.js` (nuevo)
- `server/src/db.js` (schema + ALTER + insertLead)
- `server/src/app.js` (import phone + decode)
- `server/src/email.js` (templates incluyen país)
- `server/tests/phone.test.js` (nuevo)
- `server/tests/contact.test.js` (+2 tests)
- `docs/architecture.md` (schema actualizado + sección decode + historial)
- `feature_list.json` (id=12 → done)
- `memory.md` (este archivo)

**Notas operacionales:**
- El frontend NO cambia. Sigue mandando el `telefono` en E.164 como antes; el decode es totalmente backend.
- Si más adelante queremos mostrar el país en la UI (admin/CRM), basta con leer `pais`/`pais_iso` de la BD.
- Leads existentes anteriores al feature 12 quedan con `pais=''` y `pais_iso=''` (los registros viejos `id=15,16` en la prueba). Para retro-llenarlos: `UPDATE leads SET pais=..., pais_iso=... WHERE pais IS NULL OR pais=''` ejecutando el decoder offline. No se hace automático.
- Suite total del proyecto: **30 backend + 7 client = 37 verdes**.

---

## 2026-05-26 — Feature id=15: Creación de roles (admin / editor)

**Tarea ejecutada:** Sistema de autorización por roles para los endpoints administrativos del backend. Trabajo coordinado en paralelo con feature 14 (login/users) — esta feature solo añade la columna `role` y el helper de autorización, **sin tocar** `auth.js` ni `users.js` (los crea feature 14).

**Cambios:**

1. **`server/src/db.js`** — En `ensureSchema()` se añade al final una migración idempotente:
   ```sql
   ALTER TABLE "<schema>".users ADD COLUMN IF NOT EXISTS role TEXT NOT NULL DEFAULT 'editor';
   ```
   Está envuelta en `try/catch` que ignora `42P01` (undefined_table) por si la tabla `users` aún no fue creada por feature 14 en alguna rama. Cualquier otro error sí se propaga.

2. **`server/src/roles.js`** (nuevo):
   - `ROLES = Object.freeze({ ADMIN: 'admin', EDITOR: 'editor' })`.
   - `hasRole(user, ...roles)` → `true` si `user.role` está en la lista; `false` si `user` es nulo, sin `role`, o lista vacía.
   - `requireRole(...roles)` → middleware Express. Responde `403 { ok:false, error:'forbidden' }` si el rol no está permitido; en caso contrario llama a `next()`. Asume `req.user` ya populado por `requireAuth` (feature 14).

3. **`server/scripts/seed-users.js`** (nuevo):
   - CLI que lee `MAIA_ADMIN_EMAIL` y `MAIA_ADMIN_PASSWORD` del entorno.
   - Importa dinámicamente `src/users.js`. Si feature 14 no está disponible, imprime `[seed-users] Falta src/users.js (feature 14 pendiente). No se puede crear el admin todavía.` y sale con exit 1.
   - Si el user existe, hace `UPDATE role='admin'`. Si no, lo crea via `createUser` y luego fuerza `UPDATE` por defensa (por si `createUser` no soporta el campo `role`).
   - Imprime `id` y `email` al finalizar.

4. **`server/.env.example`** — añadidas `MAIA_ADMIN_EMAIL=` y `MAIA_ADMIN_PASSWORD=`.

5. **`server/tests/roles.test.js`** (nuevo, 15 tests):
   - `ROLES`: expone admin/editor, es `Object.frozen`.
   - `hasRole`: admin↔admin, editor↛admin, editor∈[admin,editor], null, undefined, sin role, lista vacía.
   - `requireRole`: rol insuficiente → 403 + no next, rol coincide → next() sin tocar res, lista permisiva (editor∈[admin,editor]) → next, sin `req.user` → 403.
   - Migración: tras `ensureSchema`, `information_schema.columns` confirma `role NOT NULL DEFAULT editor` y un INSERT sin role recibe `editor`. ensureSchema en esquema vacío no lanza.

6. **`docs/architecture.md`** — Sección nueva "Auth & Admin — Roles" al final con tabla de roles, esquema BD, ejemplos de `hasRole`/`requireRole`, response 403 y bootstrap del admin. Entrada `id=15` añadida al historial de cambios.

**Verificación:**
- `cd server && npm test` → **49/49 verdes** (15 nuevos en roles + 9 email + 8 phone + 17 contact). Tiempo total ~2.7 s.
- Los tests de `contact.test.js`, `email.test.js` y `phone.test.js` **no se rompieron**.

**Comando para crear el primer admin:**
```bash
MAIA_ADMIN_EMAIL=admin@maiabuilder.ai \
MAIA_ADMIN_PASSWORD=cambia-esto \
node scripts/seed-users.js
```

**Archivos creados/modificados:**
- `server/src/db.js` (ALTER role en try/catch)
- `server/src/roles.js` (nuevo)
- `server/scripts/seed-users.js` (nuevo)
- `server/.env.example` (+MAIA_ADMIN_*)
- `server/tests/roles.test.js` (nuevo, 15 tests)
- `docs/architecture.md` (sección Roles + historial)
- `feature_list.json` (id=15 → in_progress; será `done` tras review)
- `memory.md` (este archivo)

**Pendiente:**
- El integrador unirá `requireRole` con el `requireAuth` que crea feature 14 en las rutas administrativas (leads, blog) — fuera de scope de esta feature.

---

## 2026-05-26 — Feature id=10: formato del correo de notificación de demos (HTML + 2 envíos)

**Tarea ejecutada:** Refactor del mailer para enviar **dos** correos por lead (notificación a ventas + confirmación al usuario), ambos con header con logo MaIA, footer con isotipo MaIA y plantillas HTML que respetan el subset transaccional documentado en `docs/design-system.md §10`.

**Cambios:**

1. **`server/src/email.js`** — reescrito:
   - `sendLead(lead, id)` ahora compone **dos** mensajes (`salesMsg` + `userMsg`) y los manda con el mismo `transporter`. Cada mensaje se evalúa de forma independiente (`sendOne`).
   - Plantillas HTML inline (sin CSS vars, sin flex/grid, todo `<table>` con `width="600"` y estilos inlined). Header `#E8440A` con logo blanco; body card blanca con borde `#F0EBE8`; footer `#FAFAF9` con isotipo y texto muted `#A89E9A`. CTA pill `#E8440A border-radius:100px`.
   - **Logos inline con CID:** se adjuntan `cid:logo-maia` (header) y `cid:isotipo-maia` (footer) desde `docs/images/{logo,isotipo}-maia.svg` con `contentType: 'image/svg+xml'`. Outlook desktop no renderiza SVG inline — si se vuelve prioridad, se puede swappear por PNG manteniendo el mismo CID (nota documentada en architecture.md).
   - `escapeHtml` extendido a `& < > " '` para anti-XSS (incluye atributos).
   - Multipart alternative completa: cada correo lleva `text` + `html`.
   - Versión usuario es deliberadamente **mínima**: saludo con primer nombre, mensaje de gracias, próximos pasos genéricos, CTA "Conocer más". **No** incluye `mensaje`, `industria`, `empresa` ni teléfono del lead (es solo confirmación, no expone datos sensibles del CRM).
   - **Nuevo contrato de respuesta:** `{ status: 'sent'|'partial'|'failed'|'skipped', sentTo: string[], messageIds: string[], results: { sales, user } }`. `partial` cuando uno de los dos sale y el otro falla.

2. **`server/src/app.js`** — actualizado el bloque de logs `[mail]`:
   - Loggea **dos líneas** (una por destinatario) con etiqueta `sales→...` / `user→...`. Si uno falla, sólo ese se loggea como `status=failed`; el otro mantiene `status=sent`.
   - Mantengo el caso especial `status=skipped` global (mailer no configurado) como una sola línea informativa.
   - **Respuesta al cliente intacta:** sigue siendo `{ ok: true, id }` — el detalle del transporte sigue server-side.
   - El resto de `app.js` (auth/cookies introducidos en paralelo por feature 14) se respeta sin tocar.

3. **`server/tests/email.test.js`** — reescrito a **9 tests** (vs 5 previos):
   - skipped sin `SMTP_HOST` (ahora también verifica `sentTo:[]` y `messageIds:[]`).
   - sent: **dos** correos con payload bien formado (subject/replyTo de ventas, subject del usuario, ambos con `attachments` CID y `cid:logo-maia` + `cid:isotipo-maia` en el HTML).
   - sales incluye datos del lead escapados (nombre con `<"O">`, empresa con `&`, comentarios con `&`).
   - **user NO incluye comentarios/industria/empresa** del lead.
   - escape XSS (`<script>` → `&lt;script&gt;`) en ambos correos.
   - `partial` cuando el 2º envío falla, y otro test simétrico cuando falla el 1º.
   - `failed` cuando ambos fallan.
   - `SMTP_SECURE="true"` interpretado como booleano + verifica que se envían 2 mensajes al `to` correcto cada uno.

4. **`docs/architecture.md`** — reemplazada la sección Email completa. Documenta el nuevo contrato (sent/partial/failed/skipped + sentTo + messageIds + results.{sales,user}) y muestra el formato de log con 2 líneas.

**Verificación ejecutada:**
- `cd server && npm test` → **49/49 verdes** (17 contact + 9 email + 8 phone + 15 roles). Los 30 baseline siguen en verde + 4 nuevos email tests (los 5 originales se reescribieron, 9 finales).

**Ejemplo de logs `[mail]` con los 2 envíos:**

```
[mail] lead=42 sales→maia@maiabuilder.ai status=sent messageId=<msg-1@test>
[mail] lead=42 user→ana@acme.com status=sent messageId=<msg-2@test>
```

Si falla solo el correo al usuario (p. ej. dirección inexistente):

```
[mail] lead=42 sales→maia@maiabuilder.ai status=sent  messageId=<msg-1@test>
[mail] lead=42 user→ana@acme.com         status=failed code=550 reason="No such recipient"
```

**Archivos modificados:**
- `server/src/email.js` (reescrito con 2 envíos, plantillas HTML completas, CID inline)
- `server/src/app.js` (sólo el bloque de logs `[mail]`, sin tocar la lógica de auth/contact)
- `server/tests/email.test.js` (9 tests, reescritos)
- `docs/architecture.md` (sección Email reemplazada)
- `feature_list.json` (id=10 → done tras review)
- `memory.md` (este archivo)

**Notas operacionales:**
- Los SVGs viven en `docs/images/`. El mailer los lee con `path.resolve(__dirname, '../../docs/images/...')` — si en deploy se mueve `/server` fuera del repo, hay que copiar los assets o setearlos via config (no se hizo aquí para no salirse del scope).
- El `replyTo` de ventas sigue apuntando al lead — si el comercial pulsa "Reply" responde al prospecto directamente.
- En tests no se valida la presencia física de los SVGs (los attachments se pasan como `{ path, cid, contentType }` y el fake transporter no abre el fichero). Para producción los archivos sí deben existir en runtime.

---

## 2026-05-26 — Feature id=14: Login admin + interfaz `/admin`

**Tarea ejecutada:** Crear el login de usuarios y la base de la interfaz `/admin` protegida. Sienta los cimientos para features 13 (mantenedor blog) y 16 (visualización de leads).

**Decisiones técnicas:**
- **Sesión** por cookie httpOnly (`maia_session`) que transporta un **JWT HS256** firmado con `AUTH_SECRET`. Expira en 7 días, `SameSite=Lax`, `Secure` solo en producción.
- **Hash** de passwords con `bcryptjs` (no nativo) y `SALT_ROUNDS = 12`.
- **No se incluye `role`** en el schema de `users` (feature 15 lo añade en paralelo con `ALTER TABLE`).
- Frontend usa `react-router-dom@^6` y `credentials: 'include'` en todos los fetch de auth (cookie viaja same-origin gracias al proxy de Vite).

**Cambios backend (`/server`):**
1. **`src/db.js`** — `ensureSchema()` añade `CREATE TABLE IF NOT EXISTS users (...)` + índice `users_email_idx`.
2. **`src/users.js`** (nuevo) — `createUser`, `findUserByEmail`, `verifyPassword` (bcrypt). `createUser` mapea violación de UNIQUE (`23505`) a `Error('email_taken')` con `err.code`.
3. **`src/auth.js`** (nuevo) — factory `createAuthRouter({ pool, schema, secret })` con `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` y helper `requireAuth`. Si `AUTH_SECRET` está vacía genera una random al boot y emite warning.
4. **`src/app.js`** — añade `cookieParser()` antes del router y monta el router de auth. No toca `/api/contact` ni los logs `[mail]`.
5. **`scripts/create-user.js`** (nuevo) — CLI para crear admin: `node scripts/create-user.js <email> <password> [name]`.
6. **`.env.example`** — añadida `AUTH_SECRET`.
7. **`tests/auth.test.js`** (nuevo, 10 tests) — esquema temporal `maia_test_*`, valida 401 sin/malos creds, 200 + cookie httpOnly, /me con/sin cookie, cookie manipulada, logout, email duplicado, hash bcrypt persistido.

**Cambios frontend (`/client`):**
1. **`react-router-dom@^6.30.3`** instalado.
2. **`src/main.tsx`** — `BrowserRouter` con rutas `/` (landing), `/admin/login` (Login) y `/admin/*` (envuelto en `AdminGuard` → `AdminLayout` con `Outlet`).
3. **`src/admin/Login.tsx`** (nuevo) — form MUI con validación cliente (`EMAIL_RE`), POST a `/api/auth/login`. Tras 200 navega a `/admin` (o a `from.pathname` si venía de `AdminGuard`).
4. **`src/admin/AdminGuard.tsx`** (nuevo) — hace `GET /api/auth/me`; mientras carga muestra `<CircularProgress />`; si 401 → `<Navigate to="/admin/login" state={{ from }} />`; si OK, renderiza `children(user)`.
5. **`src/admin/AdminLayout.tsx`** (nuevo) — sidebar con logo + nombre del user + botón "Cerrar sesión". Nav placeholders "Leads" y "Blog" deshabilitados con sufijo "próximamente" (se habilitarán en features 16 y 13).
6. **`src/admin/AdminHome.tsx`** (nuevo) — "Bienvenido, {name}".
7. **`src/lib/api.ts`** — helpers `login(email, password)`, `logout()`, `getMe()` con `credentials:'include'`.
8. **`src/admin/__tests__/Login.test.tsx`** (nuevo, 5 tests) — render, validación vacía, email inválido, submit válido con mock fetch (verifica `credentials:'include'` y body), error 401.

**Verificación ejecutada:**
- `npm test` server → **59 tests verdes** (10 auth + 17 contact + 5 email + 8 phone + 15 roles + 4 anteriores = 59 totales, 5 s).
- `npx tsc -b` client → 0 errores.
- `npx vitest run` client → **12/12 verdes** (5 Login + 7 ContactModal, 13 s).
- `npx vite build` → **661 KB / 200 KB gz** (+30 KB sobre 629 KB anteriores por react-router-dom).
- **Smoke E2E (curl con server arrancado en :3099 con `AUTH_SECRET=smoke-test-secret-xxx`):**
  - `POST /api/auth/login` con creds válidas → `200` + `Set-Cookie: maia_session=eyJhbGc...; HttpOnly; SameSite=Lax; Max-Age=604800; Path=/`.
  - `GET /api/auth/me` con cookie → `200 { user: { id:1, email:'smoke14@maia.test', name:'Smoke Tester' } }`.
  - `GET /api/auth/me` sin cookie → `401 Unauthorized`.
  - `POST /api/auth/login` con password incorrecta → `401 { error:'Credenciales inválidas' }`.
  - `POST /api/auth/logout` → `200` + `Set-Cookie: maia_session=; Expires=Thu, 01 Jan 1970 00:00:00 GMT; HttpOnly; SameSite=Lax`.
- Smoke user creado con `node scripts/create-user.js smoke14@maia.test 'Smoke-14!' 'Smoke Tester'` → `[create-user] OK id=1 ...`. Borrado al cierre con `psql DELETE FROM users`.

**Archivos creados/modificados:**
- `server/package.json` (+bcryptjs, +jsonwebtoken, +cookie-parser)
- `server/.env.example` (+AUTH_SECRET)
- `server/src/db.js` (+tabla `users` + índice)
- `server/src/users.js` (nuevo)
- `server/src/auth.js` (nuevo)
- `server/src/app.js` (+cookieParser + montaje de auth router, sin tocar `/api/contact` ni los logs `[mail]`)
- `server/scripts/create-user.js` (nuevo)
- `server/tests/auth.test.js` (nuevo, 10 tests)
- `client/package.json` (+react-router-dom@^6)
- `client/src/main.tsx` (rutas)
- `client/src/lib/api.ts` (+login/logout/getMe + AdminUser)
- `client/src/admin/{Login,AdminGuard,AdminLayout,AdminHome}.tsx` (nuevos)
- `client/src/admin/__tests__/Login.test.tsx` (nuevo, 5 tests)
- `docs/architecture.md` (sección "Auth & Admin — Login (feature 14)" + endpoints + env vars + historial)
- `feature_list.json` (id=14 → in_progress → done tras review)
- `memory.md` (esta entrada)

**Notas operacionales:**
- El primer admin se crea con `node scripts/create-user.js <email> <password> [name]` desde `/var/www/html/maia-landing/server`.
- Sin `AUTH_SECRET` definido, el server arranca con un secreto aleatorio y emite warning — todas las sesiones se invalidan al reiniciar. **Para producción es obligatorio fijarlo** (ej. `openssl rand -hex 48`).
- La cookie es `Secure` solo cuando `NODE_ENV=production`. En dev (HTTP local) eso permite que el navegador la acepte por `localhost`.
- Las secciones "Leads" y "Blog" del sidebar quedan listas (placeholders deshabilitados) para que features 16 y 13 solo tengan que activarlas.
- Suite total del proyecto: **59 backend + 12 client = 71 verdes**.

---

## 2026-05-27 — Ola 2: Features 13 + 16 (mantenedor de blog + visualización de leads)

**Contexto:** los 2 agentes paralelos que despaché para esta ola se cortaron por límite de sesión sin escribir nada. Implementé ambas features yo mismo, sin overlap entre ellas (`articles` vs `leads` — archivos distintos).

### Feature 13 — Mantenedor de Blog

- **Schema** (extendido en `db.js`): tabla `articles (id, slug UNIQUE, title, excerpt, body_md, cover_url, status, author_id FK users, published_at, created_at, updated_at)`. Índices en `(status, published_at DESC)` y `slug`.
- **`articles.js`** (nuevo): helpers puros `createArticle / updateArticle / deleteArticle / getArticleById / getArticleBySlug / listArticles / slugify`. `slugify` normaliza NFD + quita acentos + kebab-case + truncate 120.
- **`articlesRouter.js`** (nuevo): rutas públicas (GET lista/slug, solo published) + admin (CRUD bajo `requireAuth + requireRole('admin','editor')`). DELETE adicionalmente requiere role=admin.
- **UI admin (`client/src/admin/articles/`):** `ArticlesList` (tabla con badge status + Edit/Delete) y `ArticleEdit` (form create/edit con title, slug, excerpt, body_md textarea grande, cover_url, status). Delete solo visible si `user.role==='admin'`.

### Feature 16 — Visualización de leads

- **`leads.js`** (nuevo): `listLeads(pool, schema, { tipo, pais_iso, q, limit, offset })` con sanitización (limit max 200) y `getLeadById`.
- **`leadsRouter.js`** (nuevo): `GET /api/admin/leads` + `GET /api/admin/leads/:id` con guard `requireAuth + requireRole('admin','editor')`.
- **UI admin (`client/src/admin/leads/`):** `LeadsList` (tabla con TablePagination server-side, filtros tipo + país + búsqueda con debounce 300 ms) y `LeadDetailDialog` (todos los campos del lead, link `mailto:` y `wa.me/`).

### Fixes durante la integración

1. **`auth.publicUser` no incluía `role`** → todos los usuarios aparecían como `editor` aunque la DB dijera `admin`. Fix: añadir `role` al SELECT en `loadUserFromCookie` y al objeto `publicUser`.
2. **`users.findUserByEmail` tampoco incluía `role`** → el `POST /api/auth/login` retornaba `role:'editor'` para el admin. Fix: añadir `role` al SELECT.
3. **`AdminLayout` tenía Leads y Blog como placeholders deshabilitados** → habilitados con link a `/admin/leads` y `/admin/articles`.

### Cliente API (`client/src/lib/api.ts`)

Añadidos types `AdminLead`, `AdminArticle`, `ArticleInput`, `ArticleStatus`, `LeadsListResponse`, `LeadsListFilters`. Helpers: `listAdminLeads`, `getAdminLead`, `listAdminArticles`, `getAdminArticle`, `createAdminArticle`, `updateAdminArticle`, `deleteAdminArticle`. `AdminUser` ahora incluye `role: 'admin' \| 'editor'`.

### Tests

- **Backend (`server/tests/articles.test.js`):** 13 tests — slugify, GET público solo published, admin endpoints (401 sin cookie, 201 editor, 422 sin title, 409 slug duplicado, PATCH con published_at automático, DELETE 403 editor / 204 admin, 404 inexistente).
- **Backend (`server/tests/leads.test.js`):** 14 tests — listLeads (helper) con filtros tipo/pais_iso/q/paginación/limit clamp; HTTP endpoint con guard 401 y filtros via query string; GET /:id con 404.
- **Cliente (`client/src/admin/__tests__/`):** `LeadsList` (3 tests: render, search dispara fetch con ?q=, click abre detalle); `ArticlesList` (4 tests: render con badges, sin botón borrar para editor, con botón borrar para admin, cancelar el dialog).
- Suite total: **86 backend (8 archivos) + 19 client (4 archivos) = 105 verdes**.
- Aumenté `testTimeout` global de Vitest a 15 s (los 3 ContactModal tests llegaban a 8 s con la carga adicional de admin imports).

### Smoke E2E

- `POST /api/admin/articles` con cookie admin → 201, slug='hola-desde-curl', status='published'.
- `GET /api/articles` (público) → muestra el artículo recién creado.
- `GET /api/admin/leads?limit=3` → 3 últimos leads con todos los campos.
- `GET /api/admin/leads` sin cookie → 401.

### Archivos creados / modificados

- Backend: `server/src/articles.js`, `server/src/articlesRouter.js`, `server/src/leads.js`, `server/src/leadsRouter.js`, `server/tests/articles.test.js`, `server/tests/leads.test.js`. Modificados: `server/src/db.js` (tabla articles), `server/src/app.js` (montaje de routers), `server/src/auth.js` (incluir role), `server/src/users.js` (incluir role en findUserByEmail).
- Cliente: `client/src/admin/leads/LeadsList.tsx`, `client/src/admin/leads/LeadDetailDialog.tsx`, `client/src/admin/articles/ArticlesList.tsx`, `client/src/admin/articles/ArticleEdit.tsx`, `client/src/admin/__tests__/LeadsList.test.tsx`, `client/src/admin/__tests__/ArticlesList.test.tsx`. Modificados: `client/src/main.tsx` (rutas), `client/src/admin/AdminLayout.tsx` (habilitar nav), `client/src/lib/api.ts` (types + helpers admin + AdminUser.role), `client/vite.config.ts` (testTimeout 15s).
- Docs: `docs/architecture.md` (secciones "Blog (feature 13)" y "Visualización de leads (feature 16)" añadidas al final), `feature_list.json` (id=13 y id=16 → done).

### Notas operacionales

- El admin user creado por `seed-users.js` queda con `role='admin'`. Para crear un editor manualmente: `node scripts/create-user.js <email> <pass> "<name>"` — el default es `role='editor'`.
- El endpoint `GET /api/articles` (público, sin auth) ya está listo para que el feature 7 lo consuma desde la sección "Blog" de la landing pública (Ola 3).

---

## 2026-05-27 — Feature id=7: Visualización de Blog (público)

**Tarea ejecutada:** sección de blog en la landing + páginas dedicadas `/blog` y `/blog/:slug` consumiendo los endpoints públicos del feature 13 (sin auth).

**Cliente API (`client/src/lib/api.ts`):**
- Nuevo type `PublicArticle = Pick<AdminArticle, 'id'|'slug'|'title'|'excerpt'|'body_md'|'cover_url'|'published_at'|'created_at'|'updated_at'>`.
- Helper interno `publicJson()` paralelo a `apiJson()` pero con `credentials:'omit'` (los endpoints `/api/articles*` son abiertos).
- `listPublicArticles({ limit?, offset? })` → `{ rows: PublicArticle[] }`.
- `getPublicArticleBySlug(slug)` → `{ article: PublicArticle }` o 404.

**Sección landing (`client/src/components/sections/Blog.tsx`):**
- ID `#blog`, fondo `var(--bg-soft)`, padding responsive coherente con el resto.
- Carga hasta 3 artículos al montar. Si la API falla o `rows` viene vacío → `return null` (graceful degradation; la landing sigue luciendo bien sin la sección).
- Cada `BlogCard` (exportada para reuso en `/blog`): cover (`cover_url`) o fondo gradient naranja con la inicial del título; fecha localizada `es-MX`; título; excerpt (usa `excerpt` o las primeras ~140 chars de `body_md` con markdown stripped); CTA "Leer artículo →".
- Card linkea a `/blog/:slug` vía `Link component={RouterLink}` (todo el card es clickable).
- Botón "Ver todos los artículos" → `/blog` (variant outlined).

**Integración en App.tsx:** se inserta `<Blog />` después de `<Testimonials />` y antes de `<FAQ />` (que está antes de `<CTAFinal />`). Ningún otro componente se reordena ni recibe props nuevas.

**Páginas dedicadas (`client/src/pages/`):**
- **`BlogIndex.tsx`** (`/blog`):
  - Header sticky con logo MaIA (link a `/`) y botón outlined "Volver a la home".
  - Hero centrado con overline "Blog" + h1 "Ideas y guías de MaIA" + descripción.
  - Grid (`limit:50`) con `BlogCard` reusada. Estados: spinner mientras carga; `Alert` rojo si falla; mensaje "Aún no hay artículos publicados" cuando `rows: []`.
  - Footer compartido (`components/Footer.tsx`).
- **`BlogArticle.tsx`** (`/blog/:slug`):
  - Header sticky con logo + botón "Volver al blog" (link a `/blog`).
  - Estado loading → `Skeleton` de portada + título + párrafos.
  - Estado not-found (HTTP 404) → "Artículo no encontrado" + CTA primario.
  - Estado ok → portada (`cover_url`) + h1 título + fecha + cuerpo markdown.
  - Markdown renderizado con `react-markdown@9` + `remark-gfm@4` (tablas, autolinks, task lists). Estilos respetan el design system: h1/h2/h3 Inter con `letter-spacing -0.025em`, `code/pre` con fondo `#FAFAF9` + borde `var(--border)`, `blockquote` con border-left naranja, links naranja, tablas con bordes finos.
  - Container `max-width: 720px` para lectura cómoda.
  - `react-markdown` sanitiza por defecto (no HTML arbitrario).
  - Footer compartido.

**Router (`client/src/main.tsx`):** añadidas dos rutas `/blog` y `/blog/:slug` antes del bloque `/admin/*`.

**Dependencias añadidas:**
- `react-markdown@^9.1.0`
- `remark-gfm@^4.0.1`

**Tests RTL (`client/src/pages/__tests__/`):**
- **`BlogIndex.test.tsx`** (3 tests): render del overline/h1, listado de cards desde la API, estado vacío amable con `rows: []`.
- **`BlogArticle.test.tsx`** (3 tests): render del título + conversión de `## Subtitulo` a `<h2>` (validación markdown), HTTP 404 muestra "Artículo no encontrado", "Volver al blog" tiene `href="/blog"` y navega.
- Mocks con `vi.spyOn(globalThis,'fetch')`. Render en `<MemoryRouter>` + `<ThemeProvider>`.

**Verificación:**
- Backend: `npm test` → **86/86 verdes** (sin cambios; esta feature es 100% frontend).
- Cliente: `npx vitest run` → **25/25 verdes** (19 previos + 6 nuevos).
- `npx vite build` → OK (857 kB / 258 kB gz; +200 kB respecto al baseline por `react-markdown` + `remark-gfm`).
- `npx tsc -b`: los únicos errores que quedan están en `client/src/admin/__tests__/ArticlesList.test.tsx` (líneas 18 y 23, pre-existentes del feature 13 sobre la prop `context` en `<Route>` y un símbolo `renderWithUser` no usado). Esos archivos están en la lista de "Archivos PROHIBIDOS" y no se modifican.

**Cómo probar en dev:**
```bash
# terminal 1
cd server && npm run dev
# terminal 2
cd client && npm run dev
# Navegar a:
http://localhost:5173/        # landing con sección Blog (solo si hay publicados)
http://localhost:5173/blog    # listado completo
http://localhost:5173/blog/<slug-publicado>
```

**Estado vacío:** si no hay artículos publicados en la BD, la sección `<Blog />` en la landing **no se renderiza** (graceful degradation, sin huecos visuales). La página dedicada `/blog` sí muestra un mensaje amable "Aún no hay artículos publicados. Vuelve pronto: nuestro equipo está trabajando en nuevo contenido." centrado, con header y footer normales.

**Archivos creados:**
- `client/src/components/sections/Blog.tsx`
- `client/src/pages/BlogIndex.tsx`
- `client/src/pages/BlogArticle.tsx`
- `client/src/pages/__tests__/BlogIndex.test.tsx`
- `client/src/pages/__tests__/BlogArticle.test.tsx`

**Archivos modificados:**
- `client/src/lib/api.ts` (type `PublicArticle` + helpers `listPublicArticles`, `getPublicArticleBySlug`, `publicJson`).
- `client/src/App.tsx` (`<Blog />` insertado entre Testimonials y FAQ).
- `client/src/main.tsx` (rutas `/blog` y `/blog/:slug` añadidas antes de `/admin`).
- `client/package.json` (dependencias `react-markdown` + `remark-gfm`).
- `docs/architecture.md` (sección "Blog público (feature 7)" al final).
- `feature_list.json` (id=7 → `in_progress` a la espera del reviewer).
- `progress/current.md` (sesión activa).

---

## 2026-05-28 — Feature id=17: verificación del cambio a Gmail SMTP

**Tarea ejecutada:** validar que el nuevo proveedor SMTP (Gmail) entrega los correos y que los leads se siguen persistiendo en BD.

**Contexto del cambio:** el usuario cambió las credenciales SMTP en `server/.env` de `mail.maprotel.com` a Gmail (`smtp.gmail.com`, user `maiabuilderai@gmail.com` con App Password de 16 chars).

**Fix encontrado durante la verificación:**
- La config llegó con `SMTP_PORT=587` + `SMTP_SECURE=true` — combinación incorrecta (587 requiere STARTTLS = `secure=false`). Misma clase de bug que arreglamos en feature post-id=8 con maprotel.
- Adicional: la red del host bloquea SMTP saliente al **puerto 587** de `smtp.gmail.com` (TCP connect timeout reproducible). Puerto 465 sí está abierto, y `mail.maprotel.com:465` también (control).
- **Corrección aplicada:** cambiar `SMTP_PORT=465` (manteniendo `SMTP_SECURE=true` — TLS implícito SSL, correcto para 465).
- `.env.example` actualizado para documentar que Gmail :465 es la opción recomendada cuando el firewall bloquea 587.

**Sender mismatch (conocido y aceptado):** `MAIL_FROM="MaIA Landing <javier.hernandez@maprotel.com>"` no alinea con `SMTP_USER=maiabuilderai@gmail.com`. Gmail aceptó el envío sin reescribir el envelope.from (el output de `sendMail` reporta `envelope.from=javier.hernandez@maprotel.com`). Es probable que el dominio maprotel esté configurado como "Send mail as" en la cuenta Gmail, o que Gmail haya permitido el envío y delegue la decisión a SPF/DKIM del receptor.

**Verificación ejecutada:**
- `verify()` con la nueva config → ✓ OK.
- `sendMail` directo a `javier.hernandez@ximple-tech.com` con `messageId=<7bd337e9-...@maprotel.com>` aceptado.
- Reinicio del server (para recargar `.env`).
- `POST /api/contact` con payload demo completo → respuesta `{ok:true, id:29}` (201).
- Logs server: ambos envíos a ventas y al usuario con `status=sent` y `messageId` Gmail.
- DB: `SELECT * FROM leads WHERE id=29` → fila persistida con campos completos incluyendo `pais='Venezuela', pais_iso='VE'` (detectado desde `+58`).

**Archivos modificados:**
- `server/.env` — `SMTP_PORT 587 → 465`. (Mantengo `SMTP_SECURE=true`.)
- `server/.env.example` — notas sobre Gmail :465 vs :587 y firewall.
- `memory.md` — esta entrada.
- `feature_list.json` — id=17 → `done`.

**Notas operacionales:**
- Suite de tests sigue verde (no se tocó código de aplicación, solo configuración + docs).
- Si en otra red el firewall permite 587, ambas opciones son válidas — Gmail oficial sigue siendo `:587 STARTTLS` como default. En este entorno usamos `:465 SSL`.
- El warning `[auth] AUTH_SECRET no configurada` desapareció (el `.env` lo tiene fijado en hex 96 chars).

---

## 2026-05-28 — Feature id=18: Formato enriquecido en posts del blog

**Tarea ejecutada:** Implementación del feature id=18. El admin ahora edita el cuerpo de los artículos con un editor markdown enriquecido (toolbar + preview en vivo), y la vista pública renderiza el mismo markdown con soporte de imágenes con estilos, tablas y videos de YouTube/Vimeo embebidos como iframes responsivos. Sin tocar backend ni esquema.

**Decisión de librería:**
- Editor: `@uiw/react-md-editor ^4.1.1`. Razones: React 18 + TS nativos, toolbar completa "out-of-the-box" incluyendo tabla GFM, modo `preview="live"` (split view), import normal ESM-friendly con Vite, ~150 KB. Descartadas `react-mde` (sin mantenimiento reciente) y `react-markdown-editor-lite` (toolbar más pobre, sin tabla nativa).
- Render: se reutiliza `react-markdown@9` + `remark-gfm@4` ya instalados (feature 7).

**Cambios:**

1. **Nuevo `client/src/components/MarkdownRenderer.tsx`** — wrapper reusable de `react-markdown` con custom `components`:
   - `img` → `loading="lazy"`, sombra suave, `border-radius: 16px`, `max-width: 100%`.
   - `a` → si href es `http(s)://` externo, agrega `target="_blank" rel="noopener noreferrer"`. Color `primary.main`, underline en hover.
   - `blockquote` → border-left naranja, fondo `var(--bg-soft)`, itálica, padding-x.
   - `code` (inline + bloque) → fondo `#FAFAF9`, borde `var(--border)`, monoespaciado.
   - `table` → bordes finos `var(--border)`, header con fondo `var(--bg-soft)`, cabecera `font-weight: 600`.
   - `p` → si el parágrafo contiene **solo** un link a `youtube.com/watch?v=<id>`, `youtu.be/<id>` o `vimeo.com/<id>`, se reemplaza por un wrapper 16:9 con `<iframe>` (`pt: 56.25%`, `position: absolute`), `loading="lazy"`, `allowFullScreen`, `border-radius: 16px`. Resto de parágrafos quedan como `<p>`.
   - Acepta props `{ body, compact?, sx? }`. `compact` reduce tipografía/márgenes para la preview del editor.
   - Exporta helper puro `getEmbedUrl(url)` (testeable sin DOM).

2. **`client/src/pages/BlogArticle.tsx`** — el render markdown inline se reemplaza por `<MarkdownRenderer body={state.article.body_md || ''} />`. Se eliminó la constante `markdownSx` y la importación de `ReactMarkdown`/`remarkGfm` (ahora viven dentro de MarkdownRenderer).

3. **`client/src/admin/articles/ArticleEdit.tsx`** — el `<TextField multiline>` del body_md se reemplaza por `<MDEditor>` (de `@uiw/react-md-editor`) con `preview="live"`, toolbar completa (bold, italic, headings, listas, blockquote, code inline y bloque, link, imagen URL, **tabla**), `height: 420`, modo `light`. Wrapper MUI sobreescribe estilos por defecto del editor para encajar en el design system: `border-radius: 12px`, borde `var(--border)`, focus naranja. Además, debajo del editor se renderiza una sección **"Vista previa"** con `<MarkdownRenderer body={form.body_md} compact />` para que el usuario vea exactamente cómo se verá en el blog público (incluyendo los iframes de YouTube/Vimeo, que el preview interno del editor no convierte). Helper text guía sobre cómo embeber video.

4. **Tests** — nuevo archivo `client/src/components/__tests__/MarkdownRenderer.test.tsx` con 9 tests RTL:
   - `# Título` → `<h1>`
   - `## Subtítulo` → `<h2>`
   - Tabla GFM con `columnheader`/`cell`.
   - Link YouTube → `<iframe>` con `src` `youtube.com/embed/<id>` + `loading="lazy"`.
   - Link Vimeo → `<iframe>` con `src` `player.vimeo.com/video/<id>`.
   - `![alt](url)` → `<img loading="lazy" src=...>`.
   - Link externo → `target="_blank"` + `rel` con `noopener`.
   - `getEmbedUrl()` reconoce los 3 patrones (y `null` para el resto).
   - `> cita` → `<blockquote>`.

**Resultados de verificación:**
- Backend `npm test`: **86/86 verdes** (sin cambios).
- Client `npx vitest run`: **34/34 verdes** (25 previos + 9 nuevos en MarkdownRenderer).
- `npx vite build`: ✓ bundle producido, 1 727.77 KB / 567.35 KB gzip (subió ~1 060 KB desde 661 KB por la suma de `@uiw/react-md-editor` + `@uiw/react-markdown-preview` + `refractor` + `react-textarea-code-editor`).
- `npx tsc -b`: 4 warnings TS6133 pre-existentes en `client/src/components/sections/CTAFinal.tsx` (no tocado, fuera del scope de feature 18 — ya existían antes y no afectan al build de Vite).

**Archivos creados/modificados:**
- `client/src/components/MarkdownRenderer.tsx` — nuevo.
- `client/src/components/__tests__/MarkdownRenderer.test.tsx` — nuevo (9 tests).
- `client/src/admin/articles/ArticleEdit.tsx` — editor enriquecido + preview.
- `client/src/pages/BlogArticle.tsx` — delega en `<MarkdownRenderer />`.
- `client/package.json` — agrega `@uiw/react-md-editor ^4.1.1`.
- `docs/architecture.md` — nueva sección "Editor enriquecido de artículos (feature 18)" + entrada en historial + fecha actualizada.
- `memory.md` — esta entrada.
- `feature_list.json` — id=18 → `in_progress` (el reviewer decide el paso a `done`).

**Notas:**
- Backend no se tocó: `body_md TEXT` sigue almacenando markdown plano.
- Bundle podría dividirse con `manualChunks` en una iteración posterior (warning de Vite por >500 KB), pero está fuera del scope de feature 18.

---

## 2026-05-29 — Feature id=19: Eliminar post desde Admin

**Hallazgo:** la funcionalidad **ya estaba implementada en feature 13 (Ola 2)** — `client/src/admin/articles/ArticlesList.tsx` tiene botón Delete con confirmación dialog (visible solo si `user.role==='admin'`) y `server/src/articlesRouter.js` expone `DELETE /api/admin/articles/:id` con guard `requireRole('admin')`.

**Trabajo del feature 19:** solo verificación E2E + cierre formal.

**Verificación ejecutada:**
- `npx tsc -b` → 0 errores (después de limpiar CTAFinal.tsx — ver nota abajo).
- Smoke E2E real contra server :3002:
  1. Login `admin@maia.test` → cookie.
  2. POST `/api/admin/articles` → `id=3` creado.
  3. DELETE `/api/admin/articles/3` con cookie admin → `204`.
  4. GET `/api/admin/articles/3` → `404 "Artículo no encontrado"`.
- Test cubierto en `server/tests/articles.test.js` ("DELETE 403 con cookie editor", "DELETE 204 con cookie admin", "404 si no existe") — sigue verde en la suite 86/86.

**Limpieza colateral aplicada** (`client/src/components/sections/CTAFinal.tsx`):
El usuario había editado el archivo previamente, removiendo el input email y el link "ver demo con especialista", pero dejó imports (`TextField`, `Stack`) y vars (`onOpenContact`, `setEmail`) huérfanos → 4 errores TS. También el botón "Iniciar ahora" llamaba a un `submit()` que hacía `postLead` con email vacío (siempre 422). Reescribí mínimamente: el botón ahora invoca `onOpenContact` (abre `ContactModal`), eliminé los imports/state unused y el `<Box href="">` vacío. Sin cambios visuales más allá de un fix correcto del flujo.

**Archivos modificados:**
- `client/src/components/sections/CTAFinal.tsx` — cleanup (no es del feature, pero rompía `tsc -b`).
- `feature_list.json` — id=19 → `done`.
- `memory.md` — esta entrada.

---

## 2026-06-04 — Feature id=23: Cambiar el estilo de las cards de las secciones

**Tarea ejecutada:** Rediseño visual completo de las cards en las secciones Features, Pain y Solution. Iconos emoji reemplazados por iconos MUI representativos; estilos de cards actualizados para ser más llamativos.

**Secciones modificadas:**

1. **`Features.tsx`** — 6 cards rediseñadas:
   - Iconos: `PsychologyIcon`, `HubIcon`, `SecurityIcon`, `InsightsIcon`, `GroupsIcon`, `SupportAgentIcon` (reemplazan 🧠🔗🛡️📈🤝🤖).
   - Cada icono tiene un container con fondo tintado del color de su acento (6 colores distintos: naranja, violeta, azul, verde, ámbar, naranja).
   - Borde superior transparente que se vuelve del color de acento en hover.
   - Hover: `translateY(-6px)` + sombra prominente.

2. **`Pain.tsx`** — 4 cards rediseñadas:
   - Iconos: `GroupRemoveIcon`, `TrendingUpIcon`, `HourglassTopIcon`, `VisibilityOffIcon` (reemplazan ⏰💸😴📊).
   - Estilo "problema": border-left naranja de 4px, sombra naranja suave.
   - Hover: lift + sombra más intensa + fondo `var(--orange-xxl)`.

3. **`Solution.tsx`** — 3 cards de pasos rediseñadas:
   - Iconos: `CloudUploadIcon`, `AutoAwesomeIcon`, `RocketLaunchIcon` en caja gradiente naranja con sombra.
   - Badge numérico circular superpuesto (número de paso) con borde naranja sobre fondo blanco.
   - Border-top animado (gradiente naranja) que aparece en hover.
   - Hover: lift + sombra.

**Verificación:**
- `npx tsc --noEmit` → 0 errores en los archivos modificados (error preexistente en CTAFinal.tsx no relacionado).
- `npx vitest run` (client) → **34/34 verdes**.

**Archivos modificados:**
- `client/src/components/sections/Features.tsx` — rediseñado con MUI icons + nuevo estilo de card.
- `client/src/components/sections/Pain.tsx` — rediseñado con MUI icons + border-left accent.
- `client/src/components/sections/Solution.tsx` — rediseñado con MUI icons + gradient icon box + badge numérico.
- `feature_list.json` — id=23 → `done`.
- `docs/architecture.md` — historial actualizado.
- `memory.md` — esta entrada.

---

## 2026-06-04 — Feature id=24: Fondo animado Vanta.js NET en el Hero

**Tarea ejecutada:** Integración del efecto NET de Vanta.js como fondo animado del Hero, sin modificar el contenido del hero.

**Librería:** `vanta@0.5.24` + `three` (ya instalados como deps de producción del cliente). No requería deps adicionales de runtime.

**Implementación:**
- El `<Box component="section" id="hero">` pasa a `position: 'relative'` (sin background CSS propio).
- Se añade un `<Box ref={vantaRef}>` con `position: absolute, inset: 0, zIndex: 0` — aquí renderiza el canvas Vanta.
- Se añade un overlay `position: absolute, inset: 0, zIndex: 1, pointerEvents: none` con gradiente `transparent → #FFFFFF` para la transición suave al resto del contenido.
- El `<Container>` sube a `zIndex: 2` para quedar sobre el canvas.
- `useEffect` con import dinámico `import('vanta/dist/vanta.net.min')` inicializa el efecto y lo destruye en cleanup.

**Colores (conforme a los specs del feature):**
- `backgroundColor: 0xFFF8F5` (`--orange-xxl`)
- `color: 0xE8440A` (`--orange`)

**Parámetros:** `points: 12, maxDistance: 22, spacing: 18, showDots: true`.

**Tipos TypeScript:** Vanta no tiene `@types/` — se declaró un módulo en `src/vite-env.d.ts` con la firma de `NET(options)`.

**Verificación:**
- `npx tsc --noEmit` → 0 errores en archivos modificados.
- `npx vitest run` → **34/34 verdes** (sin regresiones).

**Archivos modificados:**
- `client/src/components/sections/Hero.tsx` — efecto Vanta NET + overlay gradiente.
- `client/src/vite-env.d.ts` — declaración de módulo `vanta/dist/vanta.net.min`.
- `client/package.json` — `vanta` y `three` como deps de producción.
- `feature_list.json` — id=24 → `done`.
- `docs/architecture.md` — historial actualizado.
- `memory.md` — esta entrada.

---

## 2026-06-04 — Feature id=25: Gradientes verticales en secciones

**Tarea ejecutada:** Agrega gradiente vertical a las secciones El Problema, La Solución, Funciones, Integraciones, Precios, Casos de éxito y Preguntas frecuentes. Solo se modifica el prop `background` del `Box` raíz de cada sección; el contenido no se toca.

**Patrón por tipo de fondo:**

- **Secciones con `--bg-soft`** (Pain, Features, Pricing, FAQ):
  `linear-gradient(180deg, var(--orange-xxl) 0%, var(--bg-soft) 30%, var(--bg-soft) 70%, var(--orange-xxl) 100%)`

- **Secciones con fondo blanco** (Solution, Integrations, Testimonials):
  `linear-gradient(180deg, var(--bg-soft) 0%, #FFFFFF 30%, #FFFFFF 70%, var(--bg-soft) 100%)`

**Archivos modificados (solo prop `background`):**
- `client/src/components/sections/Pain.tsx`
- `client/src/components/sections/Solution.tsx`
- `client/src/components/sections/Features.tsx`
- `client/src/components/sections/Integrations.tsx`
- `client/src/components/sections/Pricing.tsx`
- `client/src/components/sections/Testimonials.tsx`
- `client/src/components/sections/FAQ.tsx`
- `feature_list.json` — id=25 → `done`.
- `docs/architecture.md` — historial actualizado.
- `memory.md` — esta entrada.

---

## 2026-06-04 — Feature id=26: Mejora de la sección Trust (logos de empresas)

**Tarea ejecutada:** Reemplazar los nombres de texto plano de la sección "Empresas que ya automatizan con MaIA" por los logos PNG reales disponibles en `client/public/`.

**Logos usados:**
- Led Studio → `/logo-ledstudio.png`
- Instituto IPG → `/logo-ipg.png`
- IACC → `/logo-iacc.png`
- Embonor → `/logo-embonor.png`

**Estilo aplicado:**
- Logos en `<img>` con `height: 32–40px` (responsive), `objectFit: contain`.
- Estado por defecto: `filter: grayscale(1) opacity(0.5)` — apariencia neutra sobria.
- On hover: `filter: grayscale(0) opacity(1)` + `scale(1.06)` — revela el color real del logo.
- Transición `0.3s ease` en ambas propiedades.

**Archivos modificados:**
- `client/src/components/sections/Trust.tsx` — rediseñado con logos PNG.
- `feature_list.json` — id=26 → `done`.
- `docs/architecture.md` — historial actualizado.
- `memory.md` — esta entrada.
