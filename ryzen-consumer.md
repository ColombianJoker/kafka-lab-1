# QNAP Ryzen Consumer Architecture and Flow Control

The consumer service runs on a QNAP NAS powered by an x86_64 AMD Ryzen Embedded processor, deployed via QNAP Container Station.

### Flow-Rate Imbalance and Lag Dynamics

The lab configuration deliberately creates an ingestion-consumption throughput mismatch. Two independent producer containers emit messages with sleep intervals between $0$ and $0.5$ seconds, yielding an expected sleep duration of:

$$
E[S_p] = \frac{0.5}{2} = 0.25\text{ seconds}
$$

This produces an average emission rate per producer of $\lambda_p = 4\text{ msgs/sec}$, leading to a combined cluster arrival rate of:

$$
\Lambda = 2 \times 4 = 8\text{ msgs/sec}
$$

The consumer processes messages with sleep intervals between $0$ and $2.0$ seconds, resulting in an expected processing time of:

$$
E[S_c] = \frac{2.0}{2} = 1.0\text{ second}
$$

This limits the single-threaded consumer's service rate to:

$$
\mu = \frac{1}{E[S_c]} = 1\text{ msg/sec}
$$

Because the traffic intensity $\rho = \frac{\Lambda}{\mu} = 8 > 1$, the consumer operates in an overloaded state. Unprocessed messages accumulate steadily in the partition logs, creating a growing lag metric ($L = \text{LEO} - \text{Offset}$) that allows operators to observe consumer group offset commits, partition rebalances, and recovery behavior under load.

### Preventing Group Evictions and `max.poll.interval.ms` Timeouts

When a consumer loop sleeps for up to two seconds while processing individual records from a large fetch batch, it risks triggering an involuntary group rebalance. The configuration parameter `max.poll.interval.ms` sets the upper limit on the elapsed time between consecutive calls to `consumer.poll()` or `consumer.consume()`. If the batch intake size defaults to 500 records:

$$
\text{Processing Duration} = 500 \times 1.0\text{s (mean)} = 500\text{ seconds}
$$

Because 500 seconds exceeds the default 300-second threshold, the cluster coordinator will presume the processing thread has crashed. The coordinator then revokes the consumer's partition assignments and triggers a costly group rebalance (`Application maximum poll interval exceeded`).
To maintain group membership during extended message processing, the consumer application must bound its batch intake size using num_messages=5 while tuning `max.poll.interval.ms` to comfortably exceed the maximum possible batch processing duration ($5 \text{ records} \times 2.0\text{s (max)} = 10\text{ seconds}$).

### Consumer Source Code (`consumer.py`)

Save the following consumer application as `/share/Container/kafka-consumer/consumer.py`:

```python
import json
import logging
import os
import random
import signal
import sys
import time
from confluent_kafka import Consumer, KafkaError, KafkaException

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] [CONSUMER] %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger("qnap-consumer")

def main():
    bootstrap_servers = os.getenv("BOOTSTRAP_SERVERS", "kafka1.lab.internal:9092,kafka2.lab.internal:9092")
    group_id = os.getenv("GROUP_ID", "qnap-analytics-group")
    topic = os.getenv("TARGET_TOPIC", "lab-telemetry")

    conf = {
        "bootstrap.servers": bootstrap_servers,
        "group.id": group_id,
        "auto.offset.reset": "earliest",
        "enable.auto.commit": False,          # Manual synchronous offset control
        "max.poll.interval.ms": 60000,        # 60-second limit before group eviction
        "session.timeout.ms": 15000,          # Heartbeat timeout
    }

    consumer = Consumer(conf)
    consumer.subscribe([topic])

    running = True
    def stop_signal_handler(signum, frame):
        nonlocal running
        logger.info("Termination signal received. Exiting loop...")
        running = False

    signal.signal(signal.SIGTERM, stop_signal_handler)
    signal.signal(signal.SIGINT, stop_signal_handler)

    logger.info(f"Consumer started. Subscribed to topic: {topic}")

    try:
        while running:
            # Fetch a small slice of records to maintain consistent poll intervals
            records = consumer.consume(num_messages=5, timeout=1.0)
            if not records:
                continue

            for msg in records:
                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        continue
                    else:
                        raise KafkaException(msg.error())

                payload = json.loads(msg.value().decode("utf-8"))
                logger.info(
                    f"Consumed [P{msg.partition()}:Off{msg.offset()}] "
                    f"Src:{payload.get('producer_id')} User:{payload.get('user')} "
                    f"Val:${payload.get('transaction_value')}"
                )

                # Simulated processing delay: random fraction of two seconds
                processing_delay = random.uniform(0.0, 2.0)
                time.sleep(processing_delay)

            # Synchronously commit offsets after completing the batch
            consumer.commit(asynchronous=False)

    except Exception as e:
        logger.error(f"Fatal consumer error: {e}")
    finally:
        logger.info("Closing consumer and revoking partition assignments...")
        consumer.close()
        logger.info("Consumer terminated cleanly.")

if __name__ == "__main__":
    main()
```

### QNAP Container Station Packaging and Orchestration

Create `/share/Container/kafka-consumer/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install --no-cache-dir confluent-kafka

COPY consumer.py /app/consumer.py

CMD ["python", "-u", "/app/consumer.py"]
```

Create `/share/Container/kafka-consumer/docker-compose.yml` for execution in Container Station:

```yaml
services:
  telemetry-consumer:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: qnap-kafka-consumer
    restart: unless-stopped
    environment:
      - BOOTSTRAP_SERVERS=kafka1.lab.internal:9092,kafka2.lab.internal:9092
      - GROUP_ID=qnap-analytics-group
      - TARGET_TOPIC=lab-telemetry
    extra_hosts:
      - "kafka1.lab.internal:192.168.1.11"
      - "kafka2.lab.internal:192.168.1.12"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

In the QNAP Container Station interface, create an application pointing to `/share/Container/kafka-consumer`, or start the stack directly over SSH using Container Station's Docker Compose CLI:

```bash
docker compose up -d --build
```
