# Control de sentencias SQL en el backend

> Quiero hacer un proyecto con ese título. La idea es verificar errores comunes en los SQL. Por ejemplo. El 99% de las veces cuando uno hace un JOIN quiere que alguna de las dos tablas se acceda por PK. Entonces el sistema debería controlar que sea siempre así o que haya un comentario dentro de la sentencia que indique que esa vez no.

> Eso se podría hacer como una especie de linter o en el ejecutador.

Apr 26
Buena idea, y es un proyecto que se alinea bastante con tu manera de pensar el backend (controles tempranos, fallar con ruido antes que con silencio).

Antes de tirarme a diseñar, te tiro algunas preguntas y observaciones para enmarcar la cosa, porque "linter" vs "ejecutador" no es solo una decisión de implementación, cambia bastante el alcance.

## La distinción que importa: estático vs runtime

**Linter (estático)**: parseás el SQL en tiempo de build/CI, sin conexión a la DB. Ventaja: feedback temprano, no impacta producción. Desventaja: para validar "esta columna es PK" necesitás un esquema cargado de algún lado (introspección previa, archivo de definición, migrations parseadas). Y el SQL dinámico (armado por concatenación o por un query builder) se te escapa, salvo que analices el código que lo genera.

**Validador en runtime (en el ejecutador)**: interceptás cada query antes de mandarla al motor, con acceso al information_schema y al SQL final ya armado. Ventaja: ves el SQL real, incluido el dinámico, y tenés metadata de la DB a mano. Desventaja: overhead por query (mitigable con cache del AST y del esquema), y los errores aparecen cuando el código corre, no cuando se escribe.

En tu stack (Node + PG + backend-plus/full-type) yo arrancaría por el ejecutador, porque:

1. Ya tenés un punto único por donde pasan las queries (o lo podés tener fácilmente en full-type).
2. La introspección de PKs/FKs/índices en PG es trivial vía pg_catalog.
3. El SQL que tu código realmente ejecuta en producción es lo que querés validar, no una aproximación.
4. En desarrollo lo prendés en modo "fail loud", en producción lo dejás en modo "log y métrica" si querés.

Un linter estático lo podés agregar después como segunda capa para CI, reusando el mismo motor de reglas.

## Sobre la regla del JOIN por PK
La regla está bien pensada pero tiene matices que conviene explicitar desde el diseño, porque van a aparecer:

* JOIN por PK de una de las dos tablas es la regla básica. Cubre el caso típico pedidos JOIN clientes ON pedidos.cliente_id = clientes.id.
* JOIN por unique constraint es funcionalmente equivalente y debería aceptarse igual (una unique no nullable es una "PK alternativa" a efectos de cardinalidad).
* JOIN N-a-N vía tabla intermedia: cada JOIN individual de los dos sigue cumpliendo la regla (uno por PK de la tabla extremo). Bien.
* JOIN por rango o por expresión (ON a.fecha BETWEEN b.desde AND b.hasta): legítimo pero raro, va al cajón de "requiere comentario justificatorio".
* Self-join jerárquico (empleados e1 JOIN empleados e2 ON e1.jefe_id = e2.id): cumple, e2 se accede por PK.
* LEFT JOIN a tabla de detalle 1-a-N intencional: acá la regla no aplica, y es justamente cuando querés multiplicar filas. El comentario justificatorio cubre esto.

El mecanismo del comentario inline me parece la decisión correcta. Es la versión SQL del unsafe de Rust o tu encapsulamiento de magia en guarantee-type: el escape hatch existe, es local, y queda documentado en el sitio del uso.

## Otras reglas que valdrían el esfuerzo (mismo motor)
Mientras estés construyendo el andamiaje de parseo + introspección, agregar más reglas es marginal. Algunas candidatas con mucho retorno:

* UPDATE/DELETE sin WHERE (clásico, pero igual aparece). Requiere comentario.
* WHERE sobre columna no indexada en tablas grandes. Esta es más cara de validar (necesitás stats) y más ruidosa, la dejaría para una segunda etapa.
* SELECT * en código de producción. Discutible como regla pero algunos equipos la quieren.
* Comparación con NULL usando = en vez de IS NULL. Bug clásico y silencioso.
* JOIN sin condición (producto cartesiano implícito por coma o por ON true).
* Mismatch de tipos en condiciones de JOIN/WHERE (ej: varchar = int con cast implícito que mata el índice). Esta la podés validar con la metadata que ya tenés.
* Funciones sobre columnas indexadas en WHERE (WHERE lower(email) = ... cuando no hay índice funcional). Mata índices, muy común.

## Decisiones de diseño que conviene tomar temprano
**Cómo parsear SQL**. No lo hagas a mano. Para PG las opciones serias en Node son libpg-query (binding del parser oficial de Postgres, te da un AST fiel al motor) o pgsql-ast-parser (puro JS, más liviano, AST razonable). libpg-query es la respuesta correcta a largo plazo: si el parser de PG lo acepta, vos también; si no, error de sintaxis genuino. Yo arrancaría con ese.

**Sintaxis del comentario justificatorio**. Yo iría por algo estructurado y grepeable, no comentario libre. Por ejemplo:

