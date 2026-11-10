# Cluster Verification, Testing, and Failure Injection Runbook

To confirm the operational stability of the end-to-end environment, administrators must execute standard verification workflows, observe real-time consumer lag, and evaluate cluster behavior during simulated hardware failures.

### Topic Provisioning and Partition Distribution

Provision a topic configured with 2 partitions and a replication factor of 2, ensuring both physical brokers host partition replicas:

```bash
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server kafka1.lab.internal:9092 \
    --create \
    --topic lab-telemetry \
    --partitions 2 \
    --replication-factor 2 \
    --config min.insync.replicas=1
```

Confirm that partition leadership and replica placements are distributed evenly across both physical hosts:

```bash
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server kafka1.lab.internal:9092 \
    --describe \
    --topic lab-telemetry
```

The resulting output displays partition distribution, replica assignments, and the in-sync replica (ISR) set across nodes:

```text
Topic: lab-telemetry TopicId: 9vHkPq2jSraB8_8L1aZ91w PartitionCount: 2 ReplicationFactor: 2 Configs: min.insync.replicas=1
Topic: lab-telemetry Partition: 0 Leader: 1 Replicas: 1,2 Isr: 1,2
Topic: lab-telemetry Partition: 1 Leader: 2 Replicas: 2,1 Isr: 2,1
```

### Observing Consumer Lag and Member States

Because the producers emit data at an aggregate rate of approximately 8 messages per second while the consumer processes roughly 1 message per second, consumer lag will grow over time. The state of partition ownership and offset progression can be monitored using `kafka-consumer-groups.sh`:

```bash
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-consumer-groups.sh \
    --bootstrap-server kafka1.lab.internal:9092 \
    --describe \
    --group qnap-analytics-group
```

The administrative diagnostic output reveals real-time lag values:

```text
GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID
qnap-analytics-group lab-telemetry 0 128 512 384 consumer-qnap-analytics-group-1-987a... /192.168.1.50 consumer-1
qnap-analytics-group lab-telemetry 1 110 498 388 consumer-qnap-analytics-group-1-987a... /192.168.1.50 consumer-1
```

This output confirms that the single consumer instance on the QNAP NAS has claimed ownership of both topic partitions. The positive values in the LAG column reflect the designed rate difference between producer ingestion and consumer processing.

### Dynamic Failure Injection: Simulating Node Outages

Simulate an abrupt hardware failure by halting the broker service on Physical Host 2:

```bash
sudo systemctl stop kafka-broker.service
```

#### Diagnostic Evaluation

Inspecting the surviving node on Physical Host 1 reveals several operational changes:
First, both producers and consumers will experience brief transport interruptions during partition leader re-election before reconnecting to `kafka1.lab.internal:9092`. The surviving broker continues serving produce and fetch traffic for Partition 0. Partition 1 will temporarily reject writes if unclean leader elections are disabled.

Second, inspecting the topic demonstrates that the in-sync replica set shrinks to reflect the loss of Host 2:

```bash
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server kafka1.lab.internal:9092 \
    --describe \
    --topic lab-telemetry
```

Partition 0 remains fully operational with `Isr: 1`, while Partition 1 displays an offline leader state until Host 2 is restored.
Third, querying the metadata quorum status reveals that consensus is frozen:

```bash
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-metadata-quorum.sh \
    --bootstrap-server kafka1.lab.internal:9092 \
    --describe --status
```

Because the two-node cluster requires both votes to form a majority quorum ($Q = 2$), the loss of Host 2 prevents the remaining controller from committing changes. Attempting to create a new topic will block until the request times out:

```bash
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server kafka1.lab.internal:9092 \
    --create \
    --topic failure-test \
    --partitions 1 \
    --replication-factor 1
```

The command yields `org.apache.kafka.common.errors.TimeoutException`, confirming that the controller cannot commit new metadata to `__cluster_metadata` without a surviving majority quorum.

### Quorum Recovery

Bring Physical Host 2 back online:

```bash
sudo systemctl start kafka-broker.service
```

Within seconds of reconnecting, the controllers re-establish Raft consensus, synchronize missing log segments, expand the topic in-sync replica set back to `Isr: 1,2`, and resume normal cluster operations without manual administrative steps.
