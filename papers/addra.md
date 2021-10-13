# "[Addra: Metadata-private voice communication over fully untrusted infrastructure](https://www.usenix.org/system/files/osdi21-ahmad.pdf)" Notes

## Part 1

### High-Level Problem Statement

Metadata is extremely powerful and can be used to breach the privacy of users of a system in many domains, including voice calls. Existing techniques to ensure private 
communications either scale poorly or require trusted infrastructure.

### Contributions

Addra introduces two techniques to ensure that communication is private while preserving scalability and performance, related to how users push data to the server and 
how that data is retrieved by the recipient.

### Problem Statement

Voice call metadata can be collected and combined to uniquely identify traits of a user of the system, so hiding this metadata is paramount for privacy. Existing techniques 
for hiding communication-based metadata fail in mainly two areas: 1) they rely on trust assumptions such as trusted infrastructure or a maximum percentage of malicious actors, 
or 2) they scale poorly and are mainly intended for asynchronous, time-insensitive communication due to the overhead of the protocol.

### Important Terms

- Pung
- PIR cryptographic protocol
- BFV homomorphic encryption scheme

An understanding of these systems/concepts seems essential in understanding how Addra works, since Addra is so heavily reliant on these protocols and cryptographic concepts 
to ensure privacy.

### Evaluation

The authors evaluate the performance of Addra by hosting their own client-server setup via AWS using Addra, as well as a similar configuration using Pung with XPIR and SealPIR.
Using these systems, they evaluate the message latency as the number of clients/servers grow, the server-side and client-side CPU/network resource consumption, 
as well as the resource usage of the protocols themselves (FastPIR, XPIR, and SealPIR). The results do look promising, although without more knowledge about the protocols 
themselves, its hard to assess whether the test scenario was appropriate to test the advantages/disadvantages of each protocol.

### Related Work

The authors describe several previous attempts at approaching this problem and how Addra either avoids the disadvantages that previous attempts could not resolve or 
how Addra builds upon and improves the results of previous work. Onion-routing and mix-nets make trust assumptions, that the ISP is not collecting the data and that 
less than 20% of the actors in the system are malicious respectively. DC-nets do not make trust assumptions, but they scale poorly, although the scalability can be 
improved by relaxing the trust assumptions. 

One of the more related prior works was Pung, which used a similar mailbox approach as well as a tree-based retrieval scheme. However, the performance of Pung is poor 
due to its large bandwidth requirements and resource consumption, which, along with its probabilistic nature, contributes to its intended nature of an asynchronous 
communication system. 

Another fundamental work for Addra is PIR, which Addra largely uses, although Addra focuses on optimizing its performance (primarily through parallelization).

### Details

Addra supports the expected functions of voice communication (dialing, receiving calls, etc.) while ensuring that both the contents of the call and metadata 
(mainly, whether two users are communicating with each other) are private, even on compromised infrastructure. Although comprimised infrastructure can cause communications 
to fail, privacy is still maintained.

Addra utilizes a client-server model, where the server hosts mailboxes, which can be viewed as a key-value store, where the keys are the mailbox ID, and the value is 
the message. Unlike a normal mailbox though, where the recipient has a mailbox that a sender will deposit messages in, the mailboxes are intended to store messages from 
the owner of that mailbox. An alternative way of viewing the mailbox system is as a `N x M` matrix, where `N` is the number of mailboxes in the system, and `M` is the 
maximum size of the message/data to be stored in the mailbox. This view is the primary one used throughout the paper, as it makes the concept of indexes more intuitive 
later on.

At a high level, Addra consists of three main phase: 1) the registration phase, 2) the dialing phase, and 3) the communication phase.

#### Registration Phase

A user first needs to join the system by registering with the server so that it is discoverable and it can make/receive calls. This registration phase only needs to be 
ran once when a user joins. The server generates and assigns the user a mailbox ID, a unique authorization token, and the current number of mailboxes in the system. 
The mailbox ID is needed to deposit data in the user's mailbox, the authorization token ensures that data requested to be put in the mailbox is actually coming from the owner, 
and the number of mailboxes in the system is later needed as part of the encryption scheme to extract the data at the mailbox of the partner of conversation. 
The number of mailboxes will continually be updated by the server through a broadcast mechanism.

#### Dialing Phase

Addra consists of a round and multiple subrounds within a round. At the start of the round is the dialing phase. The dialing protocol is largely based off of Pung's. 
First, all senders send an encrypted message with their respective receivers' public key along with the mailbox ID of the sender and the encryption key used for the 
call to the server. The server collects these requests and broadcasts them all to the clients, which decrypt all encrypted messages to find requests for a client. 
This broadcast is costly, and is still an open problem. After a call has been confirmed, the sender generates a PIR query for the mailbox ID of the recipient, 
and the recipient also generates a PIR query for the mailbox ID of the sender. PIR stands for private information retrieval, which is a class of cryptographic 
protocols that allows a user to retrieve data at some index without any malicious actors (including the server itself) from knowing from what index the data came from. 
More information about PIR will come later in the log. This PIR query will be sent to the server, which is used by the server (without the server knowing) that these 
two participants are communicating. After the query has been sent, the client will register an asynchronous callback that will be executed when the server sends the 
data to the client to receive and decrypt the messages. 

