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

## Specification

```abnf
short             = 16*16BIT
        ; SIGNED SHORT INTEGER
		
integer           = 32*32BIT
        ; SIGNED INTEGER

long              = 64*64BIT
        ; SIGNED LONG INTEGER

length            = long
        ; Length specifies the content length in bytes.
		
length-none       = 0xFFFFFFFFFFFFFFFF
        ; SIGNED LONG INTEGER
        ; -1
        ; Denotes that there is no more data in 
        ; pipeline.
        
packet            = packet-begin 1*packet-data [packet-end]

packet-begin      = exchange-cycle flags operation-id (length-total / length-chunked)

packet-data       = state operation-id (*<length-total>length / length) <length>*<length>data

packet-end        = length-none

exchange-cycle    = integer

length-total      = length
        ; The data length in total when packet is
        ; delivered completely.
        
length-chunked    = 0x00000000
        ; Denotes that the total lenght is unknown
        ; when the flags include flag-chunked.
        ; 
        ; When chunked, what is important the length
        ; in packet-data.
		
flags             = flag-chunked 63*63undefined-flag
        ; Chunking is the only expected flag at the 
        ; moment.
        ; Disabled flags can safely be ignored.
		
flag-chunked      = flag
        ; Specifies whether or not the operation is 
        ; chunked.
		
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
		
operation-id      = integer
        ; A unique ID specifying the operation number
        ; for a given operation.
		
data              = BIT

state             = operation-id (state-none / state-close / state-cancel / (state-info state))

state-none        = 0x00000000
        ; priority 0
        ; Begins with SIGNED INTEGER "0"

state-close       = 0x00000001
        ; priority 10
        ; Begins with SIGNED INTEGER "1"

state-cancel      = 0x00000002
        ; priority 9
        ; Begins with SIGNED INTEGER "2"
        
state-info        = 0x00000003 (protocol-version)
        ; priority 8
        ; Begins with SIGNED INTEGER "3"
        
protocol-version  = integer 
```
