# Cluster Infrastructure Deployment on RHEL 10 with Podman Quadlets

Red Hat Enterprise Linux 10 integrates container execution directly into **systemd** using Podman Quadlets. Quadlets replace brittle shell startup scripts by translating declarative `.container` files into managed systemd service units during systemctl daemon-reload.

### Storage Preparation and SELinux Enforcement

RHEL 10 enforces Mandatory Access Control via SELinux. When bind-mounting host directories into Kafka containers to store partition logs and metadata segments, the underlying directories must be labeled with the container storage context (container_file_t). Appending the :Z flag to the volume definition informs Podman to configure an unshared private SELinux label on the host path.

On **Physical Host 1** (192.168.1.11 / kafka1.lab.internal):

```bash
sudo mkdir -p /var/lib/kafka/data
sudo chown -R 1000:1000 /var/lib/kafka/data
sudo chmod -R 750 /var/lib/kafka/data
```

On **Physical Host 2** (192.168.1.12 / kafka2.lab.internal):

```bash
sudo mkdir -p /var/lib/kafka/data
sudo chown -R 1000:1000 /var/lib/kafka/data
sudo chmod -R 750 /var/lib/kafka/data
```

### Static Storage Formatting and Cluster ID Generation

All KRaft nodes in a cluster must share an identical, base64-encoded 16-byte UUID `cluster.id` to establish communication. Generating this UUID beforehand guarantees deterministic initialization across both physical hosts without relying on dynamic cluster formatting tools:

```bash
export LAB_CLUSTER_ID="4L6g3nShT-eMCtK--X86sw"
```

### Quadlet Unit Declarations

On RHEL 10, cluster container units are managed system-wide by placing files in `/etc/containers/systemd/`.

#### Node 1 Specification

Create `/etc/containers/systemd/kafka-broker.container `on **Physical Host 1**:

```toml
[Unit]
Description=Apache Kafka KRaft Broker and Controller Node 1
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/apache/kafka:latest
ContainerName=kafka-node-1
Network=host
Volume=/var/lib/kafka/data:/var/lib/kafka/data:Z

# Core KRaft Topology Configuration
Environment=CLUSTER_ID=4L6g3nShT-eMCtK--X86sw
Environment=KAFKA_NODE_ID=1
Environment=KAFKA_PROCESS_ROLES=broker,controller
Environment=KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka1.lab.internal:9093,2@kafka2.lab.internal:9093

# Networking and Protocol Listeners
Environment=KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
Environment=KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka1.lab.internal:9092
Environment=KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
Environment=KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
Environment=KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# Cluster Replication and Log Defaults
Environment=KAFKA_LOG_DIRS=/var/lib/kafka/data
Environment=KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=2
Environment=KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=2
Environment=KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
Environment=KAFKA_DEFAULT_REPLICATION_FACTOR=2
Environment=KAFKA_NUM_PARTITIONS=2

[Service]
Restart=on-failure
RestartSec=10s
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

#### Node 2 Specification

Create `/etc/containers/systemd/kafka-broker.container` on **Physical Host 2**:
Ini, TOML

```toml
[Unit]
Description=Apache Kafka KRaft Broker and Controller Node 2
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/apache/kafka:latest
ContainerName=kafka-node-2
Network=host
Volume=/var/lib/kafka/data:/var/lib/kafka/data:Z

# Core KRaft Topology Configuration
Environment=CLUSTER_ID=4L6g3nShT-eMCtK--X86sw
Environment=KAFKA_NODE_ID=2
Environment=KAFKA_PROCESS_ROLES=broker,controller
Environment=KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka1.lab.internal:9093,2@kafka2.lab.internal:9093

# Networking and Protocol Listeners
Environment=KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
Environment=KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka2.lab.internal:9092
Environment=KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
Environment=KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT
Environment=KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# Cluster Replication and Log Defaults
Environment=KAFKA_LOG_DIRS=/var/lib/kafka/data
Environment=KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=2
Environment=KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=2
Environment=KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
Environment=KAFKA_DEFAULT_REPLICATION_FACTOR=2
Environment=KAFKA_NUM_PARTITIONS=2

[Service]
Restart=on-failure
RestartSec=10s
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

### Cluster Service Initialization

With the declarative Quadlet units placed in the systemd library directory, the generator must be executed to construct active service units:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kafka-broker.service
```

Verification of daemon startup and KRaft communication is conducted using standard systemd utilities:

```bash
systemctl status kafka-broker.service
journalctl -u kafka-broker.service -f
```
