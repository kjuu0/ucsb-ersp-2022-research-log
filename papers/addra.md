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

### Details

Addra supports the expected functions of voice communication (dialing, receiving calls, etc.) while ensuring that both the contents of the call and metadata (mainly, whether two users are communicating with each other) are private, even on compromised infrastructure. Although comprimised infrastructure can cause communications to fail, privacy is still maintained.

Addra utilizes a client-server model, where the server hosts mailboxes, which can be viewed as a key-value store, where the keys are the mailbox ID, and the value is the message. Unlike a normal mailbox though, where the recipient has a mailbox that a sender will deposit messages in, the mailboxes are intended to store messages from the owner of that mailbox. An alternative way of viewing the mailbox system is as a `N x M` matrix, where `N` is the number of mailboxes in the system, and `M` is the maximum size of the message/data to be stored in the mailbox. This view is the primary one used throughout the paper, as it makes the concept of indexes more intuitive later on.

At a high level, Addra consists of three main phase: 1) the registration phase, 2) the dialing phase, and 3) the communication phase.

#### Registration Phase

A user first needs to join the system by registering with the server so that it is discoverable and it can make/receive calls. This registration phase only needs to be ran once when a user joins. The server generates and assigns the user a mailbox ID, a unique authorization token, and the current number of mailboxes in the system. The mailbox ID is needed to deposit data in the user's mailbox, the authorization token ensures that data requested to be put in the mailbox is actually coming from the owner, and the number of mailboxes in the system is later needed as part of the encryption scheme to extract the data at the mailbox of the partner of conversation. The number of mailboxes will continually be updated by the server through a broadcast mechanism.

#### Dialing Phase

Addra consists of a round and multiple subrounds within a round. At the start of the round is the dialing phase. The dialing protocol is largely based off of Pung's. First, all senders send an encrypted message with their respective receivers' public key along with the mailbox ID of the sender and the encryption key used for the call to the server. The server collects these requests and broadcasts them all to the clients, which decrypt all encrypted messages to find requests for a client. This broadcast is costly, and is still an open problem. After a call has been confirmed, the sender generates a PIR query for the mailbox ID of the recipient, and the recipient also generates a PIR query for the mailbox ID of the sender. PIR stands for private information retrieval, which is a class of cryptographic protocols that allows a user to retrieve data at some index without any malicious actors (including the server itself) from knowing from what index the data came from. More information about PIR will come later in the log. This PIR query will be sent to the server, which is used by the server (without the server knowing) that these two participants are communicating. After the query has been sent, the client will register an asynchronous callback that will be executed when the server sends the data to the client.

#### Communication Phase



##### More About PIR/CPIR and FastPIR

There are various schemes for PIR, but Addra utilizes computation PIR (or CPIR) since they can be done on a single server, whereas information-theoretic PIR (IT-PIR) rely on two servers, although it is more efficient. Allowing the scheme to be carried out on a single server is better for Addra, since it could be possible that the servers or network connecting the servers can be compromised.

CPIR consists of three phases as well: 1) query, 2) answer, and 3) decode. 

###### Query

There are also several implementations for CPIR, namely XPIR and SealPIR. FastPIR, developed for Addra, although it can be used interchangeably with the other implementations, focuses on optimizing 
