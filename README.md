# Presentacion del Proyecto Streaming

Guia balanceada para exponer el pipeline de streaming (Kafka + Spark) y demostrar los requisitos cumplidos.

## 1) Objetivo de la exposicion

- Demostrar un pipeline end to end: productor -> Kafka -> procesamiento en tiempo real -> sink.
- Evidenciar ventanas de tiempo y manejo de datos tardios con watermarks.
- Mostrar que los eventos son transformados, no solo movidos.

Pregunta clave para el jurado: Que nueva informacion obtenemos al procesar el stream?

## 2) Arquitectura del pipeline (dual coding)

```
Productor Python (Binance)
        |
        v
Kafka topic: market.prices.raw
        |
        v
Spark Structured Streaming
  - parsea JSON
  - ventana tiempo
  - watermark
  - agregaciones
        |
        +--> Consola (metricas por ventana)
        +--> CSV en data/ (metricas y alertas)
        +--> Kafka topic: market.alerts
```

Mensaje clave: el valor aparece cuando agrupamos por ventanas y detectamos picos de precio.

## 3) Recorrido por codigo (chunking + mensajes clave)

### 3.1 Productor Python (eventos JSON continuos)

Archivo: `src/streams/binance_kafka_producer.py`

Fragmento clave: envio continuo a Kafka con JSON.

```python
producer = KafkaProducer(
    bootstrap_servers=[BOOTSTRAP],
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
)
...
if tick:
    producer.send(TOPIC, value=tick)
    print_event({"type": "kafka_sent", "symbol": tick["symbol"], "ts": tick["ts"]})
```

Idea para decir: "Cada evento tiene simbolo, precio y timestamp; esto es la materia prima del stream".

### 3.2 Lectura desde Kafka y parseo de JSON

Archivo: `spark/spark_processor.py`

```python
price_stream = (
    spark.readStream.format("kafka")
    .option("kafka.bootstrap.servers", KAFKA_BOOTSTRAP)
    .option("subscribe", PRICE_TOPIC)
    .option("startingOffsets", "latest")
    .load()
)

prices = (
    price_stream.selectExpr("CAST(value AS STRING) as json")
    .select(from_json(col("json"), price_schema).alias("data"))
    .select("data.*")
    .withColumn("event_time", (col("ts") / 1000).cast("timestamp"))
)
```

Idea para decir: "Convertimos el JSON del broker a columnas estructuradas para poder agregar y filtrar".

### 3.3 Ventanas de tiempo (tumbling y sliding)

```python
price_windows = (
    prices.withWatermark("event_time", "2 minutes")
    .groupBy(window(col("event_time"), "1 minute"), col("symbol"))
    .agg(
        spark_min("price").alias("min_price"),
        spark_max("price").alias("max_price"),
        spark_avg("price").alias("avg_price"),
    )
)

bet_windows = (
    bets.withWatermark("event_time", "2 minutes")
    .groupBy(window(col("event_time"), "30 seconds", "10 seconds"), col("symbol"))
    .agg(
        spark_count("amount").alias("bet_count"),
        spark_sum("amount").alias("total_amount"),
    )
)
```

Idea para decir: "Cada minuto calculamos min, max y promedio; con ventana deslizante contamos apuestas".

### 3.4 Watermarks (datos tardios)

```python
prices.withWatermark("event_time", "2 minutes")
```

Idea para decir: "Toleramos eventos atrasados hasta 2 minutos; evita recontar indefinidamente".

### 3.5 Sinks (consola + CSV + Kafka)

```python
price_windows.writeStream.outputMode("append")
    .foreachBatch(lambda df, bid: console_writer(df, bid, "PRICE_WINDOWS"))
    .start()

bet_windows.writeStream.outputMode("append")
    .foreachBatch(write_bets_csv)
    .start()

spike_windows.selectExpr("to_json(named_struct(...)) AS value")
    .writeStream.format("kafka")
    .option("topic", ALERTS_TOPIC)
    .start()
```

Idea para decir: "Mostramos en consola y persistimos CSV; ademas publicamos alertas al topico".

## 4) Demo guiada (storytelling + timing)

Duracion sugerida: 3 a 5 minutos.

1) Arranque de infraestructura
   - Mostrar `docker-compose.yml` con Kafka + Zookeeper.
2) Iniciar productor Python
   - Mostrar logs tipo `kafka_sent`.
3) Iniciar Spark
   - Mostrar salida en consola con ventanas agregadas.
4) Mostrar archivos CSV en `data/`
   - Evidenciar que se escribe por ventana.
5) Explicar alertas de picos
   - Mostrar como se detecta spike_up o spike_down.

Frase de cierre: "El pipeline transforma ticks en metricas por ventana y alertas accionables".

## 5) Evidencia de requisitos cumplidos

- Docker Compose con Kafka y Zookeeper: `docker-compose.yml`
- Productor Python continuo: `src/streams/binance_kafka_producer.py`
- Broker Kafka y topicos: configurado en codigo (topics por env)
- Procesamiento en Spark: `spark/spark_processor.py`
- Ventanas de tiempo: `spark/spark_processor.py`
- Watermarks (datos tardios): `spark/spark_processor.py`
- Sink estructurado (CSV y consola): `spark/spark_processor.py`

## 6) Preguntas de repaso (active recall)

1) Que transformacion agrega valor en nuestro stream?
2) Que diferencia hay entre tumbling y sliding windows?
3) Por que usamos watermark de 2 minutos?
4) Que evidencia muestra que los datos no solo se mueven, sino se agregan?

## 7) Nota sobre bonus

Flink y dashboards en tiempo real no estan en el repo actual. Si preguntan, explicar el alcance actual con Spark.