```sql
SELECT ...
FROM a
JOIN b ON a.x = b.y  -- @sql-lint-allow: join-no-pk reason="reporte agregado, b.y indexada"
```

Estructurado porque querés poder: filtrar por regla específica, requerir que el reason no esté vacío, y eventualmente listar todos los allow activos del codebase para revisarlos cada tanto. Comentario libre estilo "-- ya sé lo que hago" se degrada rápido.

**Granularidad del allow**. Por línea (afecta al statement que la contiene) y por bloque (un comentario al inicio de la query desactiva la regla para toda la query). Suficiente, no te compliques con scopes más finos.

**Modos de ejecución**. Tres: error (tira excepción, dev/CI), warn (loggea, no bloquea, staging), off (producción si te preocupa el overhead, aunque probablemente no haga falta). Configurable por regla, no global.

**Cache**. El AST de cada query y el snapshot del esquema. Las queries en una app suelen ser un set finito que se repite; cachear por hash del SQL te deja overhead despreciable después del warm-up.

## Una preocupación que vale la pena pensar antes
El riesgo más grande de este tipo de herramientas no es técnico, es social: que el equipo termine poniendo -- @sql-lint-allow en todos lados sin pensar, y la herramienta se vuelva ruido decorativo. Dos cosas ayudan:

1. Que el reason sea obligatorio y no vacío (y ojalá que un humano lo revise en el PR).
2. Que el set de reglas sea chico y de alta señal al inicio. Si la primera versión tira 200 warnings en el codebase existente, nadie la va a adoptar. Mejor 3 reglas que casi siempre indican un bug real, que 15 reglas que indican "estilo".

----

¿Querés que avancemos sobre alguno de estos ejes en concreto? Algunas direcciones posibles, decime cuál te interesa más:

* Esbozar la arquitectura del módulo (interfaz pública, cómo se integraría en full-type o backend-plus).
* Ir directo a un prototipo de la regla del JOIN-por-PK con libpg-query para ver si el AST te da lo que necesitás.
* Discutir cómo modelar el catálogo de reglas (tipos, configuración, extensibilidad por el usuario de la lib).
* Pensar la versión "linter estático" en paralelo, qué cosas sí y qué cosas no se pueden validar sin DB.

> El producto de esta conversación quiero que sea un CLAUDE.md para un futuro proyecto.

> La razón vacía no me molesta por ahora. Los metadatos los tengo por fuera (podríamos tenerlos en forma estática, listo para usar en un diccionario multinivel adecuado, para que no sea un lío). De acuerdo en empezarlo en el ejecutador (porque no hay construcción de SQL).

> Debería incluir los múltiples joins. La PK de `departamento d join empleados e on e.depto = d.depto` es `e.depto, e.emplado` porque ese join se podría joinear con `pedidos p on e.empleado=p.pempleado` y estaría bien (y la PK se agranda) y luego con `cliente c on c.cliente = p.cliente` y estaría bien (y la PK se mantiene)

> También hay que considerar las PK múltiples.

Apr 26
Buenísimo el refinamiento de la regla del JOIN. Lo de la "PK acumulada" del resultado del JOIN es la formulación correcta y generaliza bien. Antes de tirar el CLAUDE.md, dejame reflejar lo que entendí para asegurarme de que el documento lo capture bien, porque es el corazón de la primera regla.

## La regla, formalizada
A medida que se construye el JOIN tree de izquierda a derecha, llevamos un conjunto que llamamos "PK del resultado parcial": el conjunto mínimo de columnas que identifica unívocamente cada fila del resultado hasta ese punto.

* Estado inicial (primera tabla T0): la PK del resultado parcial es la PK de T0.
* Cada nuevo JOIN T_n ON <cond>: hay que clasificar el join en uno de tres casos según la condición, mirando sus equiparaciones de columnas.

**Caso A — la nueva tabla se accede por su PK completa**. La condición incluye igualdades que cubren todas las columnas de la PK de T_n contra columnas del lado izquierdo. La PK del resultado parcial se mantiene (no crece). Es el "join que no multiplica filas".

→ Tu ejemplo: ... JOIN cliente c ON c.cliente = p.cliente. La PK del resultado se mantiene en {e.depto, e.empleado}.

**Caso B — el lado izquierdo se accede por algo que es PK (o subconjunto cubierto por la PK actual del resultado parcial)**. Equivalente al caso A pero del otro lado: la nueva tabla puede multiplicar, y su PK se suma a la PK del resultado parcial.

→ Tu ejemplo: partiendo de departamento d, hacer JOIN empleados e ON e.depto = d.depto. d.depto es PK del resultado parcial actual (que es solo d), entonces el join es válido y la PK pasa a ser {e.depto, e.empleado} — que es la PK de e, y como e.depto = d.depto la columna de d queda funcionalmente determinada y no hace falta incluirla.

→ Y el siguiente: JOIN pedidos p ON e.empleado = p.empleado. e.empleado es parte de la PK del resultado parcial, pero no la cubre completa (falta e.depto). Acá hay que pensarlo bien: ¿es válido o no?

