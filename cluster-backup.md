# Cluster Backup, Snapshotting, and Disaster Recovery Framework

Backing up an Apache Kafka cluster running in KRaft mode requires coordinating persistent storage states with active file locks. The metadata partition log (`__cluster_metadata-0`) stores the entire cluster state, including dynamic broker registrations, topic partitions, and authorization metadata. Capturing data partition logs while leaving the metadata partition in an inconsistent state can cause unrecoverable corruption and prevent the cluster from restarting.

| **Strategy**                     | **Recovery Point Objective (RPO)**    | **Recovery Time Objective (RTO)**                | **Operational State Required**                            | **Complexity** |
| -------------------------------- | ------------------------------------- | ------------------------------------------------ | --------------------------------------------------------- | -------------- |
| **Quiesced Physical Snapshot**   | Point of shutdown/snapshot execution  | Minutes (storage extraction and container boot)  | System service stopped or frozen via fsfreeze             | Low            |
| **LVM Thin Snapshotting**        | Sub-second snapshot capture window    | Minutes (mount, tar copy, and restore)           | Filesystem frozen for milliseconds during volume creation | Medium         |
| **MirrorMaker 2 Active-Passive** | Sub-second continuous log replication | Seconds to minutes (consumer offset redirection) | Online primary and secondary clusters                     | High           |

### Point-in-Time Filesystem Snapshots on RHEL 10

Kafka uses memory-mapped files (`mmap`) to write broker page cache data to disk. Performing an uncoordinated `rsync` or `tar` backup while the broker is actively writing will result in torn writes, producing mismatched file lengths and invalid checksums between index files (`.index`, `.timeindex`) and `.log` segment files.

#### Consistent Cold Backup Procedure

To ensure an uncorrupted physical state, pause the container service to flush volatile page buffers and finalize checkpoint files:

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DEST="/mnt/nfs-backups/kafka-node-1/$(date +%Y%m%d_%H%M%S)"
mkdir -p "${BACKUP_DEST}"

# Step 1: Stop the broker service to flush mmap buffers and checkpoint logs
sudo systemctl stop kafka-broker.service

# Step 2: Archive log segments, checkpoints, and meta.properties, excluding locks
tar -czvf "${BACKUP_DEST}/kafka-broker-storage.tar.gz" \
    --exclude="/var/lib/kafka/data/*.lock" \
    /var/lib/kafka/data

# Step 3: Resume broker operations
sudo systemctl start kafka-broker.service

logger -t KAFKA_BACKUP "Node 1 storage successfully archived to ${BACKUP_DEST}"
```

#### Consistent Hot Snapshots via Filesystem Freezing and LVM

When the broker's data directory resides on an LVM logical volume, administrators can capture consistent backups without stopping the service by freezing filesystem I/O:

```bash
# Flush pending page cache writes and freeze filesystem I/O operations
sudo fsfreeze -f /var/lib/kafka/data

# Immediately generate an LVM snapshot of the storage volume
sudo lvcreate --snapshot --name kafka_snap_node1 --size 5G /dev/vg_kafka/lv_data

# Unfreeze filesystem operations; I/O pauses for only milliseconds
sudo fsfreeze -u /var/lib/kafka/data

# Mount the snapshot volume read-only and copy its contents to backup storage
sudo mount -o ro /dev/vg_kafka/kafka_snap_node1 /mnt/snapshot
sudo tar -czvf "/mnt/backups/kafka-node1-consistent.tar.gz" -C /mnt/snapshot .
sudo umount /mnt/snapshot
sudo lvremove -y /dev/vg_kafka/kafka_snap_node1
```

### Continuous Replication with MirrorMaker 2

For production disaster recovery where broker services must run without interruption, MirrorMaker 2 (MM2) provides continuous, asynchronous cross-cluster replication. Built on the Kafka Connect framework, MM2 uses dedicated connectors to replicate data, topic configurations, and consumer group offsets between clusters.
The `MirrorSourceConnector` streams records from primary cluster topics to a designated backup cluster, while the `MirrorCheckpointConnector` tracks and translates consumer group offsets across clusters. If the primary cluster suffers an unrecoverable failure, consumer applications can fail over to the disaster recovery target and resume reading from their last synchronized offset positions.
Save the following configuration as `mm2.properties`:

```properties

# Cluster connection definitions
clusters = primary, dr
primary.bootstrap.servers = kafka1.lab.internal:9092,kafka2.lab.internal:9092
dr.bootstrap.servers = dr-kafka.lab.internal:9092

# Enable replication flows from primary to disaster recovery cluster
primary->dr.enabled = true
primary->dr.topics = lab-telemetry
primary->dr.groups = qnap-analytics-group

# Connector task allocation and internal topic replication settings
tasks.max = 2
replication.factor = 1
checkpoints.topic.replication.factor = 1
heartbeats.topic.replication.factor = 1
offset-syncs.topic.replication.factor = 1

# Consumer offset synchronization settings
sync.topic.acls.enabled = false
emit.checkpoints.interval.seconds = 5
sync.group.offsets.enabled = true
sync.group.offsets.interval.seconds = 5
```

Deploy the MirrorMaker 2 instance inside an administrative container:

```bash

podman run -d --name mirrormaker2 \
  --network host \
  -v ./mm2.properties:/opt/kafka/config/mm2.properties:Z \
  docker.io/apache/kafka:latest \
  /opt/kafka/bin/connect-mirror-maker.sh /opt/kafka/config/mm2.properties
```

### Disaster Recovery Runbook: Recovering from Total Storage Failure

If a disk failure corrupts the data directory on a broker node, execute the following recovery procedure:

```bash
# Step 1: Stop the degraded broker unit
sudo systemctl stop kafka-broker.service

# Step 2: Clear all corrupted segments and index files from the host path
sudo rm -rf /var/lib/kafka/data/*

# Step 3: Extract the consistent archive into the storage root
sudo tar -xzvf /mnt/nfs-backups/kafka-node-1/latest/kafka-broker-storage.tar.gz -C /

# Step 4: Verify the cluster ID matches the cluster's defined configuration
cat /var/lib/kafka/data/meta.properties

# Step 5: Start the broker service
sudo systemctl start kafka-broker.service

# Step 6: Validate quorum convergence and controller consensus
podman exec -it kafka-node-1 /opt/kafka/bin/kafka-metadata-quorum.sh \
    --bootstrap-server kafka1.lab.internal:9092 describe --status
```
