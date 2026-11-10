# ARM64 Producer Specification and Deployment

The producer service runs within an ARM64 RHEL 10 virtual machine hosted by VMware Fusion on macOS. To test multi-source data ingestion, two independent container instances execute identical application logic while using separate identifiers (`producer-alpha` and `producer-bravo`).
### Client Library Selection
The producer is built with `confluent-kafka`, which wraps the high-performance C library `librdkafka`. Unlike pure Python implementations, `librdkafka` handles memory management natively, provides asynchronous batching optimizations (`linger.ms`, `batch.num.messages`), and supplies pre-compiled manylinux binary wheels for 64-bit ARM architectures (aarch64).
### Ingestion Source Code (`producer.py`)
Save the following Python script as `/opt/kafka-lab/producer/producer.py`:
```python
import json
import logging
import os
import random
import signal
import sys
import time
from faker import Faker
from confluent_kafka import Producer

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] [PRODUCER:%(name)s] %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger(os.getenv("PRODUCER_ID", "unidentified-producer"))

def delivery_report(err, msg):
    """Callback invoked by librdkafka on delivery success or permanent failure."""
    if err is not None:
        logger.error(f"Message delivery failed: {err}")
    else:
        logger.debug(
            f"Message delivered to {msg.topic()} [{msg.partition()}] at offset {msg.offset()}"
        )

def main():
    producer_id = os.getenv("PRODUCER_ID", "producer-default")
    bootstrap_servers = os.getenv("BOOTSTRAP_SERVERS", "kafka1.lab.internal:9092,kafka2.lab.internal:9092")
    target_topic = os.getenv("TARGET_TOPIC", "lab-telemetry")
    max_small_sleep = float(os.getenv("MAX_SMALL_SLEEP", "0.5"))

    logger.info(f"Initializing producer instance: {producer_id}")
    logger.info(f"Targeting cluster bootstrap: {bootstrap_servers}")

    conf = {
        "bootstrap.servers": bootstrap_servers,
        "client.id": producer_id,
        "acks": "all",
        "retries": 5,
        "linger.ms": 10,
        "batch.num.messages": 100,
        "queue.buffering.max.messages": 100000,
    }

    producer = Producer(conf)
    fake = Faker()

    running = True
    def stop_signal_handler(signum, frame):
        nonlocal running
        logger.info("Shutdown signal caught. Draining outgoing queues...")
        running = False

    signal.signal(signal.SIGTERM, stop_signal_handler)
    signal.signal(signal.SIGINT, stop_signal_handler)

    while running:
        try:
            # Process background delivery callbacks to release internal buffer memory
            producer.poll(0.0)

            payload = {
                "producer_id": producer_id,
                "event_id": fake.uuid4(),
                "timestamp_ns": time.time_ns(),
                "user": fake.name(),
                "email": fake.free_email(),
                "ipv4": fake.ipv4(),
                "transaction_value": round(random.uniform(5.0, 750.0), 2)
            }

            key_bytes = payload["event_id"].encode("utf-8")
            val_bytes = json.dumps(payload).encode("utf-8")

            # Route message asynchronously using Murmur2 hash on the partition key
            producer.produce(
                topic=target_topic,
                key=key_bytes,
                value=val_bytes,
                on_delivery=delivery_report
            )

            # Randomized throttle between 0 and MAX_SMALL_SLEEP
            sleep_duration = random.uniform(0.0, max_small_sleep)
            time.sleep(sleep_duration)

        except BufferError:
            logger.warning("Local queue full. Flushing and backing off...")
            producer.poll(0.5)
            time.sleep(0.1)
        except Exception as e:
            logger.error(f"Unexpected transmission error: {e}")
            break

    logger.info("Flushing buffer before exit...")
    producer.flush(timeout=10.0)
    logger.info("Producer shut down cleanly.")

if __name__ == "__main__":
    main()
```	

### Container Packaging (Dockerfile)
Create `/opt/kafka-lab/producer/Dockerfile`:

```dockerfile

FROM registry.access.redhat.com/ubi9/python-311:latest

WORKDIR /app

RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir confluent-kafka Faker

COPY producer.py /app/producer.py

USER 1001

CMD ["python3", "/app/producer.py"]
```

Build the native ARM64 image using Podman:
```bash

podman build -t localhost/kafka-lab-producer:latest /opt/kafka-lab/producer/
```

### Templated Systemd Quadlet Deployment
Using systemd template specifiers (`%i`), a single Quadlet definition automates deploying and managing both `producer-alpha` and `producer-bravo`containers.
Create `/etc/containers/systemd/kafka-producer@.container` on the ARM64 RHEL 10 VM:

```toml
[Unit]
Description=Kafka Telemetry Producer Instance %i
After=network-online.target
Wants=network-online.target

[Container]
Image=localhost/kafka-lab-producer:latest
ContainerName=kafka-producer-%i
Environment=PRODUCER_ID=producer-%i
Environment=BOOTSTRAP_SERVERS=kafka1.lab.internal:9092,kafka2.lab.internal:9092
Environment=TARGET_TOPIC=lab-telemetry
Environment=MAX_SMALL_SLEEP=0.5

[Service]
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Reload systemd and start both producer instances:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kafka-producer@alpha.service
sudo systemctl enable --now kafka-producer@bravo.service
```