Yo diría que sí es válido, y la PK del resultado parcial pasa a ser {e.depto, e.empleado, p.pedido} (asumiendo que pedido es la PK de p, o {empleado, pedido} si es compuesta). El razonamiento: cada fila del resultado parcial previo tiene un e.empleado único dentro de su e.depto, y se está joineando con todos los pedidos de ese empleado. La PK de p (o las columnas de p que la formen) se agrega al conjunto. No hay multiplicación "espuria" porque cada fila previa se cruza con un conjunto bien definido de filas de p.

Esto coincide con tu intuición. La regla generalizada queda:

_Un JOIN es válido si la condición iguala la PK completa de alguna de las dos partes (la nueva tabla, o un subconjunto de columnas que ya forma parte de la PK del resultado parcial y cubre la PK de la nueva tabla, o viceversa)._

Más precisamente, hay que verificar que al menos uno de estos se cumpla:

1. Las igualdades cubren la PK completa de T_n (caso A: el nuevo se accede por PK).
2. Las igualdades cubren alguna PK/unique de T_n por un lado, y por el otro las columnas igualadas están todas dentro de la PK del resultado parcial (caso B: la nueva tabla se está "expandiendo" controladamente sobre filas únicas previas).
Si ninguno se cumple → warning, requiere @sql-lint-allow.

**Caso C (no contemplado por la regla, requiere allow)**: ninguna de las dos puntas se accede por una PK/unique completa. Ej: JOIN ON a.fecha BETWEEN b.desde AND b.hasta, joins por columnas no únicas de ambos lados, etc.

## Detalles que conviene fijar
**PKs y uniques tratadas igual**. Una UNIQUE NOT NULL cumple el mismo rol que una PK para esta regla. El diccionario de metadata debería exponer ambas.

**PKs múltiples (compuestas)**. Cubrirlas significa que las igualdades del ON deben incluir todas las columnas de la PK. Cubrir parcialmente no cuenta como caso A.

**LEFT/RIGHT/FULL JOIN**. La regla aplica igual: el outer join no multiplica más filas que un inner si la cardinalidad está controlada, solo agrega NULLs. La validación es la misma.

**Subqueries y CTEs**. Cada una es un "JOIN tree" independiente con su propia PK del resultado. Los CTEs pueden tener PK derivada (por ejemplo, un SELECT DISTINCT), pero por ahora yo los marcaría como "PK desconocida salvo que esté declarada" y requeriría que el join contra un CTE caiga en caso A (el otro lado va por PK). Esto se puede refinar después.

**Igualdades vs otras condiciones en el ON**. Solo las igualdades simples columna-a-columna cuentan para la cobertura de PK. Condiciones extra (AND a.activo = true) son filtros, no afectan el análisis de cardinalidad.

------

¿Te cierra esta formulación? Si sí, armo el CLAUDE.md con esto como núcleo, más todo lo del andamiaje (parser, motor de reglas, formato del allow, cache, modos, integración con el ejecutador, etc.). Si querés ajustar algo de la regla antes (por ejemplo, ser más estricto en el caso B y exigir cobertura completa de la PK del resultado parcial, no solo de una PK de la nueva tabla), decime y lo incorporo.

> Sí me cierra y agreguemos ser estricto.

> Cuando hay un allow la nueva PK debería calcularse automáticamente. Por ejemplo tengo la tabla asingaciones que asigna cada empleado a una sucursal en cada trimestre y quiero saber qué filas faltan en asignación (cuya pk es trimestre, empleado), para sortear la sucursal.

> Escribiría:
```sql
SELECT e.empleado, t.trimestre, sortear_sucursal()
from empleado e CROSS JOIN trimestre t --- allow-cross-join
  LEFT JOIN asignaciones a ON e.empleado = a.empleado and t.trimestre = a.trimestre
  WHERE t.trimestre IS NULL
```

Apr 26
Excelente ejemplo, y muestra algo importante: el allow no es solo "ignorá esta verificación", es "después del salto, la PK del resultado parcial sigue siendo computable y bien definida". En tu caso, el CROSS JOIN con allow produce una PK del resultado parcial igual a {e.empleado, t.trimestre} (unión de las PKs de ambos lados), y el LEFT JOIN siguiente cae limpio en caso A porque asignaciones se accede por su PK completa.

Eso es justamente lo que hace que la regla con allow se mantenga útil: el sistema sigue rastreando la PK acumulada y las verificaciones posteriores siguen siendo significativas.

Antes de tirar el CLAUDE.md, dejame puntualizar dos cosas que me quedan dando vueltas, porque cambian cómo se redacta:

**1. Estricto en caso B — confirmemos qué quiere decir**. Con la formulación estricta, el ejemplo de JOIN pedidos p ON e.empleado = p.empleado (partiendo de {e.depto, e.empleado}) falla la regla, porque e.empleado no cubre la PK completa del resultado parcial (falta e.depto). Y eso a pesar de que p se accede por su PK completa (caso A) — ah, no, esperá. Si p.empleado es PK completa de p (porque p tiene PK simple empleado), entonces estamos en caso A puro y se acepta. La estrictez del caso B solo importa cuando A no aplica.

