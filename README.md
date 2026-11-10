# Topology Overview and Architectural Foundations

Establishing a resilient Apache Kafka testbed across heterogeneous physical hardware and virtualized execution environments requires strict separation between the cluster consensus plane, client data planes, and underlying network routing. The laboratory architecture encompasses three distinct tiers: ARM64 virtualization hosting telemetry publishers, dual bare-metal enterprise Linux hosts running a distributed Kafka cluster, and an x86_64 network-attached storage platform hosting a group consumer.

| **Host Architecture**         | **Operating System**      | **Laboratory Role**                        | **Execution Engine**               | **Primary Network Services**               |
| ----------------------------- | ------------------------- | ------------------------------------------ | ---------------------------------- | ------------------------------------------ |
| Apple Silicon (ARM64)         | RHEL 10 (VMware Fusion)   | Data Ingestion (Producers Alpha and Bravo) | Podman (Rootless Systemd Quadlets) | Dnsmasq DNS Resolver (Port 53)             |
| Bare-Metal Physical Host 1    | RHEL 10 (x86_64 or ARM64) | Kafka Broker 1 / KRaft Controller 1        | Podman (System-Level Quadlets)     | Broker (Port 9092), Controller (Port 9093) |
| Bare-Metal Physical Host 2    | RHEL 10 (x86_64 or ARM64) | Kafka Broker 2 / KRaft Controller 2        | Podman (System-Level Quadlets)     | Broker (Port 9092), Controller (Port 9093) |
| QNAP NAS (AMD Ryzen Embedded) | QTS / QuTS (x86_64)       | Data Siphon (Consumer Group Member)        | Container Station (Docker Compose) | Kafka Consumer Daemon                      |

### KRaft Quorum Dynamics in a Two-Node Topology

In Kafka Raft (KRaft) metadata mode, external ZooKeeper ensembles are replaced by an internal event-driven metadata log (`__cluster_metadata`) managed by an active controller leader and replicated to follower controllers. Raft consensus requires a strict majority quorum $Q$ defined by:

$$
Q = \left\lfloor \frac{N}{2} \right\rfloor + 1
$$

where $N$ represents the total number of voting members designated within the `controller.quorum.voters` parameter. When operating exactly two physical nodes configured with combined broker and controller roles (`process.roles=broker,controller`), $N = 2$, yielding a required quorum of:

$$
Q = \left\lfloor \frac{2}{2} \right\rfloor + 1 = 2
$$

In this two-voter configuration, both controllers must remain online and actively acknowledge metadata writes for the cluster to maintain consensus. If either physical host experiences a hardware failure, operating system crash, or network partition, the surviving controller provides only a single vote ($1 < Q$). Under these conditions, the surviving controller cannot elect itself leader or commit records to `__cluster_metadata`.
Existing partition leaders on the surviving broker will continue serving produce and fetch requests for established topics, ensuring continuous data path operations. However, all administrative mutations—such as topic provisioning, partition reassignments, leader elections for partitioned replicas, and dynamic configuration updates—halt immediately.
Achieving true metadata fault tolerance against a single node loss ($F = 1$) strictly requires an odd number of voters, such that $N = 3$ and $Q = 2$. In this laboratory topology, an operator can establish fault tolerance by deploying a third, voter-only controller (using minimal JVM heap, such as 512 MB, with `process.roles=controller`) on either the QNAP NAS or the VMware Fusion VM. If restricted strictly to the two physical machines, the deployment provides active-active data partition replication, but provides zero metadata consensus fault tolerance during an ungraceful hardware outage.

### Network Routing, Advertised Listeners, and the Dnsmasq Deception

A frequent operational misconception in distributed streaming architectures involves attempting to balance client traffic across Kafka brokers via round-robin DNS, such as defining multiple address records in Dnsmasq for a single hostname. Traditional Layer 4 proxies or round-robin DNS records cannot load balance Kafka partition traffic because Kafka does not operate as an opaque, request-response HTTP service.

| **Connection Phase**                  | **Client Operation**                                                                    | **DNS and Network Behavior**                                                          | **Broker Response / Action**                                                                                                  |
| ------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Phase 1: Initial Bootstrap**        | Client queries `bootstrap.servers` (e.g., `kafka.lab.internal:9092`).                   | Dnsmasq resolves the alias to any active broker IP address.                           | The queried broker returns a `MetadataResponse` listing all brokers, partitions, and explicit `advertised.listeners` strings. |
| **Phase 2: Socket Partition Routing** | Client closes bootstrap socket and establishes direct TCP sockets to partition leaders. | Client bypasses load balancers and resolves exact broker FQDNs advertised in Phase 1. | The specific broker holding partition leadership processes produce and fetch operations.                                      |

If a broker running inside a container advertises an internal bridge IP address, `0.0.0.0`, or `localhost`, remote clients will resolve the initial bootstrap handshake but fail on all subsequent data transmission attempts. Dnsmasq must therefore be configured to provide authoritative, forward-resolvable Fully Qualified Domain Names (FQDNs) for each physical node (for instance, mapping `kafka1.lab.internal` to `192.168.1.11` and `kafka2.lab.internal` to `192.168.1.12`). Each broker must advertise its specific host FQDN on the exposed listener port. Client traffic is balanced across brokers via internal client partitioning algorithms (such as **Murmur2** key hashing or sticky partition distribution) rather than network-level connection distribution. |

---

(c) Ramón Barrios Láscar, 2026