One may be concerned that the existence of this PIR query for some users and not for others show that some users are active and others aren't. To get around this, 
Addra simulates constant communication by having idle users dial themselves, thus mimicking communication and hiding the existence of communication for others.

#### Communication Phase

After the dialing phase ends, indicating the start of a new round, the subrounds start that make up the communication phase. In these rounds, users deposit messages 
into their mailboxes, the server takes those messages and calculates PIR answers based off of the PIR queries that were created in the dialing phase, and those PIR 
answers are returned to the users. Once the PIR answers are calculated, users will execute the asynchronous callback registered in the dialing phase and receive the PIR answer, 
then PIR decrypt the message and play it to the user. The reuse of the initial query is crucial for Addra's performance, since repeated calculations of these PIR queries are 
expensive. In fact, the calculation of these PIR queries may be slower than other systems like Pung, but the reuse means that its amortized complexity is much lower than these 
other systems.

### PIR/CPIR

PIR is fundamental to the inner workings of Addra; mainly its ability to hide the mailbox ID that a user is subscribing to. There are various schemes for PIR, 
but Addra utilizes computation PIR (or CPIR) since they can be done on a single server, whereas information-theoretic PIR (IT-PIR) rely on two servers, 
although it is more efficient. Allowing the scheme to be carried out on a single server is better for Addra, since it could be possible that the servers or 
network connecting the servers can be compromised.

CPIR consists of three phases as well: 1) query, 2) answer, and 3) decode. There are various implementations of CPIR, namely XPIR and SealPIR. Along with the overall protocol, 
Addra also introduces an optimized implementation called FastPIR that can be used interchangeably. FastPIR optimizes upon existing implementations primarily by 
avoiding generating ciphertexts for whole rows and avoiding creating large PIR answers through the use of rotations.

#### Query

If we want to query for some mailbox ID `i` and we have `N` mailboxes, we generate a one-hot-encoding for the index, which is simply a 1D vector of dimensions `1 x N` 
(although it can be depicted as a 2D vector of dimensions `2 x N/2`). This 1D vector is all `0`s except for at index `i`, which is a `1`. This differs from existing PIR 
implementations, which generally generate one ciphertext per row. Additionally, this 1D vector is broken up into `k` segments.

#### Answer

Now, given our `N` mailboxes, we simply multiply it with our PIR query to extract the contents of the mailbox we're interested in. Clearly, the `0`s will eliminate any 
data we're not interested in, and the `1` will preserve the contents of the mailbox we're interested in. The result of this matrix will generate `m` vectors, 
where `m` is the length of the message. To compress these vectors into one, we rotate and combine these vectors in a tree-based combination scheme. 
This reduces the amount of useless `0` values in our result. This vector of `0`s and the mailbox contents will be returned to the user.

#### Decode

Finally, the user will receive the PIR answer and decode it, or extract the actual information from the PIR answer. The PIR answer is rotated around the specified index, 
so the user will have to do additional rotations to obtain the mailbox contents.

#### Brakerski/Fan-Vercauteren (BFV) Encryption

The above CPIR scheme is pretty intuitive, but it seems like just a more complicated way of extracting data given some index. Also, it makes no privacy guarantees; 
a server could simply look at the query and see what indices are `1` and can then see what mailbox that user is interested in. To actually provide privacy, 
CPIR needs to be combined with a homomorphic encryption scheme, like BFV encryption. Let `enc(x)` be the homomorphic encryption of `x`. Let `f(x)` be some operation 
performed on `x`. Then homomorphic encryption is just some `enc` such that `f(enc(x)) = enc(f(x))`.

Use exponential homomorphism as a basic (and naive example). So our encryption `enc` is just `enc(x) = 2^x`. We have `x`, and we want to find `x + y`, 
but we don't have `y`, the server (which can't be trusted!) does. So, we send our encrypted `x`, `2^x` to the server, and the server, which has encrypted `enc(y) = 2^y`, 
just knows to multiply the two values together. It does not know the actual value of `x`, or `y`, or `x + y`. It just knows `2^y` and `2^x * 2^y = 2^(x + y)`.
Now with `2^(x + y)`, we can easily decrypt it and find `x + y`. CPIR relies on an encryption scheme that has this property.

So, the general steps of CPIR is still the same, but we introduce BFV encryption, so now the server does not know what mailbox ID we're querying for, 
nor the contents of the actual mailbox, it just returns the user the encrypted data that the user can decrypt.

BFV's rotation property is essential for the rotation/compression scheme outlined in the answer/decode steps.

## Part 2

### 2.1: Clarifying Terms

#### CPIR cryptographic protocol

