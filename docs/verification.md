# Verificación — Cómo demostrar que el trabajo funciona

> Regla de oro: **el agente no dice "funciona", lo demuestra**.
> Toda feature termina con evidencia ejecutable, no con afirmaciones.

## Niveles de verificación

### Nivel 1 — Tests unitarios (obligatorio)

1. Cubre el camino feliz.
2. Cubre al menos un camino de error si la función puede fallar.


### Nivel 2 — Test de integración del API REST (obligatorio para features de backend)

Las features que añaden endpoints se verifican levantando el servidor real
contra una base de datos/archivo temporal y haciendo peticiones HTTP:

```js
// tests/integration/notes.test.js (backend)
import request from "supertest";
import { createApp } from "../../src/app.js";
import { mkdtempSync, rmSync } from "fs";
import { tmpdir } from "os";
import { join } from "path";

let app, tmpDir;

beforeAll(() => {
  tmpDir = mkdtempSync(join(tmpdir(), "notes-test-"));
  process.env.DATA_DIR = tmpDir;
  app = createApp();
});

afterAll(() => rmSync(tmpDir, { recursive: true }));

it("POST /notes crea una nota y devuelve id", async () => {
  const res = await request(app)
    .post("/notes")
    .send({ title: "hola", body: "mundo" });
  expect(res.status).toBe(201);
  expect(res.body).toHaveProperty("id");
});
```

**Frontend — test de componente con React Testing Library:**
```jsx
// client/src/components/__tests__/NoteForm.test.jsx
import { render, screen, fireEvent } from "@testing-library/react";
import NoteForm from "../NoteForm";

it("llama onSubmit con title y body al enviar", async () => {
  const onSubmit = vi.fn();
  render(<NoteForm onSubmit={onSubmit} />);

  fireEvent.change(screen.getByLabelText(/título/i), {
    target: { value: "hola" },
  });
  fireEvent.click(screen.getByRole("button", { name: /guardar/i }));

  expect(onSubmit).toHaveBeenCalledWith(
    expect.objectContaining({ title: "hola" })
  );
});
```

---

### Nivel 3 — Smoke test manual (opcional pero recomendado)

Ejecuta el navegador y evalua los features desarrollados y si funcionan correctamente.

---

## Anti-patrones (no hacer)

- ❌ "He añadido el endpoint, debería funcionar." → falta test ejecutable.
- ❌ Test que solo verifica que la función no lanza excepción → debe
  comprobar el resultado concreto (`status`, `body`, texto en pantalla).
- ❌ `mock` del filesystem o de la base de datos → usa directorios temporales
  reales (`fs.mkdtempSync`) o una DB en memoria (SQLite `:memory:`, etc.).
- ❌ `mock` de `fetch`/`axios` en tests de integración → usa `supertest`
  contra el servidor real.
- ❌ Marcar la feature como `done` sin pasar `node init.js`.

---