Reformulo la regla estricta para que quede sin ambigüedad:

_Un JOIN es válido si se cumple al menos una de estas dos condiciones:_

* _(A) Las igualdades del ON cubren la PK (o una UNIQUE NOT NULL) completa de la nueva tabla T_n._
* _(B estricto) Las igualdades del ON cubren la PK completa del resultado parcial y cubren la PK (o una UNIQUE NOT NULL) completa de T_n._

El caso B estricto es raro en la práctica (es básicamente "joineo dos cosas únicas por sus claves únicas") pero lo dejamos por completitud. El 99% de los joins legítimos van a caer en caso A.

Eso significa que tu ejemplo de empleados→pedidos JOIN pedidos p ON e.empleado = p.empleado solo se acepta sin allow si la PK de pedidos es exactamente {empleado} o un superconjunto cubierto. Si la PK de pedidos fuera {pedido} (con empleado siendo FK no única), entonces no cumple ni A ni B y requiere allow. ¿Estamos de acuerdo en eso? Me parece coherente con la intuición: si un empleado tiene muchos pedidos, ese join multiplica filas y eso debería ser explícito.

**2. Cómputo de la PK después de un allow**. Esto necesita una regla por tipo de allow, porque distintos casos producen distintas PKs:

* allow-cross-join: PK resultante = unión de la PK del resultado parcial y la PK de la nueva tabla. (Tu ejemplo.)
* allow-join-no-pk (join por condición no-PK, ej: rangos, FKs no únicas): PK resultante = unión de PKs de ambos lados, conservadoramente. Es la opción segura: nunca subestima la cardinalidad.
* allow-join-explicit-pk: el usuario declara la PK resultante en el comentario. Útil cuando el desarrollador sabe algo que el sistema no puede inferir (ej: "este self-join está acotado a un solo registro por la condición extra").

La opción default debería ser siempre la conservadora: unión de PKs. Solo si el desarrollador la declara explícitamente, se usa la declarada. Esto encaja con tu filosofía de no introducir magia: el sistema hace lo seguro, y el escape para hacer algo más fino requiere declaración explícita.

¿Te suena bien? Si sí, con esto y todo lo previo armo el CLAUDE.md completo. Decime también un par de cosas operativas para el documento:

* Nombre del proyecto / repo: ¿algo como sql-guard, sql-lint-runtime, otro? Veo que tenés tradición de nombres descriptivos (guarantee-type, backend-plus).
* ¿Standalone o submódulo de full-type/backend-plus? Mi instinto dice standalone (una lib que después se integra), pero vos conocés mejor el ecosistema.
* ¿TypeScript con la misma rigurosidad de guarantee-type (todo explícito, magia encapsulada en pocos lugares)? Asumo que sí salvo que digas otra cosa.

> Esperá, hacéme las preguntas de a una así puedo ir contestándolas. Ahora solo contesto lo de las PK. En mi ejemplo `departamento d join empleados e on e.depto = d.depto join pedidos p on e.empleado=p.pempleado join cliente c on c.cliente = p.cliente` Ocurre que las pk son depratamentos(departamento), empleados(empleado), pedidos(pedido) clientes(cliente).

> Entoncoes el primer join está bien porque joinea con d.depto (pk completa)
El segundo join está bien porque joinea la subquery de la izquiera cuya pk es e.empleado con p.empleado que no es pk.

> ¿por qué e.empleado es pk del primer join? porque si a departamento entro por pk no necesito agregar la pk de departamento en la pk de empleado, todos los datos de la tabla departamento y todos los campos de empleado que no son pk son dependencias funcionales de la pk de la tabla principal (empleado).

Apr 26
Ah, perfecto. Ahora veo mi error y es importante.

Lo que estás describiendo es dependencia funcional: cuando entrás a una tabla por su PK completa, todos los demás atributos de esa tabla (incluida su PK) quedan funcionalmente determinados por la PK por la que entraste. Entonces no hace falta acumular: la PK del resultado parcial se mantiene en la PK de la tabla "principal", no se hace unión.

Reformulo el cómputo de la PK del resultado parcial:

Estado inicial. Primera tabla T0: PK del resultado parcial = PK de T0.

**Cada JOIN T_n ON <cond>:**

* Caso A — T_n se accede por su PK completa. Las igualdades cubren la PK de T_n con columnas del lado izquierdo. Significa que cada fila del resultado parcial previo se cruza con a lo sumo una fila de T_n. La PK del resultado parcial se mantiene igual (las columnas de T_n, incluida su PK, quedan funcionalmente determinadas por la PK previa).
* Caso B — T_n se "expande" sobre el resultado parcial previo. Las igualdades del ON igualan algunas columnas de T_n contra columnas que están dentro de la PK del resultado parcial previo. Cada fila previa se cruza con todas las filas de T_n que matcheen. La PK del resultado parcial pasa a ser la PK previa unida con la PK de T_n (menos las columnas redundantes por la igualdad).

Aplicando a tu ejemplo:

