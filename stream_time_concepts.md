# CONCEPTOS ADICIONALES STREAMS

### Tiempo

En Kafka Streams, el tiempo es un concepto crítico a la hora de procesar flujos. La noción del tiempo y cómo se modela e integra es funadmental.
Kafka Streams soporta las siguientes nociones temporales:

#### event-time

El punto en el tiempo en el que se produjo un evento o registro de datos (es decir, fue creado originalmente por la fuente).
Para ello, es necesario incruster marcas de tiempo en los registros de datos en el momento en que se producen.

#### processing-time

El momento en que el evento es procesado por la aplicación de procesamiento (cuando el registro está siendo consumido).

Por ejemplo, en una aplicación que lee datos, el processing-time podría ser de milisegundos, segundos, horas, etc. después de haberse producido el evento.

#### ingestion-time

Es el instante en que un evento es almacenado en una partición de un tópico de Kafka. El broker de Kafka incrustará el timestamp en el momento en que lo escribe en el tópico.
En general el ingestion-time debería ser muy cercano al event-time.


### Timestamps de salida

A los registros producidos se les asignará el timestamp siguiendo los siguientes criterios:

- Cuando se generan nuevos registros fruto de un procsamiento directo, el timestamp heredará de la entrada directamente. En operaciones flatMap y resto de operaciones que emiten varios eventos a partir de uno recibido, el timestam será el del evento que genera los múltiples eventos.
- Para agregaciones, el timestamp será el del último registro que sea agregado.
- Para uniones stream-stream y table-table, el timestamp será el mayor de la unión
- Para uniones stream-table, el timestamp será el del evento del stream.

> ![](static/informacion.png) [Más información de conceptos temporales en KStreams](https://kafka.apache.org/23/documentation/streams/core-concepts#streams_time)

## [VOLVER](readme)