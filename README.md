# CoolSocket ABNF Specification

CoolSocket is a birectional TCP messaging library aiming to be fast and reliable all the while being simple. 

CoolSocket does not take on any existing messaging library, such as ZeroMQ, and tries to serve a specific purpose, that is, user-endedness (being specific to data exchange operations under the influence of end-user). This brings a few challenges that, I believe, I have been able tackle down successfully. 

Those challenges were:

* Cancellability: cancelling a specific read/write operation while keeping the connection itself intact
* Closeability: ability to shut down a connection gracefully with mutual aggreement of both parties without errors. 

The two above ensure that when a connection fails while in progress, the recovery becomes a consideration or if not that, a more correct error can be presented to user, such as, connection closed unexpectedly.

* Bidirectional communication: client or server can both write or read in the same session.
* Data is data: data consists of bytes even if it is a string or any other type and treated as such.
* Single read is possible when possible: data reaches to its destination when it can with a single read/write operation, when it cannot, sending/receiving as chunks is natural part of the process, meaning we don't rely on the developer to divide the data into chunks.

## Terminology

* _connection_ — A connection between two parties maintaining a communication using CoolSocket.
* _receiver_ — The one who will read a `<packet>`.
* _sender_ — The one who will write a `<packet>`.

## Specification

```abnf 
packet            = packet-begin 1*packet-data [packet-end]
        ; The total amount of <packet-data> will 
        ; differ and depend on the delivery
        ; of the <length-total>.
        ;
        ; <packet-end> MUST only be present 
        ; when the packet is <flag-chunked>.
        
packet-begin      = exchange-cycle flags packet-id (length-total / length-chunked) state state
        ; If <flags> include <flag-chunked>,
        ; <length-chunked> SHOULD be present 
        ; instead of <length-total>, however,
        ; when this is the case, it MAY safely 
        ; be ignored. 
        ;
        ; The <packet-id> MUST be consistent
        ; across a <packet>. 
        ; 
        ; The two <state>s at the end are not a 
        ; typo. If this is the 'receiver', it 
        ; MUST read <state> and write one. If
        ; this is the 'sender', the same thing
        ; MUST occur in reverse order.

packet-data       = state packet-id (*<length-total>length / length) <length>data
        ; The <state> here MUST change direction
        ; once <exchange-cycle> value is reached.
        ; Every time <packet-data> is exchanged,
        ; a counter SHOULD increment once to keep
        ; track of the cycle. A 'receiver' MUST
        ; only read until a cycle is complete. 
        ; Once a cycle is completed, the reversal
        ; MUST occur only once and afterwards, it
        ; SHOULD reset the counter, repeating the
        ; process again.
        ; 
        ; If <packet-id> is inconsistent and
        ; differs from the one that was present
        ; in <packet-begin>, you SHALL reject
        ; this and SHOULD close the 'connection'
        ; because this is a programmer error.
        ;
        ; If this is not <flag-chunked> and 
        ; <length-total> has become available
        ; in <packet-begin>, <length> SHOULD not
        ; be greater than it. With each 
        ; <packet-data>, <length-total> SHOULD
        ; be consumed <length> amount. 
        ; 
        ; <data> MUST be <length> in length. 
        ; The length MUST NOT be larger than
        ; <length-total> if not <flag-chunked>.

packet-end        = length-eof
        ; Denotes the end of a <packet>.

exchange-cycle    = integer
        ; The number of cycles before reversing
        ; the <state> direction in <packet-data>.
        ;
        ; This ensures a 'receiver' can send 
        ; <state>s without hurting performance 
        ; too much. 
		
flags             = flag-chunked 63undefined-flag
        ; <flag-chunked> is the only expected flag at the 
        ; moment.
        ;
        ; These enable or disable features depending
        ; before beginning to exchange a packet.
		
flag-chunked      = flag
        ; Specifies whether or not the operation is 
        ; chunked.
        ;
        ; A chunked operation happens when 
        ; <length-total> is unknown and 'sender'
        ; expects to reach a EOF and report it
        ; via <packet-end>.
        
		
undefined-flag    = flag-disabled
        ; An undefined flag specifies a bit sequence
        ; spared for the future but not used at the 
        ; moment.
		
flag              = flag-enabled / flag-disabled
        ; Flags are bits, each of which enables or 
        ; disables a feature.
		
flag-enabled      = %b1
        ; BINARY ONE
		
flag-disabled     = %b0
        ; BINARY ZERO
		
packet-id         = integer
        ; A unique ID specifying the <packet>
        ; a <packet-data> is intended for.
        
length-total      = length
        ; The body length in total when packet is
        ; delivered completely.
        
length-chunked    = 0x00000000
        ; Denotes that the total length is unknown
        ; when the flags include <flag-chunked>.
        ; 
        ; This doesn't cover a packet with 0 <length-total>.
		
data              = 8BIT
        ; A data is a byte.

state             = packet-id (state-none / state-close / state-cancel / (state-info state-info state))
        ; If this is 'receiver', the two 
        ; <state-info>s at the end are in the
        ; order of read and write. For 'sender',
        ; the same SHOULD occur in reverse order.
        ; 'receiver' SHOULD write its respective
        ; info of the same type requested by 
        ; 'sender'. For instance, if the first 
        ; <state-info> contains <protocol-version>, 
        ; 'receiver' must send <protocol-version> 
        ; in theirs.

state-none        = 0x00000000
        ; priority 0
        ; Begins with SIGNED INTEGER "0"
        ;
        ; Denotes that there isn't anything
        ; to do.

state-close       = 0x00000001
        ; priority 10
        ; Begins with SIGNED INTEGER "1"
        ;
        ; You MUST close the connection 
        ; afterwards and MUST disallow any
        ; exchange of data.
        
state-cancel      = 0x00000002
        ; priority 9
        ; Begins with SIGNED INTEGER "2"
        ;
        ; The <packet> that this appeared
        ; on MUST be cancelled and should
        ; no longer be used. The connection
        ; MAY be retained.
        
state-info        = 0x00000003 (protocol-version)
        ; priority 8
        ; Begins with SIGNED INTEGER "3" 
        
protocol-version  = 0x00000000 integer
        ; The CoolSocket protocol version exchanged
        ; between the two to enable compatibility 
        ; when needed.

length-eof       = 0xFFFFFFFFFFFFFFFF
        ; A <long> with the value of "-1".
        ;
        ; Denotes that there is no more data in 
        ; pipeline.
		
integer           = 32BIT
        ; SIGNED INTEGER

long              = 64BIT
        ; SIGNED LONG INTEGER

length            = long
        ; Length specifies the content length in 
        ; bytes.
