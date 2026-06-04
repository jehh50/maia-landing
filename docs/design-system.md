# Design System — MaIA Landing

> Extraído desde `client/src/theme/theme.ts` + `client/src/styles/globals.css` + `legacy.html` (2026-05-26).
> Fuente única de verdad para colores, tipografía y componentes en la landing y los correos transaccionales.

---

## 1. Marca

| Token | Hex | Uso |
|-------|-----|-----|
| `brand.orange`       | `#E8440A` | Color primario de marca. CTAs, focus rings, énfasis. |
| `brand.orangeHover`  | `#D03A08` | Hover de botones primarios. |
| `brand.orangeLight`  | `#FF6B35` | Variante clara (gradient text, acentos). |
| `brand.orangeXL`     | `#FFF0EB` | Tinta muy clara (chip backgrounds, hover de outlined). |
| `brand.orangeXXL`    | `#FFF8F5` | Tinta súper clara (fondo de sección hero). |

**Gradient corporativo:** `linear-gradient(135deg, #E8440A 0%, #FF6B35 100%)` — usado en `gradient-text` y CTAs grandes.

**Logotipo:** `ma**IA**` con la sílaba IA en `brand.orange`. SVGs en `/client/public/`:
- `logo-maia.svg` — logo completo (navbar, footer, mail header).
- `isotipo-maia.svg` — isotipo solo (favicon, mail footer compacto).

---

## 2. Neutros y superficies

| Token | Hex | Uso |
|-------|-----|-----|
| `surface.main`       | `#FFFFFF` | Fondo de tarjetas, modales, página. |
| `surface.soft`       | `#FAFAF9` | Fondo de secciones alternadas. |
| `surface.tint`       | `#FFF5F1` | Fondo cálido para hover/áreas destacadas. |
| `border`             | `#F0EBE8` | Bordes sutiles, dividers. |
| `border2`            | `#E5DDD9` | Bordes más visibles (inputs outlined, pills). |
| `text.primary`       | `#1A1410` | Texto principal, headlines. |
| `text.secondary`     | `#4A3F3A` | Texto secundario. |
| `muted`              | `#7A6E6A` | Descripciones, captions. |
| `muted2` (`disabled`)| `#A89E9A` | Texto deshabilitado, metadata. |

---

## 3. Estados

| Token | Hex | Uso |
|-------|-----|-----|
| `success.main`  | `#16A34A` | Estados positivos, ahorro, "Activo". |
| `success.light` | `#DCFCE7` | Fondo de alerts/chips de éxito. |
| `success.dark`  | `#14532D` | Texto sobre `success.light`. |
| `error.main`    | `#DC2626` | Validación de formularios, fallos. |
| `error.light`   | `#FEE2E2` | Fondo de alerts de error. |

---

## 4. Tipografía

**Familia principal:** `"Inter", system-ui, -apple-system, sans-serif`
Carga vía Google Fonts (pesos 300–800 + itálica 400). En correos transaccionales, **siempre** declarar el fallback `Arial, sans-serif` porque muchos clientes (Outlook desktop, algunos webmails) no descargan web fonts.

### Escala tipográfica

| Estilo | Tamaño (clamp) | Peso | Letter-spacing | Line-height |
|--------|----------------|------|----------------|-------------|
| `h1`     | `clamp(2.2rem, 5.5vw, 3.8rem)` | 700 | `-0.025em` | 1.2 |
| `h2`     | `clamp(1.7rem, 3.5vw, 2.6rem)` | 700 | `-0.025em` | 1.2 |
| `h3`     | `1.15rem` (18.4px)              | 600 | —          | 1.3 |
| `h4`     | `1rem`     (16px)               | 600 | —          | — |
| `body`   | `1rem`     (16px)               | 400 | —          | 1.6 |
| `body sm`| `0.875rem` (14px)               | 400 | —          | 1.6 |
| `caption`| `0.75rem`  (12px)               | 500 | `0.02em`   | 1.4 |
| `overline`| `0.75rem` (12px)               | 600 | `0.08em` UPPER | 1.4 |
| `button` | `0.9375rem`(15px) / `1rem` lg   | 600 | —          | — |

