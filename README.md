Lamport Signatures for Py3k
==================
by Cathal Garvey
Released under the GNU General Public License 3.0
A novice's implementation of the Quantum-Computer-Resistant Lamport Signature scheme in Python 3

**Don't use or rely on this code. It was an experiment when I was inexperienced and looking into
post-quantum crypto, but I was not then and am still not now competent enough to be trusted with
security-critical cryptographic constructions.**

Dependencies:
* None if using Python 3.3
* PyCrypto if using a lesser version of Python: Necessary for its Crypto-Secure Pseudo-Random Number Generator.
* In the absence of up-to-date python or PyCrypto, lamport.py will attempt to use /dev/urandom if run on Linux, but it is possible that the urandom pool will be exhausted if using to generate many keys (i.e. generating a merkle tree.)

What is this?
-------------
### Lamport Signatures
Lamport Signatures are an elegant use of cryptographically-secure hash functions. That is, functions that may be applied to data which generate an output of a known size, which is unique to the input, without revealing anything useful about the input itself, and without being able to deliberately design an input to get a desired output. So, while the hash of a certain piece of data may be a constant thing, you cannot theoretically determine what data has been hashed unless you have a copy of the original data to compare against on hand. Also, you cannot (with any reasonable amount of effort) find an alternative input which generates the same hash output, except through a level of brute-force trial-and-error which would take longer than a human lifespan at least with exceptionally powerful computing resources at your disposal.

In order to use this to generate a secure digital signature scheme, the Lamport Signature scheme uses hashes to declare knowledge of a set of secret numbers, some of which are then revealed along with a message that is intended to be signed. Specifically, the scheme uses large sets of pairs of numbers, and reveals one from each pair depending on the value of the message to be signed. Readers can then compare the message, the secret numbers, and the previously and publicly disclosed hashes of the secret keys, to prove that the message could only have been signed by a person who knew all of the secret numbers; ergo, the "owner" of the private key.

### Example
For example, say that Alice wants to write a blogpost and digitally sign it. Alice has previously declared that she has a set of 512 secret number pairs, with each number being 512 bits in size, by releasing a corresponding set of 512-bit hashes for the secret numbers. Because she is using a cryptographically secure hash function, the hashes can be used to identify these secret numbers when they are eventually partially disclosed, but cannot be used to actually determine what the numbers are before that occurs.

Alice writes her blogpost, and then calculates a 512-bit hash for her blogpost when she is happy with the result. For each bit of the blogpost hash (that is, each 1 or 0), she chooses, respectively, either the first or second secret number in the set of 512 secret number pairs. These numbers she will disclose as her signature.

Bob, an avid reader, wants to verify that Alice wrote the blogpost (because there have been copycats in the past!). He calculates the hash of the blogpost (which generates the same result as Alice did during the signing process), and for each bit of the 512-bit message hash, he fetches the publicly-disclosed hash of Alice's number set. He compares these publicly disclosed hashes to the hash value of the corresponding numbers disclosed in Alice's signature. As Alice was the author, the values match, and he considers the signature valid.

Bob knows that Alice wrote the message because in order to generate a false message, a forger would have to either:
A) Know the secret numbers in order to reveal the appropriate number for each bit of the message hash,
B) Find alternative numbers that generate hashes identical to Alice's disclosed public hash set,
C) Find an alternative message that generates a hash identical to the message hash (and have it be readable and convincing)
D) Piece together a set of known numbers that generate identical hashes to those needed for the signature.

Because Alice uses a cryptographically secure hash algorithm, B and C are unfeasable. A is unlikely as long as Alice carefully guards her private number set (her "private key").

### Single-Use Limitation of Lamport Signatures
However, D reveals the primary drawback of the Lamport Signature Scheme: if Alice uses her private number set more than once, she risks revealing enough secret numbers to allow a forger to generate falsified messages using the numbers revealed in previous posts. The Lamport Signature scheme is therefore considered a "one-time" signature system.

This is a hugely significant drawback of the scheme. The benefit of public keys is generally that they can be verified to belong to a person once, and used thereafter to sign a large or infinite number of messages. If they can only be used once, the burden of proving that a public key corresponds to an authentic private key belonging to the target person becomes highly onerous. Also, public keys in other signature schemes can benefit from a "web of trust", wherein other persons, having verified that Alice owns a public/private key pair, can sign her public key to indicate to others that it is a trustworthy key; that it belongs to the person it appears to.

If a key can only be used once, then a web of trust cannot happen as the act of signing other keys exhausts a key.

The Merkle Tree System
----------------------
### Why are Merkle Trees Useful?
To resolve this "one-use-only" problem of Lamport Keys, a system was created by Ralph Merkle to generate a large number of Lamport Keys at one time and lay claim to the entire set publicly. Although the "public key" in this case now corresponds to a large but finite set of separate keypairs, it allows reasonable use of Lamport Keys in establishing Webs of Trust and delaying the burden of re-verifying new keys for known persons.

