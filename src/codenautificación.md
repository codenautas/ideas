# Codenautificación

Guía genérica para **alinear una herramienta/paquete viejo o desalineado con la forma de codenautas**.
Pensada como punto de partida para que un agente de IA (o una persona) la siga paso a paso.

No es un script automático: en cada fase hay **decisiones que requieren confirmación del responsable**.
La regla de oro es la de codenautas: *primero acordar qué hacer, después tocar código*.

---

## Principios de trabajo (aplican a todas las fases)

- **Acordar antes de programar.** Diagnosticar y proponer primero; implementar recién con luz verde.
- **Por fases y revisable.** Diffs chicos y humanamente revisables. No reformatear, no reordenar, no tocar
  indentación ni comillas, no aplicar linters por gusto. El *trailing whitespace* no importa (se normaliza con `appease`).
- **No fallar en silencio.** Nada de `catch` vacíos, promesas sin manejo de error, callbacks que ignoran el `err`.
- **No commitear ni tocar el índice de git.** El responsable revisa y commitea. Sólo se usa git para mover/renombrar.
- **Verificar de verdad.** Correr los tests y la herramienta; reportar resultados reales, no asumidos.
- **Contradicciones = parar y avisar.** Si la herramienta hace algo distinto a lo que su guía/doc dice
  (warnings raros, salida inesperada, comportamiento que obliga a un *workaround*), **no lo resuelvas en silencio**:
  puede ser un bug de la herramienta que conviene corregir en su origen. Señal de alarma: si te encontrás
  leyendo el código de la herramienta para esquivar lo que documenta, frená.

---

## Fase 0 — ¿Reemplazar o modernizar?

Antes de invertir en modernizar, ver si la herramienta ya no hace falta.

1. Entender **qué hace** exactamente (leer el código y los tests, no asumir).
2. Buscar (con datos, no de memoria) si existe un **reemplazo mantenido** que cubra lo mismo:
   - paquete oficial/upstream vivo, forks comunitarios, o funcionalidad ya cubierta por otra dependencia.
   - comparar feature por feature contra lo que el fork local agrega.
3. Si hay reemplazo directo, mantenido y equivalente → **proponer el cambio** y frenar acá.
4. Si **no** lo hay (caso típico de un fork con agregados propios) → seguir con la modernización.
5. Si la herramienta era un fork revisar si está atrasado respecto del **upstream** (commits/fixes que valga la pena traer).
   Confirmar fechas reales del upstream; muchos están congelados hace años.

---

## Fase 1 — Diagnóstico del código

Revisar sin cambiar todavía. Reportar y acordar antes de tocar.

- **Smells / correctitud:** precedencia de operadores dudosa, condiciones que nunca se cumplen,
  valores que se guardan sin validar (`undefined`, vacío), *race conditions* (p. ej. responder antes
  de persistir estado asíncrono).
- **No fallar en silencio:** detectar manejo de error ausente y proponer warn/propagación.
- **Tipado:** si es TypeScript, `strict` y sin `any`. Si los fuentes están en JS quedan en JS (a veces solo los tests están en TS, a veces todo menos los tests).
- **Seguridad / dependencias:** `npm audit`. Distinguir vulnerabilidades de **runtime** (en `dependencies`)
  de las de **devDeps** (suelen irse al cambiar el framework de test).
- **Tests existentes:** ¿corren realmente? ¿cubren todo? Es común encontrar tests que el `npm test`
  ni siquiera ejecuta (rutas mal, `NODE_PATH`, frameworks muertos).

---

## Fase 2 — Tooling

Alinear el andamiaje con los repos de referencia (p. ej. `best-globals`, `cast-error`, `login-plus`).

- **Framework de test → mocha + nyc.** Reescribir los tests (con `assert` nativo o `expect.js`, si se usa `discrepances` dejarlo).
  Apuntar a **buena cobertura** (objetivo 100% cuando sea razonable).
- **Scripts** (sin hardcodear rutas a `node_modules/.bin`, que rompen multiplataforma):
  ```json
  "test":    "mocha --reporter spec --bail --check-leaks test/",
  "test-ci": "nyc --reporter=lcov --reporter=text node_modules/mocha/bin/_mocha --reporter spec --check-leaks test/"
  ```
  (el `test-ci` corre en CI Linux; el `test` pelado es el portable).
- **Arreglar los `require`** de los tests para que `npm test` corra **todos** los archivos.
- **`overrides`** para vulnerabilidades transitivas (espejar las de los hermanos y verificar que siguen siendo necesarios):
  ```json
  "overrides": { "diff": ">=8.0.2", "serialize-javascript": ">=7.0.0", "uuid": ">=14.0.0" }
  ```
