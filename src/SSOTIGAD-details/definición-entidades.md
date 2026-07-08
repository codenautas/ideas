# Definición de entidades — ejemplo: profesores y materias

> Bajada a tierra de [SSOTIGAD](../SSOTIGAD.md), sección 3.1 (Entidades), cumpliendo la forma de definición de la sección 2.3 (variables tipadas del lenguaje, serializables, comportamientos referenciados por nombre). Los ejemplos están en TypeScript siguiendo la conclusión provisoria de la sección 5; la decisión de lenguaje sigue abierta.

## 1. Qué muestra este ejemplo

Dos entidades: **materias** (mínima) y **profesores** (con más atributos). Entre las dos cubren:

- PK natural simple (2 casos).
- `isName` e `inComboBox` (declarados en materias, consumidos por la FK de profesores).
- FK con combo box derivado automáticamente.
- Columna calculada referenciada por nombre.
- Valor por defecto constante y valor por defecto referenciado por nombre.
- Label, descripción y grupo de columna.

Queda afuera **subgrillas** y **PK compuesta**: requieren una tercera entidad (la asignación profesor↔materia, mencionada en la sección 3.4 del documento principal). Podría ser el próximo detalle de esta carpeta.

## 2. Tipado mínimo de soporte

Solo lo necesario para que el ejemplo compile mentalmente. No es una propuesta cerrada de API.

```ts
// Placeholder: el catálogo real de tipos de dominio es tema de la sección 2.5
// (composición y reusabilidad, ambas pendientes).
type TipoDeDominio =
    | {dominio: 'codigo', largo: number}
    | {dominio: 'texto', largo: number}
    | {dominio: 'entero', min?: number, max?: number}
    | {dominio: 'fecha'}
    | {dominio: 'email'};

type CampoDef = {
    tipo: TipoDeDominio;
    label: string;
    descripcion?: string;
    grupoDeColumna?: string;
    // constante serializable, o referencia por nombre a un método registrado aparte (2.3)
    valorPorDefecto?: string | number | boolean | {metodo: string};
    isName?: true;
    inComboBox?: true;
    // columna calculada: el cálculo se referencia por nombre, nunca se embebe (2.3)
    calculado?: {metodo: string};
};

type EntidadDef = {
    nombre: string;
    pk: string[];
    campos: Record<string, CampoDef>;
    // a nivel de entidad (no de campo) para poder soportar FKs compuestas
    fks?: {campos: string[], referencia: string}[];
};
```

Nótese que `EntidadDef` es JSON plano por construcción: no hay ningún lugar donde pueda colarse una función.

## 3. La entidad materias

```ts
export const materias = {
    nombre: 'materias',
    pk: ['materia'],
    campos: {
        materia: {
            tipo: {dominio: 'codigo', largo: 10},
            label: 'Materia',
            descripcion: 'Código de la materia en el plan de estudios',
        },
        nombre: {
            tipo: {dominio: 'texto', largo: 100},
            label: 'Nombre',
            isName: true,
        },
        horasSemanales: {
            tipo: {dominio: 'entero', min: 1, max: 60},
            label: 'Horas semanales',
            inComboBox: true,
        },
    },
} as const satisfies EntidadDef;
```

- `nombre` es el `isName` del código `materia`: cualquier grilla que muestre el código agrega al lado esta columna, salvo override (3.1).
- `horasSemanales` está marcado `inComboBox`: cuando otra entidad referencie a materias por FK, el desplegable muestra código + nombre + horas semanales.

## 4. La entidad profesores

```ts
export const profesores = {
    nombre: 'profesores',
    pk: ['profesor'],
    campos: {
        profesor: {
            tipo: {dominio: 'codigo', largo: 10},
            label: 'Profesor',
            descripcion: 'Legajo docente',
        },
        apellido: {
            tipo: {dominio: 'texto', largo: 60},
            label: 'Apellido',
        },
        nombres: {
            tipo: {dominio: 'texto', largo: 60},
            label: 'Nombres',
        },
        nombreCompleto: {
            tipo: {dominio: 'texto', largo: 122},
            label: 'Nombre completo',
            isName: true,
            calculado: {metodo: 'nombreCompleto'},
        },
        email: {
            tipo: {dominio: 'email'},
            label: 'Email',
            grupoDeColumna: 'Contacto',
        },
        telefono: {
            tipo: {dominio: 'texto', largo: 30},
            label: 'Teléfono',
            grupoDeColumna: 'Contacto',
        },
        fechaIngreso: {
            tipo: {dominio: 'fecha'},
            label: 'Ingreso',
            descripcion: 'Fecha de ingreso a la institución',
            valorPorDefecto: {metodo: 'fechaDeHoy'},
        },
        activo: {
            tipo: {dominio: 'entero', min: 0, max: 1}, // pendiente: tipo de dominio booleano
            label: 'Activo',
            valorPorDefecto: 1,
        },
        materiaPrincipal: {
            tipo: {dominio: 'codigo', largo: 10},
            label: 'Materia principal',
            descripcion: 'Materia en la que se concentra su dedicación',
        },
    },
    fks: [
        {campos: ['materiaPrincipal'], referencia: 'materias'},
    ],
} as const satisfies EntidadDef;
```

