# Error Containment
# Overview

Consider a two-story house with four bathrooms, where one bathroom is broken. Closing the entire house because of a single unusable bathroom would be absurd. Instead, the faulty bathroom should be isolated until it is repaired, while the rest of the house remains functional.

Modern computer systems face a similar problem. Large numbers of interconnected devices work together to provide system functionality. Yet due to single faulty device sometimes the entire system shuts down.

A better approach is to gracefully handle device failures without disrupting overall system operation. Although losing a device may reduce performance or disable certain features, the system can continue running—avoiding loss of unsaved data and preventing interruptions to real-time services such as banking or stock trading, etc.

**Error containment** is the mechanism that enables this behavior. It detects errors, isolates the faulty device, and either resets it for recovery or logically removes it from the system. This prevents the fault from propagating and affecting other components.

# Layers of Responsibility in Error Containment

Error containment involves both **hardware** and **software**, with each layer contributing to detecting, isolating, and handling errors efficiently.

**1. Hardware - Physical & Transaction Layers**

The hardware is responsible for detecting and isolating errors at the lowest level:

* **Physical Layer:** Detects signal-level bit flips using techniques like **Parity Checks** and **Error-Correcting Code (ECC)** to ensure data integrity.

* **Transaction Layer:** Detects errors in **Transaction Layer Packets (TLPs)**. If a TLP is found to have bad data, the packet is discarded, and an error report is sent via **Advanced Error Reporting (AER).**

**2. Firmware/BIOS**

* The firmware is responsible for detecting errors and initiating recovery:

* The **BIOS/UEFI** checks error conditions, often using the **AER** mechanism.

* Errors are logged, and recovery attempts are made when possible.

* The **ACPI (Advanced Configuration and Power Interface)** tables are used to pass error details to the OS.

**3. Device Drivers**

* Device drivers handle error detection and recovery in user-space:

* Drivers read error details from the **ACPI tables** provided by BIOS.

* Error handling routines within the driver may attempt to recover from errors or reset devices.

**4. User Applications**

* Applications receive error logs and handle user-level actions:

* Errors are logged as events or hardware errors within the application.

* These logs provide users with the information needed to take corrective actions if necessary.


<!--
Feature	| Error Containment (General PCIe)	                       | Downstream Port Containment (DPC)
-------------------------------------------------------------------|--------------------------------------
Scope	| Broad PCIe error-handling framework	                   | Specific mechanism on downstream ports
Purpose	| Prevent error propagation across the PCIe hierarchy	   | Automatically isolate a faulty device/port
Trigger	| Any PCIe error (correctable, non-fatal, fatal)           | Uncorrectable error detected on a downstream port
          depending on subsystem	
Action	| Logging, reporting, recovery attempts	                   | Disables the link below the port and blocks TLPs
Defined 
in	    |A ER (Advanced Error Reporting),                          | PCIe spec	PCIe spec, DPC capability
Typical 
UseCase | General reliability and debugging	                       | Hot-plug, device failure isolation, preventing data corruption
______________________________________________________________________________________________________________
-->