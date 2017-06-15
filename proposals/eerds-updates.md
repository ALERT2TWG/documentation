# End-to-End Reliable Datagram Service Updates

Submitted for review by Blue Water Design. 

David Van Wie

Adam Torgerson

R Chris Roark

## Introduction

The End-to-End Reliable Datagram Service (EERDS) is a part of the ALERT2
protocol specification designed to enable reliable data delivery across an
unreliable transport medium. At present, it has not been enabled or used in the
field. However, similar results have been achieved without support of the MANT
layer or the ALERT2 protocol, using the application-layer only. 

In the long run, we believe that the right answer is to implement EERDS at
the MANT layer rather than leaving it to the application developer.  There are
two primary motivations for this. First, defining the interface in the protocol
means that ALERT2 systems will have a uniform implementation that is
transparent to application device developers and end users.  Any application
developer will be able to take advantage of the reliable datagram service
without needing to perform additional development, and system integrators and
maintainers will only need to understand and configure a single set of
parameters for reliable operation.  The second advantage is that the protocol
can make more efficient use of the channel bandwidth by providing support for
the service at the MANT layer.

In this proposal, we address issues identified during the implementation of the
EERDS protocol as defined in the current MANT specification, and propose
updates to the protocol that will increase efficiency, particularly with an eye
to ensuring that the protocol can function at a system-wide scale. 

## Proposed Updates

### Application Protocol Device interaction

The MANT protocol specifies that the Intelligent Network Device (IND) should
inform the Application Protocol Device (APD) when the EERDS service is
requested and a message is either successfully delivered or dropped. The API
for that interaction was not defined. 

We propose the following:

  * Following submission of an Application Layer PDU, after creation of the
    MANT PDU, the IND should send a receipt to the APD containing the source
    address, destination address, and MANT PDU ID of the newly created MANT. 
    
  * When the IND receives acknowledgment for a message or drops a message, it
    should send the same receipt to the API with the appropriate type.

  * The following types should be added to the API to support this interaction: 

Event|Type|Length|Value
-----|----|------|-----
EERDS message submitted|129|5|[Source Addr][Dest Addr][MANT PDU]
EERDS message acknowledged|130|5|[Source Addr][Dest Addr][MANT PDU]
EERDS message dropped|131|5|[Source Addr][Dest Addr][MANT PDU]


### Multiple ACKs

In the scenario where EERDS is being used to deliver data reliably from
reporting sites to a base station, the base station will enqueue the ACKs that
need to be sent during the next available TDMA slot.  In the current protocol,
each of those ACKs would produce a 9-byte MANT PDU (which could also contain a
MANT payload).  To acknowledge six messages would require sending 54 bytes of
data. 

We propose changing the protocol to allow one MANT PDU to acknowledge multiple
messages. In the proposed format when the ACK flag is set:
  
  * The Destination Address Included bit should be set to 0

  * The Destination Address field of the MANT PDU should be absent

  * The Payload Length field should be set to 3 * [Number of ACK bits]

  * The MANT Payload should be populated with [Dest Addr], [PDU ID] pairs,
    where the [Dest Addr] field is populated with the 2-byte source address
    of the IND that originated the message and the [PDU ID] field is
    populated with the PDU ID of the message being acknowledged. 

Using this scheme, the same six ACKs would require 24 bytes of data: six bytes
of MANT header, plus 12 bytes of ACK payload. 

Because the payload field will be overloaded to contain ACKs, the IND needs to
handle this specially when decoding. In binary mode, the IND should not report
a MANT message with the ACK flag set to the APD (it will already send an ACK or
a NAK receipt). In ASCII mode the IND should report the P message as normal and
then report a new 'K' message type. K messages would look like: ```K,[Msg Source
Addr],[Dest Addr],[PDU ID]```. One K message will be produced per ACK
contained in the incoming message. 

The protocol hints at the possibility of bundling additional MANT data with the
ACK message, and these protocol changes eliminate that possibility. However, in
the case where data is bundled with an ACK there are some conflicts that are
not easily resolved. Both the destination address and MANT PDU ID must match
those of the message being acknowledged, which means that the EERDS service
will not work with the bundled payload data. It may be cleanest to remove the
possibility of bundling payload data with an ACK message regardless of the
inclusion of multiple ACKs. 

### Destination Mask / Multicast

There are many cases where it may be desirable to send the same message or
command to multiple receivers. For example, one might want to turn on multiple
sets of flashers when a river gets above a certain level, display a message on
all of the variable message signs in an area, or update the encryption key to
all two-way capable sites. At present, this requires sending individual MANT
PDUs, one per site, and that doesn't scale well to a larger system, especially
with longer control messages. 

In order to facilitate sending one message to multiple hosts (Multicast), we
propose the following scheme: 

  * The reserved bit immediately preceding the ACK bit becomes a "Multicast
    flag". 

  * Immediately following the optional "Destination Address" field, we add an
    optional 16-bit "Destination Mask" field, that is only present in the
    "Multicast flag" is set. 

  * Upon receipt of a Multicast message, sites check if the message matches
    their source address using the following expression: 
    `[IND Source Addr] & [Destination Mask] == [Msg Dest Addr] & [Destination Mask]`. 

  * If the Destination Address is not included in the message, but Multicast is
    set, the destination address is assumed to be 0.

This allows system integrators a high degree of flexibility in setting up their
source addressing. For example, one could define source addresses according to
the following pattern: 

bits|meaning
----|-------
15-13|site type
12-10|drainage or region
10-0|ID

This configuration would allow 8 site types and 8 regions. To send a Multicast
message to all sites of type 2 in region 5, one would set the Destination
Address and mask to: 

format|address|mask
------|-------|----
binary|010 101 00 0000 0000|111 111 00 0000 0000
hex|0x5400|0xfc00
decimal|21504|64512

Some care will need to be taken in how addresses are allocated to ensure
that they remain in compliance with the Soure Address Management System (SAMS). 

When sending a Multicast message, if EERDS is enabled, the requesting APD will
also need to specify the source addresses of the devices from which the IND
should expect an ACK. If one or more of the intended recipients fails to ACK 
the message, the multicast will be sent again unmodified. 

