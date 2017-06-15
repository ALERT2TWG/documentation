# ALERT2 Authentication and Encryption Proposal

Submitted for review by Blue Water Design. 

David Van Wie

Adam Torgerson

R Chris Roark

## Introduction

As usage of the ALERT2 protocol becomes more widespread, it is becoming
desirable to use it in command and control applications: turning on flashers,
opening and closing gates, or driving warning sirens, for example.
Authentication -- the process of verifying that a request is genuine -- is an
essential part of any command and control system. Recent attacks [1] have shown
that even public safety systems are not immune from intentional abuse and
hacking. 

In this proposal, we describe how industry-standard encryption algorithms could
be applied to the ALERT2 protocol to provide both authentication (is this
message from a genuine source) and encryption (keeping the message contents
secret). 

## Discussion

### Encryption Algorithm 

The is a wide range of different encryption technologies available. Critical
attributes of an encryption technology for use in the ALERT2 space include: 

  * Performance - the algorithm must be suitable for use on low-power embedded
    systems

  * Efficient operation on small blocks of data - the chosen encryption
    technology must not dramatically increase the size of the message

  * Symmetric - because we only want known sites to be able to transmit to other
    sites, a public/private key system is not required for this application 

  * Broadly accepted - selecting an algorithm with broad acceptance and a
    high level of security will make it easier for the protocol to gain
    acceptance in different applications. 

There are a wide number of different algorithms available that meet these
criteria, including the Advanced Encryption Standard (AES, a.k.a. Rijndael),
Serpent, Blowfish and Twofish. These algorithms, with some level of variation,
all meet the first three requirements. In the fourth category, the edge goes to
AES because it is the standard adopted by National Institute of Standards and
Technology (NIST) in 2001 [2], and has been approved for encryption of
classified data by the NSA [3]. 

The AES algorithm performs well on all of the critical attributes described
above. There are a large number of freely available open source implementations
of AES [4] optimized for a variety of different embedded platforms -- including
the widely-used AVR32 -- in addition to variants optimized for PCs. AES is a
block cipher (operating on fixed blocks of 128 bits), but there are several
established techniques to transform a block cipher into a stream cipher (where
the number of output bits matches exactly the number of input bits). 

### Encryption in the AirLink Layer

There are two places in the protocol where encryption could be added: at the
AirLink layer or at the application layer. We propose adding encryption to the
AirLink layer, so that the change would be transparent to the MANT and
Application layers. In this way, our proposal follows the successful
implementation of encryption in many other applications, such as HTTP/HTTPS. 

There are two main advantages to implementing encryption in the AirLink layer:
uniformity of implementation, and transparency to lower layers of the protocol.
Encryption configuration and key management can be accomplished through the
ALERT2 API, providing a standard means to configure system security.  Further,
by offering encryption as a service on the IND, we ensure that any device
capable of using the ALERT2 protocol can be used securely, and free the
manufacturer from the responsibility of properly implementing encryption on all
of the different devices.

A single, uniform implementation should make it easier for systems and
integrators to adopt the changes, which in turn reduces the potential that
critical public-safety infrastructure is disrupted by malicious actors. 

### Authentication

This proposal assumes a simple authentication scheme: if a message can be
successfully decrypted (see the discussion on Encryption/Decryption
Steps below), it is considered to be genuine. In order for an
encrypted message to be transmitted and received successfully, both the sender
and the receiver must use the same key. 

In short, anyone possessing the encryption key is authenticated. This means
that system maintainers must have a process for retiring keys in the case that
a key is compromised, and should cycle keys on a regular schedule. 

Another consideration - it is difficult to prevent an attacker with physical
access to a programmed ALERT2 modem from recovering an encryption key. However,
ALERT2 modem designers should take reasonable precautions to secure the
encryption keys.

### Replay Attacks

