# EasyLink Custom RS485 Communication Protocol Specification

**Protocol Version:** 1.0
**Document Version:** 1.0
**Last Updated:** 2024-03-XX 
**Author/Maintainer:** Hamzah Faleel / Jesper-919 (Your Name/Handle)

## 1. Introduction & Overview

This document specifies the **EasyLink protocol**, a custom binary communication protocol designed for reliable Master-Slave data exchange over a **half-duplex RS485 serial bus**. It is intended for use in wired sensor networks, IoT applications, and control systems, such as environmental monitoring in greenhouses.

The protocol emphasizes simplicity for implementation on resource-constrained microcontrollers (e.g., ATmega series) while providing essential features like node addressing, command-based interaction, data integrity checking, acknowledgements, and an automated node ID assignment mechanism.

### 1.1. Key Characteristics

*   **Physical Layer:** RS485, Half-Duplex.
*   **Serial Configuration:** 9600 baud, 8 data bits, No parity, 1 stop bit (8-N-1). This is the recommended default; implementations may support other baud rates if agreed upon by all devices on a given bus.
*   **Addressing:**
    *   Each Slave Node has a unique 1-byte ID (1 to 254).
    *   ID `0xFF` (255) is reserved for broadcast messages from the Master to all Slaves.
    *   ID `0x00` (0) is typically reserved for the Master or indicates an unassigned node.
*   **Error Checking:** A 1-byte XOR checksum is used for basic data integrity.
*   **Reliability:** Most unicast commands (Master to specific Slave ID) expect an Acknowledge (ACK) packet from the Slave, which includes the original Message ID from the Master's command.
*   **Packet Delimiting:** Packets are framed with unique Start and End bytes.

## 2. Packet Structure

All EasyLink communication packets adhere to the following general structure:

| Offset (from Start) | Field Name         | Size (Bytes) | Description                                                                                                |
| :------------------ | :----------------- | :----------- | :--------------------------------------------------------------------------------------------------------- |
| 0                   | **Start Byte**     | 1            | `0xAA` - Indicates the beginning of a packet.                                                                |
| 1                   | **Target/Sender ID**| 1            | For Master->Node: Target Node ID (1-254, or 0xFF for Broadcast). For Node->Master: Sender Node's own ID.       |
| 2                   | **Firmware Version**| 1            | Firmware version of the sending device (e.g., `0x01` for v1.0). Used for compatibility awareness.            |
| 3                   | **Message ID (MsgID)**| 1            | Master: Unique ID (1-254, wraps around) for commands expecting ACK/response. Node: Echoes Master's MsgID in ACK/response. Use 0x00 for Master broadcasts or Node-initiated broadcasts (like ID_REPORT). |
| 4                   | **Command Code**   | 1            | Defines the action to be performed or the type of message (see Section 3).                                    |
| 5 to (N-3)          | **Payload**        | 0 to X bytes | Optional data associated with the command. Max payload = `MAX_PACKET_SIZE - 8`.                          |
| N-2                 | **Checksum**       | 1            | XOR checksum of bytes from `Target/Sender ID` (Offset 1) through the last `Payload` byte.                    |
| N-1                 | **End Byte**       | 1            | `0x55` - Indicates the end of a packet.                                                                      |

*   **Minimum Packet Length:** 7 bytes (Start, ID, FW, MsgID, Cmd, Checksum, End - for commands with no payload).
*   **Current `MAX_PACKET_SIZE`:** 16 bytes (allows for a maximum payload of `16 - 8 = 8` bytes). This can be adjusted.

### 2.1. Checksum Calculation

The 1-byte checksum is calculated as the bitwise XOR sum of all bytes in the packet *starting from the `Target/Sender ID` field (Offset 1) up to and including the last byte of the `Payload`*. The Start Byte (Offset 0) and the End Byte are **not** included in the checksum calculation.

**Formula:** `Checksum = Byte[1] ^ Byte[2] ^ Byte[3] ^ Byte[4] ^ Payload[0] ^ ... ^ Payload[LastPayloadByte]`

## 3. Command Set

