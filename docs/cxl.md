# Overview
**CXL (Compute Express Link)** is built on the **PCIe bus interface** and designed for **high-speed, data-intensive** applications. It is a **high-bandwidth, low-latency protocol** that reuses the PCIe physical layer while adding **memory and cache coherency protocols.** These enhancements enable efficient support for **accelerators** and **memory expanders.**

# Protocols
**CXL.io**
* PCIe-compatible I/O protocol
* Used for device discovery and configuration, including **MMIO/MMCFG** (Memory Mapped IO) register access, PCIe **interrupts**, and **DMA (Direct Memory Access)**
* Maintains backward compatibility with existing software and tools in the PCIe ecosystem

**CXL.mem**
* Allows the CPU to access **device-attached** memory
* Device memory space is mapped to system address usable space to OS
* Enabled **memory expansion**

**CXL.cache**
* Enables **devices to cache host memory** and the **host to cache device memory**
* Maintains **cache coherancy** with the CPU

