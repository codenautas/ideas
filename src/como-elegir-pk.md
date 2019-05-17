# ¿Cómo elegir la *primary key*?

## Clave natural

**Siempre que se disponga debe preferirse una clave natural**. 
O sea un campo o conjunto de campos que ya forme parte de los datos 
de la entidad que se está modelando 
(aún cuando originalmente se hubiera decidido que no era útil 
almacenar esos campos). 
Por ejemplo para codificar países una buena opción es utilizar el 
código [ISO 3166](https://es.wikipedia.org/wiki/ISO_3166-1),
aún cuando ese código no signifique mucho en la organización. 

## Claves compuestas

A veces la clave natural pueden ser un conjunto de camos.
Todas las bases de datos modernas (y los frameworks u ORMs potentes)
permiten definir claves primarias consistentes en varios campos.

Las claves compuestas son útiles cuando se quiere modelar la
pertenencia. Por ejemplo los países se dividen en estados o provincias, 
cada uno de esos estados o provincias puede tener un código
que los identifique dentro del país, entonces podría elegirse como
*PK* el par código de país, código de provincia/estado. 
Otro ejemplo puede ser el de una factura de venta, 
cada factura tiene un número que es su PK, cada ítem dentro de la factura
tiene un número de orden dentro de la factura, entonces la *PK* puede ser
número de factura, número de órden. Incluso si la empresas tuviera
sucursales, la *PK* de sucursal tendría que ser parte de la *PK*
de factura y por lo tanto de la *PK* de ítem de factura. 

## Claves durante el tiempo

Un requisito para una buena clave es que no se repita. 
Que no se repita ni con los datos actuales ni con los datos futuros. 

  1. **Tiene que ser inconcebible que pueda repetirse**. 
  2. Todos los datos lo tienen que tener
  3. Ese dato tiene que conocerse
  4. No tiene que cambiar a lo largo del tiempo 
     (o sea tiene que servir para) identificar la entidad

Por ejemplo en Argentina el número de documento de identidad 
de una persona no es único (por error administrativo en la época en que los 
documentos se confeccionaban manualmente). Además no todas las personas
lo tienen (si nuestro sistema está relacionado con turistas porque ellos 
no tienen por qué tener documento argentino). El pasaporte tampoco sirve
porque no todos lo tienen y además cambia con cada renovación. 

En una biblioteca universitaria no podemos usar como clave principal
para identificar a los lectores el número de libreta universitaria 
porque eso impediría prestarle libros a docentes o a alumnos de otras 
universidades.

En algunas empresas tampoco sirve el número de empleado para identificar
a los miembros de un equipo de trabajo (aún cuando sean todos empleados)
porque el área que asigna los números (alguna oficina de personal)
quizás tarde algunos días en asignarle el número y haya necesidad de 
incorporarlo al equipo real antes de que eso suceda. 

Hay que preveer qué se hará con las entidades que dejan de existir, 
porque su historia persiste. 
Un cliente deja de serlo pero por cuestiones legales e impositivas
las facturas que le hemos hecho necesitan conocer todos los datos del cliente. 
Los países pueden cambiar y dejar de existir. Por ejemplo Yugoslavia
ya no existe, tenemos que seguir pudiendo usar ese código para mencionar
hechos del pasado (ej: nacimientos de personas), pero ya no como destino
de entrega de productos. 

## Claves artificiales

En [por qué no usar claves autonuméricas](no-usar-auto-id.md) hay una
serie de razones por las cuáles no deben usarse si no son necesarias. 
pero cuando hay que poner una clave artificial porque no existe en
la entidad a modelar un campo o conjunto de campos adecuados las opciones son:

  1. Usar un campo *autonumérico* (*serial* en postgresql) basado en un
     generador de secuencias. Es la mejor opción cuando se necesita una 
     clave simple (de un único campo)
     Tienen un problema, en algunas circunstancias pueden dejar agujeros 
     y eso podría confundir o molestar a los usuarios.
     La razón de los agujeros es que la secuencia que genera los números
     no forman parte de la transacción, el número es generado y entregado
     al cliente para que identifique el registro temporal, si la inserción
     del registro falla o si está dentro de una transacción que finalmente
     falla el número no se recupera. Esto es una ventaja en bases de datos
     que realizan muchas inserciones en cortos períodos de tiempo porque
     no es necesario que las inserciones se esperen unas a otras 
     (el motor de base de datos las puede ejecutar en paralelo).
  2. Usar una secuencia estricta en órden creciente. 
     Puede ser una buena opción en tablas con claves compuestas 
     (por ejemplo el número de renglón de las facturas).
     Algo similiar a `insert into (id, dato) select max(id)+1, dato from T`
     La ventaja es que los únicos agujeros son debido a registros borrados. 
     La desventaja es que el manejo de inserciones en paralelo no se puede 
     hace en forma eficiente (sin bloquear datos de más). 
  3. Usar la concatenación o un JSON o un hash de algunos de los datos
     de la entidad. Es más complejo, 
     crea la tentación de obtener algunos datos de ese campo construido 
     (cuando forma parte de la clave de una tabla hija). 