### 3.1. Master to Node Commands (and Node Responses)

| Code | Name          | Dir. | Payload (Master->Node)             | Expected Node Response (Payload)                               | Description                                                                  |
| :--- | :------------ | :--- | :--------------------------------- | :----------------------------------------------------------- | :--------------------------------------------------------------------------- |
| 0x01 | `CMD_RUN`     | M->N (BC) | None                               | None (Broadcast, no ACK)                                     | Master signals end of ID assignment/config. Nodes in `ASSIGNING_ID` move to `IDLE`. |
| 0x02 | `CMD_WAKE`    | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID)                                | Wakes a specific node from `IDLE` to `ACTIVE` state.                       |
| 0x03 | `CMD_READ`    | M->N | None                               | `CMD_READ` (echoes MsgID) with Sensor Data (e.g., 8 bytes)     | Requests current sensor readings from an `ACTIVE` node.                     |
| 0x04 | `CMD_STATUS`  | M->N | None                               | `CMD_STATUS` (echoes MsgID) with Status Data (3 bytes: State, FW, ID) | Requests the node's operational status.                                      |
| 0x05 | `CMD_DISABLE` | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID)                                | Commands a specific node to enter `DISABLED` state (minimal operation).       |
| 0x06 | `CMD_ENABLE`  | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID)                                | Re-enables a `DISABLED` node, transitioning it to `IDLE`.                    |
| 0x07 | `CMD_REBOOT`  | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID, sent before reboot)              | Commands a specific node to perform a software reset.                          |
| 0x08 | `CMD_IDLE`    | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID)                                | Commands an `ACTIVE` node to enter `IDLE` state.                             |
| 0x10 | `CMD_LED_ON`  | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID)                                | Turns ON the debug LED on a specific node.                                   |
| 0x11 | `CMD_LED_OFF` | M->N | None                               | ACK (`CMD_ACK`, echoes MsgID)                                | Turns OFF the debug LED on a specific node.                                  |
| 0x14 | `CMD_ID_ERASE`| M->N (BC) | None                               | None (Broadcast, no ACK)                                     | All nodes erase their stored ID, reset `lastBroadcastedID`, and enter `UNINITIALIZED` (typically reboot). |
| 0x15 | `CMD_ID_RECHECK`| M->N (BC) | None                               | None (Broadcast, no ACK)                                     | Initiates the "Smart Re-Assign" sequence. Nodes set `recheckIDFlag` and reset `lastBroadcastedID`. |

### 3.2. Node to Master / Node Broadcast Commands

| Code | Name            | Dir.    | Payload Description                               | Notes                                                                                        |
| :--- | :-------------- | :------ | :------------------------------------------------ | :------------------------------------------------------------------------------------------- |
| 0x09 | `CMD_ACK`       | N->M    | None                                              | Node acknowledges a unicast command from Master. `Target/Sender ID` field is Node's own ID. `MsgID` field echoes the Master's original `MsgID`. |
| 0x0A | `CMD_ID_REPORT` | N->M (BC) | 1 byte: Node's newly assigned or confirmed ID.    | Node broadcasts this after successful ID assignment/confirmation during an assignment sequence. `Target/Sender ID` field is `BROADCAST_ID`. `MsgID` is typically `0x00`. |

## 4. Message ID (MsgID) Handling

*   **Master Initiated Commands:** For commands sent to a specific node (unicast) that require a response (ACK or data), the Master MUST generate a unique `MsgID`. This `MsgID` typically increments from 1 to 254 and then wraps around. `MsgID = 0x00` can be reserved for Master-initiated broadcasts that do not expect specific ACKs (like `CMD_RUN`, `CMD_ID_ERASE`, `CMD_ID_RECHECK`).
*   **Node Responses:** When a Node sends an `ACK` or a data response (e.g., a `CMD_READ` response containing sensor data, or a `CMD_STATUS` response), it **MUST** copy the `MsgID` from the Master's original command into the `MsgID` field of its response packet. The `Target/Sender ID` field in the Node's response will be its own unique `myID`.
*   **Node Initiated Broadcasts:** For broadcasts initiated by a Node (e.g., `CMD_ID_REPORT`), the `MsgID` field should be set to `0x00`. The `Target/Sender ID` field for such broadcasts should be `BROADCAST_ID` (0xFF).

