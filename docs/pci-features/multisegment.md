# PCIe Multi-Segment Architecture

# Overview
A PCI Express (PCIe) segment (also referred as a PCIe domain) represents an independent PCIe hierarchy with its own configuration space and bus numbering. In a single PCIe segment, bus numbering is limited to 8 bits, allowing a maximum of 256 busses (0 - 255).

In large-scale server and high-performance computing systems, the number of required PCIe devices, bridges, and switches may exceed the addresssing capacity of a single segment. This limitation is primarily due to the finite bus number defined in the PCIe specification more than three decades ago. 

To address this scalability constraint, systems may implement multiple PCIe segments.
Each segment:
* Has an independent bus number space (0-255)
* Maintains seperate configuration address spaces
* Viewed and managed as an independent PCIe hierarchy by system Firmware and the operating system.
* Typically originates from a separate Root Complex or logically partitioned Root Port

The Firmware and Operating System identifies each segment using a Segment Number. The PCIe segment details are reported to Operating System through ACPI MCFG table.
The MCFG table contains:
* PCIe Configuration Space Base Address
* PCIe Segment Group Number
* Start (base) Bus Number
* End Bus Number
