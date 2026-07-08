# Definición de entidades — ejemplo: profesores y materias

> Bajada a tierra de [SSOTIGAD](../SSOTIGAD.md), sección 3.1 (Entidades), cumpliendo la forma de definición de la sección 2.3 (variables tipadas del lenguaje, serializables, comportamientos referenciados por nombre). Los ejemplos están en TypeScript siguiendo la conclusión provisoria de la sección 5; la decisión de lenguaje sigue abierta.

## 1. Qué muestra este ejemplo

Tres entidades: **materias** y **profesores**, y **asignacionProfesores** que representa la relación muchos a muchos entre las dos. Entre las tres cubren:

- PK natural simple (materias, profesores) y PK natural compuesta (asignacionProfesores).
- Tipos de dominio referenciados por nombre contra un catálogo de la aplicación.
- `isName` e `inComboBox` (declarados en materias y profesores, consumidos por las FKs de asignacionProfesores).
- FKs con combo box derivado automáticamente.
- Columna calculada referenciada por nombre.
- Valor por defecto constante y valor por defecto referenciado por nombre.
- Campo nulleable.
- Label, descripción y grupo de columna.

Queda afuera **subgrillas**: con estas tres entidades ya sería posible (las asignaciones como subgrilla dentro de profesores), y también la grilla-matriz de la sección 3.4; pueden ser el próximo detalle de esta carpeta.

## 2. El catálogo de tipos de dominio

El dato más importante de un campo es su tipo de dominio, y es un **string**. Los parámetros del tipo (min, max, decimales, validación) no viajan en cada campo: se definen una sola vez, a nivel aplicación, en un catálogo de tipos de dominio. Es la idea de la sección 2.5: "Edad" es un tipo que internamente sabe su rango; acá "horasSemanales" sabe el suyo.

```ts
// Atributos con los que se define un tipo de dominio.
// Provisorio: asume composición a partir de bases primitivas (pendiente 2.5).
type DefinicionDeTipoDeDominio = {
    base: 'texto' | 'entero' | 'fecha' | 'booleano';
    min?: number;
    max?: number;
    // validación custom referenciada por nombre (2.3)
    validacion?: string;
};

// El catálogo de la aplicación. También es JSON plano.
const tiposDeDominio = {
    codigo:         {base: 'texto'},
    texto:          {base: 'texto'},   // sin largo: en 2026 ya no tiene sentido
    email:          {base: 'texto', validacion: 'email'},
    fecha:          {base: 'fecha'},
    booleano:       {base: 'booleano'},
    entero:         {base: 'entero'},
    horasSemanales: {base: 'entero', min: 1, max: 60},
} as const satisfies Record<string, DefinicionDeTipoDeDominio>;
```

Las definiciones de entidad se tipan paramétricamente contra ese catálogo: `CampoDef` recibe el catálogo como parámetro de tipo y el campo `tipo` es el `keyof` de ese parámetro. Así `tipo` sigue siendo un string plano en la definición serializable, pero un error de tipeo (`'texo'`) no compila, y el editor autocompleta los nombres del catálogo.

```ts
type CampoDef<Tipos extends Record<string, DefinicionDeTipoDeDominio>> = {
    tipo: keyof Tipos & string;
    label: string;
    descripcion?: string;
    grupoDeColumna?: string;
    // constante serializable, o referencia por nombre a un método registrado aparte (2.3)
    valorPorDefecto?: string | number | boolean | {metodo: string};
    permiteNull?: true;
    isName?: true;
    inComboBox?: true;
    // columna calculada: el cálculo se referencia por nombre, nunca se embebe (2.3)
    calculado?: {metodo: string};
};

type EntidadDef<Tipos extends Record<string, DefinicionDeTipoDeDominio>> = {
    nombre: string;
    pk: string[];
    campos: Record<string, CampoDef<Tipos>>;
    // a nivel de entidad (no de campo) para poder soportar FKs compuestas
    fks?: {campos: string[], referencia: string}[];
};
```

Nótese que tanto el catálogo como las entidades son JSON plano por construcción: no hay ningún lugar donde pueda colarse una función.

## 3. La entidad materias

```ts
export const materias = {
    nombre: 'materias',
    pk: ['materia'],
    campos: {
        materia: {
            tipo: 'codigo',
            label: 'Materia',
            descripcion: 'Código de la materia en el plan de estudios',
        },
        nombre: {
            tipo: 'texto',
            label: 'Nombre',
            isName: true,
        },
        horasSemanales: {
            tipo: 'horasSemanales',
            label: 'Horas semanales',
            inComboBox: true,
        },
    },
} as const satisfies EntidadDef<typeof tiposDeDominio>;
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
            tipo: 'codigo',
            label: 'Profesor',
            descripcion: 'Legajo docente',
        },
        apellido: {
            tipo: 'texto',
            label: 'Apellido',
        },
        nombres: {
            tipo: 'texto',
            label: 'Nombres',
        },
        nombreCompleto: {
            tipo: 'texto',
            label: 'Nombre completo',
            isName: true,
            calculado: {metodo: 'nombreCompleto'},
        },
        email: {
            tipo: 'email',
            label: 'Email',
            grupoDeColumna: 'Contacto',
        },
        telefono: {
            tipo: 'texto',
            label: 'Teléfono',
            grupoDeColumna: 'Contacto',
        },
        fechaIngreso: {
            tipo: 'fecha',
            label: 'Ingreso',
            descripcion: 'Fecha de ingreso a la institución',
            valorPorDefecto: {metodo: 'fechaDeHoy'},
        },
        activo: {
            tipo: 'booleano',
            label: 'Activo',
            valorPorDefecto: true,
        },
    },
} as const satisfies EntidadDef<typeof tiposDeDominio>;
```