## 5. Data Type Representation in Payload (Example for `CMD_READ` Response)

The `CMD_READ` response from a Node to the Master contains sensor data. The exact format and order depend on the sensors implemented. A common example with a total payload of 8 bytes:

1.  **Temperature (Bytes 0-3 of Payload):**
    *   Type: 32-bit IEEE 754 Floating Point.
    *   Byte Order (Example): Big-Endian for the two 16-bit words, and Big-Endian within each word.
        *   If float `T = 23.75` (Hex: `0x41BD0000`):
            *   Payload Byte 0: `0x41` (High byte of high word)
            *   Payload Byte 1: `0xBD` (Low byte of high word)
            *   Payload Byte 2: `0x00` (High byte of low word)
            *   Payload Byte 3: `0x00` (Low byte of low word)
2.  **Soil Moisture (Bytes 4-5 of Payload):**
    *   Type: 16-bit Unsigned Integer (e.g., raw ADC value 0-1023).
    *   Byte Order (Example): High Byte first.
        *   If value = `500` (Hex: `0x01F4`):
            *   Payload Byte 4: `0x01`
            *   Payload Byte 5: `0xF4`
3.  **Light Level (Bytes 6-7 of Payload):**
    *   Type: 16-bit Unsigned Integer (e.g., raw ADC value or sensor output).
    *   Byte Order (Example): High Byte first.

*(Implementations MUST agree on the exact byte order for multi-byte values).*

## 6. Auto Node ID Assignment Sequence

This sequence allows nodes to obtain unique IDs based on their physical position in a daisy chain. It can be initiated in two modes:

### 6.1. Full Erase and Re-Assignment (`CMD_ID_ERASE`)

1.  **Master Broadcasts `CMD_ID_ERASE`:** TargetID = `0xFF`, MsgID = `0x00`.
2.  **All Nodes:** Upon receiving `CMD_ID_ERASE`:
    *   Erase their stored `myID` from EEPROM (e.g., write `0xFF`).
    *   Set internal `myID = 0`.
    *   Set internal `lastBroadcastedID = 0`.
    *   Transition to `UNINITIALIZED` state.
    *   Perform a software reboot.
3.  **Master Pulses `ASSIGN_MASTER_OUT_PIN`:** After a short delay (e.g., 500ms) to allow nodes to reboot, the Master briefly pulses its `ASSIGN_MASTER_OUT_PIN` HIGH.
4.  **Nodes Assign Sequentially (Initial Assignment Logic):**
    *   Each node in `UNINITIALIZED` state waits for its `ASSIGN_IN_PIN` to go HIGH.
    *   It then uses the `lastBroadcastedID` (heard from the `CMD_ID_REPORT` of the previous node, or 0 for the first node) to calculate `myID = lastBroadcastedID + 1`.
    *   Saves `myID` to EEPROM.
    *   Broadcasts `CMD_ID_REPORT` with its new `myID` (TargetID=`0xFF`, MsgID=`0x00`, Payload=`myID`).
    *   Sets its `ASSIGN_OUT_PIN` HIGH to trigger the next node.
    *   Transitions to `ASSIGNING_ID` state.
5.  **Master Monitors `CMD_ID_REPORT`:** The master listens for these broadcasts, tracks assigned IDs, checks for collisions/gaps, and updates its `highestReportedID`. The assignment listening phase ends after a timeout (e.g., 10 seconds of no new ID reports).
6.  **Master Broadcasts `CMD_RUN`:** TargetID = `0xFF`, MsgID = `0x00`.
7.  **Nodes:** Nodes in `ASSIGNING_ID` state transition to `IDLE` state.

### 6.2. Smart Re-Check / Update Sequence (`CMD_ID_RECHECK`)