### Estilos especiales

- **Gradient text:** `bg-clip: text` con el gradient corporativo, sobre headlines del hero.
- **`label` (chip pill):** 12px, peso 600, uppercase, letter-spacing `0.08em`, padding `4px 14px`, border-radius `100px`, fondo `brand.orangeXL` con borde `rgba(232,68,10,0.2)` y texto `brand.orange`.

---

## 5. Espaciado y layout

| Token | Valor | Uso |
|-------|-------|-----|
| Container max-width | `1140px` | Ancho máximo de contenido en todas las secciones. |
| Container padding   | `24px` (horizontal) | Padding lateral en mobile/desktop. |
| Sección padding-y   | `clamp(56px, 8vw, 80px)` | Padding vertical de cada `<section>`. |
| Stack gap (xs)      | `8px / 12px / 16px` | Gaps cortos entre elementos relacionados. |
| Stack gap (md)      | `24px / 32px / 40px` | Separación entre bloques. |

### Breakpoints

| Nombre | Min-width | Notas |
|--------|-----------|-------|
| `xs` | `0` |
| `sm` | `640px` | Tarjetas pasan a 2 col. Pills layout. |
| `md` | `860px` | Navbar desktop visible; menú móvil oculto. |
| `lg` | `1200px` | Pricing pasa a 4 col. |

---

## 6. Radios

| Token | Valor | Uso |
|-------|-------|-----|
| `shape.borderRadius` | `12px` | Default global (`var(--radius)`). |
| `radius-lg`          | `20px` | Modales, hero demo window. |
| `radius-pill`        | `100px` | Botones, chips, badges. |

---

## 7. Sombras (elevation)

| Token | Valor | Uso |
|-------|-------|-----|
| `shadow.sm` | `0 1px 3px rgba(0,0,0,0.06), 0 1px 2px rgba(0,0,0,0.04)` | Cards en reposo. |
| `shadow.md` | `0 4px 16px rgba(0,0,0,0.08), 0 2px 4px rgba(0,0,0,0.04)` | Cards en hover, navbar scrolled. |
| `shadow.lg` | `0 12px 40px rgba(0,0,0,0.10), 0 4px 8px rgba(0,0,0,0.04)` | Modales, demo window. |
| `shadow.brand` | `0 4px 16px rgba(232,68,10,0.25)` (hover `0 6px 20px rgba(232,68,10,0.35)`) | Botones primarios. |

> ⚠️ En clientes de correo (Outlook especialmente) las `box-shadow` se ignoran en su mayoría. Para emails usar bordes finos en lugar de elevación.

---

## 8. Componentes clave

### Button (`MuiButton`)

| Variant | Background | Color | Border | Padding | Border-radius | Font |
|---------|------------|-------|--------|---------|---------------|------|
| `contained primary`  | `#E8440A` (hover `#D03A08`) | `#FFFFFF` | — | `11px 22px` | `100px` | 15/600 |
| `contained primary lg` | `#E8440A` | `#FFFFFF` | — | `14px 28px` | `100px` | 16/600 |
| `outlined`           | `transparent` (hover `#FFF8F5`) | `#1A1410` | `1px solid #E5DDD9` (hover `#E8440A`) | `11px 22px` | `100px` | 15/600 |

### Chip / Label / Pill

- **Brand chip:** ver "label" en Tipografía especial (sección 4).
- **Status chip success:** fondo `#DCFCE7`, texto `#16A34A`, padding `2px 10px`, radius `100px`, font 12/600.

### Input (`MuiTextField`)

- **Default size:** `small`. Padding `11px 14px`.
- **Border:** `1px solid #E5DDD9` (focus `#E8440A` + box-shadow `0 0 0 3px rgba(232,68,10,0.12)`).
- **Invalid:** `border-color: #DC2626 + box-shadow: 0 0 0 3px rgba(220,38,38,0.10)`.
- **Border-radius:** `12px`.

### Card

