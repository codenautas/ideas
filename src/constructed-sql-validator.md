# constructed-sql-validator

Librería standalone de Node/TypeScript para validar sentencias SQL en el backend, detectando errores comunes antes de ejecutarlas contra Postgres.

## Propósito

El objetivo es chequear, sobre el SQL ya armado, un conjunto de reglas que típicamente indican un bug real (no cuestiones de estilo). La regla central es de cardinalidad: en un JOIN, el 99% de las veces se quiere acceder a una de las dos partes por su PK; si no es así, hay que declararlo explícitamente.

La librería expone una función pura `validateSql(sql, catalog)` que devuelve un resultado con los errores encontrados. La integración con el ejecutador real (qué hacer con los errores, cuándo invocarla, cache, etc.) queda del lado del usuario de la lib.

## Estado y alcance inicial

Esta primera versión cubre:

- Reglas sobre `SELECT`/`INSERT`/`UPDATE`/`DELETE` que contengan joins.
- INNER JOIN y LEFT JOIN (se tratan igual para el cálculo de PK).
- Sintaxis `ON ...` y `USING (...)`.
- Una regla léxica sobre `CREATE TRIGGER` (nombre del trigger vs nombre de la función).

Queda fuera del alcance inicial, para etapas posteriores:

- RIGHT JOIN y FULL JOIN.
- Subqueries y CTEs (sólo se mencionan: cada una es un árbol de joins independiente con su propia PK).
- Construcción dinámica de SQL (no aplica: la lib valida el SQL final ya armado).
- Linter estático (esta lib es para runtime; un linter estático podría reusar el motor más adelante).
- Otras reglas posibles: `UPDATE`/`DELETE` sin `WHERE`, comparación con `NULL` usando `=`, `SELECT *`, mismatch de tipos en `ON`, funciones sobre columnas indexadas, etc.

## Stack

