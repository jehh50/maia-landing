# Arquitectura — MaIA Landing Page

> Última actualización: 2026-06-04 (feature id=27)

---

## Resumen del proyecto

Landing page estática para **MaIA**, una plataforma de agentes de IA para equipos latinoamericanos (desarrollada por Ximple Solutions). El objetivo es capturar leads y demostrar el valor del producto.

**URL en producción:** `app.maiabuilder.ai`  
**Email de contacto:** `maia@maiabuilder.ai`

---

## Stack tecnológico

### Frontend (`/client`)

| Capa | Tecnología |
|------|-----------|
| Bundler | **Vite 5** |
| Framework | **React 18** |
| Lenguaje | **TypeScript 5** (strict) |
| UI Library | **Material UI v6** (`@mui/material` + `@emotion`) |
| Iconos | `@mui/icons-material` |
| Estilos | Theme MUI + `styles/globals.css` (variables CSS, animaciones `fadeUp`/`blink`/`reveal`) |
| Tests | Vitest + `@testing-library/react` + jsdom |
| Fuentes | Google Fonts — Inter (300–800 + itálica) |
| Servidor estático | Apache/Nginx sirviendo `client/dist/`, o Vite dev server (`:5173`) con proxy `/api` |

> El antiguo `index.html` monolítico (~1900 líneas con CSS+JS inline) se conserva en la raíz como `legacy.html` para comparar.

### Backend (`/server`)

| Capa | Tecnología |
|------|-----------|
| Runtime | Node.js ≥ 22 (ESM) |
| Framework HTTP | Express 4 |
| Base de datos | **PostgreSQL** (≥ 14) externo, vía `pg` (Pool). Conexión por `DATABASE_URL` |
| Esquema | Configurable con `DB_SCHEMA` (default `public`). Tabla `leads` con `BIGSERIAL` + `TIMESTAMPTZ` |
| Email | **Nodemailer** (SMTP); se omite si `SMTP_HOST` está vacío |
| Variables de entorno | `dotenv` (ver `server/.env.example`) |
| Tests | Vitest + Supertest contra un esquema PG temporal `maia_test_*` (DROP CASCADE al final) |

> Despliegue recomendado: el sitio estático sigue servido por Apache/Nginx; el API Node escucha en `localhost:3001` y se expone bajo `/api/*` a través de un reverse-proxy.

> `AGENT.md` mencionaba Vite y ahora el frontend es efectivamente una app Vite + React + TS en `/client`. El backend tiene su propio `package.json` en `/server`.

---

## Estructura de archivos

```
maia-landing/
├── legacy.html              # Versión monolítica anterior (HTML+CSS+JS inline) — sólo referencia
├── AGENT.md                 # Instrucciones para agentes IA que trabajen en el repo
├── memory.md                # Registro de tareas ejecutadas por agentes
├── feature_list.json        # Lista de features del proyecto
├── CHECKPOINT.md            # Checkpoints de evaluación
├── client/                  # Frontend Vite + React + TS + MUI
│   ├── package.json
│   ├── vite.config.ts       # Proxy /api → http://localhost:3001 en dev
│   ├── tsconfig.json
│   ├── index.html
│   ├── public/              # logo-maia.svg, isotipo-maia.svg
│   ├── README.md
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── theme/theme.ts   # Theme MUI (brand tokens, Inter, palette extendida)
│       ├── lib/api.ts       # postLead(), EMAIL_RE
│       ├── hooks/useReveal.ts
│       ├── styles/globals.css
│       ├── components/
│       │   ├── Navbar.tsx
│       │   ├── Footer.tsx
│       │   ├── WhatsAppFloat.tsx
│       │   ├── ContactModal.tsx        # Dialog MUI (centro de la app)
│       │   ├── sections/{Hero,Trust,Pain,Solution,Features,Integrations,ROI,Pricing,Testimonials,FAQ,CTAFinal}.tsx
│       │   └── __tests__/ContactModal.test.tsx
│       └── test/setup.ts
├── docs/
│   ├── architecture.md      # Este archivo
│   ├── design-system.md     # Tokens visuales (colores, tipo, espaciado, sombras, componentes, email subset)
│   ├── verification.md      # Política de tests / verificación
│   └── images/
│       ├── logo-maia.svg    # Logo SVG completo (navbar, footer, demo sidebar)
│       └── isotipo-maia.svg # Isotipo SVG para favicon
├── progress/
│   └── current.md           # Sesión activa
├── server/                  # Backend Node.js (Express)
│   ├── package.json
│   ├── .env.example
│   ├── README.md
│   ├── src/
│   │   ├── app.js           # Factory de Express (exportada para tests)
│   │   ├── server.js        # Entrypoint (dotenv + ensureSchema + listen)
│   │   ├── db.js            # pg.Pool + ensureSchema + insertLead
│   │   └── email.js         # Nodemailer SMTP wrapper (no-op si SMTP_HOST vacío)
│   └── tests/
│       ├── contact.test.js  # Vitest + Supertest contra PG real (8 tests)
│       └── email.test.js    # Vitest sobre el mailer Nodemailer (5 tests, transporter mockeado)
└── .vscode/                 # Configuración del editor (no relevante para producción)
```

---

## Estructura interna del frontend (`/client/src`)

### Composición (`App.tsx`)

`App.tsx` mantiene 2 piezas de estado global:
- `contactOpen` / `contactTipo` → controla el `ContactModal`.
- `isAnnual` → toggle de precios anual/mensual (compartido entre `ROI` y `Pricing`).

Las secciones se componen en orden:

```
<Navbar /> → <Hero /> → <Trust /> → <Pain /> → <Solution /> →
<Features /> → <Integrations /> → <ROI /> → <Pricing /> →
<Testimonials /> → <FAQ /> → <CTAFinal /> → <Footer /> →
<WhatsAppFloat /> + <ContactModal />
```

Cualquier sección que necesite abrir el modal recibe `onOpenContact` por prop.

### ContactModal

`Dialog` MUI con formulario controlado (`useState`). Valida nombre y email con `EMAIL_RE`. Submit hace `POST /api/contact` vía `lib/api.ts#postLead`. Muestra `Alert` MUI con éxito/error. Cierra automáticamente 2.2 s después del éxito.

### Theme (`theme/theme.ts`)

Extiende `palette` con `brand` (`orange`, `orangeHover`, `orangeLight`, `orangeXL`, `orangeXXL`, `green`, `greenLight`) y `surface` (`main`, `soft`, `tint`). Sobrescribe `MuiButton`, `MuiContainer`, `MuiTextField`. Tipografía Inter, `borderRadius: 12`.

### Estilos globales (`styles/globals.css`)

Mantiene variables CSS (`--orange`, `--bg-soft`, ...) usadas vía `sx={{ color: 'var(--orange)' }}` cuando el theme MUI no las cubre. Animaciones: `fadeUp`, `blink`, `reveal` (scroll-triggered).

### API client (`lib/api.ts`)

```ts
postLead(payload): Promise<{ ok, status, data }>
```

URL base: `${VITE_API_BASE ?? ''}/api/contact`. En dev, Vite hace proxy a `:3001`.

---

## Estructura interna del legacy (`legacy.html`)

### Secciones HTML (en orden de aparición)