- **Eliminar infra vieja:** `.travis.yml`, `Makefile` y el framework de test viejo. Appveyor puede quedar porque es la forma elegida de testear en windows.
- **Regenerar `package-lock.json`** (`npm install`) y confirmar `0 vulnerabilities`.
- **No escribir los GitHub Actions a mano:** los controla qa-control (Fase 3).

---

## Fase 3 — qa-control

Es un proceso **iterativo**: correr, leer, corregir, repetir.

```sh
npx qa-control .            # diagnostica (no escribe nada)
npx qa-control . --verbose  # muestra qué regla dispara cada warning
npx qa-control . --code     # muestra el código del warning (para silenciar)
npx qa-control . --fix      # genera/corrige lo automatizable (badges, workflows GHA)
```

Pasos típicos:

1. **Agregar la sección `qa-control`** a `package.json` y bumpear el devDep a la última versión:
   ```json
   "qa-control": {
     "profile": "minimum",      // minimum | (otros perfiles activan eslint, etc.)
     "run-in": "server",        // server | both | client  (según dónde corre)
     "type": "lib",             // lib | app | cmd-tool | web
     "gha": "all"               // all | skip
   }
   ```
2. **`repository` en forma corta** `owner/repo` (no `git://…`, no objeto largo).
3. **Sección `files`** (whitelist de publicación, p. ej. `["lib"]`) en vez de depender de `.npmignore`.
4. **`.gitignore`**: además de `node_modules`, agregar lo obligatorio y los artefactos:
   `coverage`, `*.lcov`, `.nyc_output`, `local-*`, `*-local.*`, `cucardas.log`.
5. **Discrepancias legítimas → `silenced`.** Si un warning refleja algo que **a propósito** no se va a cambiar
   (ej.: el nombre del repo difiere del nombre del paquete porque el paquete ya está publicado), se silencia con
   su código:
   ```json
   "qa-control": { ..., "silenced": ["invalid_repository_section_in_package_json"] }
   ```
   Obtener el código con `--code`. **Validar** que el silenciado realmente destrabe la corrida
   (si la herramienta lo silencia pero igual frena, es un bug de la herramienta: parar y avisar — ver Principios).
6. **`--fix` genera lo automatizable:** badges (cucardas) en el doc principal y los workflows GHA
   (apuntan al workflow reutilizable `codenautas/.github/...@v1`). **No** escribir esos YAML a mano.
7. Re-correr hasta **"Done without warnings!"**.

---

## Fase 4 — Documentación multilang

codenautas usa documentación bilingüe con la herramienta `multilang` (siempre en el `PATH`).

- **`LEEME.md` es la fuente** (castellano + inglés en un solo archivo). **`README.md` se genera**:
  ```sh
  multilang LEEME.md
  ```
  No editar `README.md` a mano.
- **Estructura de `LEEME.md`** (modelar contra un hermano, p. ej. `best-globals/LEEME.md`):
  ```
  <!--multilang v0 es:LEEME.md en:README.md -->
  # nombre-paquete
  <!--lang:es-->
  descripción en español
  <!--lang:en--]
  description in english
  [!--lang:*-->

  <!-- cucardas -->        <!-- marcador: --fix mete acá los badges -->

  <!--multilang buttons-->
  ... pie de selección de idioma ...
  ```
  - Bloque bilingüe: `<!--lang:es-->` … `<!--lang:en--]` … `[!--lang:*-->`.
  - Contenido común a ambos idiomas: `<!--lang:*-->`.
- Aprovechar para **limpiar typos** y dejar el contenido prolijo (es contenido, no formato).
- Los **badges** salen del `npx qa-control . --fix` (Fase 3), no se escriben a mano.

---

## Convenciones de `package.json` (referencia rápida)

- `license: "MIT"` (no el array deprecado `licenses`).
- `contributors: [...]` (no campos inventados como `first-author`).
- `repository: "owner/repo"`.
- `main`, y `types` si se publica `.d.ts`.
- `files: [...]` como whitelist de publicación.
- `engines` acorde (p. ej. `"node": ">= 18"`).
- `scripts`: `test`, `test-ci` (y opcional `test-cov`).
- `overrides`, `devDependencies` (mocha, nyc, qa-control), sección `qa-control`.

---

## Checklist final

- [ ] Fase 0 decidida (reemplazo vs modernizar) y acordada.
- [ ] Diagnóstico reportado y acordado.
- [ ] Código corregido sin fallar en silencio; tipado si aplica.
- [ ] Tests en mocha+nyc, todos corriendo, cobertura alta/100%.
- [ ] Infra vieja eliminada (travis/makefile/framework viejo).
- [ ] `npm audit` → 0 vulnerabilidades; `package-lock.json` regenerado.
- [ ] `qa-control` → "Done without warnings!".
- [ ] `LEEME.md` fuente + `README.md` generado con `multilang`.
- [ ] `.gitignore` completo.
- [ ] **No** se commiteó nada: queda en el working tree para revisión.