1. departamento d → PK parcial = {d.depto}.
2. JOIN empleados e ON e.depto = d.depto. La PK de e es {e.empleado}, el ON no la cubre. Pero d.depto (lado izquierdo) es la PK del resultado parcial. Esto es un caso de expansión, caso B. Resultado: PK parcial = {d.depto, e.empleado}. Y como d.depto = e.depto, podemos representarla como {e.depto, e.empleado} indistintamente. Acá viene tu punto sutil: una vez que tenemos {e.depto, e.empleado}, ¿es equivalente a {e.empleado}? Solo si e.empleado es PK de empleados (que lo es). Porque entonces e.depto es funcionalmente determinado por e.empleado (es un atributo de la tabla cuya PK es e.empleado). Entonces la PK parcial se simplifica a {e.empleado}. Esto es el paso clave que me había salteado: después de cada join, simplificar la PK parcial eliminando columnas que estén funcionalmente determinadas por otras columnas de la PK parcial.
3. JOIN pedidos p ON e.empleado = p.empleado. La PK de p es {p.pedido}, no la cubre. e.empleado (lado izquierdo) sí está en la PK parcial actual {e.empleado}. Caso B. Resultado: {e.empleado, p.pedido}. Simplificación: p.empleado = e.empleado, y p.empleado es atributo de p cuya PK es {p.pedido}, entonces p.empleado queda determinado por p.pedido → la PK parcial se simplifica a {p.pedido}.
4. JOIN cliente c ON c.cliente = p.cliente. PK de c es {c.cliente}, el ON la cubre completa. Caso A. PK parcial se mantiene en {p.pedido}.

Eso da el resultado intuitivo: cada fila del resultado final está identificada por el pedido. Que es lo que uno espera.

--------------

Entonces dos cambios en mi modelo previo, los dos importantes:

1. Caso A no agrega nada a la PK parcial. (Esto ya lo tenía bien.)
2. Caso B agrega la PK de T_n, pero después hay que simplificar usando dependencias funcionales: cualquier columna de la PK parcial que sea atributo no-PK de una tabla cuya PK también está en la parcial, se elimina.

Y la regla de validez del JOIN sigue siendo:

_El JOIN es válido si se cumple A, o bien las igualdades del lado izquierdo del ON están todas dentro de la PK del resultado parcial actual (caso B legítimo)._

Con eso, tu ejemplo pasa los tres joins sin necesidad de allow, y la PK final queda en {p.pedido}. ¿Era eso lo que querías que entendiera?

> Escribís mucho, no me das tiempo a contestar. Solo leí hasta la sección Cada JOIN T_n ON <cond>: pero no Aplicando a tu ejemplo. Porque ya está mal lo de arriba.

> Esto tiene que ser un diálogo donde yo te vaya confirmando las ideas de a una.

> Está mal en el caso genérico vos tenés un arbol que une T1 y T2 (a join b) join (c join d) Lo de izquiera a derecha es un caso particular.

> T1 y T2 son indistinguibles (cada una puede ser una tabla o un join de múltiples niveles) Si T1 es accedida por PK la pk es la de T2, no se unen las pk ni a izquierda ni a derecha.

> Por ahora solo hablamos de inner join (los left y right join son distintos porque sí son distinguibles). Pero lo vemos después

Apr 26
Tenés razón. Disculpá lo largo y la asimetría falsa.

Reformulo, corto:

Para INNER JOIN entre T1 y T2 (cada uno puede ser una tabla o un sub-árbol con su PK ya calculada), son indistinguibles. La regla:

* Si el ON cubre la PK completa de T1 → PK del resultado = PK de T2.
* Si el ON cubre la PK completa de T2 → PK del resultado = PK de T1.
* Si cubre ambas → PK del resultado = cualquiera de las dos (son equivalentes).
* Si no cubre ninguna → no es válido, requiere allow.

¿Voy bien?

> Sí, para poder hacerlo de etapas cortas podemos decir que para el caso no válido habrá un calculador de pk y que también puede ser sobreescrito. Para cross join es la suma de las dos pk, para otros casos dará "no puedo calcularlo aún". Y además del /* allow-no-pk-join "razón" */ se podrá escribir /* new-pk e.depto, t.trimestre */.

Apr 26
Perfecto, claro.

Resumo para confirmar:

**Cómputo de la PK del resultado de un JOIN entre T1 y T2:**

1. Si el ON cubre la PK de T1 → PK = PK(T2).
2. Si el ON cubre la PK de T2 → PK = PK(T1).
3. Si cubre ambas → PK = PK(T1) (equivalente a PK(T2)).
4. Si no cubre ninguna → requiere allow. Y entra el calculador:
    * CROSS JOIN: PK = PK(T1) ∪ PK(T2).
    * Otros casos: "no puedo calcularlo aún" — el usuario debe declarar la PK con /* new-pk ... */ o el resto de las verificaciones del árbol quedan inválidas/desactivadas a partir de ese punto.

**Comentarios disponibles:**

* `/* allow-no-pk-join "razón" */` — autoriza el join que no cumple la regla.
* `/* new-pk col1, col2, ... */` — sobreescribe la PK calculada (o la provee cuando el calculador no puede).

