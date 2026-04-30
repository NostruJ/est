# ESTUDIO - GambleBitCoin

Material de estudio basado en las preguntas de repaso del proyecto GambleBitCoin. Incluye teoría y ejemplos de código real del proyecto.

---

## Índice

1. [¿Qué garantiza que una apuesta se procese con el precio correcto?](#1-qué-garantiza-que-una-apuesta-se-procese-con-el-precio-correcto)
2. [¿Cómo maneja el sistema la desconexión de Binance?](#2-cómo-maneja-el-sistema-la-desconexión-de-binance)
3. [¿Qué diferencia hay entre el estado en Redis y el estado en memoria?](#3-qué-diferencia-hay-entre-el-estado-en-redis-y-el-estado-en-memoria)
4. [¿Por qué usamos Socket.IO en lugar de HTTP polling?](#4-por-qué-usamos-socketio-en-lugar-de-http-polling)
5. [¿Cómo se asegura que no haya apuestas después del bloqueo?](#5-cómo-se-asegura-que-no-haya-apuestas-después-del-bloqueo)
6. [¿Qué rol juega Kafka en la arquitectura?](#6-qué-rol-juega-kafka-en-la-arquitectura)
7. [¿Qué funcionalidad tiene Spark en el proyecto?](#7-qué-funcionalidad-tiene-spark-en-el-proyecto)
8. [Funcionalidad de SRC y sus Instancias](#8-funcionalidad-de-src-y-sus-instancias)
9. [Cómo Funciona Docker y Funcionalidad de sus Contenedores](#10-cómo-funciona-docker-y-funcionalidad-de-sus-contenedores)
10. [Anexos](#11-anexos)

---

## 1) ¿Qué garantiza que una apuesta se procese con el precio correcto?

### Teoría:
El sistema utiliza un **ciclo de ronda temporal** con tres fases bien definidas:
1. **Fase abierta** (20 segundos): Se aceptan apuestas con el precio inicial
2. **Fase de bloqueo** (5 segundos): No se aceptan más apuestas, se prepara la resolución
3. **Fase de resolución**: Se compara el precio inicial vs el precio final de Kafka

La clave está en que el precio se toma directamente del stream de Kafka (Binance) y no se manipula manualmente.

### Código:

**Archivo: `src/services/marketService.js` (líneas 240-287)**
```javascript
startRoundLoop() {
  setInterval(async () => {
    const now = Date.now();

    for (const symbol of this.symbols) {
      const market = this.state[symbol];
      const round = market.currentRound;

      if (!round) {
        await this.openRound(symbol);
        continue;
      }

      const msLeft = Math.max(0, round.endAt - now);
      const secondsLeft = Math.ceil(msLeft / 1000);

      // Bloquear apuestas a los 15 segundos (20 - 5)
      if (!round.locked && now >= round.lockAt) {
        round.locked = true;
        await this.repo.saveRound(symbol, round);
        this.io.to(this.room(symbol)).emit("round_locked", {
          symbol,
          roundId: round.id,
          secondsLeft,
        });
      }

      // Actualizar temporizador en frontend
      this.io.to(this.room(symbol)).emit("round_timer", {
        symbol,
        roundId: round.id,
        secondsLeft,
        lock: round.locked,
      });

      // Resolver ronda cuando termine el tiempo
      if (now >= round.endAt) {
        await this.settleRound(symbol);
      }
    }
  }, 1000);
}
```

**Archivo: `app.js` (líneas 150-157) - Precios de Kafka:**
```javascript
onMessage: (topic, payload) => {
  if (topic === config.kafkaPriceTopic) {
    marketService.onPriceTick({
      symbol: String(payload.symbol).toUpperCase(),
      price: Number(payload.price),
      ts: Number(payload.ts || Date.now()),
      source: payload.source || "kafka",
    });
    return;
  }
};
```

### Garantía:
El precio se toma del stream de Kafka en tiempo real, no se manipula manualmente. La ronda se resuelve automáticamente al finalizar el tiempo comparando `startPrice` vs `endPrice`.

---

## 2) ¿Cómo maneja el sistema la desconexión de Binance?

### Teoría:
El sistema tiene un **doble mecanismo de tolerancia a fallos**:
1. **Python-binance (Principal)**: Conexión WebSocket a Binance API
2. **Fallback Generator (Respaldo)**: Si no hay precios nuevos en 12 segundos (`PRICE_STALE_MS`), genera precios simulados

Esto asegura que el servicio nunca se detenga, incluso sin conexión a Binance.

### Código:

**Archivo: `src/services/marketService.js` (líneas 290-316)**
```javascript
startFallbackPriceGenerator() {
  // Solo activa si está habilitado en .env
  if (!this.config.enablePriceFallback) return;

  setInterval(() => {
    const now = Date.now();

    for (const symbol of this.symbols) {
      const market = this.state[symbol];
      
      // Verificar si el precio está estancado (más de 12 segundos)
      const isStale = now - market.latestPriceTs > this.config.priceStaleMs;
      if (!isStale) continue;

      // Generar precio simulado basado en drift aleatorio
      const seed = market.latestPrice ||
        (symbol === "ETHUSDT" ? 3200 : symbol === "SOLUSDT" ? 140 : 620);
      const drift = seed * 0.0006;
      const randomDelta = (Math.random() - 0.5) * drift;
      const next = Math.max(0.0001, seed + randomDelta);

      this.onPriceTick({
        symbol,
        price: Number(next.toFixed(6)),
        ts: now,
        source: "fallback", // Marca la fuente del precio
      });
    }
  }, 1000);
}
```

**Configuración en `.env`:**
```env
ENABLE_PRICE_FALLBACK=true  # Cambiar a true para activar
PRICE_STALE_MS=12000        # 12 segundos sin precio = activar fallback
```

**Archivo: `src/streams/binanceKafkaProducer.js` (líneas 20-35)**
```javascript
start() {
  const scriptPath = path.join(process.cwd(), "src", "streams", "binance_kafka_producer.py");
  
  const startProcess = () => {
    const env = {
      ...process.env,
      BINANCE_SYMBOLS: this.symbols.map(s => s.toLowerCase()).join(","),
    };
    
    // Iniciar script Python que conecta a Binance
    this.child = spawn(this.pythonCmd, [scriptPath], {
      cwd: process.cwd(),
      env,
      stdio: ["ignore", "pipe", "pipe"],
    });
  };
}
```

### Nota:
Tu `.env` actual tiene `ENABLE_PRICE_FALLBACK=false`. Cámbialo a `true` para que el generador de respaldo funcione cuando Binance no esté disponible.

---

## 3) ¿Qué diferencia hay entre el estado en Redis y el estado en memoria?

### Teoría:

| Característica | Memoria (this.state) | Redis |
|-----------------|----------------------|-------|
| **Persistencia** | Volátil (se pierde al reiniciar) | Persistente (sobrevive reinicios) |
| **Velocidad** | Muy rápido (acceso directo) | Rápido (pero con latencia de red) |
| **Compartido** | Solo una instancia | Compartido entre múltiples instancias |
| **Uso** | Estado temporal, precios actuales | Balances, rondas, historial |

### Código:

**Memoria (this.state) - `src/services/marketService.js` (líneas 16-24):**
```javascript
this.state = {};
for (const symbol of symbols) {
  this.state[symbol] = {
    latestPrice: null,        // Precio actual
    latestPriceTs: 0,         // Timestamp del último precio
    tickSize: DEFAULT_TICK_SIZE[symbol],
    currentRound: null,        // Ronda actual en curso
  };
}
```

**Redis - `src/repositories/RedisRepository.js` (líneas 17-19, 45-55):**
```javascript
// Claves en Redis
roundCurrentKey(symbol) {
  return `round:${symbol}:current`; // Ej: "round:ETHUSDT:current"
}

userKey(userId) {
  return `user:${userId}`; // Ej: "user:123"
}

// Guardar usuario en Redis (persistente)
async saveUser(user) {
  await this.redis.hset(this.userKey(user.id), {
    id: user.id,
    name: user.name,
    balance: String(user.balance),  // Persiste saldo
    blocked: String(user.blocked),
  });
  // Guardar índice de nombre único
  await this.redis.set(this.userNameIndexKey(user.name), user.id);
}
```

### Casos de uso:
- **Memoria**: Precios en tiempo real, ronda actual (cambian constantemente)
- **Redis**: Saldo de usuarios, historial de rondas, leaderboard, chat (deben persistir)

---

## 4) ¿Por qué usamos Socket.IO en lugar de HTTP polling?

### Teoría:

| Método | HTTP Polling | Socket.IO |
|--------|--------------|-----------|
| **Dirección** | Cliente → Servidor (pull) | Servidor → Cliente (push) |
| **Latencia** | Alta (depende del intervalo de poll) | Baja (milisegundos) |
| **Overhead** | Alto (headers HTTP en cada request) | Bajo (solo el evento) |
| **Conexión** | Nueva conexión cada poll | Conexión persistente (WebSocket) |

### Código:

**Servidor emite eventos en tiempo real - `src/services/marketService.js`:**
```javascript
// Actualizar precio en todos los clientes suscritos
this.io.to(this.room(symbol)).emit("price_update", {
  symbol,
  price: market.latestPrice,
  timestamp: market.latestPriceTs,
  source: payload.source,
});

// Notificar nueva ronda abierta
this.io.to(this.room(symbol)).emit("round_open", round.toJSON());

// Notificar resultado de la ronda
this.io.to(this.room(symbol)).emit("round_result", {
  symbol,
  round: { id: round.id, startPrice, endPrice, result },
  payouts,
});
```

**Cliente escucha eventos (frontend) - `src/public/main.js`:**
```javascript
// Escuchar actualizaciones de precio
socket.on("price_update", (data) => {
  updateChart(data.price);        // Gráfica se actualiza instantáneamente
  updateCurrentPrice(data.price); // Mostrar precio actual
});

// Escuchar apertura de ronda
socket.on("round_open", (round) => {
  updateRoundUI(round);          // Actualizar interfaz
  enableBetting();               // Habilitar botones de apuesta
});
```

**Configuración Socket.IO - `app.js` (líneas 66-68):**
```javascript
const app = express();
const server = http.createServer(app);
const io = new Server(server); // Socket.IO sobre HTTP server
```

### Ventaja clave:
Latencia de **milisegundos** vs **segundos** de polling. Ideal para apuestas en tiempo real.

---

## 5) ¿Cómo se asegura que no haya apuestas después del bloqueo?

### Teoría:
El sistema implementa una **validación triple** en el servicio de apuestas:
1. **Estado de la ronda**: `round.status !== "open"`
2. **Bandera de bloqueo**: `round.locked === true`
3. **Tiempo límite**: `Date.now() >= round.lockAt` (15 segundos después del inicio)

Aunque haya retraso de red, si el tiempo expiró, la apuesta se rechaza.

### Código:

**Archivo: `src/services/betService.js` (líneas 15-29)**
```javascript
async placeBet({ userId, symbol, side, amount }) {
  // Validar símbolo y lado (sube/baja)
  if (!this.symbols.includes(symbol)) {
    throw new Error("Invalid market");
  }
  if (!this.sides.includes(side)) {
    throw new Error("Invalid side");
  }

  // Obtener ronda actual
  const round = this.marketService.getCurrentRound(symbol);
  
  // Validación 1: ¿Ronda abierta?
  if (!round || round.status !== "open") {
    throw new Error("Round is not open");
  }
  
  // Validación 2 y 3: ¿Bloqueada por tiempo?
  if (round.locked || Date.now() >= round.lockAt) {
    throw new Error("Bet lock is active (last 5s)");
  }

  // Validar saldo y monto
  const user = await this.repo.getUser(userId);
  if (!user.canBet(numericAmount)) {
    throw new Error(`Invalid bet. Amount range is ${this.config.betMin}-${this.config.betMax}`);
  }

  // Validar una sola apuesta por ronda
  const existing = await this.repo.getBetForUser(symbol, round.id, user.id);
  if (existing) {
    throw new Error("Only one bet per round");
  }

  // Procesar apuesta...
}
```

**Configuración de tiempos - `.env`:**
```env
ROUND_SECONDS=20   # Ronda abierta por 20 segundos
LOCK_SECONDS=5     # Bloqueo de 5 segundos antes de resolver
```

### Garantía:
Aunque haya retraso de red, si el tiempo expiró (`Date.now() >= round.lockAt`), la apuesta se rechaza automáticamente.

---

## 6) ¿Qué rol juega Kafka en la arquitectura?

### Teoría:
Kafka actúa como el **sistema nervioso** que interconecta todos los componentes:
1. **Ingesta**: Python envía precios de Binance a `market.prices.raw`
2. **Procesamiento**: Node.js consume y emite eventos
3. **Eventos**: Se publican apuestas, rondas y alertas a diferentes topics
4. **Desacoplamiento**: Permite que productores y consumidores escalen independientemente

### Topics de Kafka:
| Topic | Propósito | Productor | Consumidor |
|-------|-----------|-----------|------------|
| `market.prices.raw` | Precios de Binance | Python | Node.js |
| `market.bets.events` | Eventos de apuestas | Node.js | Analytics (futuro) |
| `market.round.events` | Eventos de rondas | Node.js | Analytics (futuro) |
| `market.alerts` | Alertas de anomalías | Node.js | Frontend |

### Código:

**Consumidor Kafka - `app.js` (líneas 145-165):**
```javascript
const streamConsumer = new KafkaStreamConsumer({
  kafkaMirror,
  topics: [config.kafkaPriceTopic, config.kafkaAlertsTopic],
  groupId: "market-consumer",
  onMessage: (topic, payload) => {
    // Procesar precios
    if (topic === config.kafkaPriceTopic) {
      marketService.onPriceTick({
        symbol: payload.symbol,
        price: payload.price,
        ts: payload.ts,
        source: payload.source,
      });
      return;
    }

    // Procesar alertas
    if (topic === config.kafkaAlertsTopic) {
      io.emit("anomaly_alert", payload); // Alerta al frontend
    }
  },
});
await streamConsumer.start();
```

**Productor Kafka (apuestas) - `src/services/betService.js` (líneas 69-80):**
```javascript
await this.kafkaMirror.send(
  this.config.kafkaBetTopic,
  {
    type: "bet_placed",
    ts: Date.now(),
    symbol,
    roundId: round.id,
    userId: user.id,
    userName: user.name,
    side,
    amount: numericAmount,
    balanceAfter: user.balance,
  }
);
```

**Configuración de topics - `.env`:**
```env
KAFKA_BROKERS=localhost:9092
KAFKA_PRICE_TOPIC=market.prices.raw
KAFKA_BET_TOPIC=market.bets.events
KAFKA_ROUND_TOPIC=market.round.events
KAFKA_ALERTS_TOPIC=market.alerts
```

### Beneficios:
- **Escalabilidad**: Múltiples consumidores pueden leer del mismo topic
- **Persistencia**: Los mensajes se guardan por configurable time
- **Desacoplamiento**: Productor y consumidor no se conocen directamente

---

## 7) ¿Qué funcionalidad tiene Spark en el proyecto?

### Teoría:
Spark actúa como un **motor de análisis de datos en tiempo real** que procesa streams de Kafka para generar métricas, detectar anomalías y crear reportes. Es un componente **adicional** para análisis de datos, no es esencial para el funcionamiento de la aplicación principal (que usa Node.js).

**Nota importante:** Spark NO está incluido en el `docker-compose.yml` actual. Se ejecuta por separado para análisis de datos históricos y tendencias.

### Código:

**Archivo: `spark/spark_processor.py`**

#### 7.1) Procesamiento de Streams desde Kafka (líneas 78-124)

Spark lee datos en tiempo real desde dos topics de Kafka:
- **`market.prices.raw`**: Precios de criptomonedas
- **`market.bets.events`**: Eventos de apuestas

```python
# Leer precios de Kafka
price_stream = (spark.readStream.format("kafka")
    .option("kafka.bootstrap.servers", KAFKA_BOOTSTRAP)
    .option("subscribe", PRICE_TOPIC)
    .option("startingOffsets", "latest")
    .load())

# Parsear JSON y convertir timestamp
prices = (price_stream.selectExpr("CAST(value AS STRING) as json")
    .select(from_json(col("json"), price_schema).alias("data"))
    .select("data.*")
    .withColumn("event_time", (col("ts") / 1000).cast("timestamp")))

# Leer apuestas de Kafka
bet_stream = (spark.readStream.format("kafka")
    .option("subscribe", BET_TOPIC)
    .load())

bets = (bet_stream.selectExpr("CAST(value AS STRING) as json")
    .select(from_json(col("json"), bet_schema).alias("data"))
    .select("data.*"))
```

#### 7.2) Ventanas de Tiempo - Time Windows (líneas 93-140)

Spark agrupa datos en ventanas para análisis temporal:

**A) Ventanas de precios (1 minuto - tumbling window):**
```python
price_windows = (prices.withWatermark("event_time", "2 minutes")
    .groupBy(window(col("event_time"), "1 minute"), col("symbol"))
    .agg(spark_min("price").alias("min_price"),
         spark_max("price").alias("max_price"),
         spark_avg("price").alias("avg_price")))
```

**B) Ventanas de apuestas (30 segundos - sliding window con 10 segundos de deslizamiento):**
```python
bet_windows = (bets.withWatermark("event_time", "2 minutes")
    .groupBy(window(col("event_time"), "30 seconds", "10 seconds"), col("symbol"))
    .agg(spark_count("amount").alias("bet_count"),
         spark_sum("amount").alias("total_amount")))
```

#### 7.3) Detección de Anomalías - Spike Detection (líneas 142-174)

Detecta picos de precio (subidas o bajadas) basado en porcentajes configurables:

```python
spike_windows = (prices.withWatermark("event_time", "2 minutes")
    .groupBy(window(col("event_time"), spike_window), col("symbol"))
    .agg(spark_min("price").alias("min_price"),
         spark_max("price").alias("max_price"))
    .withColumn("spike_up",
        expr(f"(max_price - min_price) / min_price * 100 >= {SPIKE_UP_PCT}"))
    .withColumn("spike_down",
        expr(f"(min_price - max_price) / max_price * 100 <= -{SPIKE_DOWN_PCT}"))
    .filter("spike_up OR spike_down")
    .withColumn("alert_type", expr("CASE WHEN spike_up THEN 'SPIKE_UP' ELSE 'SPIKE_DOWN' END"))
```

**Configuración en `.env`:**
```env
PRICE_SPIKE_UP_PCT=2.0      # Subida de 2% detecta spike
PRICE_SPIKE_DOWN_PCT=2.0    # Bajada de 2% detecta spike
PRICE_SPIKE_WINDOW_SEC=30    # Ventana de 30 segundos
```

#### 7.4) Watermarks - Manejo de Datos Tardíos (línea 94, 127, 144)

Spark tolera datos atrasados hasta 2 minutos:
```python
.withWatermark("event_time", "2 minutes")
```
Esto evita recalcular indefinidamente si llegan eventos con retraso.

#### 7.5) Reportes de Tendencias - Anomaly Report (líneas 176-207)

Genera un reporte de tendencia por ventana de 1 minuto:
- Compara primer vs último precio
- Determina tendencia: UP, DOWN o HOLD

```python
anomaly_report = (prices.groupBy(window(col("event_time"), "1 minute"), col("symbol"))
    .agg(spark_first("price").alias("first_price"),
         spark_last("price").alias("last_price"))
    .withColumn("trend", expr(
        "CASE WHEN last_price > first_price THEN 'UP' "
        "WHEN last_price < first_price THEN 'DOWN' "
        "ELSE 'HOLD' END")))
```

#### 7.6) Múltiples Sinks - Salidas de Datos (líneas 209-381)

Spark escribe los resultados en **4 destinos diferentes**:

| Sink | Datos | Código (líneas) | Descripción |
|------|-------|-------------------|-------------|
| **Consola** | Métricas en tiempo real | 209-236 | Imprime en consola para monitoreo |
| **CSV** | `data/metrics_prices/`, `data/metrics_bets/`, `data/alerts/`, `data/anomaly_report/` | 238-312 | Persistencia para análisis posterior |
| **Kafka** | Alertas de picos a `market.alerts` | 336-357 | Re-inyecta alertas al pipeline |
| **Kafka** | Reportes de anomalías a `market.alerts` | 359-381 | Reportes de tendencias a Kafka |

**Ejemplo de escritura a Kafka (líneas 336-357):**
```python
alerts_to_kafka = (spike_windows.selectExpr(
    "to_json(named_struct(...)) AS value")
    .writeStream.format("kafka")
    .option("kafka.bootstrap.servers", KAFKA_BOOTSTRAP)
    .option("topic", ALERTS_TOPIC)
    .outputMode("append")
    .start())
```

### Resumen de Funcionalidad de Spark:

1. ✅ **Consume streams de Kafka** (precios y apuestas)
2. ✅ **Agrega datos por ventanas de tiempo** (1 min, 30 seg, 30 seg)
3. ✅ **Detecta anomalías** (picos de precio > 2%)
4. ✅ **Genera reportes de tendencias** (UP/DOWN/HOLD)
5. ✅ **Escribe resultados** a consola, CSV y Kafka
6. ✅ **Maneja datos tardíos** con watermarks de 2 minutos

### ¿Cómo ejecutar Spark?

Spark no está en Docker Compose. Para ejecutarlo:

```bash
# Instalar dependencias
pip install pyspark kafka-python

# Ejecutar el procesador
python spark/spark_processor.py
```

O con Spark distribuido:
```bash
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 spark/spark_processor.py
```

---

## 8) Funcionalidad de SRC y sus Instancias

El directorio `src/` contiene toda la lógica de la aplicación Node.js organizada por responsabilidades.

### Estructura General:
```
src/
├── config/           # Configuración global
├── models/          # Modelos de datos (POJOs)
├── services/        # Lógica de negocio
├── repositories/     # Acceso a datos (Redis)
├── controllers/     # Endpoints HTTP
├── sockets/         # Tiempo real (Socket.IO)
├── streams/         # Kafka y Binance
└── public/          # Frontend (HTML, CSS, JS)
```

---

### 📁 **config/** - Configuración Global

**`constants.js`** (14 líneas):
- Define símbolos: `SYMBOLS = ["ETHUSDT", "SOLUSDT", "BNBUSDT"]`
- Lados de apuesta: `SIDES = ["up", "down", "hold"]`
- Tick sizes: `DEFAULT_TICK_SIZE = { ETHUSDT: 0.01, SOLUSDT: 0.001, BNBUSDT: 0.01 }`

**`logger.js`** (17 líneas):
- Funciones: `log(scope, ...args)`, `warn(scope, ...args)`, `error(scope, ...args)`
- Ejemplo: `log("app", "running on port 3000")`

---

### 📁 **models/** - Modelos de Datos (POJOs)

**`Round.js`** (16 líneas):
```javascript
class Round {
  constructor({ id, symbol, startPrice, startAt, endAt, lockAt }) {
    this.id = id;
    this.symbol = symbol;
    this.startPrice = Number(startPrice);
    this.endPrice = null;
    this.status = "open"; // "open" | "locked" | "closed"
    this.locked = false;
  }
}
```

**`User.js`** (33 líneas):
```javascript
class User {
  canBet(amount) { /* valida saldo y límites */ }
  debit(amount) { this.balance -= amount; if (balance <= 0) blocked = true; }
  credit(amount) { this.balance += amount; }
}
```

**`Bet.js`** (14 líneas):
- Representa una apuesta: `{ roundId, userId, side, amount, timestamp }`

**Otros modelos:**
- `ChatMessage.js`: Mensaje de chat
- `Leaderboard.js`: Leaderboard
- `PriceStream.js`: Stream de precios

---

### 📁 **services/** - Lógica de Negocio

**`marketService.js`** (~320 líneas) - **Núcleo del sistema:**
- `startRoundLoop()`: Ciclo de rondas (20s abierto → 5s bloqueo → resolver)
- `onPriceTick()`: Procesa precios de Kafka, actualiza estado
- `settleRound()`: Resuelve ronda comparando `startPrice` vs `endPrice`
- `startFallbackPriceGenerator()`: Genera precios simulados si Binance falla
- `hydrateTickSizes()`: Obtiene tick sizes de Binance API

**`betService.js`** (~92 líneas):
- `placeBet()`: Valida y procesa apuestas (verifica ronda abierta, saldo, límites)
- Envía evento a Kafka topic `market.bets.events`

**Otros servicios:**
- `userService.js`: CRUD de usuarios
- `chatService.js`: Gestión de mensajes de chat
- `leaderboardService.js`: Cálculo de rankings
- `priceUtils.js`: Utilidades para precios

---

### 📁 **repositories/** - Acceso a Datos

**`RedisRepository.js`** (~150 líneas):
- Todas las operaciones Redis centralizadas
- Claves: `user:{id}`, `round:{symbol}:current`, `round:{id}:{symbol}:bets`
- Métodos: `saveUser()`, `getUser()`, `saveBet()`, `getBetForUser()`, `getLeaderboard()`

---

### 📁 **controllers/** - Endpoints HTTP

**`healthController.js`**:
- `GET /health`: Verifica estado de Redis, Kafka y símbolos

**`userController.js`**:
- `POST /api/user`: Crea o obtiene usuario
- `GET /api/user/:id`: Obtiene perfil de usuario

**`adminController.js`**:
- Endpoints admin para gestionar rondas y estado del sistema

---

### 📁 **sockets/** - Tiempo Real (Socket.IO)

**`registerSocketHandlers.js`**:
- `subscribe_market`: Suscribe cliente a un símbolo (se une a `room:{symbol}`)
- `place_bet`: Procesa apuesta via Socket.IO
- `chat_message`: Envía mensaje al chat en vivo
- `get_leaderboard`: Obtiene rankings

**Eventos emitidos al frontend:**
```javascript
io.to(room).emit("price_update", { symbol, price, timestamp });
io.to(room).emit("round_open", round);
io.to(room).emit("round_locked", { roundId, secondsLeft });
io.to(room).emit("round_result", { result, payouts });
io.to(room).emit("chat_update", message);
```

---

### 📁 **streams/** - Kafka y Binance

**`kafkaMirror.js`**:
- Wrapper de kafkajs para producer/consumer
- Métodos: `connect()`, `send(topic, message)`, `createConsumer()`

**`kafkaStreamConsumer.js`**:
- Consume topics de Kafka (precios y alertas)
- Callback `onMessage(topic, payload)` para procesar mensajes

**`binanceKafkaProducer.js`** (81 líneas):
- Spawnea el script Python `binance_kafka_producer.py`
- Maneja reinicios automáticos (máximo 6) si el proceso muere
- Pasa variables de entorno: `BINANCE_SYMBOLS`, `BINANCE_API_KEY`, etc.

**`binance_kafka_producer.py`** (~130 líneas):
- Conecta a Binance WebSocket API
- Envía ticks de precio a Kafka topic `market.prices.raw`
- Maneja reconexión automática

---

### 📁 **public/** - Frontend

**`index.html`**:
- Página principal con gráficas Chart.js
- Interfaz de apuestas (sube/baja/hold)
- Chat en vivo y leaderboard

**`client.js`** (main.js):
- Escucha eventos Socket.IO: `price_update`, `round_open`, `round_result`
- Actualiza gráficas y UI en tiempo real
- Maneja interacciones de usuario (apostar, chat)

**`styles.css`**:
- Estilos responsive para la interfaz

---

### Resumen de Funcionalidad por Instancia:

| Instancia | Función Principal | Líneas |
|-----------|-------------------|---------|
| **app.js** | Entry point, configura Express + Socket.IO | ~196 |
| **marketService.js** | Ciclo de rondas, precios, resolución | ~320 |
| **betService.js** | Validar y procesar apuestas | ~92 |
| **RedisRepository.js** | Persistencia en Redis | ~150 |
| **kafkaMirror.js** | Conexión a Kafka | ~100 |
| **registerSocketHandlers.js** | Tiempo real con Socket.IO | ~200 |
| **binance_kafka_producer.py** | Puente Binance → Kafka | ~130 |

**Total: 23 archivos en `src/`** cubriendo toda la arquitectura del proyecto.

---

## 9) Anexos

### URLs del Proyecto:
- **Aplicación principal**: http://localhost:3000
- **Kafka UI**: http://localhost:8080
- **Redis**: localhost:6379
- **Kafka Broker**: localhost:9092

### Estructura de Archivos Clave:
```
GambleBitCoin/
├── app.js                          # Entry point principal
├── docker-compose.yml              # Orquestación de servicios
├── Dockerfile                      # Imagen de la app
├── .env                            # Configuración
├── src/
│   ├── services/
│   │   ├── marketService.js        # Gestión de rondas y precios
│   │   ├── betService.js           # Lógica de apuestas
│   │   └── userService.js         # Gestión de usuarios
│   ├── streams/
│   │   ├── binanceKafkaProducer.js # Spawn de Python
│   │   └── kafkaStreamConsumer.js  # Consumidor Kafka
│   ├── repositories/
│   │   └── RedisRepository.js      # Acceso a Redis
│   ├── sockets/
│   │   └── registerSocketHandlers.js # Eventos Socket.IO
│   └── public/
│       └── main.js                 # Frontend JavaScript
└── spark/                          # (No usado actualmente)
```

### Comandos Útiles:
```powershell
# Iniciar todos los servicios
docker-compose up -d

# Ver logs de la aplicación
docker logs gamblebitcoin --tail 50

# Reiniciar la aplicación
docker-compose restart gamblebitcoin

# Conectar a Redis CLI
docker exec -it mi-redis redis-cli

# Ver topics en Kafka UI
# Ir a http://localhost:8080
```

### Configuración Crítica (.env):
```env
# Tiempos de ronda
ROUND_SECONDS=20   # 20 segundos para apostar
LOCK_SECONDS=5     # 5 segundos de bloqueo

# Límites de apuestas
BET_MIN=10
BET_MAX=300
INITIAL_BALANCE=2000

# Respaldo de precios
ENABLE_PRICE_FALLBACK=true  # Activar generador simulado
PRICE_STALE_MS=12000        # 12 segundos sin precio = usar fallback
```

---

## 10) Cómo Funciona Docker y Funcionalidad de sus Contenedores

### Teoría de Docker Compose:

**Docker Compose** permite definir y ejecutar **múltiples contenedores** como una aplicación única. En este proyecto:

1. **Definición**: `docker-compose.yml` define 5 servicios (contenedores)
2. **Orquestación**: Administra redes, volúmenes y dependencias entre contenedores
3. **Ejecución**: `docker-compose up -d` inicia todos los servicios en segundo plano

### Red Docker:

Todos los contenedores se conectan a una red interna (`gamblebitcoin_default`) y pueden comunicarse usando **nombres de contenedor** como hostnames:
- `mi-redis:6379` (en lugar de `localhost:6379` dentro de Docker)
- `kafka:29092` (broker interno)

---

### Contenedores y su Funcionalidad:

#### 10.1) **mi-redis** - Redis 7.2 Alpine
```yaml
image: redis:7.2-alpine
ports: "6379:6379"
```

**Funcionalidad:**
- Base de datos en memoria para **persistencia rápida**
- Almacena: saldos de usuarios, rondas actuales, historial, chat, leaderboard
- El nombre `mi-redis` se usa en otros contenedores como hostname

**Datos almacenados (RedisRepository.js):**
```javascript
user:{id} → Hash { id, name, balance, blocked }
round:{symbol}:current → Hash { id, startPrice, endPrice, status }
round:{id}:{symbol}:bets → Hash de apuestas
chat:{symbol}:messages → Lista de mensajes
leaderboard:{symbol} → Sorted Set (ranking)
```

**Comando útil:**
```powershell
docker exec -it mi-redis redis-cli
> KEYS *
> HGETALL user:123
```

---

#### 10.2) **zookeeper-1** - Confluent Zookeeper 7.5.0
```yaml
image: confluentinc/cp-zookeeper:7.5.0
environment:
  ZOOKEEPER_CLIENT_PORT: 2181
```

**Funcionalidad:**
- **Coordinador de Kafka** (requerido por Kafka 7.5.0)
- Gestiona: brokers, topics, particiones, configuración del cluster
- Puerto: `2181` (interno, no expuesto al host)
- **No se usa directamente**, es infraestructura de soporte para Kafka

**Nota:** Kafka moderno puede funcionar sin Zookeeper (KRaft mode), pero este proyecto usa la versión tradicional.

---

#### 10.3) **kafka-1** - Confluent Kafka 7.5.0
```yaml
image: confluentinc/cp-kafka:7.5.0
ports: "9092:9092"
environment:
  KAFKA_BROKER_ID: 1
  KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:29092
  KAFKA_NUM_PARTITIONS: 3
```

**Funcionalidad:**
- **Message broker** para el pipeline de streaming
- **Topics configurados:**
  - `market.prices.raw` - Precios de Binance (3 particiones)
  - `market.bets.events` - Eventos de apuestas
  - `market.round.events` - Eventos de rondas
  - `market.alerts` - Alertas de anomalías

**Listeners (2 configurados):**
1. `localhost:9092` - Para aplicaciones en el HOST (fuera de Docker)
2. `kafka:29092` - Para contenedores Docker internos

**Comandos útiles:**
```powershell
# Ver topics
docker exec -it kafka-1 kafka-topics --bootstrap-server localhost:9092 --list

# Consumir topic
docker exec -it kafka-1 kafka-console-consumer --bootstrap-server localhost:9092 --topic market.prices.raw
```

---

#### 10.4) **kafka-ui-1** - Kafka UI Latest
```yaml
image: provectuslabs/kafka-ui:latest
ports: "8080:8080"
environment:
  KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
```

**Funcionalidad:**
- **Interfaz gráfica web** para monitorear Kafka
- URL: http://localhost:8080
- Permite:
  - Ver todos los topics y particiones
  - Leer mensajes en tiempo real
  - Ver configuración de brokers
  - Monitorear consumidores y lag

**Uso:** Abrir http://localhost:8080 en el navegador

---

#### 10.5) **gamblebitcoin** - Aplicación Node.js (Build from Dockerfile)
```yaml
build: .
container_name: gamblebitcoin
depends_on:
  - mi-redis
  - kafka
environment:
  REDIS_URL: redis://mi-redis:6379
  KAFKA_BROKERS: kafka:29092
  PORT: 3000
ports: "3000:3000"
```

**Dockerfile (construcción de la imagen):**
```dockerfile
FROM node:18-alpine
RUN apk add --no-cache python3 py3-pip
RUN pip3 install --no-cache-dir --break-system-packages python-binance kafka-python
COPY package*.json ./
RUN rm -f package-lock.json && npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

**Funcionalidad:**
- **Aplicación principal** que integra todo
- Inicia `app.js` que:
  1. Conecta a Redis (`mi-redis:6379`)
  2. Conecta a Kafka (`kafka:29092`)
  3. Spawnea Python para Binance WebSocket
  4. Inicia servidor HTTP en puerto 3000
  5. Configura Socket.IO para tiempo real
- **Depende de:** `mi-redis` y `kafka` (espera a que estén listos)

**URLs:**
- App: http://localhost:3000
- API Health: http://localhost:3000/health

---

### Diagrama de Comunicación:

```
[Binance API]
     |
     v (WebSocket)
[gamblebitcoin container] ← Python script envía a Kafka
     |
     +--→ [kafka-1:9092] → Topics: market.prices.raw
     |         ^
     |         | (consume)
     +--→ [gamblebitcoin] → Procesa precios, apuestas
     |         |
     |         v (Socket.IO)
     +--→ [Frontend:3000] → Gráficas, apuestas, chat
     |
     +--→ [mi-redis:6379] → Persistencia (usuarios, rondas)
     |
     +--→ [kafka-1] → Topics: market.bets.events, market.alerts
     
[kafka-ui-1:8080] ← Monitorea [kafka-1]
[zookeeper-1:2181] ← Coordina [kafka-1]
```

---

### Comandos Clave:

```powershell
# Iniciar todos los servicios
docker-compose up -d

# Ver logs de la app
docker logs gamblebitcoin --tail 50

# Reiniciar un servicio
docker-compose restart gamblebitcoin

# Ver todos los contenedores
docker ps

# Detener todo
docker-compose down
```

---

### Resumen de Puertos:

| Servicio | Puerto Host | Puerto Contenedor | Uso |
|----------|--------------|-------------------|-----|
| **gamblebitcoin** | 3000 | 3000 | Aplicación web |
| **mi-redis** | 6379 | 6379 | Redis DB |
| **kafka-1** | 9092 | 9092/29092 | Kafka Broker |
| **kafka-ui-1** | 8080 | 8080 | Interfaz Kafka |
| **zookeeper-1** | - | 2181 | Coordinación (interno) |

---

### Nota Importante:
Los contenedores se comunican internamente usando los nombres definidos en `container_name:` (mi-redis, kafka-1, zookeeper-1), no `localhost`.

---

## 11) Anexos

### URLs del Proyecto:
