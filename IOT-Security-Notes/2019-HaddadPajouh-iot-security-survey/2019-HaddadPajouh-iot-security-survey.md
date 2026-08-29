# Paper overview 
Paper name: A survey on internet of things security: Requirements, challenges, and solutions (2019).  

Main target: Identifying and analyzing security requirements and vulnerabilities of the IoT environments, as well as a taxonomy based 
on a three layer architecture for IoT security requirements and vulnerabilities.  

| IoT Layer | Security Requirements | Key Threats & Attacks | Countermeasures & Solutions |
| :--- | :--- | :--- | :--- |
| **Edge / Device** | Multi-factor authentication (MFA), network access control (NAC), threat hunting, lightweight cryptography | Signal jamming, identity spoofing, insecure boot, sleep deprivation attacks | Signal strength measurement, hardware TPM modules, integrated AI/IDS agents |
| **Network** | Traffic shaping, traffic monitoring, network anomaly detection | Replay attacks, buffer overflows, RPL routing attacks (sinkhole/wormhole), Sybil attacks, session hijacking | Timestamps & hash checksums, lightweight IPsec, signature-based IDS/IPS |
| **Application** | Application verification, API security (OAuth2/OpenID Connect), digital forensics | Unencrypted CoAP protocol, weak web interfaces (SQLi, XSS), middleware & OS vulnerabilities | Channel encryption (DTLS/TLS), application firewalls, periodic firmware updates, API access controls |

**Notes**
* **Resource Bottleneck**: Edge devices cannot support traditional, heavy encryption algorithm; lightweight cryptographic algorithms are required.
* **Interface Exposure**: The primary attack surface at the application layer remains unencrypted API endpoints and broken access management.
* **Modern Defense Trend**: Effective threat mitigation relies heavily on embedding AI and machine learning into intrusion detection systems (IDS) across both the edge and network layers.

**Limitations**
* **Heavy Resource Demand**: Advanced AI/ML threat-detection models consume too much energy and processing power, making them impractical for low-spec, MCU-based IoT hardware.
* **High False Alarm Rates**: Because IoT environments constantly shift, intrusion detection systems (IDS) often trigger false positives by misidentifying normal network activity as a threat.
* **System Fragmentation**: A widespread lack of standardized protocols across proprietary OSs and firmware forces many defense tools to run only on specific systems like Linux.

**Practical Takeaways**
* **System Hardening**: Lock down hardware using secure chips (TPM/TEE), encrypt CoAP network traffic with DTLS, and restrict API access using OAuth2/OpenID.
* **Penetration Testing**: Focus audit scopes on top risk areas, such as unencrypted API endpoints, misconfigured middleware, and physical edge threats like signal jamming or tampered boot sequences.
* **Possible future development**: Develop lightweight Federated Learning models designed for edge devices, enabling local threat detection within strict memory and battery limits.