Los dos son independientes: podés tener allow sin new-pk (si el calculador automático te da una PK aceptable, como en cross join), new-pk sin allow (para sobreescribir la PK calculada por el sistema en un join válido, aunque esto sería raro), o ambos.

¿Confirmás? ¿Y el comportamiento cuando hay allow sin new-pk y el calculador no puede inferir — querés que sea error, warning, o que simplemente desactive validaciones posteriores marcando la PK como "desconocida"?

> Sí. Confirmo. Tiene que dar un error de la misma jerarquía de cuando no hay ningún comentario y algo falla.

> También hay que incluir el uso de USING (field1, field2) en vez de ON  .

> Y en LEFT JOIN  es igual a un inner join, pensé que era distinto pero creo que no lo es

Apr 26
Perfecto, anotado todo:

* Allow sin new-pk + calculador no puede inferir → error (misma jerarquía que un join que falla la regla sin allow).
* USING (col1, col2) se trata como equivalente a ON t1.col1 = t2.col1 AND t1.col2 = t2.col2 para el análisis de cobertura de PK.
* LEFT JOIN se trata igual que INNER JOIN para el cálculo de PK y validación.
Una observación rápida sobre LEFT JOIN antes de seguir, y después te hago las preguntas operativas de a una.

LEFT JOIN coincide con INNER en la cardinalidad siempre que el lado derecho se acceda por PK (caso A con T2 = lado derecho). Cada fila izquierda matchea con 0 o 1 fila derecha → no multiplica. Eso es el 99% de los LEFT JOIN reales.

El único caso donde difiere es el LEFT JOIN "expansivo" (lado derecho sin acceso por PK, multiplicando filas), pero eso ya cae en el caso 4 (no cumple la regla, requiere allow). Así que en la práctica la regla unificada funciona bien.

El caso especial real del LEFT JOIN es el de tu ejemplo de asignaciones faltantes con WHERE a.trimestre IS NULL — semánticamente es un anti-join. Pero para el cálculo de PK del resultado parcial no cambia nada: la PK se calcula igual, el WHERE filtra después.

¿Te cierra o querés ajustar algo? Si cierra, te hago la primera pregunta operativa.

> sí que en el USING lo que heredás como pk es col1, col2 no t1.col1, t1.col2 (o sea quedan sin alias)

Apr 26
Cierto, buen punto. Con USING (col1, col2) las columnas del join quedan unificadas (sin alias) en el resultado — es como SQL las expone también (un solo col1, no t1.col1 y t2.col1 por separado). Entonces si esas columnas formaban parte de la PK de alguno de los lados, en la PK del resultado parcial aparecen sin alias.

Anotado.

Ahora sí, primera pregunta operativa para el CLAUDE.md:

**¿Nombre del proyecto / repo?** Algunas opciones que se me ocurren siguiendo tu estilo descriptivo (guarantee-type, backend-plus):

* sql-guard
* sql-sentinel
* safe-sql
* pk-aware-sql

¿Alguno te gusta o tenés otro pensado?

> sql-strict-plus

Apr 26
Anotado: sql-strict-plus.

Segunda pregunta:

¿Standalone o submódulo de full-type/backend-plus?

Mi instinto es standalone: una lib publicable por separado en npm, que después full-type (o backend-plus, o cualquier otro proyecto) la integre como dependencia. Ventajas: testeable aislada, reusable, sin acoplar el ciclo de releases.

¿Standalone o lo querés como parte de algo existente?

> standalone

Apr 26
Anotado: standalone.

Tercera pregunta:

¿TypeScript con la misma rigurosidad de guarantee-type? Es decir: tipos explícitos en todos lados, sin any, sin casts vía unknown, magia encapsulada en pocos lugares claramente delimitados. Y mantener tu estilo de código (uso de var, etc.).

¿Confirmás, o hay algo distinto para este proyecto?

> sí

Apr 26
Anotado.

Cuarta pregunta:

¿Qué parser de SQL usar? Las opciones serias en Node para PG:

* libpg-query: binding del parser oficial de Postgres. AST 100% fiel al motor, si PG lo acepta vos también. Tiene dependencia nativa (compilación al instalar).
* pgsql-ast-parser: puro JS, más liviano, sin compilación nativa. AST razonable pero no idéntico al oficial.
* Otro que ya conozcas o prefieras.

¿Cuál preferís? Considerando tu entorno Windows para desarrollo, libpg-query puede dar dolores de cabeza con la compilación nativa en Windows; pgsql-ast-parser es más limpio en ese aspecto.

> libpq-query qué binarios necesita para compilar?

Apr 26
Para libpg-query en Node, la compilación nativa requiere:

En Windows:

* Python (3.x), porque node-gyp lo usa.
* Visual Studio Build Tools con el workload "Desktop development with C++" (compilador MSVC + Windows SDK). Es bastante pesado (varios GB).
* Alternativamente, instalar todo eso vía npm install --global windows-build-tools (deprecado pero todavía circula) o manualmente desde el instalador de VS Build Tools.

En Linux (tu server):

* build-essential (gcc, make), python3. Mucho más liviano y suele estar ya instalado o a un apt install de distancia.

