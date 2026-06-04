# SESION ACTUAL

Feature en curso: 18 — Agregar formato enriquecido a la creación de post para el blog

Plan:
- Crear `client/src/components/MarkdownRenderer.tsx` reusable con custom components (img lazy, table, blockquote, code, YouTube/Vimeo → iframe responsivo) usando `react-markdown` + `remark-gfm`.
- Reemplazar el render markdown de `BlogArticle.tsx` por `<MarkdownRenderer body={...} />`.
- Instalar `@uiw/react-md-editor` y usarlo en `ArticleEdit.tsx` (toolbar con bold/italic/headers/listas/quote/code/link/imagen/tabla, preview en vivo via la lib + opción de previa con MarkdownRenderer).
- Añadir tests RTL en `client/src/components/__tests__/MarkdownRenderer.test.tsx` (h1, h2, tabla GFM, YouTube iframe, img lazy).
- Actualizar `docs/architecture.md` (sección feature 18) y `memory.md`. Estado feature 18 → `in_progress`.
