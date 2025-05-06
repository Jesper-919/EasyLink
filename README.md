# EasyLink-Protocol Specification

**Version:** 1.0
**Status:** In Development / Stable (Choose one, and update as needed)
**Author:** Jesper-919 (Your GitHub Handle or Name)
**License:** [MIT License](LICENSE) *(Assuming you add an MIT LICENSE file)*

---

## 🌟 Overview

Welcome to the **EasyLink-Protocol** repository! This repository contains the official specification for **EasyLink**, a custom binary communication protocol designed for robust and straightforward data exchange over an **RS485 half-duplex serial bus**.

EasyLink is primarily intended for Master-Slave communication in wired sensor networks and IoT applications, with a particular focus on environments like greenhouses where reliable, addressable communication with multiple sensor nodes is crucial.

The protocol aims to be:
*   **Simple:** Easy to understand and implement on resource-constrained microcontrollers (like ATmega328P).
*   **Reliable:** Incorporates checksums for data integrity and an ACK-based mechanism for command verification.
*   **Scalable:** Supports unique node addressing and broadcast capabilities, allowing for networks of multiple nodes.
*   **Flexible:** Designed to handle sensor data retrieval, node control commands, node ID assignment, and potentially in-field firmware updates.

---

## 📖 Full Protocol Specification

The complete and detailed specification for the EasyLink protocol, including packet structure, command codes, data types, and operational sequences, can be found in the following document:

➡️ **[EasyLink Protocol Detailed Specification (PROTOCOL.md)](PROTOCOL.md)**

Please refer to `PROTOCOL.md` for all technical details required to implement or understand EasyLink.

---

## 🎯 Purpose & Intended Use Cases

EasyLink was developed to provide a reliable communication backbone for the "LeafLink" Greenhouse Monitoring System (or similar projects). Its primary use cases include:

*   **Greenhouse Environmental Monitoring:** Collecting data (temperature, humidity, soil moisture, light levels, etc.) from multiple distributed sensor nodes.
*   **Actuator Control:** Sending commands to nodes to control devices (e.g., turn LEDs on/off, control relays - though not yet implemented in base commands).
*   **Node Management:**
    *   Automatic Node ID assignment in a daisy-chain configuration.
    *   Querying node status and firmware versions.
    *   Remotely rebooting or disabling/enabling nodes.
*   **DIY IoT Networks:** Serving as a foundation for custom wired IoT solutions where simplicity and direct control are preferred over more complex industrial or wireless protocols.

---

## ✨ Key Features of the EasyLink Protocol

*   **RS485 Physical Layer:** Optimized for the robustness of RS485 differential signaling.
*   **Master-Slave Architecture:** Clear roles for command initiation and response.
*   **Custom Binary Packet Structure:** Efficient data transmission. Includes:
    *   `START_BYTE` and `END_BYTE` for packet framing.
    *   `Target/Sender ID` for addressing specific nodes or using broadcast.
    *   `Firmware Version` field for compatibility checks.
    *   `Message ID (MsgID)` for correlating commands with acknowledgements and responses.
    *   `Command Code` byte defining the action.
    *   Flexible `Payload` for data transfer.
    *   `XOR Checksum` for basic data integrity validation.
*   **Acknowledged Commands:** Most unicast commands expect an ACK from the target node, confirming receipt and successful parsing before the node acts on the command.
*   **Node Auto-Identification:** A daisy-chain "Assign Pin" mechanism allows nodes to determine their unique IDs based on their physical position in the chain, with support for full network ID erasure or smart re-checking.
*   **Defined Command Set:** Includes commands for:
    *   Finalizing ID assignment (`CMD_RUN`)
    *   Controlling node power states (`CMD_WAKE`, `CMD_IDLE`, `CMD_DISABLE`, `CMD_ENABLE`)
    *   Requesting data (`CMD_READ`) and status (`CMD_STATUS`)
    *   Remote control (`CMD_LED_ON`, `CMD_LED_OFF`, `CMD_REBOOT`)
    *   Node ID management (`CMD_ID_REPORT`, `CMD_ID_ERASE`, `CMD_ID_RECHECK`)

---

## 🔧 Getting Started & Implementation

To implement the EasyLink protocol in your own Master or Node devices:

1.  **Review the [Full Protocol Specification (PROTOCOL.md)](PROTOCOL.md)** carefully to understand packet structures, command codes, and expected behavior.
2.  **Hardware Requirements:**
    *   A microcontroller with at least one UART (Hardware Serial recommended for RS485).
    *   An RS485 transceiver IC (e.g., MAX485, MAX13487, SP3485) with direction control.
    *   For the auto-ID feature, a dedicated "Assign Pin" for input and output per node.
3.  **Firmware:**
    *   Implement functions to assemble and send EasyLink packets.
    *   Implement a robust parser to receive, validate (checksum), and decode incoming packets.
    *   Create handlers for each relevant command code.
    *   Manage the RS485 transceiver's direction enable (DE/RE) pin correctly (HIGH for transmit, LOW for receive).
    *   Implement the Message ID generation (master) and echoing (node) logic for reliable ACKs.

Sample firmware implementations for ATmega328P (Node) and ESP32/Arduino Uno (Master) may be found in the [main LeafLink project repository](LINK_TO_YOUR_MAIN_PROJECT_REPO_HERE_IF_APPLICABLE). *(You can add this link later)*

---

## 🔀 Protocol Versioning

*   The current version of this protocol specification is **1.0**.
*   The `Firmware Version` field in the packet header is intended to allow for future protocol updates while maintaining some level of backward compatibility or awareness between devices running different protocol versions.

---

## 🤝 Contributing (Optional - Add if you want contributions)

This protocol is currently developed for the LeafLink project. If you have suggestions, find issues, or wish to contribute to the protocol specification:

1.  Please open an **Issue** in this repository to discuss your ideas or report problems.
2.  Fork the repository, make your changes, and submit a **Pull Request**.

---

## 📜 License

This EasyLink Protocol Specification is licensed under the **[MIT License](LICENSE)**. You are free to use, adapt, and implement this protocol in your own projects, both commercial and non-commercial, provided you include the original copyright notice and license text.
