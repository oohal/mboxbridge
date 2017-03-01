Copyright 2016 IBM

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

# Intro
The autotools of this requires the autoconf-archive package for your
system

This is a protocol description using the mailbox registers on Aspeed 2400/2500
chips for managing the LPC Firmware space.

## Hardware Details

The mailbox consists of 16 (8 bit) data registers (see Layout for their use).
Mailbox interrupt enabling, masking and triggering is done using a pair of
control registers, one accessible by the host and the other by the BMC.
Interrupts can also be raised per write to each data register, for BMC and
host. Write tiggered interrupts are configured using two 8 bit registers where
each bit represents a data register and if an interrupt should fire on write.
Two 8 bit registers are present to act as a mask for write triggered
interrupts.

## Low Level Protocol Flow

The protocol itself consists of:

	1. Commands sent from the Host to the BMC
	2. Responses sent from the BMC to the Host
	3. Asyncronous events raised by the BMC.

## Sending commands (Host -> BMC)

To initiate a request the host must set a command code (see
Commands) into mailbox data register 0. It is also the hosts
responsibility to generate a unique sequence number into mailbox
register 1. After this any command specific data should be written
(see Layout). The host must then generate an interrupt to the BMC by
using bit 0 of its control register and wait for an interrupt in the
response. Generating an interrupt automatically sets bit 7 of the
corresponding control register. This bit can be used to poll for
messages.

On receiving an interrupt (or polling on bit 7 of its Control
Register) the BMC should read the message from the general registers
of the mailbox and perform the necessary action before responding. On
responding the BMC must ensure that the sequence number is the same as
the one in the request from the host. The BMC must also ensure that
mailbox data regsiter 13 is a valid response code (see Responses). The
BMC should then use its control register to generate an interrupt for
the host to notify it of a response.

## Sending Responses (BMC -> Host)

BMC to host communication is also possible for notification of events
from the BMC. This requires that the host have interrupts enabled on
mailbox data register 15 (or otherwise poll on bit 7 of mailbox status
register 1). On receiving such a notification the host should read
mailbox data register 15 to determine the event code was set by the
BMC (see BMC Event notifications in Commands for detail). After
performing the necessary action the host should send a BMC_EVENT_ACK
message to the BMC with which bit it has actioned.


# High Level Protocol Flow

The commands from the Host fall into rougly three categories:

	1. Informational
	2. Window management
	3. Write handling

The "window" refers to the part of the LPC FW space that has defined contents.
The Host cannot make any assumptions about the contents of the FW space outside
of the window and the Host must not write to a window that has been opened
for reading. The exact behaviour of accessing outside the window is explictly
undefined by the protocol to allow some flexibility in how the BMC implements
the protocol.

The Host opens a window using the ```CREATE_READ_WINDOW''' or
```CREATE_WRITE_WINDOW''' commands. There is only one window open at a time and
issuing a ```CREATE_*_WINDOW command''' will forcibly close the existing window
even if opening the new window fails. The Host specifies the location inside the
flash that it wishes to access for either reading or writing as a part of the
command, and may provide a hint indicating how large the window should be.
However, the BMC is free to ignore this. As a part of the response the BMC
provides the offset into the FW space where the window begins and the actual
size of the window.

## Read Handing

This window design allows read window requests to be serviced by exposing
the entire SPI Flash MMIO range on the LPC FW space. The advantage of the
window method over directly mapping the entire flash is that it allows us to
selectively overwrite parts of flash without reprogramming the backing storage.
This is convnienet 

## Write Handling

Write handling is somewhat more complicated because the BMC has no way to
determine when the Host has written into the FW space. The Host sends the
MARK_DIRTY command to notify the BMC which parts of the write window have
been updated. Once marked the BMC may write the window contents to permernant
storage (if required) at any time. Alternatively, the Host can force a flush
with the WRITE_FLUSH command, or by closing the window.

---

## Layout
```
Byte 0: COMMAND
Byte 1: Sequence
Byte 2-12: Arguments
Byte 13: Response code
Byte 14: Host controlled status reg
Byte 15: BMC controlled status reg
```
## Commands
```
RESET_STATE          1
GET_MBOX_INFO        2
GET_FLASH_INFO       3
CREATE_READ_WINDOW   4
CLOSE_WINDOW         5
CREATE_WRITE_WINDOW  6
MARK_WRITE_DIRTY     7
WRITE_FLUSH          8
BMC_EVENT_ACK        9
```
## Sequence
Unique message sequence number to be allocated by the host at the
start of a command/response pair. The BMC must ensure the responses to
a particular message contain the same sequence number that was in the
request from the host.

