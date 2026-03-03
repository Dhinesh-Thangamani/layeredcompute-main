# Posted Transaction

In PCIe, posted transaction is a request sent by the originator that does not require a completion response from the target. These are often referred to as "Fire-and-Forget" transactions. 

Although the originator does not receive a completion status, reliable delivery is ensured by the PCIe link-level protocol mechanisms (ACK/NACK and replay buffers). So, the transaction is guaranteed to reach the destination at the data link layer, but the requestor does not receive an explicit confirmation from the target device that the operation has been processed. 

The primary advantage of posted transactions is lower latency and reduced protocol overhead, since no completion packet needs to be generated or processed. 

Common examples of posted transactions in PCIe include:

* **Memory Write**
* **Message Signaled Interrupts (MSI)**
* **Message Signaled Interrupts - Extended (MSI-X)**

# Non-Posted Transaction
