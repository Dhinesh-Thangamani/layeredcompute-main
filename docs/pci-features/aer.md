# Advanced Error Reporting
# Overview

Advanced Error Reporting is error handling and reporting mechanism in PCIe devices. This capability is reported in extended capability register space. AER handles and reports all three classification of errors.

**1. Uncorrectable Fatal Error:** PCIe non-recoverable fatal errors are serious hardware failure in communication which cannot be corrected or recovered. Unresponsive device, instable link, data corruption, protocol violations are few examples of fatal errors. System reboot or hardware reset is required to recover the system to funcational state. 

**2. Uncorrectable Non-Fatal Error** PCIe non-fatal errors are less severe errors, however they are not automatically correctable by hardware or firmware. This may not require system reboot. The system continues to operate. The primary difference between fatal and non-fatal is the impact to overall system. The non-fatal does not impact overall system integrity. Does not require system or PCIe hierarchy reset. The error is confined to single device or single transaction. ECRC, unsupported request are the examples of non-fatal errors.

**3. Correctable Error** PCIe correctable errors are automatically corrected by the hardware. The error is still reported to Advanced Error Reporting mechanism. 

 
