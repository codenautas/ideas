<style>
.war li{
    list-style-image: url(warning.png)
}
</style>

# Por qué no usar *id* autonuméricos en las bases de datos

En este texto vamos a intentar anotar las **razones** por las cuales se podría decidir 
**no usar un id autonumérico**.

## Por qué mucha gente usa los *id* autonuméricos

  * Porque es comprensible por cualquier programador.
  * Porque no hay que recordar cuál es la *clave natural* de cada tabla antes de interactuar con la base de datos. 
  * Porque es más fácil cuando se va a utilizar un framework o librería.
  * Porque es una opción que está siempre. En cualquier contexto puede utilizarse un id autonumérico.
  
## Razones para **no usar** *id* autonuméricos

Hay razones suficiente para no usar *id autonuméricos* casi en ninguna circunstancia. 
Algunas son de orden netamente prácticas y otras son de orden teóricas 
(pero que también tienen consecuencias prácticas):

### Cuando en la tabla existe una [*clave natural*](https://es.wikipedia.org/wiki/Clave_natural)

Cuando una tabla ya tiene una *clave natural* 
(por ejemplo el Código [ISO 3166](https://es.wikipedia.org/wiki/ISO_3166-1) de país en la tabla países 
o la [*CUIT*](https://es.wikipedia.org/wiki/Clave_%C3%9Anica_de_Identificaci%C3%B3n_Tributaria) 
en la tabla clientes o el legajo del empleado en la tabla de empleados), 
hay más razones para no usar un *id autonumérico*:

<div class=war>

  * Al agregar un *id* se está agregando una [*clave candidata*](https://es.wikipedia.org/wiki/Llave_candidata) 
    a la tabla, ambas la *clave natural* y el *id* son *claves candidatas*, eso agrega complegidad al modelo de datos.
    Este problema puede empeorar si no se designa la *clave candidata* como 
    [*clave única*](https://es.wikipedia.org/wiki/Clave_primaria#Definiendo_claves_%C3%BAnicas) 
    dentro de la base de datos.

A veces la *clave natural* no está almacenada en la base de datos porque al diseñar la tabla no se pensó 
que ese dato fuera importante. Por ejemplo el código ISO de país. 
**Es recomendable antes de decidir usar un id autonumérico como clave investigar qué claves naturales existen 
y evaluar cuál es la conveniente**.

## Desarrollo utilizando metodologías ágiles

Si se están usando metodologías ágiles de desarrollo es normal no tener definido el modelo de datos completo
y ya tener algunos módulos del sistema funcionando en producción. 
A veces se pueden tener funcionando módulos en forma separada o con distinto nivel de integración o madurez. 
Durante esta etapa la estructura de la base de datos varía, se incorporan otros módulos 
o se fusionan subsistemas. 

Podemos encontrar las siguientes ventajeas de orden práctico de usar claves naturales: 

  * Cada sucursal tiene, provisionalmente, su propio sistema separado, 
    luego se juntarán las bases de datos en un sistema centralizdo o distribuido pero coordinado. 
    Si por ejemplo una misma empresa es cliente en dos sucursales distintas,
    al unir o coordinar las bases de datos tendrá que haber un único registro para ese cliente,
    si el sistema de cada sucursal asignó un id autonumérico es probable que ese id se distinto 
    en cada sistema y al unificar habrá que renumerar todos los registros relacionados 
    (ventas, presupuestos, etc) en alguna de las dos sucursales.
  * Se le puede pedir a algún usuario experto que vaya adelantando tiempo 
    preparando las tablas referenciales y sus tablas relacionadas en Excel 
    para incluirlas después directamente en el sistema.
  * Se puede decidir unir dos tablas que en principio parecía que iban a estar separadas
    (por ejemplo clientes y proveedores unirlas en la tabla "cuentas")
  * Cuando se tienen claves compuestas jerarquizadas 
    (ej: para provincias/estados, o como se llame la primer subdivisión nacional, 
    teniendo como clave principal una clave compuesta por los campos código de país + código de provincia
    y su correspondiente clave foránea entre el campo país y la tabla países) 
    se tiene la ventaja adicional de poder filtrar fácilmente por la jerarquía
    para por ejemplo agregar seguridad por filas 
    o particionar la base de datos en distintos servidores 
    (por ejemplo se puede tener distintos permisos para el contenido referido a cada país,
    si el código de país está en todas las claves de las tablas donde eso tiene sentido
    cada usuario podrá ver o modificar los registros de las tablas hijas referidas a esos países).
    Implementar esa partición con id autonuméricos implica tener que consultar la anidación
    de tablas o agregar el campo por el que se desaría filtrar o particionar en cada tabla
    violando así la [tercera forma normal](https://es.wikipedia.org/wiki/Tercera_forma_normal)
    (por ejemplo si fuera necesario agregar el id de país en todas las tablas).
    **Agregar redundancia** en la base de datos **es un potencial peligro**
    que solo debería hacerse en casos justificados explícitamente 
    (en general por cuestiones de performance). 
  * Al mirar las tablas en forma directa (durante el desarrollo) las claves naturales dan 
    más información que los id autonuméricos (se le puede preguntar en voz alta a un usuario 
    "qué cliente tiene tal CUIT" y podrá ir a buscarlo a algún lugar, pero no a qué cliente se 
    le asignó tal id porque ese id solo tiene sentido dentro del sistema).

</div>
