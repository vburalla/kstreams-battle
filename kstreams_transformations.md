# Transformaciones de datos

Kafka Streams proporciona un conjunto de operaciones de transformación para los objetos `KStream` y `KTable`. Estas operaciones permiten a los desarrolladores aplicar su lógica de negocio a los datos que fluyen a través del stream.

La API de Kafka Streams dispone de una fluent API que permite encadenar operaciones de transformación. En función de la operación que se aplique se devolverá el objeto que corresponda. Por ejemplo, si se aplica una operación de `map` sobre un `KStream` se devolverá un nuevo `KStream` con los datos transformados. Si en cabio aplicamos un `groupBy` con un count sobre un `KStream` se devolverá un `KTable` con el conteo de ocurrencias basado en el campo de agrupación.

Las operaciones de transformación se pueden dividir en dos categorías: `stateless` y `stateful`.

## Stateless
Las operaciones stateless son aquellas que no requieren mantener un estado interno. Estas operaciones se aplican a cada registro de forma independiente, de forma que no se necesita información adicional de otros registros para procesar el actual. Algunas de las operaciones stateless más comunes son: `map`, `filter`, `flatMap`, `branch`, `selectKey`, etc.

## Stateful
Las operaciones stateful son aquellas que requieren mantener un estado interno. Estas operaciones requieren de una state store para poder almacenar la información de registos anteriores para aplicar su lógica. Algunas de las operaciones stateful más comunes son: `aggregate`, `reduce`, `count`, `join`, etc.