## Responses
```
SUCCESS       1
PARAM_ERROR   2
WRITE_ERROR   3
SYSTEM_ERROR  4
TIMEOUT       5
```

## Information
- All multibyte messages are LSB first(little endian)
- All responses must have a valid return code in byte 13

Only one window can be open at once.

### Commands in detail
```
	Command:
		RESET_STATE
		Data:
			-
		Response:
			-
		Notes:
			This command is designed to inform the BMC that it should put
			host LPC mapping back in a state where the SBE will be able to
			use it. Currently this means pointing back to BMC flash
			pre mailbox protocol. Final behavour is still TBD.


	Command:
		GET_MBOX_INFO
		Arguements:
			Args 0: API version

		Response:
			Args 0: API version
			Args 1-2: default read window size as number of blocks
			Args 3-4: default write window size as number of block
			Args 5: Block size as power of two.


	Command:
		CLOSE_WINDOW
		Arguments:
			-
		Response:
			-
		Notes:
			Closes active window. Renders the LPC mapping unusable.
			If the current window is a write window any dirty data
			will be flushed.

	Command:
		GET_FLASH_INFO
		Arguments:
			-
		Response:
			Args 0-3: Flash size in bytes
			Args 4-7: Erase granule in bytes

			XXX: Why is this in bytes?


	Command:
		CREATE_WRITE_WINDOW
		CREATE_READ_WINDOW
		Arguments:
			Args 0-1: Read window offset as number of blocks
			Args 2-3: Requested read window size in blocks. (v2)
		Response:
			Args 0-1: Start block of this window on the LPC bus
			Args 2-3: Actual size of the window in blocks. (v2)
		Notes:
			The requested window size is only a hint. The response
			indicates the actual size of the window. The BMC may
			want to use the requested size to pre-load the remainder
			of the request.

			The format of the CREATE_{READ,WRITE}_WINDOW commands
			are identical.


	Command:
		MARK_WRITE_DIRTY
		Data:
			Data 0-1: Where within window as number of blocks
				What the fuck? On what planet is <blocks, bytes> useful?
			Data 2-5: Number of dirty bytes
		Response:
			-
		Notes:
			The BMC has no method for intercepting writes that occur
			over the LPC bus. The Host must explicitly notify the
			where and when a write has occured so that it may flush
			it to persistence.

			This command accepts a <bytes, bytes> arguments rather
			than blocks to allow finer grained handling of writes.
			The BMC is free to mark entire blocks as dirty in response
			to this command.

			Where within the window is the index of the first dirty
			block within the window - zero refers to the first block of
			the mapping.
			This command marks bytes as dirty but does not nessesarily
			flush them to flash. It is expected that this command will
			respond quickly without actually performing a write to the
			backing store.


	Command:
		WRITE_FLUSH
		Data:
			-
		Response:
			-
		Notes:
			Flushes any data in the current window that has been
			marked dirty.

	Command:
		ERASE (v2 only)
		Data:
			0-1: Window block offset
			2-3: block count
		Response:
			-
		Notes:
			Some applications assume Flash-like semantics where
			writes must be preceeded by an erase. This command
			provides a fast method for erasing blocks so the Host
			does not have to emulate these semantics over the LPC.

			Additionally marks the blocks as dirty. The BMC may
			delay doing the actual erase until a FLUSH.

			XXX: Should we nail down this more and force the BMC to
			     set the contents of an erased block to 0xFFs. I
			     think that's what pflash expects.

			XXX: How does this interact with the window? Looks like
			     we're forcing the BMC to do a RMW cycle under the
			     hood if the write window is not aligned to the
			     erase granule. That's ok I suppose.

	Command:
		BMC_EVENT_ACK
		Data:
			Bits in the BMC status byte (mailbox data register 15) to ack
		Response:
			*clears the bits in mailbox data register 15*
			-
		Notes:
			The host will use this command to acknowledge BMC events
			supplied in mailbox register 15.


	BMC notifications:
		If the BMC needs to tell the host something then it simply
		writes to Byte 15. The host should have interrupts enabled
		on that register, or otherwise be polling it.
		 -[bit 0] BMC reboot. A BMC reboot informs the host that its
		  windows/dirty bytes/in flight commands will be lost and it
		  should attempt to reopen windows and rewrite any data it had
		  not flushed.
		Futhur details TBD
```
