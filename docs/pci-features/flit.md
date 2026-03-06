# Flit Mode Transactions
# Overview

Flit mode, introduced in PCIe Gen6, stands for Flow Control Unit, which enables data transactions using fixed-size packets of 256 bytes. This fixed size allows for improved flow control and efficiency, as each transaction is handled in predictable, standardized units. The main advantage of flit mode is that it reduces overhead and improves data throughput by ensuring consistency in packet size, making it easier for the system to manage and transfer data with lower latency.

In contrast, non-flit mode (also known as legacy mode) uses variable-sized packets, which can lead to inefficiencies due to the need for additional processing to handle packets of different sizes. This can increase latency and reduce overall system performance.

While flit mode provides significant benefits in performance and data handling, PCIe Gen6 remains backward-compatible with earlier versions, allowing devices to use the legacy mode if needed. Flit mode's fixed-size packets make it especially well-suited for high-performance applications, such as those in data centers or high-speed computing environments, where low latency and high data throughput are crucial.