| ID / selector | Descripción |
|---------------|-------------|
| `nav#navbar` | Barra de navegación fija con glassmorphism; se oscurece al hacer scroll |
| `#mobileMenu` | Menú móvil full-screen (visible < 860 px) |
| `#hero` | Hero section: headline, sub-copy, CTAs, estadísticas animadas y demo window |
| `#trust` | Barra de logos de clientes (estática, sin scroll infinito) |
| `.section` (Pain) | 4 pain cards mostrando el problema |
| `#solution` | Grid de 3 pasos + visual interactivo de agentes |
| `#features` | 5 feature cards (se eliminó "Workflows visuales sin código") |
| `#integrations` | Grid de 15 pills de integraciones (se eliminaron Instagram DM, Oracle, Monday) |
| `#roi` | Calculadora de ROI interactiva (sliders) |
| `#pricing` | 4 planes con toggle mensual/anual |
| `#testimonials` | 3 tarjetas de testimonios |
| `#faq` | 7 preguntas tipo acordeón |
| `#cta-final` | CTA final con captura de email |
| `#contactModal` | Modal de contacto / agenda de demo (formulario nombre, empresa, email, teléfono, mensaje) |
| `footer` | Footer con 4 columnas + redes sociales |
| `.wa-float` | Botón flotante de WhatsApp |

### Sistema de diseño (CSS tokens en `:root`)

| Token | Valor | Uso |
|-------|-------|-----|
| `--orange` | `#E8440A` | Color de marca principal |
| `--orange-h` | `#D03A08` | Hover del botón primario |
| `--orange-l` | `#FF6B35` | Variante clara |
| `--bg` | `#FFFFFF` | Fondo general |
| `--bg-soft` | `#FAFAF9` | Fondo de secciones alternadas |
| `--text` | `#1A1410` | Texto primario |
| `--text2` | `#4A3F3A` | Texto secundario |
| `--muted` | `#7A6E6A` | Texto terciario / descripciones |
| `--green` | `#16A34A` | Estados positivos / ahorro |
| `--radius` | `12px` | Bordes estándar |
| `--shadow-*` | sm / md / lg | Sistema de sombras |

### JavaScript (funciones principales)

| Función | Propósito |
|---------|-----------|
| `openMenu()` / `closeMenu()` | Control del menú móvil |
| `toggleFaq(item)` | Acordeón de FAQ (solo 1 abierto a la vez) |
| `toggleBilling()` | Cambia entre precios mensuales/anuales |
| `calcROI()` | Calculadora de ROI en tiempo real (sliders) |
| `sendROI()` | POST `/api/contact` con `tipo='email'` (captura desde ROI) |
| `startTrial()` | POST `/api/contact` con `tipo='email'` (captura desde CTA final) |
| `postLead(payload)` | Helper compartido — `fetch` POST JSON al backend |
| `openContactModal(tipo)` | Abre el modal de contacto/demo (focus + reset de estado) |
| `closeContactModal()` | Cierra el modal (click overlay, botón ✕, tecla Escape) |
| Listener `submit` en `#contactForm` | Valida y envía el formulario al backend |
| `counter(el, target, suffix, dec)` | Animación de contadores numéricos |
| `IntersectionObserver` (×2) | Scroll reveal para `.reveal` + activación de contadores |

**Configuración del API base (frontend):** por defecto los `fetch` apuntan a `/api/contact` (ruta relativa → reverse-proxy). Para apuntar a otro host puede setearse `window.MAIA_API_BASE` antes de que cargue el script principal (ej. `<script>window.MAIA_API_BASE='https://api.maiabuilder.ai'</script>`).

---

## Patrones de UI notables

- **Logo:** SVG externo (`docs/images/logo-maia.svg`) usado en navbar, footer y demo sidebar
- **Favicon:** Isotipo SVG (`docs/images/isotipo-maia.svg`) referenciado en `<head>`
- **Animaciones:** `fadeUp` (entrada de hero), `blink` (dot verde del badge), `reveal` (scroll-triggered via IO)
- **Demo window:** Mockup del panel de la app real (sidebar + lista de agentes)
- **Responsive:** Breakpoints en `640px` y `860px` con media queries inline

---

## Links externos relevantes

- Demo en vivo (YouTube): `https://www.youtube.com/watch?v=EngW7tLk6R8`
- WhatsApp ventas: `https://wa.me/message/maiabuilder`
- Email: `maia@maiabuilder.ai`

> Los antiguos links a Calendly fueron sustituidos por el modal `#contactModal` (feature id=3).

---

## Backend API (`/server`)

### Endpoints

| Método | Ruta | Descripción | Códigos |
|--------|------|-------------|---------|
| `POST` | `/api/contact` | Crea un lead; persiste en PostgreSQL y envía email | `201` ok · `422` email inválido · `500` DB error |
| `POST` | `/api/auth/login`  | Login admin; setea cookie `maia_session` (JWT HS256, httpOnly, 7 días) | `200` ok · `400` campos faltantes · `401` credenciales inválidas |
| `POST` | `/api/auth/logout` | Borra la cookie de sesión | `200` ok |
| `GET`  | `/api/auth/me`     | Devuelve el usuario logueado a partir de la cookie | `200` ok · `401` sin sesión |
| `GET`  | `/api/health`  | Health check (`{ ok, db, mailer }`) | `200` ok · `503` si DB no responde |

### Payload `POST /api/contact`

```json
{
  "nombre":    "Ana Pérez",
  "empresa":   "Acme",
  "email":     "ana@acme.com",
  "telefono":  "+525512345678",
  "industria": "Tecnología / SaaS",
  "mensaje":   "Quiero un demo",
  "tipo":      "demo"
}
```

### Validación (feature 11)

`POST /api/contact` aplica reglas según el flujo:

| Flujo | Email | Nombre | Teléfono | Industria | Mensaje |
|-------|-------|--------|----------|-----------|---------|
| `tipo='demo'` o `'contacto'` | requerido (regex `EMAIL_RE`) | requerido | requerido (regex `^\+\d{7,15}$`, normalizado quitando espacios/`-()`) | opcional | opcional |
| `tipo='email'` (CTA final / ROI) | requerido | opcional | opcional | opcional | opcional |

Errores 422 incluyen `{ error: <msg>, field: 'email' \| 'nombre' \| 'telefono' }` para que el frontend pueda enfocar el campo correcto. `tipo` se normaliza a `demo` si está fuera del set `{demo, email, contacto}`.

`tipo` aceptados: `demo` (modal demo), `contacto` (modal "hablar con ventas"), `email` (capturas one-shot: CTA final + ROI calculator).

Los campos string se truncan: nombre/empresa/industria 120, teléfono 40, mensaje 2000.

Respuesta `201`:

```json
{ "ok": true, "id": 42 }
```

### Persistencia (PostgreSQL externo)

Conexión a una instancia PostgreSQL ≥ 14 vía `pg.Pool` con `DATABASE_URL`. Compatible con Supabase, Neon, RDS, Cloud SQL, etc. (poner `PGSSL=true` para esos providers).

El servidor llama `ensureSchema()` al arrancar — crea el esquema, la tabla y los índices si no existen (operación idempotente, segura para arranques repetidos).

```sql
CREATE TABLE leads (
  id          BIGSERIAL PRIMARY KEY,
  nombre      TEXT,
  empresa     TEXT,
  email       TEXT NOT NULL,
  telefono    TEXT,
  pais        TEXT,                          -- añadido en feature 12 ("Venezuela", "México", ...)
  pais_iso    TEXT,                          -- añadido en feature 12 (ISO 3166-1 alpha-2: "VE", "MX", ...)
  industria   TEXT,                          -- añadido en feature 11
  mensaje     TEXT,                          -- "comentarios" en la UI
  tipo        TEXT NOT NULL DEFAULT 'demo',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX leads_email_idx      ON leads (email);
CREATE INDEX leads_created_at_idx ON leads (created_at DESC);
```

