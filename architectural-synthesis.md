# Architectural Synthesis

Deploying a multi-node Apache Kafka cluster across heterogeneous hardware provides valuable insights into streaming architectures, container orchestration, and consensus protocols:

RHEL 10 Podman Quadlets eliminate brittle container startup scripts by converting declarative `.container` files directly into systemd services. Combining this native service management with SELinux context configuration (:Z) provides a reliable foundation for stateful broker containers on enterprise Linux.

KRaft mode removes the need for ZooKeeper, simplifying cluster management. However, administrators must plan controller topologies around Raft quorum requirements ($Q = \lfloor N/2 \rfloor + 1$). While a two-node cluster provides active-active data replication, it requires an odd number of controllers ($N = 3$) to maintain metadata consensus through a node failure.

Client network design must reflect Kafka's two-phase connection process. While DNS round-robin can resolve initial bootstrap requests, partition data routing relies entirely on the hostnames or IP addresses advertised in advertised.listeners.

Consumer application design requires balancing batch fetch sizes (`num_messages`) against processing delays to prevent `max.poll.interval.ms` timeouts. Configuring these parameters properly prevents slow consumers from causing unnecessary consumer group rebalances and partition reassignments.

Finally, disaster recovery strategies must protect both partition data and the unified `__cluster_metadata` log. Quiesced cold backups and LVM snapshots ensure consistent point-in-time recovery, while MirrorMaker 2 enables continuous, active-passive cross-cluster replication without interrupting running services.