- `nombreCompleto` es una columna calculada (3.1); el cálculo no está acá, solo su nombre.
- `fechaIngreso` tiene un valor por defecto que remite a un método (`fechaDeHoy`), mientras que `activo` tiene un valor por defecto constante — los dos casos previstos en 3.1.
- `email` y `telefono` comparten el grupo de columna "Contacto": en la grilla aparecen bajo un sobretítulo común.

## 5. La entidad asignacionProfesores

La relación muchos a muchos entre profesores y materias, con datos propios.

```ts
export const asignacionProfesores = {
    nombre: 'asignacionProfesores',
    pk: ['profesor', 'materia'],
    campos: {
        profesor: {
            tipo: 'codigo',
            label: 'Profesor',
        },
        materia: {
            tipo: 'codigo',
            label: 'Materia',
        },
        puedeDictarla: {
            tipo: 'booleano',
            label: 'Puede dictarla',
        },
        prioridad: {
            tipo: 'entero',
            label: 'Prioridad',
            permiteNull: true,
        },
    },
    fks: [
        {campos: ['profesor'], referencia: 'profesores'},
        {campos: ['materia'], referencia: 'materias'},
    ],
} as const satisfies EntidadDef<typeof tiposDeDominio>;
```

- La PK es natural y **compuesta**: el par [profesor, materia] identifica la asignación; no hace falta ningún id sintético.
- `profesor` y `materia` son FKs: se renderizan automáticamente como combos editables-y-desplegables (3.1). El de materia muestra `materia` + `nombre` (el `isName`) + `horasSemanales` (el `inComboBox`); el de profesor muestra `profesor` + `nombreCompleto` — que es una columna calculada.
- `prioridad` es nulleable; el resto de los campos son NOT NULL por defecto.

## 6. Los comportamientos nombrados, aparte

Los métodos referenciados por nombre viven fuera de la definición serializable (2.3). Cómo se organiza este registro (centralizado vs. distribuido) está pendiente; a modo ilustrativo:

```ts
registrarCalculo('nombreCompleto',
    (fila: {apellido: string, nombres: string}) => `${fila.apellido}, ${fila.nombres}`
);

registrarValorPorDefecto('fechaDeHoy', () => fechaActual());

registrarValidacion('email', (valor: string) => /* ... */);
```

El tipado del parámetro `fila` debería derivarse de la definición de la entidad (condición 5.1.1 del documento principal), no escribirse a mano como acá. Cómo se liga el registro con ese tipo derivado es parte del diseño pendiente del registro.

## 7. Qué se deriva automáticamente

De las tres constantes anteriores, sin ninguna mención adicional a ningún campo (criterio 2.1), el framework deriva:

- **Tipos del lenguaje**: `type Materia = FilaDe<typeof materias>` produce `{materia: string, nombre: string, horasSemanales: number}` — es la razón de las tres condiciones de la sección 5.1. En asignacionProfesores, `prioridad` se deriva como `number | null` por el `permiteNull`.
- **SQL**: `CREATE TABLE` (incluyendo la PK compuesta y las FKs), y los `INSERT`/`SELECT`/`UPDATE` del CRUD.
- **Endpoints CRUD** para las tres entidades.
- **Grilla**: columnas con sus labels, tooltips desde las descripciones, sobretítulo "Contacto" agrupando email y teléfono, edición inline.
- **Formulario de alta/edición**: con `fechaIngreso` precargada con la fecha del día y `activo` en verdadero.
- **Combos** en las FKs de asignacionProfesores, tipeables, con código + `isName` + campos `inComboBox`.
- **Validación** en ambos extremos (navegador y servidor, por el isomorfismo de 2.4): rangos y validaciones nombradas salen del catálogo de tipos de dominio; la obligatoriedad, de `permiteNull`.

Verificación del criterio SSOT (2.1): `horasSemanales` aparece **una sola vez** como campo — en la definición de materias. La columna SQL, la validación de rango (vía el catálogo de tipos), la columna de grilla y su aparición en los combos que referencian materias se derivan de esa única mención.

## 8. Cuestiones que este ejercicio dejó a la vista

Decisiones que tomé para poder escribir el ejemplo y que hay que confirmar o cambiar:

1. **FK a nivel de entidad, no de campo** (`fks: [{campos, referencia}]`): lo elegí para que las FKs compuestas se declaren igual que las simples. La alternativa (marca `fk` dentro del campo) es más local pero no generaliza bien a claves compuestas.
2. **Nombres técnicos en camelCase** (`horasSemanales`, `asignacionProfesores`): ¿la columna/tabla SQL se deriva a `horas_semanales` / `asignacion_profesores` automáticamente, o el nombre técnico es único y se usa idéntico en ambos mundos?
3. **Parámetros del tipo solo en el catálogo**: `horasSemanales` (1–60) vive en el catálogo estilo "Edad" (2.5). ¿Qué pasa cuando un campo necesita un rango puntual no reutilizable: se crea igual un tipo en el catálogo, o se admite algún override por campo? Es la pregunta de composición de la sección 2.5.
4. **Nullability**: inventé `permiteNull?: true` con NOT NULL como default. ¿Confirmamos ese default?
5. **Singular/plural**: la entidad se llama `profesores` (plural, como la tabla) y el tipo derivado sería `Profesor` (singular). ¿Se deriva automáticamente (problemático en español) o la definición declara ambos nombres?
6. **`isName` sobre columna calculada**: en profesores el `isName` es `nombreCompleto`, que es calculada. Asumí que es válido; implica que el combo de las FKs a profesores muestra un valor calculado (y que ese cálculo tiene que poder resolverse también al derivar el combo).