In addition to traditional attacks on the encryption algorithm or the key, a
security solution for ALERT2 needs to be concerned with *replay attacks*. A
replay attack is a type of attack where the attacker makes no attempt to
decrypt a message, but rather replays an encrypted message at a later time. If
an attacker were able to identify and record ALERT2 control messages ("Turn on
the warning siren!"), he or she could attempt to trigger the same action by
playing back the unaltered message at a later date. 

A common solution to the replay attack problem is to use a *nonce*[7] -- a
number associated with the transmission that may only be used once. The nonce
can either be sent, in plain text, along with the encrypted message or derived
from some piece of shared knowledge (e.g., the time). 

### AES Modes 

Block ciphers, such as AES, do not provide a complete encryption solution in
isolation. The essence of the issue is that the same 128-bit input sequence
produces the the same 128-bit output sequence. This can cause information to
"leak" through the encryption if, for example, the cipher is being used to
encrypt data that has long strings of repeated values. There are a large number
of different "modes of operation" that can be used with any block cipher to
address this problem.[8]

Of the different modes of operation, counter mode (CTR) is particularly well suited to
low-bandwidth applications because it introduces no message-size overhead. In
CTR mode the algorithm requires the use *nonce*, addressing the replay attack
vector. The *nonce* need not be secret, but should never be reused with the
same key. In order to successfully decrypt the message, the *nonce* plus the
key must be used. 

### The Nonce

There are two different strategies that could be employed to define the *nonce* used. 

The first method uses **time** (plus, optionally, an additional piece of
user-defined information) as the Nonce value. The advantage of this method is
that time is already known to both the sender and the receiver, so no
additional overhead is required to send the nonce. Further, time monotonically
increasing, which means that we are guaranteed never to reuse a value, and no
additional logic is needed to determine the validity of the nonce. The
disadvantage is that it requires that both the sender and a receiver to have an
accurate clock. 

An alternative method would be to send a **token** with the encrypted message
as a part of the AirLink header. The nonce sent might look like a 2-byte source
address (of the IND sending the message, not necessarily the originating IND), a
validation byte, plus a 3-byte counter. Every time the sender encrypts a
message, it increments the counter value. Every time the receiver decodes a
message, it checks that the counter value is strictly greater than the
previous value received from the same source address. If it is greater, the
receiver updates the counter value and processes the message. If not, it
discards the message.  This method has the advantage that it offers more
flexibility than the time-based encryption, but comes at a cost of 6-bytes of
overhead per AirLink frame. 

## Proposed Updates

### AirLink Layer Changes

The AirLink header currently has 4 bits of reserved data. We propose that the
header be updated to include a bit that indicates if encryption is in use. 

The header would then look like: 

Byte|Bits|Purpose|Description / Notes
----|----|-------|-------------------
0   | 7-6|Version|Current version is 0
0   |   5|Encryption|0=plaintext; 1=encrypted
0   | 4-3|Reserved|Value must be 0
0   | 2-0|Length|Most significant bits of length, with byte 1
1   | 7-0|Length|Least significant bites of length, with the lower 2 bits of byte 0

### Encryption / Decryption Step

Encryption and Decryption should be implemented using AES-128 in the CTR mode.

If implementing the nonce via the **time** mode, the first 64-bits of the nonce
shall be the time of the start of the transmission, measured in integer quarter
seconds since Jan 1, 1970 UTC. In order to accommodate using the same encryption
key on multiple radio channels where two transmissions could, without
interference, occur at the same time, users should be able to specify a 4-byte
"salt" value, to be used as the high bits in the second half of the nonce.
Finally, the remaining bits of the nonce should be set to 0. 
If implementing the nonce via the **token** mode, the first 64-bits of the
nonce shall be the token (with the most significant bits always being 0),
followed by an optional "salt" value, followed by 0's. 

There are several different places in the AirLink processing where the
encryption step would be inserted. This requires careful consideration, as the
encryption algorithm itself is unable to tell if a message was successfully
decrypted or not. Thus, it is up the user of the encryption algorithm to perform
validation of the decrypted message. 

In **time** mode, there is not a lot of redundancy present to determine if a
message was decrypted properly -- checking that the MANT length matches the
AirLink length would have a false-acceptance rate of approximately 1 in 4000.
For this reason, in **time** mode it is recommended that encryption be
performed after the Reed-Solomon encoding, and the decryption performed
before. In this way, the Reed-Solomon coding checks the validity of the
decryption. However, it should be noted that the "encrypted message" flag must
be used before the RS coding is applied, so the chance of a bit error is
somewhat increased. 

In **token** mode, there is an extra "validation" byte added to the token. This
should be populated with the least significant bits of the first originating
IND's source address. By using this byte and the length calculations, the false
acceptance rate falls to roughly 1 in one million. Because the included nonce
value should receive the full benefit of ALERT2's forward error correction, in
token mode encryption should take place before the Reed-Solomon encoding and
that decryption take place after the Reed-Solomon decoding. 

It is worth noting that CTR-mode encryption preserves bit-flips (that is, a
flipped bit in the encrypted text results in an encrypted bit in the plain
text), so forward error correction can be applied before or after decryption.

### API and Configuration Changes

It will be necessary to add API commands to enable or disable
encryption and to set the encryption key on an IND. However, unlike
all the other API messages, we suggest that there should not be a way
to get the encryption key from a device, in order to help keep the key
secret. When setting a new encryption key, users should be able to
specify an effective date as well. This would allow technicians to
visit sites to update keys over a number of days or weeks, and then
have the new keys take effect at the same time across the entire
system.

Control sites should implement the MANT command protocol (0x80) to allow setting
encryption keys over the air. The command protocol should require encrypted 
communication. 

Decoders should be configurable to require all incoming messages be encrypted,
or to require encrypted messages from certain source addresses in mixed-mode systems.

## References

[1] https://www.nytimes.com/2017/04/08/us/dallas-emergency-sirens-hacking.html

[2] http://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.197.pdf

[3] https://en.wikipedia.org/wiki/Advanced_Encryption_Standard

[4] https://en.wikipedia.org/wiki/AES_implementations

[5] https://tools.ietf.org/html/draft-ietf-tls-ctr-01

[6] Helger Lipmaa, Phillip Rogaway, and David Wagner. Comments to NIST concerning AES modes of operation: CTR-mode encryption. 2000

[7] https://en.wikipedia.org/wiki/Cryptographic_nonce

[8] https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation
