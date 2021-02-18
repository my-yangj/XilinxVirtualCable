# XilinxVirtualCable v1.1


**Description**:  Xilinx Virtual Cable (XVC) is a TCP/IP-based protocol that 
acts like a JTAG cable and provides a means to access and debug your 
FPGA or SoC design without using a physical cable. 

This capability helps facilitate hardware debug for designs that:
* Have the FPGA in a hard-to-access location, where a "lab-PC" is not close by
* Do not have direct access to the FPGA pins – e.g. the JTAG pins are only accessible via a local processor interface
* Need to efficiently debug Xilinx FGPA or SoC systems deployed in the field to save on costly or impractical travel and reduce the time it takes to debug a remotely located system.

## Key Features & Benefits
* Ability to debug a system over an internal network, or even the internet.
* Debug via Vivado Logic Analyzer IDE exactly as if directly connected to design via standard JTAG or parallel cable
* Extensible to allow for safe, secure connections

# XVC 1.1 Protocol
```
XVC 1.1 Protocol 
Copyright 2015-2021 Xilinx, Inc.  All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## Overview

Xilinx Virtual Cable (XVC) is a TCP/IP-based communication protocol that acts like a JTAG cable and provides a means to access and debug your FPGA or SoC design without using a physical cable. In this document are the general details of this XVC 1.1 protocol.

The XVC protocol is used by a TCP/IP client and server to transfer low level JTAG vectors from a high level application to a device. The client, for instance Xilinx's EDA tools, will connect to an XVC server using standard TCP/IP client/server connection methods. Once connected, the client will issue an XVC message (getinfo:) to the server requesting the server version. After the client validates the protocol version, a new message is sent to the XVC server to set the JTAG tck rate for future shift operations. 

The shift message is the main command used between the client and the server to transfer low level JTAG vectors. The client will issue shift operations to determine the JTAG chain composition and then perform various JTAG instructions for instance driving pins or programming a device. The mrd and mwr messages can be used to read/write debug addresses.

In summary, XVC 1.1 protocol defines a simple JTAG communication method with sufficient capabilities for high level clients, like Xilinx Vivado and Vitis tools, to perform complex functions like programming and debug of devices. In this document is a basic description of the protocol. The intent of this document is to provide a blueprint for users of the XVC 1.1 protocol to create their own custom clients and servers.

## Protocol

The XVC 1.1 communication protocol consists of the following five messages:

```
getinfo:
mrd:<flags><address><num bytes>
mwr:<flags><address><num bytes><data>
settck:<period in ns>
shift:<num bits><tms vector><tdi vector>
```

For each message the client is expected to send the message and wait for a response from the server.  The server needs to process each message in the order received and promptly provide a response. Note that for the XVC 1.1 protocol only one connection is assumed so as to avoid interleaving locking and interleaving issues that may occur with concurrent client communication.

### MESSAGE: "getinfo:"

The primary use of "getinfo:" message is to get the XVC server version. The server version provides a client a way of determining the protocol capabilities of the server.

**Syntax**

Client Sends:
```
"getinfo:"
```

Server Returns:
```
“xvcServer_v1.1:<xvc_vector_len>\n”
```

Where:
```
<xvc_vector_len> is the max width of the vector that can be shifted 
                 into the server
```

### MESSAGE: "mrd:"

The primary use of "mrd:" message is to read an address. 

**Syntax**

Client Sends:
```
"mrd:<flags><address><num bytes>"
```

Server Returns:
```
“<data><status>"
```

Where:
```
<flags> ULEB128 bit field for future flag use
<address> ULEB128 starting address for memory read
<num bytes> ULEB128 number of bytes to read
<data> byte vector of data read
```

### MESSAGE: "mwr:"

The primary use of "mwr:" message is to write at an address. 

**Syntax**

Client Sends:
```
"mwr:<flags><address><num bytes><data>"
```

Server Returns:
```
“<status>"
```

Where:
```
<flags> ULEB128 bit field for future flag use
<address> ULEB128 starting address for memory write
<num bytes> ULEB128 number of bytes to write
<data> byte vector to write
```

### MESSAGE: "settck:"

The "settck:" message configures the server TCK period. When sending JTAG vectors the TCK rate may need to be varied to accommodate cable and board signal integrity conditions. This command is used by clients to adjust the TCK rate in order to slow down or speed up the shifting of JTAG vectors.

**Syntax:**

Client Sends:
```
"settck:<set period>"
```

Server Returns:
```
“<current period>”
```

Where:
```
<set period>      is TCK period specified in ns. This value is a little-endian 
                  integer value.
<current period>  is the value set on the server by the settck command. If 
                  the server cannot set the value then it will return the 
                  current value.
```

### MESSAGE: "shift:"

The "shift:" message is used to shift JTAG vectors in and out of a device. The number of bits to shift is specified as the first shift command parameter followed by the TMS and TDI data vectors. The TMS and TDI vectors are sized according to the number of bits to shift, rounded to the nearest byte. For instance if shifting in 13 bits the byte vectors will be rounded to 2 bytes. Upon completion of the JTAG shift operation the server will return a byte sized vector containing the sampled target TDO value for each shifted TCK clock.

**Syntax:**

Client Sends:
```
"shift:<num bits><tms vector><tdi vector>"
```

Server Returns:
```
“<tdo vector>”
```

Where:
```
<num bits>   : is a integer in little-endian mode. This represents the number 
               of TCK clk toggles needed to shift the vectors out
<tms vector> : is a byte sized vector with all the TMS shift in bits Bit 0 in 
               Byte 0 of this vector is shifted out first. The vector is 
               num_bits and rounds up to the nearest byte.
<tdi vector> : is a byte sized vector with all the TDI shift in bits Bit 0 in 
               Byte 0 of this vector is shifted out first. The vector is 
               num_bits and rounds up to the nearest byte.
<tdo vector> : is a byte sized vector with all the TDO shift out bits Bit 0 in 
               Byte 0 of this vector is shifted out first. The vector is 
               num_bits and rounds up to the nearest byte.
```

# Note
XVC server 1.1 for Versal performs reads and writes (*mrd* and *mwr*) as multi-word transactions. On some platforms performing accesses unaligned to 64-bits addresses may throw "Bus Error". In such cases, uncomment *ENABLE_SINGLE_WORD_RW* definition in *xvc_mem.c* to perform single word (32-bits) read/write transactions.