PIR stands for private information retrieval. It allows clients to query some database server for some item `i` without the server knowing what `i` is. 
There are two types of PIR schemes; information-theoretic PIR (ITPIR) and computational PIR (CPIR). Addra utilizes CPIR for its retrieval scheme, 
as ITPIR requires multiple replicated databases whereas CPIR can be implemented with only one.

Sources:

- https://en.wikipedia.org/wiki/Private_information_retrieval
- https://people.csail.mit.edu/silvio/Selected%20Scientific%20Papers/Private%20Information%20Retrieval/Computationally%20Private%20Information%20Retrieval%20with%20Polylogarithmic%20Communication.pdf

#### BFV homomorphic encryption scheme

The BFV homomorphic encryption scheme is a homomorphic encryption scheme developed by Brakerski and Fan-Vercauteren (where the name comes from). 
Homomorphic encryption schemes are based off of the homomorphism property, which is a mapping between two objects of the same types that preserves the operations 
of the two objects. So, if `*` is some operation supported by two types `x` and `y` (like multiplication), and `f` is the homomorphic mapping, then `f(x * y) = f(x) * f(y)`.
This property makes it possible to compute on encrypted data, which makes things more secure. The BFV encryption scheme offers relatively high performance compared 
to other homomorphic encryption schemes.

Sources:

- https://en.wikipedia.org/wiki/Homomorphic_encryption
- https://inferati.azureedge.net/docs/fhe-bfv.pdf
- https://medium.com/privacy-preserving-natural-language-processing/homomorphic-encryption-for-beginners-a-practical-guide-part-1-b8f26d03a98a
- https://www.ic.unicamp.br/~reltech/PFG/2018/PFG-18-28.pdf

### 2.2: Paper Reflections

> What is your takeaway message from this paper?

Preserving privacy in both the actual data/contents to be retrieved as well as the overall metadata of the transaction itself is possible and potentially scalable.

> What is the motivation for this work (both people and technical problem), and its distillation into a research question? Why doesn't the problem have a trivial solution? 
> What are the previous solutions and why are they inadequate?

Metadata can be collected and analyzed to extrapolate specific information about certain users of a system, even if the specific data the users transmit are hidden. 
Although this problem can technically be solved trivially by just sending a copy of the entire database to a client and allow the clients to query the copy locally,
it's not feasible in production, nor is it scalable due to the high communication cost. 

Previous solutions either make trust assumptions (tolerating 20% of the infrastructure to be malicious or having a single trusted server) or scale very poorly/are intended
for asynchronous communication (Pung).

> What is the proposed solution? Why is it believed it will work? How does it represent an improvement? How is the solution achieved?

The proposed solution is named Addra, a three-staged round-based protocol that allows for performant data transmission through the use of a mailbox architecture while preserving
the metadata privacy between two users. In addition, a faster CPIR scheme aptly named FastPIR is created to further enhance the scalability of Addra. Addra preserves
metadata privacy primarily through the use of BFV homomorphic encryption, as well as some additional techniques like cover noise to limit the amount of metadata that can
be obtained. 

The only way for a server to know which mailbox some user is querying for is to be able to decrypt the query, but since only the users have access to their encryption keys,
this should be impossible for the server.

> What is the author's evaluation of the solution? What logic, argument, evidence, artifacts (e.g., a proof-of-concept system), 
> or experiments are presented in support of the idea?

The author proves that metadata and data privacy are preserved by constructing a rigorous proof in an appendix section. It takes the approach of setting up "games" and proving
that malicious actors cannot retrieve metadata/data in these games.

In addition, the scalability/performance of Addra was measured in a simulated load test hosted by AWS.

> What is your analysis of the identified problem, idea and evaluation? Is this a good idea? What flaws do you perceive in the work? 
> What are the most interesting or controversial ideas? For work that has practical implications, ask whether this will work, who would want it, 
> what it will take to give it to them, and when it might become a reality?

The problem is an interesting one, and creating a performant and secure solution to it can have a wide range of applications in any privacy-focused context.
It is especially important these days, since more and more data is being moved to cloud-based solutions (so companies/individuals don't have control over their data)
and overall privacy is becoming a greater concern these days.

The idea and evaluation looks extremely promising. The main flaw is the high amount of data that needs to be transmitted in order to participate in the system,
since cover traffic is needed to maintain true metadata privacy. Also, although Addra scales to tens of thousands of users, it would be even better if we can improve
the scalability of the system to scale to even greater numbers.

> What are the paper's contributions (author's and your opinion)? Ideas, methods, software, experimental results, experimental techniques...?

The paper's main contributions are Addr and FastPIR.

> What are the future directions for this research (author's and hours, perhaps driven by shortcomings or other critiques)?

Currently, Addra hasn't been tested in a real voice-call environment, so that can be something to look into. Additionally, the authors are reseraching more into how 
FastPIR can be further optimized, possibly through the use of recursion or additional parallelization techniques.

> What questions are you left with? What questions would you like to raise in an open discussion of the work (review interesting and controversial points above)? 
> What do you find difficult to understand? List as many as you can.

The questions I have are mostly with how BFV encryption actually works, as well as FastPIR.