- Fondo: `#FFFFFF`.
- Borde: `1px solid #F0EBE8`.
- Padding: `28px–32px` (responsive).
- Border-radius: `12px`.
- Hover (cuando aplica): `transform: translateY(-4px)` + `shadow.md`.

---

## 9. Animaciones

| Nombre | Duración | Easing | Uso |
|--------|----------|--------|-----|
| `fadeUp`  | 0.6s | `ease` | Entrada inicial del hero (delays `.1s / .2s / .3s / .4s`). |
| `blink`   | 1.5s loop | `ease-in-out` | Dot verde del badge "Nuevo". |
| `reveal`  | 0.55s | `ease` | Scroll-triggered via `IntersectionObserver`. |
| Hover btn | 0.2s | `ease` | Transform + shadow. |
| Modal open | 0.25s | `ease` | Dialog MUI. |

---

## 10. Tokens para correos transaccionales

Los correos HTML deben usar un **subset robusto** del design system. Reglas:

- **Width:** body wrapper `max-width: 600px` centrado.
- **Familia:** `font-family: 'Inter', Arial, sans-serif;` siempre con fallback.
- **No CSS vars, no flex/grid:** usar `<table>` con `cellpadding/cellspacing/border="0"` y `width="600"` para layout. Inlinea todos los estilos críticos (`style="..."`).
- **No `box-shadow`** (mal soporte). Usar bordes finos.
- **No SVGs externos en `<img>`:** Outlook desktop no los renderiza. Usar PNG inline con CID o sustituir por texto con estilo si pesa demasiado.
- **Colores hex completos** (no `#fff`, sí `#FFFFFF`) para Outlook.
- **Dark mode hints:** declarar `meta name="color-scheme"` y `meta name="supported-color-schemes"` con `light`.

### Subset recomendado de tokens

| Uso en correo | Token |
|---------------|-------|
| Header background | `#E8440A` (sólido) o gradient (puede degradar a sólido en Outlook). |
| Header texto      | `#FFFFFF`. |
| Body background   | `#FAFAF9`. |
| Card background   | `#FFFFFF` + `1px solid #F0EBE8`. |
| Texto primario    | `#1A1410`. |
| Texto secundario  | `#4A3F3A`. |
| Etiqueta de campo | `#7A6E6A`, 12px, uppercase, `letter-spacing: 0.06em`. |
| Botón CTA         | bg `#E8440A`, texto `#FFFFFF`, padding `14px 28px`, `border-radius: 100px`. |
| Footer background | `#FAFAF9`. |
| Footer texto      | `#A89E9A`, 12px. |
| Link              | `#E8440A`, sin underline (excepto hover, que en email no aplica). |

### Plantilla mínima de wrapper

```html
<!doctype html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="color-scheme" content="light">
  <meta name="supported-color-schemes" content="light">
  <title>MaIA</title>
</head>
<body style="margin:0;padding:0;background:#FAFAF9;font-family:'Inter',Arial,sans-serif;color:#1A1410;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0">
    <tr><td align="center" style="padding:24px;">
      <table role="presentation" width="600" cellpadding="0" cellspacing="0" border="0" style="background:#FFFFFF;border:1px solid #F0EBE8;border-radius:12px;overflow:hidden;">
        <!-- header / body / footer -->
      </table>
    </td></tr>
  </table>
</body>
</html>
```

---

## 11. Cómo extender el design system

- **Frontend (React):** los tokens nuevos van en `client/src/theme/theme.ts`. Si son colores aumenta el módulo `Palette`/`PaletteOptions` (ver bloque `declare module`). Si son utilidades visuales (animaciones, gradient text), van en `client/src/styles/globals.css` como CSS vars / classes.
- **Backend (correos):** los tokens hex van directo en los templates HTML inlined dentro de `server/src/email.js` (sin abstracciones, porque cada cliente de email rasguña por separado).
- **Docs:** cuando agregues un token nuevo, **edita este archivo** y referencia desde el commit / PR. Mantén la tabla ordenada por categoría.

---

## 12. Historial de versiones

| Fecha | Cambio |
|-------|--------|
| 2026-05-26 | Versión inicial extraída del theme MUI + globals.css del cliente Vite (feature id=6). |