Lo que pasa en la práctica: el paquete trae el código C del parser de Postgres embebido (no necesita PG instalado en la máquina), pero compila al hacer npm install. En CI/CD y en Linux es indoloro. En Windows es donde duele: bajar VS Build Tools la primera vez es tedioso.

Hay una alternativa intermedia: libpg-query publica prebuilt binaries para las plataformas/versiones de Node más comunes (vía prebuild-install o similar). Si tu combinación de Node + arquitectura + OS está cubierta por un prebuild, no compila nada — solo descarga el binario. En la práctica esto cubre Windows x64 con versiones recientes de Node sin que tengas que instalar VS Build Tools. Solo cae a compilación si no hay prebuild para tu combo.

Resumen práctico: probablemente en tu Windows funcione directo gracias a los prebuilds. Si fallara, te pediría VS Build Tools.

¿Vamos con libpg-query entonces, o preferís evitar el riesgo y arrancar con pgsql-ast-parser?

> qué limitaciones tiene pgsql-ast-parser ? (ej, tiene USING?)

Apr 26
Voy a verificar eso, no quiero contestarte de memoria sobre features específicos de la lib.


Searched the web
Searched the web
Resumen rápido sobre pgsql-ast-parser:

* USING está soportado desde la versión 7.0.1 (fue un issue, ya cerrado).
* Soporta SELECT, INSERT, UPDATE, DELETE, joins (LEFT/RIGHT/FULL/INNER/CROSS), CTEs (WITH), subqueries, UNION, GROUP BY, window functions, etc.
* Tiene astVisitor y astMapper que son herramientas muy cómodas para recorrer/transformar el AST. Para tu caso (recorrer joins y validar) astVisitor es exactamente lo que necesitás.
* Tiene parseWithComments() que te devuelve los comentarios — crítico para vos, porque las directivas /* allow-no-pk-join */ y /* new-pk ... */ viven en comentarios.
* Soporta locationTracking: true para obtener línea/columna de cada nodo, útil para reportar errores con ubicación.

Limitaciones reales:

* No es 100% fiel al parser oficial de PG. Features muy nuevos o sintaxis exótica de versiones recientes pueden faltar.
* Algunos PL/pgSQL constructs (procedures, triggers) son limitados — pero vos lo vas a usar para SQL del backend, no para definir código stored.
* Es mantenido por una sola persona (Olivier Guimbal), con releases razonables pero no tan frecuentes como un proyecto corporativo.

Para tu uso (validar queries de aplicación: SELECT/INSERT/UPDATE/DELETE con joins, CTEs, subqueries) cubre todo lo que necesitás, sin compilación nativa.

¿Vas con pgsql-ast-parser?

> Creo que podemos ir con pgsql-ast-parser

Apr 26
Anotado: pgsql-ast-parser.

Quinta pregunta:

¿Cómo se le pasa el catálogo de PKs/uniques a la lib?

Dijiste que los metadatos los tenés por fuera, listos para un diccionario multinivel. ¿Pensás algo como esto?

```ts
{
  schema: {
    tabla: {
      pk: ['col1', 'col2'],
      uniques: [['otra_col'], ['col_a', 'col_b']]
    }
  }
}
```

¿O preferís otra estructura? (Por ejemplo, sin schema como nivel, o con más metadata por columna como nullable, tipo, etc.)

> esa está bien.

Apr 26
Anotado.

Sexta pregunta:

¿Cuál es la interfaz pública principal de la lib? Es decir, ¿cómo se invoca desde el código del usuario?

Algunas opciones (no excluyentes):

A) Función pura de validación, que recibe SQL + catálogo y devuelve resultado:

```ts
const result = validateSql(sqlString, catalog);
// result: { valid: true } | { valid: false, errors: [...] }
```

El usuario decide qué hacer con el resultado (tirar excepción, loggear, etc.).

B) Wrapper del ejecutador, que recibe el cliente de PG y lo "envuelve":

```ts
const safeClient = wrapClient(pgClient, catalog, { mode: 'error' });
await safeClient.query(sql, params); // valida antes de ejecutar
```

C) Las dos cosas: la función pura como API base, y el wrapper como conveniencia construida encima.

Mi instinto dice C (función pura como núcleo, wrapper opcional encima), pero vos conocés mejor cómo se va a integrar con backend-plus/full-type.

> A. Es más fácil para teastear. Hace una sola cosa.

Apr 26
Anotado: solo función pura validateSql(sql, catalog). El wrapping al ejecutador queda como problema del usuario de la lib (o de full-type cuando la integre).

Séptima pregunta:

¿Qué devuelve exactamente validateSql en caso de error?

Algunas dimensiones a definir:

* ¿Tira excepción o devuelve un resultado? Inclino por devolver resultado ({ valid, errors }), más testeable y deja al usuario decidir.
* ¿Un solo error o lista de errores? Es decir, ¿se detiene en el primer problema o sigue analizando y reporta todos los joins problemáticos de la query?
* ¿Qué información trae cada error? Mínimo: tipo de regla violada (ej: 'join-no-pk'), ubicación (línea/columna del nodo), mensaje. ¿Algo más? ¿La query parcial hasta ese punto, las tablas involucradas, la PK calculada hasta ahí?