Indeed, now that more than one key can be simultaneously associated with a person, that person could even use their (hopefully well-established and peer-signed) keyset to sign their next keyset when their existing one is coming close to exhaustion, helping somewhat to overcome the finite nature of Merkle-Lamport keysets.

### How does a Merkle Tree Work?
In order to claim many keys in one move, the Merkle Tree scheme uses hashes of the disclosed pubkey sets for each keypair to create the "leaves" of a new hash-tree. For each pair of pubkey-hash-leaves of this tree, a hash value is computed, and this "layer" of the tree has half as many members as the first. This layer is then pairwise-hashed to generate another layer of half the size, and so on until only one hash remains at the top of the tree; this hash alone is considered the "pubkey" of the individual laying claim to the keyset.

Because the hash function is cryptographically secure, forgers cannot (reasonably):
A) Find other key-sets or partial key-sets that generate matching hashes which cascade upwards to the same "root" hash.
B) Find other valid key-sets that match leaf-node hashes for given key-sets.
C) Convert any leaf-node hashes into key-sets of private keys.

Using a Merkle Tree therefore, a user can disclose a root hash alone, or the upper sets of hashes in the tree, or the entire tree, but each leaf-node can be used as part of the overall tree, exhausting that leaf but not the tree itself.

### Using a Merkle-Tree
When a user wants to use a keypair to sign a message, the user includes the Lamport Signature as normal, and also includes the nodes of the tree that are necessary to compute upwards to the "root" node. That is, for each node of the tree between the used keypair and the root, the other hash needed to compute the ensuing hash-node is provided in the signature. In this way, readers need not know the entire tree (which is good, because these trees can be huge) in order to verify that:
A) The signature matches the message given a certain Lamport Pubkey
B) The Lamport Pubkey Hash can be combined with other hashes in the Merkle Tree to compute upper nodes of the tree.
C) The uppermost node, when computed with the provided pubkey and node-hashes, is equal to the published pubkey of the author.

This use of a hash-tree is still computationally time-consuming compared to some other signature schemes, and requires more space per signature. Signature size in bits is: 512 * 512 bits of data for the Lamport Key, plus 512 * (Tree Layers) additional bits for each hash required to constitute the public key "root" node. For a Merkle Tree consisting of 1024 keypairs, there are 10 additional hashes required to constitute the root, so signatures are (512*512 + 5120) bits in size, amounting to about 33.4kb of data. 512 hash operations will be needed to generate the hashes of the disclosed secret key numbers, and 11 hash operations will be needed to traverse the Merkle tree. Coupled with comparison operations and other code, these signatures can become quite costly to deploy and verify.

Why is the Lamport-Merkle Scheme Useful?
----------------------------------------
### Simplicity
One reason the scheme is valuable is that the computational methods needed can be found or implemented on almost any system; most of the required functionality of a Lamport Signature is a hash operation and a cryptographically secure random number generation, both of which are often included on operating systems as a matter of course, and which are required for the basic functions of the operating system itself. Making use of these built-in functions is usually possible with relatively little effort.

### Quantum Computation Resistance
The primary reason the Lamport-Merkle scheme gains attention however is that it is considered resistant to Quantum cryptanalysis. Most prevalent systems for cryptography in use today which are used for message signing and verification of authenticity are vulnerable to an algorithm called "Grover's Algorithm", which, when run on a feasable quantum computer, will enable effortless cracking of these systems. A useful quantum computer is considered to be many years away, but as soon as one appears, all current methods of cryptography in wide use become obsolete.

### Modern Relevance
This is relavant today due to the unprecedented ability of governments and other potentially repressive organisations to perform near-total data retention of network traffic. For example, encrypted traffic can be intercepted at fiber-optic links connecting countries or continents, through satellite downlinks, or through monitoring of airwaves or direct monitoring of wifi network traffic. This data can be logged indefinitely, and likely will be, especially for individuals of note to these regimes (often though not always due to an interest in or willingness to defend free speech or other liberties).

This means that data retained today, which may contain private or personally sensitive information, will likely someday be cracked and laid bare before organisations that have little regard for civil rights, privacy or personal dignity, unless a switch is made to quantum-resistant algorithms such as the Lamport-Merkle system.

Can it Encrypt Things?
----------------------
Traditionally, to send a secure message to someone over an insecure channel each participant sends a public key which can be used to encrypt messages one-way-only, whereas the key owner has additional data (the private key) that allows decryption of the message.

These systems are usually only used to encrypt the secret key of a symmetric cipher like AES, which is then used by either participant to encrypt future messages.

Sadly, the Lamport scheme cannot be used to encrypt data, so another system will be needed to complement or replace this scheme for secure asymmetric encryption to establish shared secret symmetric cipher keys.
Other options of note include NTRU, which is patent-encumbered but seems to be viable for asymmetric, quantum-resistant encryption. Open Source C-code implementations of NTRU are available, but use of these may be illegal depending on the state of software patenting in your locale, until the patents expire.