> `ensureSchema()` ejecuta `ALTER TABLE leads ADD COLUMN IF NOT EXISTS {industria,pais,pais_iso} TEXT` para migrar tablas creadas en features previos. Operación idempotente.

### Detección de país desde el teléfono (feature 12)

Cuando el endpoint recibe `telefono` en formato E.164 (`+<country><number>`), `src/phone.js#detectCountry(phone)` lo decodifica con [`libphonenumber-js`](https://www.npmjs.com/package/libphonenumber-js) y devuelve `{ iso, name }` (ej. `+584242848748` → `{ iso: 'VE', name: 'Venezuela' }`). El nombre se localiza al español con `Intl.DisplayNames(['es'], { type: 'region' })` (nativo de Node ≥ 18). Si el teléfono no es válido devuelve `{ iso: '', name: '' }` (no falla la inserción). Ambos campos se persisten en `pais` y `pais_iso`, y se incluyen en el cuerpo del email enviado a ventas.

La tabla vive en el esquema definido por `DB_SCHEMA` (default `public`). En tests se usa un esquema `maia_test_<timestamp>_<rand>` que se borra con `DROP SCHEMA ... CASCADE` al terminar.

### Persistencia (PostgreSQL)

| Variable | Default | Descripción |
|----------|---------|-------------|
| `PGHOST` | localhost | Host PostgreSQL |
| `PGPORT` | `5432` | Puerto PostgreSQL |
| `PGUSER` | postgres | Usuario PostgreSQL |
| `PGPASSWORD` | 123456 | Contraseña PostgreSQL |
| `PGDATABASE` | maia-landing | Nombre de la base de datos PostgreSQL |



### Email (Nodemailer / SMTP) — 2 correos por lead (feature 10)

Se usa `nodemailer` con transporte SMTP estándar. El módulo `src/email.js` expone `createMailer({ host, port, secure, user, pass, from, to, transporter? })` que devuelve un objeto con `sendLead(lead, id)`.

**`sendLead(lead, id)` envía dos correos por cada lead recibido:**

1. **Notificación a ventas** → `to: MAIL_TO`. Contiene los datos del lead en una tabla HTML (nombre, empresa, email, teléfono, país, industria, tipo, comentarios). `replyTo` apunta al email del lead para permitir respuesta directa.
2. **Confirmación al usuario** → `to: lead.email`. Mensaje amistoso "gracias por agendar tu demo, te contactamos en menos de 24 h". **No** contiene `mensaje`, `industria` ni `empresa` del lead — es solo confirmación.

Ambos correos comparten:
- **Header** con bg `#E8440A` y logo MaIA blanco (`cid:logo-maia`, attachment inline desde `docs/images/logo-maia.svg`, `contentType: image/svg+xml`).
- **Footer** `#FAFAF9` con isotipo MaIA (`cid:isotipo-maia`, desde `docs/images/isotipo-maia.svg`) y texto muted `#A89E9A`.
- **Body card** blanca con borde fino `#F0EBE8`, sin `box-shadow`, ancho 600 px centrado.
- CTA pill `#E8440A` `border-radius: 100px`.
- Multipart alternative: `text/plain` + `text/html`. Clientes que no muestran HTML reciben la versión texto.

> Outlook desktop no renderiza SVG inline. Si el cliente del usuario tiene Outlook como prioridad, sustituir los logos por PNG fallback (manteniendo CID). El subset se documenta en `docs/design-system.md` §10.

Todos los strings del usuario se escapan con `escapeHtml` (cubre `& < > " '`) antes de inyectarse al HTML — anti-XSS.

#### Contrato de respuesta (`sendLead`)

```ts
// Ambos correos enviados OK
{
  status: 'sent',
  sentTo: ['ventas@maia', 'lead@empresa'],
  messageIds: ['<m1@..>', '<m2@..>'],
  results: { sales: { status:'sent', messageId, response, accepted, rejected, to },
             user:  { status:'sent', ... } }
}
// Uno de los dos falló; el otro salió
{
  status: 'partial',
  sentTo: ['ventas@maia'],          // sólo los que sí salieron
  messageIds: ['<m1@..>'],
  results: { sales: { status:'sent', ... },
             user:  { status:'failed', statusCode, reason, body, to } }
}
// Ambos fallaron
{ status: 'failed', sentTo: [], messageIds: [],
  results: { sales: {...failed}, user: {...failed} } }
// Mailer global deshabilitado (sin SMTP_HOST)
{ status: 'skipped', reason: 'SMTP_HOST no configurada',
  sentTo: [], messageIds: [],
  results: { sales: { status:'skipped' }, user: { status:'skipped' } } }
```

`app.js` consume este objeto y loggea **un log por destinatario** con prefijo `[mail]`:

```
[mail] lead=42 sales→maia@maiabuilder.ai status=sent messageId=<msg-1@test>
[mail] lead=42 user→ana@acme.com         status=sent messageId=<msg-2@test>
```

Si uno falla y el otro no, solo se loggea como `status=failed` el que falló (incluye `code` + `reason`). Si el mailer está deshabilitado (sin SMTP_HOST), se loggea una sola línea informativa. **La respuesta al cliente sigue siendo `{ ok: true, id }`** — sin detalles del transporte.

Si `SMTP_HOST` está vacío, `createMailer` devuelve un mailer no-op y el lead se guarda igual (`201`). Si los envíos fallan con SMTP configurado, se loggea pero la respuesta sigue siendo `201` — el lead ya está guardado y el correo es best-effort.

#### Proveedores SMTP probados

Cualquier proveedor SMTP estándar funciona. Ejemplos comentados en `server/.env.example`:
- **Gmail** (con App Password): `smtp.gmail.com:465` SSL (recomendado, `SMTP_SECURE=true`) o `:587` STARTTLS (`SMTP_SECURE=false`) si el firewall lo permite. En el feature 17 se detectó que el puerto 587 estaba bloqueado en el entorno (TCP timeout reproducible), por eso la config en producción usa `:465`. Para que SPF/DKIM alineen, `MAIL_FROM` debe coincidir con `SMTP_USER` o ser un alias verificado en "Send mail as".
- **Brevo** (ex-Sendinblue): `smtp-relay.brevo.com:587`.
- **Mailgun**: `smtp.mailgun.org:587`.
- **Amazon SES**: `email-smtp.us-east-1.amazonaws.com:587`.
- **Resend**: `smtp.resend.com:587`.

### Variables de entorno

| Variable | Default | Descripción |
|----------|---------|-------------|
| `PORT` | `3001` | Puerto HTTP |
| `DATABASE_URL` | — (requerido) | Connection string PostgreSQL (`postgres://user:pass@host:port/db` o socket Unix `postgres:///db?host=/var/run/postgresql`) |
| `DB_SCHEMA` | `public` | Esquema donde vive la tabla `leads` |
| `PGSSL` | `false` | `true` fuerza TLS (necesario en Supabase/Neon/RDS) |
| `CORS_ORIGIN` | `*` | Origen(es) permitidos; coma-separado |
| `SMTP_HOST` | _vacío_ | Host SMTP — vacío = no envía email |
| `SMTP_PORT` | `587` | 587 para STARTTLS, 465 para SSL implícito |
| `SMTP_SECURE` | `false` | `true` ⇒ conecta con TLS desde el inicio (puerto 465) |
| `SMTP_USER` / `SMTP_PASS` | _vacío_ | Auth SMTP; si vacío, conexión sin auth |
| `MAIL_FROM` | `noreply@maiabuilder.ai` | Sender autorizado en el proveedor SMTP |
| `MAIL_TO` | `maia@maiabuilder.ai` | Destinatario de notificaciones |
| `AUTH_SECRET` | _vacío_ | Secreto HS256 para firmar el JWT de sesión admin. Si está vacío, se genera uno aleatorio al boot (warning) y las cookies emitidas no sobreviven a un reinicio. |
| `TEST_DATABASE_URL` | usa `DATABASE_URL` | Opcional: sobrescribe la URL solo para tests |

### Desarrollo / despliegue

```bash
cd server
cp .env.example .env       # editar DATABASE_URL si la BD no es local
npm install
npm run dev                # node --watch src/server.js → http://localhost:3001
npm test                   # vitest — 13 tests verdes (8 contact + 5 email; PG accesible)
```

> En el primer arranque se ejecuta `ensureSchema()` y se crea la tabla `leads` (idempotente). Si quieres migrarla a otro proyecto basta con dejar que el servidor arranque contra la nueva `DATABASE_URL`.

En producción, Apache/Nginx sirve los estáticos y reverse-proxy de `/api/*` → `localhost:3001`.

---

## Deuda técnica y observaciones

1. **CSS/JS inline:** Dificulta el mantenimiento. Candidato a separar en `styles.css` y `main.js`.
2. **Logos de clientes (trust bar):** Son placeholders con emojis, no SVGs reales.
3. **Sin `<link rel="canonical">`** ni OG tags (Open Graph / Twitter Card) — SEO mejorable.
4. **Sin rate-limiting / captcha en `/api/contact`:** El backend no tiene anti-spam (solo límites de longitud). Candidato para `express-rate-limit` o hCaptcha/Turnstile cuando se publique.
5. **`MAIA_API_BASE` opcional:** No hay archivo de configuración por entorno en el frontend; se asume same-origin.

---

## Historial de cambios

| Fecha | Feature | Cambio |
|-------|---------|--------|
| 2026-05-26 | id=1 | Análisis inicial del proyecto, creación de `docs/architecture.md` |
| 2026-05-26 | id=2 | Logo reemplazado por SVG; favicon añadido; botón Navbar "Prueba gratis" → "Agenda un demo"; "Ver demo en vivo" apunta a YouTube; hero CTA primario → "Agenda un demo"; feature "Workflows visuales" eliminada; integraciones Instagram DM, Oracle, Monday eliminadas; botones "Probar gratis 14 días" eliminados de planes de precios; badge CTA final → "Sin riesgos" |
| 2026-05-26 | id=3 | Backend Node.js/Express creado en `/server`: `POST /api/contact` con persistencia SQLite y envío SMTP vía Nodemailer (7 tests Vitest/Supertest verdes). Modal `#contactModal` añadido a `index.html` con formulario completo (nombre, empresa, email, teléfono, mensaje). Botones "Agenda un demo" del navbar, menu móvil, hero, pricing y CTA final ahora abren el modal en vez de Calendly. `startTrial()` y `sendROI()` reconectadas al nuevo endpoint. Endpoint PHP previo (`/api/contact.php`) eliminado. |
| 2026-05-26 | id=4 | Motor de BBDD migrado de SQLite a **PostgreSQL externo**. `better-sqlite3` reemplazado por `pg` (Pool). Conexión por `DATABASE_URL`. Nueva tabla con `BIGSERIAL` + `TIMESTAMPTZ` e índices en `email` y `created_at DESC`. Esquema configurable por `DB_SCHEMA`. `ensureSchema()` se ejecuta al arrancar el servidor. Health check ahora reporta estado de la DB y devuelve `503` si no responde. Tests reescritos para correr en un esquema temporal real (`maia_test_*`) — 8/8 verdes. `/data/` ya no se usa para persistencia (queda solo para artefactos futuros). |
| 2026-05-26 | id=5 | **SendGrid** sustituye a Nodemailer/SMTP en `src/email.js`. Variables `SMTP_*` eliminadas; nuevas `SENDGRID_API_KEY` + `SENDGRID_SANDBOX`. El mailer ahora envía además HTML escapado (anti-XSS) junto al texto plano. Inyección de cliente para tests (no se llama a la red). 4 tests nuevos en `tests/email.test.js`. Suite total: 12/12 verdes. Script `dev-fullstack.js` añadido para servir landing + API en un solo puerto en desarrollo. |
| 2026-05-26 | id=6 | **Migración a Vite + React 18 + TypeScript + Material UI** en `/client/`. Toda la landing fue componetizada (13 secciones + Navbar/Footer/WhatsAppFloat/ContactModal). Theme MUI con tokens de marca extendidos; `globals.css` para animaciones. `lib/api.ts` con types compartidos. Tests RTL del ContactModal (6 tests verdes, mock fetch). Vite proxy `/api` → `:3001` en dev. Build prod: `460 KB / 141 KB gz`. El antiguo `index.html` monolítico se conserva como `legacy.html`. `dev-fullstack.js` ahora sirve `client/dist` (con fallback a legacy si no hay build) y SPA-fallback para rutas no-API. Tests del proyecto: 12 backend + 6 client = **18 verdes**. |
| 2026-05-26 | post-id=6 | **Observability del envío sin filtración al cliente.** `sendLead()` ya no lanza: devuelve `{ status: 'sent'\|'skipped'\|'failed', ... }`. `app.js` loggea con prefijo `[mail]` el resultado estructurado (incluye reason y body del provider en failures) pero **la respuesta al cliente sigue siendo `{ ok, id }`** — sin info interna del transporte. Test nuevo cubre el caso `failed` cuando el provider lanza 401. |
| 2026-05-26 | id=8 | **Migración SendGrid → Nodemailer/SMTP.** Motivo: la cuenta SendGrid agotó créditos en producción y el upgrade no era viable en este momento. `@sendgrid/mail` desinstalado, `nodemailer` reinstalado. `src/email.js` reescrito manteniendo el mismo contrato de respuesta (`status: sent\|skipped\|failed`), los builders `buildSubject/buildText/buildHtml` (HTML escapado anti-XSS) e inyección de `transporter` para tests sin red. Variables `SENDGRID_*` eliminadas; restauradas `SMTP_HOST/PORT/SECURE/USER/PASS`. `.env.example` documenta ejemplos de Gmail/Brevo/Mailgun/SES/Resend. Suite: **13 backend** (5 email + 8 contact) + 6 client = **19 verdes**. |
| 2026-05-26 | design-system | **`docs/design-system.md`** creado como fuente única de verdad para colores, tipografía, espaciado, sombras, componentes y subset para correos HTML. Extraído del theme MUI + `globals.css` + legacy.html. |
| 2026-05-26 | id=11 | **Actualización del modal de agendamiento.** Añadidos campos `industria` (Select MUI con lista cerrada de 12 + Otro, opcional) y nueva input `MuiTelInput` con bandera y prefijo de país (default MX, preferidos: MX/CO/PE/CL/AR/VE/US/ES). El label "Mensaje" se renombró a "Comentarios". Validación cliente y servidor: `nombre`, `email` y `telefono` son ahora requeridos (excepto para `tipo='email'` de CTA final/ROI). Errores 422 incluyen `field` para foco en cliente. Backend: nueva columna `industria TEXT` con `ALTER TABLE ADD COLUMN IF NOT EXISTS`. Tests: **15 contact + 5 email = 20 backend** + **7 client = 27 verdes**. Bundle: 629 KB / 189 KB gz (subió por mui-tel-input). |
| 2026-05-26 | id=12 | **Detección de país en backend desde el prefijo del teléfono.** `libphonenumber-js` instalado. Nuevo módulo `src/phone.js#detectCountry(phone)` que devuelve `{ iso, name }` (ISO 3166-1 + nombre en español vía `Intl.DisplayNames`). Columnas nuevas `pais TEXT` y `pais_iso TEXT` añadidas vía `ALTER TABLE`. El correo a ventas ahora incluye una fila "País" con el nombre + código ISO. Tests: **8 phone + 17 contact + 5 email = 30 backend verdes**. Verificado E2E con `+58→Venezuela`, `+52→México`, `+1→Estados Unidos`, `+34→España`, `+57→Colombia`, `+51→Perú`. |
| 2026-05-26 | id=15 | **Roles de usuarios (admin/editor).** Nueva columna `users.role TEXT NOT NULL DEFAULT 'editor'` añadida vía `ALTER TABLE ADD COLUMN IF NOT EXISTS` dentro de `ensureSchema()` con try/catch tolerante (ignora `42P01` por si feature 14 todavía no creó `users`). Nuevo módulo `server/src/roles.js` con `ROLES = { ADMIN, EDITOR }`, `hasRole(user, ...roles)` y `requireRole(...roles)` (middleware Express → 403 `forbidden` si el rol no está permitido; asume `req.user` populado por `requireAuth` de feature 14). Script `server/scripts/seed-users.js` para asegurar el admin inicial leyendo `MAIA_ADMIN_EMAIL`/`MAIA_ADMIN_PASSWORD` (importa dinámicamente `users.js`; si feature 14 no está disponible aún, sale con mensaje claro). Tests: **15 nuevos en `tests/roles.test.js`** (ROLES inmutable, `hasRole` con admin/editor/null/lista/sin role/vacío, `requireRole` next vs 403 y sin `req.user`, migración: columna `role` existe con default `editor`, ensureSchema tolerante a esquemas vacíos). Suite total backend: **49 verdes**. |
| 2026-05-28 | id=18 | **Editor enriquecido para artículos del blog.** Nuevo componente reusable `client/src/components/MarkdownRenderer.tsx` (`react-markdown` + `remark-gfm` con custom components para `img` lazy, `a` externo target=_blank, `blockquote` naranja, `table` con bordes finos, y conversión de un parágrafo con un único link de YouTube/Vimeo a `<iframe>` 16:9 responsivo con `loading="lazy"`). `BlogArticle.tsx` ahora delega en `<MarkdownRenderer body={body_md} />`. `ArticleEdit.tsx` reemplaza el `<TextField multiline>` por `@uiw/react-md-editor` (toolbar con bold/italic/headings/listas/blockquote/code/link/imagen/tabla, preview lateral en vivo) + sección "Vista previa" debajo con `<MarkdownRenderer compact />` para validar YouTube/Vimeo embebidos. Dependencias añadidas: `@uiw/react-md-editor ^4.1.1`. **9 nuevos tests RTL en `MarkdownRenderer.test.tsx`** (h1, h2, tabla GFM, YouTube iframe, Vimeo iframe, img lazy, link externo noopener, `getEmbedUrl()`, blockquote). Suite client: 25 previos + 9 = **34 verdes**. Backend: 86 sin cambios. Bundle prod: 1 727 KB / 567 KB gz. |
| 2026-05-26 | id=14 | **Login admin + interfaz `/admin` protegida.** Backend: nueva tabla `users (id, email UNIQUE, password_hash, name, created_at)` creada por `ensureSchema()`. Hash de password con `bcryptjs` (salt rounds = 12). Endpoints `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me`; sesión por cookie httpOnly `maia_session` con JWT firmado en `AUTH_SECRET` (HS256, 7 días, SameSite=Lax, Secure en producción). Helper `requireAuth(req,res,next)` exportado por `createAuthRouter`. Script CLI `server/scripts/create-user.js <email> <password> [name]` para crear el primer admin. **10 nuevos tests en `tests/auth.test.js`** (401 sin/malos creds, 200 + cookie en login OK, /me con y sin cookie, cookie manipulada, logout limpia cookie, email duplicado, hash bcrypt persistido). Suite total backend: **59 verdes**. Frontend: `react-router-dom@^6` instalado; rutas `/` (landing), `/admin/login`, `/admin` (protegida). Nuevos componentes en `client/src/admin/` (`Login`, `AdminGuard`, `AdminLayout`, `AdminHome`). Helpers `login/logout/getMe` en `lib/api.ts` usan `credentials:'include'`. AdminLayout deja placeholders "Leads" y "Blog" para features 16 y 13. **5 nuevos tests RTL en `client/src/admin/__tests__/Login.test.tsx`** (render, validaciones vacías + email inválido, POST `/api/auth/login` con `credentials:'include'`, error 401). Suite total client: **12 verdes**. Bundle prod: 661 KB / 200 KB gz. |

---

## Auth & Admin — Roles (feature 15)

Modelo simple de **autorización basada en roles** para los endpoints administrativos.

### Roles soportados

| Rol      | Descripción                                                                 |
|----------|-----------------------------------------------------------------------------|
| `admin`  | Acceso total: gestión de usuarios, leads, configuración, blog.              |
| `editor` | Acceso limitado: redactar/publicar artículos del blog. Rol por defecto.     |

### Esquema en base de datos

La tabla `users` (creada por feature 14) recibe una columna adicional:

```sql
ALTER TABLE "<schema>".users ADD COLUMN IF NOT EXISTS role TEXT NOT NULL DEFAULT 'editor';
```

Esta migración vive en `server/src/db.js#ensureSchema` envuelta en `try/catch` que ignora `42P01` (undefined_table), permitiendo arranques en los que la tabla `users` todavía no fue creada.

### API de autorización (`server/src/roles.js`)

```js
import { ROLES, hasRole, requireRole } from './roles.js';

ROLES.ADMIN   // 'admin'
ROLES.EDITOR  // 'editor'

hasRole({ role: 'admin' }, 'admin');             // true
hasRole({ role: 'editor' }, 'admin');            // false
hasRole({ role: 'editor' }, 'admin', 'editor');  // true (lista)
hasRole(null, 'admin');                          // false
```

`requireRole(...roles)` es un **middleware Express** que asume un `requireAuth` previo ya populó `req.user`:

```js
router.delete('/api/leads/:id', requireAuth, requireRole('admin'), handler);
router.patch('/api/posts/:id',  requireAuth, requireRole('admin', 'editor'), handler);
```

Respuesta cuando el rol no está permitido (`403`):

```json
{ "ok": false, "error": "forbidden", "message": "No tienes permisos para realizar esta acción." }
```

### Bootstrap del admin inicial

Script CLI `server/scripts/seed-users.js`:

```bash
MAIA_ADMIN_EMAIL=admin@maiabuilder.ai \
MAIA_ADMIN_PASSWORD=cambia-esto \
node scripts/seed-users.js
```

- Si el user existe → `UPDATE role='admin'`.
- Si no existe → lo crea con `role='admin'` (reusa `createUser` de feature 14 por **import dinámico**; si `src/users.js` aún no existe, sale con mensaje claro y exit code 1).
- Imprime al final `id` y `email` del admin asegurado.

Las variables nuevas viven en `.env.example`:

```env
MAIA_ADMIN_EMAIL=
MAIA_ADMIN_PASSWORD=
```

---

## Auth & Admin — Login (feature 14)

Autenticación de administradores con sesión por **cookie httpOnly** que transporta un JWT firmado.

### Esquema en base de datos

```sql
CREATE TABLE "<schema>".users (
  id            BIGSERIAL PRIMARY KEY,
  email         TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  name          TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX users_email_idx ON "<schema>".users (email);
```

La columna `role` la añade feature 15 en otra `ALTER TABLE` (sin acoplarse a feature 14).

### Endpoints (`server/src/auth.js`)

| Método | Ruta                  | Body                          | Respuesta OK                                                | Errores |
|--------|-----------------------|-------------------------------|-------------------------------------------------------------|---------|
| `POST` | `/api/auth/login`     | `{ email, password }`         | `200 { ok:true, user:{ id, email, name } }` + `Set-Cookie`  | `400` campos faltantes · `401` credenciales inválidas |
| `POST` | `/api/auth/logout`    | —                             | `200 { ok:true }` + cookie expirada                         | — |
| `GET`  | `/api/auth/me`        | _(cookie)_                    | `200 { user:{ id, email, name } }`                          | `401` sin cookie / cookie inválida |

### Cookie de sesión

- **Nombre:** `maia_session`.
- **Atributos:** `HttpOnly`, `SameSite=Lax`, `Path=/`, `Max-Age=604800` (7 días). `Secure` se activa cuando `NODE_ENV=production`.
- **Contenido:** JWT HS256 firmado con `AUTH_SECRET` (`{ sub, email, iat, exp }`).

### Hashing de contraseñas (`server/src/users.js`)

`bcryptjs` con `SALT_ROUNDS = 12`. API:

```js
import { createUser, findUserByEmail, verifyPassword } from './users.js';

const user = await createUser(pool, { email, password, name }, { schema });
const found = await findUserByEmail(pool, email, { schema });
const ok    = await verifyPassword(found, plaintext);
```

`createUser` lanza `Error('email_taken')` (con `err.code = 'email_taken'`) si el email ya existe.

### Helper `requireAuth`

`createAuthRouter({ pool, schema })` devuelve además un middleware `requireAuth(req, res, next)` que:

1. Lee `req.cookies.maia_session`.
2. Verifica el JWT con `AUTH_SECRET` (HS256).
3. Si OK, popula `req.user = { id, email, name }` y llama `next()`.
4. Si KO, responde `401 { error: 'No autenticado' }`.

Las features 13 (mantenedor blog) y 16 (leads admin) lo encadenan con `requireRole(...)` de feature 15:

```js
router.get('/api/admin/leads', requireAuth, requireRole('admin'), handler);
```

### Bootstrap del primer admin

```bash
node scripts/create-user.js <email> <password> "Nombre opcional"
```

Carga `.env`, ejecuta `ensureSchema()` y crea un usuario. Útil para sembrar el primer admin antes de tener UI de gestión de usuarios.

### Frontend (`client/src/admin/`)

- **Router** (`main.tsx`): `BrowserRouter` con rutas `/` (landing), `/admin/login` (Login) y `/admin/*` (envuelto en `AdminGuard` → `AdminLayout` con `Outlet`).
- **`AdminGuard`**: hace `GET /api/auth/me`; si responde 401 redirige a `/admin/login` (recordando `from` en `location.state`); si 200, renderiza children con el `user`.
- **`AdminLayout`**: sidebar minimal (logo + nombre + botón "Cerrar sesión") + `<Outlet />`. Las entradas "Leads" y "Blog" están como placeholders deshabilitados (se activarán en features 16 y 13).
- **`Login.tsx`**: formulario MUI con email + password, validación cliente (`EMAIL_RE`), POST a `/api/auth/login` vía `lib/api.ts#login()` (que usa `credentials:'include'` para que el navegador envíe/reciba la cookie).
- **`AdminHome.tsx`**: página inicial "Bienvenido, {name}".

Helpers en `lib/api.ts`:

```ts
login(email, password): Promise<{ ok, status, data: { user } | { error } }>
logout(): Promise<{ ok, status, data }>
getMe():  Promise<{ ok, status, data: { user } | { error } }>
```

Todos usan `credentials:'include'` para que la cookie `maia_session` viaje en cada request (mismo origin en dev gracias al proxy `/api` de Vite).

---

## Blog (feature 13)

CRUD de artículos accesible para `admin` y `editor`. Solo `admin` puede borrar.

### Schema (`articles`)

```sql
CREATE TABLE articles (
  id           BIGSERIAL PRIMARY KEY,
  slug         TEXT UNIQUE NOT NULL,
  title        TEXT NOT NULL,
  excerpt      TEXT,
  body_md      TEXT NOT NULL,           -- Markdown del cuerpo
  cover_url    TEXT,
  status       TEXT NOT NULL DEFAULT 'draft',   -- 'draft' | 'published'
  author_id    BIGINT REFERENCES users(id) ON DELETE SET NULL,
  published_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX articles_status_idx ON articles (status, published_at DESC);
CREATE INDEX articles_slug_idx   ON articles (slug);
```

Cuando un draft pasa a `published` por primera vez, `published_at` se setea automáticamente a `NOW()`.

### Endpoints (`server/src/articlesRouter.js`)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| `GET`    | `/api/articles`              | público                        | Lista publicados, `?limit&offset` |
| `GET`    | `/api/articles/:slug`        | público                        | Uno publicado; 404 si draft |
| `GET`    | `/api/admin/articles`        | editor/admin                   | Lista TODOS (drafts + publicados) |
| `GET`    | `/api/admin/articles/:id`    | editor/admin                   | Uno por id |
| `POST`   | `/api/admin/articles`        | editor/admin                   | Crea (si no hay slug, `slugify(title)`). 422 si falta title/body_md. 409 slug duplicado. |
| `PATCH`  | `/api/admin/articles/:id`    | editor/admin                   | Update parcial. Marca `published_at` al primer "published". |
| `DELETE` | `/api/admin/articles/:id`    | **solo admin**                 | 403 si role=editor |

`server/src/articles.js` expone los helpers puros (`createArticle`, `updateArticle`, `deleteArticle`, `getArticleBySlug/Id`, `listArticles`, `slugify`).

### UI admin (`client/src/admin/articles/`)

- **`ArticlesList.tsx`** — tabla MUI con title/slug/status/updated. Botón "Nuevo artículo". Edit y Delete por fila (Delete solo si `user.role==='admin'`).
- **`ArticleEdit.tsx`** — formulario create/edit con title, slug (autogenerado y editable), excerpt, body_md (textarea grande, fuente monospace), cover_url, status. Submit POST/PATCH.
- Rutas: `/admin/articles`, `/admin/articles/new`, `/admin/articles/:id`.

---

## Visualización de leads (feature 16)

### Endpoints (`server/src/leadsRouter.js`)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| `GET` | `/api/admin/leads`      | editor/admin | Lista con filtros `?q&tipo&pais_iso&limit&offset`. Devuelve `{ rows, total, limit, offset }`. Limit default 50, max 200. |
| `GET` | `/api/admin/leads/:id`  | editor/admin | Detalle de un lead. 404 si no existe. |

`q` matchea ILIKE (case-insensitive) contra `nombre`, `email`, `empresa`. `tipo` y `pais_iso` son filtros exactos.

`server/src/leads.js` expone `listLeads(pool, schema, filters)` y `getLeadById(pool, schema, id)`.

### UI admin (`client/src/admin/leads/`)

- **`LeadsList.tsx`** — tabla MUI con TablePagination server-side (limit, offset). Filtros encima: input búsqueda (debounce 300 ms), Select tipo, Select país (autocompletado desde el set actual). Click en fila abre `LeadDetailDialog`.
- **`LeadDetailDialog.tsx`** — Dialog MUI con todos los campos del lead, link `mailto:` y `https://wa.me/<telefono>` cuando aplica.
- Ruta: `/admin/leads`.

---

## Blog público (feature 7)

Visualización pública de los artículos publicados (consume los endpoints del feature 13). Sin auth, sin cookies.

### Cliente API (`client/src/lib/api.ts`)

```ts
export type PublicArticle = Pick<AdminArticle,
  'id' | 'slug' | 'title' | 'excerpt' | 'body_md' | 'cover_url' |
  'published_at' | 'created_at' | 'updated_at'
>;

listPublicArticles({ limit?, offset? }): Promise<{ rows: PublicArticle[] }>
getPublicArticleBySlug(slug): Promise<{ article: PublicArticle }>
```

Ambos usan un helper interno `publicJson()` que es como `apiJson` pero con `credentials:'omit'` (los endpoints públicos del blog no requieren cookie de sesión).

### Sección en la landing (`client/src/components/sections/Blog.tsx`)

- ID `#blog`. Padding vertical responsive, fondo `var(--bg-soft)`. Header overline "Blog" + h2 "Ideas y guías de MaIA" + párrafo descriptivo.
- Llama a `listPublicArticles({ limit: 3 })` al montar.
- Cada card (`BlogCard`, exportada para reuso en `/blog`) muestra cover (`cover_url`) o fondo gradient naranja con la primera letra del título cuando falta. Incluye fecha (`published_at` o `created_at`), título, excerpt (usa `excerpt` o las primeras ~140 chars de `body_md` con markdown stripped) y CTA "Leer artículo →".
- Card linkea a `/blog/:slug` vía `Link component={RouterLink}`.
- Botón "Ver todos los artículos" → `/blog` (variante outlined).
- **Graceful degradation:** si la API falla o `rows` viene vacío, el componente retorna `null` y la landing no muestra la sección.
- Animaciones de scroll vía `useReveal()` (clase `.reveal`).
- Posición en la landing: `<Testimonials /> → <Blog /> → <FAQ /> → <CTAFinal />`.

### Páginas dedicadas (`client/src/pages/`)

- **`BlogIndex.tsx`** (`/blog`):
  - Header sticky con logo MaIA (link a `/`) y botón outlined "Volver a la home".
  - Hero centrado con overline "Blog", h1 "Ideas y guías de MaIA" y párrafo descriptivo (fondo `var(--bg-soft)`).
  - Grid de todos los artículos publicados (`limit: 50`). Estados: spinner mientras carga, `Alert` rojo si falla, mensaje "Aún no hay artículos publicados." cuando `rows: []`.
  - Reutiliza `BlogCard` exportada por `Blog.tsx`.
  - Footer compartido (`components/Footer.tsx`).

- **`BlogArticle.tsx`** (`/blog/:slug`):
  - Header sticky con logo + botón outlined "Volver al blog" (link a `/blog`).
  - Estado `loading` → `Skeleton` de portada + título + párrafo.
  - Estado `not-found` (HTTP 404) → "Artículo no encontrado" + CTA primario "Volver al blog".
  - Estado `error` → `Alert` con mensaje genérico.
  - Estado `ok` → portada opcional (`cover_url`) + h1 título + fecha + cuerpo markdown renderizado con `react-markdown@9` + `remark-gfm@4` (tablas, autolinks, listas con tareas).
  - Container con `max-width: 720px` para lectura cómoda.
  - Estilos del markdown respetan el design system: h1/h2/h3 con tipografía Inter (`letter-spacing -0.025em`), `code/pre` con fondo `#FAFAF9` y borde `var(--border)`, `blockquote` con border-left naranja, links naranja, tablas con bordes finos. `react-markdown` sanitiza por defecto (no HTML arbitrario).
  - Footer compartido.

### Router (`client/src/main.tsx`)

```tsx
<Route path="/blog"        element={<BlogIndex />} />
<Route path="/blog/:slug"  element={<BlogArticle />} />
```

Ambas rutas se añaden **antes** del bloque `/admin/*` y conviven con la landing `/` sin tocar la lógica existente.

### Dependencias añadidas

```
react-markdown ^9.1.0
remark-gfm     ^4.0.1
```

### Tests RTL (`client/src/pages/__tests__/`)

- **`BlogIndex.test.tsx`** — 3 tests:
  - Render del overline "Blog" + h1 "Ideas y guías de MaIA".
  - Listado de cards con datos mockeados (verifica llamada a `/api/articles`).
  - Estado vacío amable cuando `rows: []`.
- **`BlogArticle.test.tsx`** — 3 tests:
  - `body_md = '## Subtitulo\n\nCuerpo...'` produce `<h2>Subtitulo</h2>` en el DOM (validación del render markdown).
  - HTTP 404 → muestra "Artículo no encontrado".
  - "Volver al blog" tiene `href="/blog"` y navega correctamente (RouterLink).

Mock con `vi.spyOn(globalThis, 'fetch')`, render en `<MemoryRouter>` + `<ThemeProvider>`. Total client: 19 previos + 6 nuevos = 25 verdes.

---

## Editor enriquecido de artículos (feature 18)

### Resumen

El campo `body_md` de un artículo se sigue persistiendo en backend como `TEXT` (sin cambios en schema). En frontend se enriqueció en dos lados:

1. **Edición (`/admin/articles/new` y `/admin/articles/:id`)** — el `<TextField multiline>` simple se reemplaza por `@uiw/react-md-editor` con toolbar (bold, italic, headings, listas, blockquote, code inline + bloque, link, imagen, **tabla**), preview en vivo lateral y atajos de teclado. Debajo se renderiza una vista previa adicional con el mismo `MarkdownRenderer` que usa el blog público para que el editor vea exactamente el resultado final (incluye embebido de YouTube/Vimeo).
2. **Visualización pública (`/blog/:slug`)** — el render markdown ya no vive inline en `BlogArticle.tsx`; ahora delega en el componente reusable `MarkdownRenderer`.

### Componente `MarkdownRenderer` (`client/src/components/MarkdownRenderer.tsx`)

```tsx
<MarkdownRenderer body={string} compact?={boolean} sx?={SxProps} />
```

- Usa `react-markdown@9` + `remark-gfm@4` (tablas, listas con tareas, autolinks).
- **Custom components**:
  - `img` → `loading="lazy"`, `max-width: 100%`, `border-radius: 16px`, sombra suave.
  - `a` → si la URL es externa (http/https), abre con `target="_blank" rel="noopener noreferrer"`. Color naranja de marca, underline en hover.
  - `blockquote` → border-left naranja, fondo `var(--bg-soft)`, itálica.
  - `code` (inline + bloque) → fondo `#FAFAF9`, borde `var(--border)`, fuente monoespaciada.
  - `table` → bordes finos (`var(--border)`), header con fondo `var(--bg-soft)`.
  - `p` → si el parágrafo contiene **solo** un link a YouTube (`youtube.com/watch?v=` o `youtu.be/`) o Vimeo (`vimeo.com/<id>`), se reemplaza por un `<iframe>` responsivo 16:9 (`pt: 56.25%`, `position: absolute`) con `loading="lazy"` y `allowFullScreen`. Resto de parágrafos render normal.
- `compact` reduce tamaños de tipografía y márgenes para encajar en la vista previa del editor.
- Exporta también el helper puro `getEmbedUrl(url)` (usado en tests) que devuelve la URL del iframe a partir de un link de YouTube/Vimeo, o `null` si no aplica.

### Editor en `ArticleEdit.tsx`

- Librería: **`@uiw/react-md-editor` ^4.1.1** (un solo paquete ~150 KB con toolbar y preview integrado; trae su propia clase CSS `markdown-editor.css` que se importa en el componente). Elegida sobre `react-mde` y `react-markdown-editor-lite` por: (a) compatibilidad nativa con React 18 + TS, (b) toolbar más completa "out of the box" (tabla incluida), (c) modo `preview="live"` lateral, (d) Vite/ESM friendly.
- Modo `preview="live"`: muestra editor + preview lado a lado. La toolbar incluye bold/italic/h1-h3/listas/quote/code inline + bloque/link/imagen (URL)/tabla/fullscreen.
- `data-color-mode="light"` fuerza el tema claro coherente con el design system.
- Wrapper MUI sobreescribe los estilos por defecto: `border-radius 12px`, borde `var(--border)`, focus naranja (`primary.main`).
- Helper text bajo el editor: _"Soporta tablas, listas, código, imágenes (URL) y videos de YouTube/Vimeo"_.
- Adicionalmente se muestra una sección **"Vista previa"** debajo del editor con `<MarkdownRenderer body={form.body_md} compact />` para validar la conversión de YouTube/Vimeo a iframe — algo que el preview interno de la librería **no** hace.

### Dependencias añadidas

```
@uiw/react-md-editor ^4.1.1
```

Arrastra `@uiw/react-markdown-preview`, `rehype-raw`, `rehype-sanitize`, `refractor` y `react-textarea-code-editor` como dependencias transitivas. Nota: aunque el editor incluye `rehype-raw`/`rehype-sanitize`, el render público (`BlogArticle` → `MarkdownRenderer`) **no** los activa: solo se soporta HTML "embebido" cuando viene de patrones controlados (links de YouTube/Vimeo convertidos a iframe en el custom `p`). Esto mantiene la sanitización por defecto de `react-markdown` (`html` plano se descarta) y evita XSS.

### Tests RTL (`client/src/components/__tests__/MarkdownRenderer.test.tsx`)

9 tests nuevos:
- `# Título` → `<h1>`.
- `## Subtítulo` → `<h2>`.
- Tabla GFM (`| a | b |` + separador + fila) → `<table>` con `columnheader` y `cell`.
- Link de YouTube en su propia línea → `<iframe>` con `src` `youtube.com/embed/<id>` y `loading="lazy"`.
- Link de Vimeo → `<iframe>` con `src` `player.vimeo.com/video/<id>`.
- `![alt](url)` → `<img>` con `loading="lazy"` + `src` correcto.
- Link externo `[text](https://...)` → `target="_blank"` + `rel` con `noopener`.
- `getEmbedUrl()` reconoce los 3 patrones soportados y devuelve `null` para el resto.
- `> cita` → `<blockquote>`.

Suite total client: 25 previos + 9 nuevos = **34 verdes**. Backend: 86 verdes (sin cambios).

### Cómo verificar visualmente

```bash
# Backend
cd server && npm run dev          # API en :3001

# Frontend
cd client && npm run dev          # Vite en :5174
```

1. Abrir `/admin/login`, autenticarse.
2. Ir a `/admin/articles/new`. La barra de herramientas del editor permite insertar negrita, headings, listas, código, link, imagen (por URL) y **tabla GFM**.
3. Pegar `https://www.youtube.com/watch?v=EngW7tLk6R8` en su propia línea → la sección "Vista previa" debajo muestra el iframe 16:9.
4. Guardar y abrir `/blog/<slug>` → el artículo público se ve con la misma maquetación (tablas, imágenes con sombra, video embebido).

---

## Rediseño de cards de secciones (feature id=23)

### Secciones afectadas

| Sección | Archivo | Cambio |
|---------|---------|--------|
| Features | `client/src/components/sections/Features.tsx` | 6 iconos MUI con contenedor tintado, hover con borde de acento por color |
| Pain | `client/src/components/sections/Pain.tsx` | 4 iconos MUI, border-left naranja, hover con lift + fondo cálido |
| Solution | `client/src/components/sections/Solution.tsx` | 3 iconos MUI en caja gradiente naranja + badge numérico superpuesto |

### Iconos usados

**Features:** `PsychologyIcon` (memoria), `HubIcon` (integraciones), `SecurityIcon` (privacidad), `InsightsIcon` (analytics), `GroupsIcon` (hand-off), `SupportAgentIcon` (autoservicio).

**Pain:** `GroupRemoveIcon` (equipo abrumado), `TrendingUpIcon` (costos), `HourglassTopIcon` (espera), `VisibilityOffIcon` (sin visibilidad).

**Solution:** `CloudUploadIcon` (conectar datos), `AutoAwesomeIcon` (diseñar agente), `RocketLaunchIcon` (lanzar y medir).

### Patrones de diseño aplicados

- **Features**: `borderTop: '3px solid transparent'` → `borderTopColor: accentColor` en hover; container de icono con `background: ${accent}18`.
- **Pain**: `borderLeft: '4px solid var(--orange)'` fijo; hover cambia fondo a `var(--orange-xxl)`.
- **Solution**: icono en `Box` con `background: linear-gradient(135deg, var(--orange), var(--orange-l))` + `boxShadow`; badge numérico posicionado `absolute` con borde naranja; `::before` pseudo-elemento border-top gradiente que aparece en hover.

---

## Fondo animado Hero — Vanta.js NET (feature id=24)

### Dependencias añadidas

| Paquete | Versión | Uso |
|---------|---------|-----|
| `vanta` | `^0.5.24` | Efectos de fondo animados |
| `three` | última estable | Motor 3D requerido por Vanta |

### Arquitectura del efecto

```
<Box id="hero" position="relative">
  <Box ref={vantaRef} position="absolute" inset=0 zIndex=0 />  ← canvas Vanta NET
  <Box position="absolute" inset=0 zIndex=1 pointerEvents="none"
       background="linear-gradient(transparent 40%, #FFFFFF 100%)" />  ← overlay fade
  <Container zIndex=2>
    ... contenido hero sin modificaciones ...
  </Container>
</Box>
```

### Parámetros del efecto NET

```js
{
  color: 0xE8440A,       // --orange
  backgroundColor: 0xFFF8F5, // --orange-xxl
  points: 12,
  maxDistance: 22,
  spacing: 18,
  showDots: true,
}
```

### Tipos TypeScript

Vanta no publica `@types/`. La firma del módulo se declara en `src/vite-env.d.ts`:
```ts
declare module 'vanta/dist/vanta.net.min' {
  function NET(options: VantaNetOptions): VantaEffect;
  export default NET;
}
```

El efecto se inicializa con `import()` dinámico (evita SSR issues) y se destruye en el cleanup del `useEffect`.

---
