# OPERACIONES STATEFUL

Las operaciones stateful requieren de un estado previo para realizar su procesado. Ya sea registros procesados anteriormente en el stream o registros de otro stream o tabla procesado con el que se han de cruzar datos. Por ello todas las operaciones stateful requieren una state store para almacenar este estado.

Las operaciones stateful se pueden dividir en 3 categorías:
- Agregaciones
- Uniones (joins)
- Ventanas temporales (windowing)

![](static/stateful_operations.png)

Disponemos de operaciones stateful tanto para `KStream` como para `KTable`. Ademas también trabajaremos con `KGroupedStream` y `KGroupedTable` como objetos intermedios de una agregación en curso.

## Agregaciones

En las agregaciones se realizan operaciones sobre los registros con misma key. Las operaciones de aggregación disponibles en el DSL de Kafka Streams son:

### [toTable](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KStream.html#toTable--)
Transforma un KStream en un KTable. La clave y el valor de la tabla resultante son los mismos que los del stream original.
Dado que pasamos a una table solo conservaremos el último valor de cada clave.

La firma del método es la siguiente:
> KTable<K,V> toTable()
> KTable<K,V> toTable(Materialized<K,V,?> materialized)

Para esta operación en concreto la tabla puede estar o no materializada. Utilizando la firma en la que parametrizamos la materialización forzamos a que la tabla se materialice.

### [groupByKey](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KStream.html#groupByKey--)

Operación intermedia que agrupa los registros por clave. Devuelve un KGroupedStream que es un stream de registros agrupados por clave. A partir de este objeto dispondremos de una serie de operaciones de agregación que comentaremos más adelante.

La firma del método es la siguiente:
> KGroupedStream<K,V> groupByKey()

### [groupBy](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KStream.html#groupBy-org.apache.kafka.streams.kstream.KeyValueMapper-)

Operación intermedia que agrupa los registros acorde a un valor. A diferencia de `groupByKey`, `groupBy` permite elegir el valor por el que se van a agregar los registros, no estando limitados solos a la key de los registros. Esta operación esta disponible tanto para KStream como KTable, y devolverá un KGroupedStream o un KGroupedTable en función del objeto al que se aplique la operación. A partir de este objeto dispondremos de una serie de operaciones de agregación que comentaremos más adelante.

La firma del método es la siguiente:
> <KR> KGroupedStream<KR ,V> groupBy(KeyValueMapper<? super K,? super V,? extends KR> selector)
> <KR> KGroupedTable<KR ,V> groupBy(KeyValueMapper<? super K,? super V,? extends KR> selector)


### [aggregate](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html#aggregate-org.apache.kafka.streams.kstream.Initializer-org.apache.kafka.streams.kstream.Aggregator-)

Operación que permite calcular agregaciones de registros según nuestra lógica de negocio. Para realizar una agregación necesitaremos especificar como mínimo un inicializador y un agregador. El inicializador se encargará de inicializar el estado de la agregación cuando llegue la primera ocurrencia de una key, mientras que el agregador se encargará de realizar la agregación de los registros.

La firma del método es la siguiente:
> <VR> KTable<K ,VR> aggregate(Initializer<VR> initializer, Aggregator<? super K,? super V,VR> aggregator)

Adicionalmente para un KGroupedTable necesitaremos especificar también una función de restar (subtractor) para cuando se elimine o se actualice un registro de la tabla.

La firma del método es la siguiente:
><VR> KTable<K ,VR> aggregate(Initializer<VR> initializer, Aggregator<? super K,? super V,VR> adder, Aggregator<? super K,? super V,VR> subtractor, Materialized<K ,VR,KeyValueStore/><org.apache.kafka.common.utils.Bytes ,byte[]>> materialized)

A continuación un ejemplo visual de como se computa una agregación en una KTable:
![](static/ktable_aggregation.png)

Como podemos ver en el step 4, al actualizarse el valor de una key se llama a la función subtract para restar el valor antiguo y a la función add para sumar el nuevo valor. En el step 5 también podemos ver que en caso de un borrado (tombstone) se llama a la función subtract para restar el valor eliminado.


### [count](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html#count--)

Operación que permite contar el número de registros por clave. Devuelve un KTable con la clave y el número de ocurrencias de la misma. Disponible tanto para KGroupedStream como para KGroupedTable.

La firma del método es la siguiente:
> KTable<K,Long> count()

### [reduce](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html#reduce-org.apache.kafka.streams.kstream.Reducer-)

Operación que permite reducir los registros de una clave a un único valor. A diferencia de `aggregate` el valor de la KTable resultante debe ser el mismo que el del KGroupedStream o KGroupedTable origen.

La firma del método es la siguiente:
> KTable<K,V> reduce(Reducer<V> reducer)

### [cogroup](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html#cogroup-org.apache.kafka.streams.kstream.Aggregator-)