Mi sugerencia base:

```ts
type ValidationResult =
  | { valid: true; resultPk: string[] }
  | { valid: false; errors: ValidationError[] };

type ValidationError = {
  rule: 'join-no-pk' | 'allow-without-pk-resolution' | ...;
  message: string;
  location: { line: number; column: number };
  context: { /* tablas involucradas, PK parcial al momento del error, etc */ };
};
```

¿Te sirve esa forma o querés ajustar?

> Sí, pero no agreguemos lo que no necesitamos. O sea está bien no lanzar. Pero location no va (habría que ir recalculándola si no dentro de cada una si la función pudiera ser recursiva)

> Y lo de context podría ser. No es necesario tampoco. Podrían ser un nice to have.

Apr 26
Anotado:

```ts
type ValidationResult =
  | { valid: true; resultPk: string[] }
  | { valid: false; errors: ValidationError[] };

type ValidationError = {
  rule: string;
  message: string;
};
```

Sin location, sin context (este último queda como posible nice-to-have futuro). Lista de errores (no se detiene en el primero).

Octava pregunta:

¿Cómo se identifican las reglas? Necesitamos un identificador para cada regla, que aparece tanto en el rule del error como en los comentarios /* allow-... */.

Hasta ahora aparecieron en la conversación:

* allow-no-pk-join (join que no cumple la regla de PK)
* allow-cross-join (cross join explícito)
* new-pk col1, col2 (declaración explícita de PK del resultado)

¿Querés que mantenga esos nombres? ¿Hay un esquema de naming preferido (kebab-case está bien, o preferís otro)? ¿Algún nombre que cambiarías?

> Otra regla es que los trigger se llamen igual que la función a la que llaman

May 4
Buena. Esa es una regla nueva y de naturaleza distinta: no es sobre cardinalidad ni sobre el árbol de joins, es una regla léxica/estructural sobre CREATE TRIGGER. Encaja bien en el motor general de reglas pero opera sobre otro tipo de statement.

La anoto. Identificador propuesto: trigger-name-matches-function.

Para que quede claro: el trigger CREATE TRIGGER foo BEFORE INSERT ON tabla EXECUTE FUNCTION foo() cumple. CREATE TRIGGER foo BEFORE INSERT ON tabla EXECUTE FUNCTION bar() falla.

¿Y el allow correspondiente sería /* allow-trigger-name-mismatch "razón" */?

Vuelvo a la octava pregunta entonces, ahora más amplia:

Convención de nombres para reglas y allows. Lo que llevamos:

* Regla join-no-pk → allow /* allow-no-pk-join "razón" */
* Regla cross-join (¿implícita?) → allow /* allow-cross-join "razón" */
* Regla trigger-name-mismatch → allow /* allow-trigger-name-mismatch "razón" */
* Directiva /* new-pk col1, col2 */ (no es un allow, es una declaración)

Noto inconsistencia entre join-no-pk (nombre de la regla) y allow-no-pk-join (allow). ¿Querés que sean simétricos, ej: regla no-pk-join y allow allow-no-pk-join? ¿O regla join-no-pk y allow allow-join-no-pk?

> Me generás un documento .md con todo esto que hablamos?

6:19 PM

Estructuró documento técnico con reglas de validación SQL y decisiones pendientes
Estructuró documento técnico con reglas de validación SQL y decisiones pendientes
Ahí lo tenés. Dejé marcadas como "Decisiones pendientes" las cosas que quedaron abiertas (incluida la convención de nombres regla/allow que no terminamos de cerrar). Si querés ajustar alguna sección, decime y lo edito.


Claude
Document · MD
> ¿Y se puede bajar esta conversación completa?

6:26 PM

Evaluó opciones nativas y alternativas para descargar conversación
Evaluó opciones nativas y alternativas para descargar conversación
Sí, hay algunas opciones, aunque ninguna ideal:

Opción nativa (todo el historial junto): Settings → Privacy → Export data, desde claude.ai en el navegador o desde Claude Desktop (no está disponible en iOS/Android). Te llega por mail un link que dura 24hs con un ZIP que contiene un JSON con todas las conversaciones de la cuenta. El formato es estructurado pero no muy legible al ojo desnudo. No te deja bajar una sola conversación.

Manual, para esta sola conversación: Ctrl+A en la página de la conversación y pegar en un editor; o Ctrl+P → "Guardar como PDF" desde el navegador. Lo segundo preserva el layout y es lo más práctico para archivar este chat puntual.

Extensiones de Chrome de terceros: hay varias (AI Chat Exporter, Claude Toolbox, AI Toolbox) que agregan un botón para exportar la conversación actual a PDF/Markdown/JSON. No son oficiales de Anthropic.

Para tu caso, si lo que querés es guardar esta conversación como referencia del proceso de diseño del proyecto (complementando el CLAUDE.md), el "Guardar como PDF" del navegador es probablemente lo más rápido y suficiente.


Fable 5 is temporarily unavailable.
Learn more(opens in new tab)




Claude is AI and can make mistakes. Please double-check cited sources.
