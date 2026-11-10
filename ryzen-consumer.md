# QNAP Ryzen Consumer Architecture and Flow Control

The consumer service runs on a QNAP NAS powered by an x86_64 AMD Ryzen Embedded processor, deployed via QNAP Container Station.

### Flow-Rate Imbalance and Lag Dynamics

The lab configuration deliberately creates an ingestion-consumption throughput mismatch. Two independent producer containers emit messages with sleep intervals between $0$ and $0.5$ seconds, yielding an expected sleep duration of:

$$
E[S_p] = \frac{0.5}{2} = 0.25\text{ seconds}
$$