- `nombreCompleto` es una columna calculada (3.1); el cálculo no está acá, solo su nombre.
- `fechaIngreso` tiene un valor por defecto que remite a un método (`fechaDeHoy`), mientras que `activo` tiene un valor por defecto constante — los dos casos previstos en 3.1.
- `email` y `telefono` comparten el grupo de columna "Contacto": en la grilla aparecen bajo un sobretítulo común.
- `materiaPrincipal` es FK a materias: se renderiza automáticamente como combo editable-y-desplegable mostrando `materia` + `nombre` (el `isName`) + `horasSemanales` (el `inComboBox`), sin que profesores tenga que declarar nada de eso.

## 5. Los comportamientos nombrados, aparte

Los métodos referenciados por nombre viven fuera de la definición serializable (2.3). Cómo se organiza este registro (centralizado vs. distribuido) está pendiente; a modo ilustrativo:

```ts
registrarCalculo('nombreCompleto',
    (fila: {apellido: string, nombres: string}) => `${fila.apellido}, ${fila.nombres}`
);

registrarValorPorDefecto('fechaDeHoy', () => fechaActual());
```

El tipado del parámetro `fila` debería derivarse de la definición de la entidad (condición 5.1.1 del documento principal), no escribirse a mano como acá. Cómo se liga el registro con ese tipo derivado es parte del diseño pendiente del registro.

## 6. Qué se deriva automáticamente

De las dos constantes anteriores, sin ninguna mención adicional a ningún campo (criterio 2.1), el framework deriva:

- **Tipos del lenguaje**: `type Materia = FilaDe<typeof materias>` produce `{materia: string, nombre: string, horasSemanales: number}` — es la razón de las tres condiciones de la sección 5.1.
- **SQL**: `CREATE TABLE`, y los `INSERT`/`SELECT`/`UPDATE` del CRUD.
- **Endpoints CRUD** para ambas entidades.
- **Grilla**: columnas con sus labels, tooltips desde las descripciones, sobretítulo "Contacto" agrupando email y teléfono, edición inline.
- **Formulario de alta/edición**: con `fechaIngreso` precargada con la fecha del día y `activo` en 1.
- **Combo de materias** en el campo `materiaPrincipal`, tipeable, con código + nombre + horas semanales.
- **Validación** en ambos extremos (navegador y servidor, por el isomorfismo de 2.4): largos, rangos, formato de email — todo sale del tipo de dominio.

Verificación del criterio SSOT (2.1): `horasSemanales` aparece **una sola vez** en todo el sistema — en la definición de materias. La columna SQL, la validación de rango, la columna de grilla y su aparición en el combo de profesores se derivan de esa única mención.

## 7. Cuestiones que este ejercicio dejó a la vista

Decisiones que tomé para poder escribir el ejemplo y que hay que confirmar o cambiar:

1. **FK a nivel de entidad, no de campo** (`fks: [{campos, referencia}]`): lo elegí para que las FKs compuestas se declaren igual que las simples. La alternativa (marca `fk` dentro del campo) es más local pero no generaliza bien a claves compuestas.
2. **Nombre técnico de campos en camelCase** (`horasSemanales`): ¿la columna SQL se deriva a `horas_semanales` automáticamente, o el nombre técnico es único y se usa idéntico en ambos mundos?
3. **El tipo de dominio como objeto con discriminante** (`{dominio: 'texto', largo: 100}`): las opciones viajan dentro del mismo objeto. Alternativa: nombre de tipo + opciones separadas.
4. **Falta un tipo de dominio booleano**: lo simulé con entero 0/1 en `activo`, que es justamente el tipo de deuda conceptual que el framework quiere evitar. El catálogo de tipos de dominio (2.5) lo tiene que resolver.
5. **Singular/plural**: la entidad se llama `profesores` (plural, como la tabla) y el tipo derivado sería `Profesor` (singular). ¿Se deriva automáticamente (problemático en español) o la definición declara ambos nombres?
6. **`isName` sobre columna calculada**: en profesores el `isName` es `nombreCompleto`, que es calculada. Asumí que es válido; implica que el combo de una eventual FK a profesores muestra un valor calculado.