Operación que permite realizar una agregación entre dos KGroupedStream. Se requiere que ambos KGroupedStream tengan la misma key, el valor puede variar. Adicionalmente se requiere especificar un agregador que se encargará de realizar la agregación entre los valores. Esta operación nos devuelve un CogroupedKStream, que nos permitiría seguir realizando cogroup con otros streams o finalizar la agregación con un `aggregate`.

La firma del método es la siguiente:
>ª <Vout> CogroupedKStream<K,Vout> cogroup(Aggregator<? super K,? super V,Vout> aggregator)
> <VIn> CogroupedKStream<K,VOut> cogroup(KGroupedStream<K,VIn> groupedStream, Aggregator<? super K,? super VIn,VOut> aggregator)

## [Joins](https://kafka.apache.org/24/documentation/streams/developer-guide/dsl-api.html#joining)

En muchas ocasiones, se necesita mergear los datos de un tópico con los de otro u otros. Para ello, se emplean los joins. Existen distintos tipos de join: 
* Inner join(): emite un evento cuando ambas fuentes tienen registros con la misma clave.
* Left join(): emite un evento con cada evento de la fuente de la izquierda o primaria. Si la otra fuente no tiene valor para una determinada clave, se interpreta como un nulo.
* Outerjoin(): emite un evento con cada entrada. Si sólo una fuente tiene datos para una clave, la otra se interpretará como nulo.

Las distintas uniones que se pueden dar, se muestran en la siguiente tabla:
![](static/joins.png)

### primary-key joins

Como condición para este tipo de uniones:

* Los tópicos de entrada deben estar co-particionados, lo cual asegura que los registros con la misma key irán a la misma partición.
* Todos los productores de los tópicos deben usar la misma estrategia de particionado, por el mismo motivo del punto anterior.
* Los tópicos de entrada usan el mismo tipo de keys.

El co-particionado no es necesario en las uniones KStream-GlobalKtable, ya que la GlobalKTable tiene los datos de todas las particiones.

Como ejemplo de unión, podemos ver la unión de 2 KTables:
> KeyValue<K, LV> leftRecord = ...;
> KeyValue<K, RV> rightRecord = ...;
> ValueJoiner<LV, RV, JV> joiner = ...;
>
> KeyValue<K, JV> joinOutputRecord = KeyValue.pair(
>    leftRecord.key, /* ambas claves deben ser iguales */
>
>    joiner.apply(leftRecord.value, rightRecord.value)
>  );

### [foreign-key joins](https://kafka.apache.org/24/documentation/streams/developer-guide/dsl-api.html#streams-developer-guide-dsl-joins-ktable-ktable-fk-join)

No obstante, hay muchos casos en los que necesitamos unir 2 tópicos que no comparten key o que uno de ellos tenga más campos en la clave que el otro.
Para esos casos, existe el Foreing-Key join en Kafka Streams.
Son uniones KTable-KTable sin ventana temporal, con 2 tópicos de entrada (left y right). El tópico right será el que tiene la clave primaria y se aplicará una función foreign-key extractor al tópico de la izquierda.
Este tipo de unión no requiere de co-particionado y la salida, será un KTable.
El KTable de la izquierda, puede contener múltiples registros que mapean a la misma clave de la KTable de la derecha, por lo que una actualización del tópico de la derecha producirá un resultado único a la salida, pero un cambio en un registro de la derecha, producirá múltiples registros de salida.

Un ejemplo de foreign-key join sería el siguiente:

> KTable<String, Long> left = ...;
>
> KTable<Long, Double> right = ...;
>
>//Este foreignKeyExtractor simplemente usa el valor de la izquierda como clave para mapear la clave de la derecha
>
>Function<Long, Long> foreignKeyExtractor = (x) -> x;
>
> KTable<String, String> joined = left.join(right,
>
>    foreignKeyExtractor,
>
>    (leftValue, rightValue) -> "left=" + leftValue + ", right=" + rightValue /* ValueJoiner */
>
>     );

A modo ilustrativo de las fk-joins:

![](static/fkJoin_example.png)


> ![](static/informacion.png)
>
>[Más información sobre uniones](https://www.confluent.io/blog/>crossing-streams-joins-apache-kafka/)
>
>[La evolución hasta llegar a las foreign-key-joins](https://www.confluent.io/blog/data-enrichment-with-kafka-streams-foreign-key-joins/)

### [windowedBy](https://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html#windowedBy-org.apache.kafka.streams.kstream.Windows-)

Operación que nos permite iniciar una agregación por ventanas temporales. Esta ventana de tiempo nos permite agrupar los registros por un periodo de tiempo. Explicaremos en más detalle las operaciones que podemos realizar a partir de aquí en la sección de ventanas temporales.

La firma del método es la siguiente:
> <W extends Window> TimeWindowedKStream<K ,V> windowedBy(Windows<W> windows)

## [VOLVER](readme.md)