1.  **Master Broadcasts `CMD_ID_RECHECK`:** TargetID = `0xFF`, MsgID = `0x00`.
2.  **All Nodes:** Upon receiving `CMD_ID_RECHECK`:
    *   Set an internal flag `recheckIDFlag = true`.
    *   Reset internal `lastBroadcastedID = 0`.
    *   Nodes *do not* erase their existing `myID` or change their current operational state yet.
3.  **Master Pulses `ASSIGN_MASTER_OUT_PIN`:** Briefly pulses its `ASSIGN_MASTER_OUT_PIN` HIGH.
4.  **Nodes Check/Assign Sequentially:**
    *   Each node waits for its `ASSIGN_IN_PIN` to go HIGH.
    *   If `myID == 0` (e.g., a newly inserted node): Proceeds with initial assignment as in 6.1.4.
    *   If `myID != 0` AND `recheckIDFlag` is true:
        *   It calculates `expectedID = lastBroadcastedID + 1`.
        *   If `myID != expectedID`, it updates its `myID` to `expectedID` and saves it to EEPROM.
        *   It broadcasts `CMD_ID_REPORT` with its (potentially new) `myID`.
        *   Sets its `ASSIGN_OUT_PIN` HIGH.
        *   Clears its `recheckIDFlag`.
    *   If `myID != 0` and `recheckIDFlag` is false (or already processed): It simply passes the `ASSIGN_OUT` signal HIGH.
5.  **Master Monitors and Finalizes:** Same as steps 6.1.5 to 6.1.7.

## 7. Node Operational States

*   **`UNINITIALIZED`:** Default state on first boot or after ID erase. Node has `myID = 0`. Actively listens for `ASSIGN_IN_PIN` and `CMD_ID_REPORT` broadcasts.
*   **`ASSIGNING_ID`:** Node has calculated and saved an ID during an assignment sequence. It has activated its `ASSIGN_OUT_PIN` and is waiting for the master's `CMD_RUN` broadcast to finalize and move to `IDLE`.
*   **`IDLE`:** Node has a valid ID and is fully initialized. It is in a low-activity/power-saving state, listening for commands. The debug LED is typically OFF.
*   **`ACTIVE`:** Node has been woken by a `CMD_WAKE` command. It is fully operational and ready to respond quickly to sensor read requests or other commands. The debug LED is typically ON solid.
*   **`DISABLED`:** Node has been disabled by a `CMD_DISABLE` command. It ignores most commands except `CMD_ENABLE`, `CMD_STATUS`, and `CMD_REBOOT`. Enters a deep power-saving state if possible. Debug LED is OFF.

## 8. Error Handling & Safe Mode (Node Implementation Detail)

*   Nodes should implement a "Safe Mode" for unrecoverable errors (e.g., EEPROM write failure).
*   In Safe Mode, the node might blink an LED in a specific pattern (SOS) and only listen for a minimal set of recovery commands (like `CMD_ENABLE` or `CMD_REBOOT`).
*   Checksum failures on received packets result in the packet being silently discarded by the node (no NACK is sent by default to keep protocol simple). The Master relies on ACK timeouts to detect communication failures.

## 9. Protocol Version History

*   **v1.0 (YYYY-MM-DD):** Initial protocol definition.
    *   Defined packet structure with Start/End bytes, Target/Sender ID, Firmware Version, Message ID, Command Code, Payload, and XOR Checksum.
    *   Established initial command set for node control, data retrieval, and ID assignment (`CMD_RUN`, `CMD_WAKE`, `CMD_READ`, `CMD_STATUS`, `CMD_DISABLE`, `CMD_ENABLE`, `CMD_REBOOT`, `CMD_IDLE`, `CMD_ACK`, `CMD_ID_REPORT`, `CMD_LED_ON`, `CMD_LED_OFF`).
    *   Defined Auto Node ID Assignment sequence using an "Assign Pin" and `CMD_ID_REPORT` broadcasts.
    *   Added `CMD_ID_ERASE` and `CMD_ID_RECHECK` for more flexible ID management.
    *   Specified node operational states.
    *   Outlined data type representation for sensor values (float, uint16).