- **Lenguaje**: TypeScript, con la misma rigurosidad que `guarantee-type`: todos los tipos explícitos, sin `any`, sin casts vía `unknown`, sin magia salvo en lugares puntuales claramente encapsulados.
- **Parser SQL**: [`pgsql-ast-parser`](https://www.npmjs.com/package/pgsql-ast-parser). Puro JS, sin compilación nativa, soporta `USING`, `astVisitor`/`astMapper`, `parseWithComments()`. Suficiente para el alcance inicial.
- **Distribución**: librería standalone publicable en npm. No es submódulo de `full-type` ni de `backend-plus`; esos proyectos la integran como dependencia.
- **Plataformas**: cross-platform (Windows para desarrollo, Linux para servidor). Scripts pueden ser específicos; el código no.

## La regla principal: validación de JOINs por PK

### Formulación

Para INNER JOIN entre dos sub-árboles `T1` y `T2` (cada uno puede ser una tabla simple o el resultado de joins previos), el JOIN es válido si:

1. La condición del `ON` (o `USING`) **cubre la PK completa de `T1`**, o
2. La condición del `ON` (o `USING`) **cubre la PK completa de `T2`**.

`T1` y `T2` son simétricos. No hay un lado "izquierdo" ni "derecho" privilegiado; el orden sintáctico no importa.

Si no se cumple ninguna de las dos condiciones, el JOIN **no es válido** y requiere una directiva `/* allow-... */` (ver más abajo).

LEFT JOIN se trata igual que INNER JOIN para esta regla.

### Equivalencias

- Una `UNIQUE NOT NULL` se trata como una PK para todos los efectos de esta regla.
- `USING (col1, col2)` equivale a `ON t1.col1 = t2.col1 AND t1.col2 = t2.col2`.

### Cálculo de la PK del resultado del JOIN

Cada JOIN produce un resultado parcial que tiene a su vez una PK. Esa PK es la que se usa para validar joins posteriores.

Sea `cubre(ON, T)` cierto cuando las igualdades del `ON` cubren la PK completa de `T`:

| `cubre(ON, T1)` | `cubre(ON, T2)` | PK del resultado | Validez |
|---|---|---|---|
| sí | no | `PK(T2)` | válido |
| no | sí | `PK(T1)` | válido |
| sí | sí | `PK(T1)` (equivalente a `PK(T2)`) | válido |
| no | no | (ver "Joins no válidos con allow") | requiere allow |

Cuando el JOIN usa `USING (col1, col2)`, las columnas mencionadas en el `USING` se exponen en el resultado **sin alias** (igual que en SQL estándar). Si esas columnas forman parte de la PK heredada, en la PK del resultado aparecen sin alias.

### Joins no válidos: comportamiento con allow

Cuando el JOIN no cumple la regla, se requiere una directiva `/* allow-... */`. Hay además un **calculador automático de PK** que intenta inferir la PK del resultado parcial:

- **CROSS JOIN** (con `/* allow-cross-join */`): PK del resultado = `PK(T1) ∪ PK(T2)`.
- **Otros casos** (con `/* allow-join-no-pk */`): el calculador devuelve "no puedo calcularlo". El usuario debe declarar la PK explícitamente mediante `/* new-pk ... */` en la misma cláusula del JOIN.

Si hay un allow que no permite calcular la PK automáticamente y no hay un `new-pk` que la declare, eso constituye un **error de la misma jerarquía** que la falta del allow (no es válido seguir validando joins posteriores con una PK indefinida).

### Ejemplo: cadena de joins válida sin allow

```sql
SELECT *
FROM departamento d
JOIN empleados e ON e.depto = d.depto
JOIN pedidos   p ON e.empleado = p.empleado
JOIN cliente   c ON c.cliente = p.cliente
```

PKs: `departamento(depto)`, `empleados(empleado)`, `pedidos(pedido)`, `cliente(cliente)`.

- `departamento d` → PK = `{d.depto}`.
- `JOIN empleados e ON e.depto = d.depto`: cubre PK de `d`. PK del resultado = PK de `e` = `{e.empleado}`.
- `JOIN pedidos p ON e.empleado = p.empleado`: cubre PK del resultado parcial (`e.empleado`). PK del resultado = PK de `p` = `{p.pedido}`.
- `JOIN cliente c ON c.cliente = p.cliente`: cubre PK de `c`. PK del resultado = PK del lado de `p` = `{p.pedido}`.

PK final: `{p.pedido}`. Ningún allow necesario.

### Ejemplo: caso con allow y new-pk implícita por cross join

```sql
SELECT e.empleado, t.trimestre, sortear_sucursal()
FROM empleado e
CROSS JOIN trimestre t   /* allow-cross-join "generar combinaciones faltantes" */
LEFT JOIN asignaciones a ON e.empleado = a.empleado AND t.trimestre = a.trimestre
WHERE a.trimestre IS NULL
```

- `empleado e` → PK = `{e.empleado}`.
- `CROSS JOIN trimestre t` con allow: PK = `{e.empleado, t.trimestre}` (unión automática).
- `LEFT JOIN asignaciones a ON ...`: el `ON` cubre la PK completa de `a` (`{trimestre, empleado}`). Válido. PK final se mantiene en `{e.empleado, t.trimestre}`.

### Ejemplo: allow con new-pk explícita

Cuando el calculador no puede inferir la PK (allow distinto de cross join), el usuario debe declararla:

```sql
SELECT ...
FROM a
JOIN b ON a.fecha BETWEEN b.desde AND b.hasta   /* allow-join-no-pk "rango temporal" new-pk a.id, b.id */
```

## Otras reglas

### `trigger-name-mismatch`

En `CREATE TRIGGER`, el nombre del trigger debe coincidir con el nombre de la función que invoca. Es una regla léxica simple sobre el AST de `CREATE TRIGGER`.

- `CREATE TRIGGER foo BEFORE INSERT ON t EXECUTE FUNCTION foo()` → válido.
- `CREATE TRIGGER foo BEFORE INSERT ON t EXECUTE FUNCTION bar()` → requiere `/* allow-trigger-name-mismatch "razón" */`.

## Directivas en comentarios

Todas las directivas viven en comentarios SQL `/* ... */`. La librería las extrae usando `parseWithComments()` del parser y las asocia al nodo del AST correspondiente (el JOIN, el CREATE TRIGGER, etc.).

| Directiva | Significado |
|---|---|
| `/* allow-join-no-pk "razón" */` | Autoriza un JOIN que no cubre la PK de ninguno de los dos lados. |
| `/* allow-cross-join "razón" */` | Autoriza un `CROSS JOIN`. El calculador infiere la PK como unión. |
| `/* allow-trigger-name-mismatch "razón" */` | Autoriza un `CREATE TRIGGER` con nombre distinto al de la función. |
| `/* new-pk col1, col2, ... */` | Declara explícitamente la PK del resultado del JOIN. Obligatorio cuando hay `allow-join-no-pk` (porque el calculador no puede inferirla). Puede combinarse con cualquier allow para sobreescribir la PK calculada. |

Notas:

- La razón puede estar vacía por ahora (no se valida que tenga contenido en esta versión).
- `new-pk` no es un allow; es una declaración de PK del resultado que puede acompañar a un allow o usarse para sobreescribir la PK calculada.
- Convención de nombres: `kebab-case`. Cada regla tiene un identificador (ej: `join-no-pk`) y el allow correspondiente lo prefija con `allow-` (ej: `allow-join-no-pk`). **Confirmar**: simetría exacta regla/allow.

## Interfaz pública

Una sola función:

```ts
function validateSql(sql: string, catalog: Catalog): ValidationResult;
```

### Tipos

```ts
type Catalog = {
    [schema: string]: {
        [table: string]: {
            pk: string[];
            uniques: string[][];
        };
    };
};

type ValidationResult =
    | { valid: true; resultPk: string[] }
    | { valid: false; errors: ValidationError[] };

type ValidationError = {
    rule: string;
    message: string;
};
```

### Decisiones de diseño

- **Función pura**, sin efectos. Recibe SQL y catálogo, devuelve resultado. Fácil de testear.
- **No tira excepciones**. El usuario decide qué hacer con los errores.
- **Lista de errores, no se detiene en el primero**. Una query con varios joins problemáticos reporta todos.
- **Sin `location`**. Recalcularla recursivamente complica las funciones internas. Si más adelante se necesita, se agrega.
- **Sin `context`** (tablas involucradas, PK parcial al momento del error, etc.). Posible *nice to have* futuro, no es necesario ahora.
- **`resultPk`** se devuelve sólo cuando la validación es exitosa. Es la PK del resultado completo de la query y puede ser útil para quien encadene validaciones o quiera saber qué identifica cada fila del resultado.

## Catálogo de metadatos

El catálogo se le pasa a `validateSql` como parámetro. Es responsabilidad del usuario armarlo (puede ser estático en código, o generado a partir de introspección de PG, o lo que sea). Estructura multinivel: `schema → tabla → { pk, uniques }`.

```ts
const catalog: Catalog = {
    public: {
        empleados: {
            pk: ['empleado'],
            uniques: [['legajo']]
        },
        asignaciones: {
            pk: ['trimestre', 'empleado'],
            uniques: []
        },
        // ...
    }
};
```

PKs simples y compuestas se manejan igual (siempre son `string[]`). Una `UNIQUE NOT NULL` se trata como PK alternativa a efectos de la regla.

## Estilo de código

- TypeScript con todos los tipos explícitos.
- Prohibido `any`. Prohibido `as unknown as ...`.
- Sin magia salvo en puntos muy concretos, claramente encapsulados (modelo `guarantee-type`).
- Uso de `var` (consistente con el resto del ecosistema `codenautas`).
- Cross-platform donde sea posible; cualquier cosa específica de un OS va a scripts separados.
- Versionado: arranca en `3.x` si se incorpora al ecosistema `codenautas` con la migración ESM/TS 6 (a confirmar al momento del primer release).

## Decisiones pendientes

- Convención exacta de nombres regla/allow (simetría estricta `join-no-pk` / `allow-join-no-pk`, o asimetría tolerada).
- Manejo de RIGHT JOIN y FULL JOIN.
- Tratamiento de subqueries y CTEs en el árbol de joins.
- Diseño de cache de AST (necesario sólo si el overhead aparece como problema; mientras tanto, sin cache).
- Si la lib se integra al ecosistema `codenautas` o se publica como paquete independiente sin ese prefijo.
