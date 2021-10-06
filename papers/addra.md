# "[Addra: Metadata-private voice communication over fully untrusted infrastructure](https://www.usenix.org/system/files/osdi21-ahmad.pdf)" Notes

## Part 1

### High-Level Problem Statement

Metadata is extremely powerful and can be used to breach the privacy of users of a system in many domains, including voice calls. Existing techniques to ensure private communications either scale poorly or require trusted infrastructure.

### Contributions

Addra introduces two techniques to ensure that communication is private while preserving scalability and performance, related to how users push data to the server and how that data is retrieved by the recipient.

### Problem Statement

Voice call metadata can be collected and combined to uniquely identify traits of a user of the system, so hiding this metadata is paramount for privacy. Existing techniques for hiding communication-based metadata fail in mainly two areas: 1) they rely on trust assumptions such as trusted infrastructure or a maximum percentage of malicious actors, or 2) they scale poorly and are mainly intended for asynchronous, time-insensitive communication due to the overhead of the protocol.

### Important Terms

- Pung
- PIR cryptographic protocol
- FastPIR cryptographic protocol
- BFV homomorphic encryption scheme
- Homomorphic rotation operations

An understanding of these systems/concepts seems essential in understanding how Addra works, since Addra is so heavily reliant on these protocols and cryptographic concepts to ensure privacy.

### Evaluation

The authors evaluate the performance of Addra by hosting their own client-server setup via AWS using Addra, as well as a similar configuration using Pung with XPIR and SealPIR. Using these systems, they evaluate the message latency as the number of clients/servers grow, the server-side and client-side CPU/network resource consumption, as well as the resource usage of the protocols themselves (FastPIR, XPIR, and SealPIR). The results do look promising, although without more knowledge about the protocols themselves, its hard to assess whether the test scenario was appropriate to test the advantages/disadvantages of each protocol.

### Related Work

The authors describe several previous attempts at approaching this problem and how Addra either avoids the disadvantages that previous attempts could not resolve or how Addra builds upon and improves the results of previous work. Onion-routing and mix-nets make trust assumptions, that the ISP is not collecting the data and that less than 20% of the actors in the system are malicious respectively. DC-nets do not make trust assumptions, but they scale poorly, although the scalability can be improved by relaxing the trust assumptions. 

One of the more related prior works was Pung, which used a similar mailbox approach as well as a tree-based retrieval scheme. However, the performance of Pung is poor due to its large bandwidth requirements and resource consumption, which, along with its probabilistic nature, contributes to its intended nature of an asynchronous communication system. 

Another fundamental work for Addra is PIR, which Addra largely uses, although Addra focuses on optimizing its performance (primarily through parallelization).
