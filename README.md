# ADB Protocol Documentation
ADB (Android Debug Bridge) and it's protocol is what you're computer uses to 
communicate with Android devices.  The protocol itself is an 
[application layer](https://en.wikipedia.org/wiki/Application_layer) protocol, 
which can sit inside TCP or USB.  Googles documentation for the protocol and 
processes that use it can be found in their ADB code repository as **text** files:
* [protocol](https://android.googlesource.com/platform/system/core/+/master/adb/protocol.txt)
* [ADB overview](https://android.googlesource.com/platform/system/core/+/master/adb/OVERVIEW.TXT)
* [sync](https://android.googlesource.com/platform/system/core/+/master/adb/SYNC.TXT)

Hopefully this document will clear up the lack of actual documentation and 
details regarding the implementation of the ADB protocol.
The packet capture I refer to that was captured during a connection sequence 
using Googles ADB implementation is in this repo under `adbCapture.pcapng`.

## Packet Format
ADB packets, they kind of suck.  
>   unsigned command;       /* command identifier constant        */  
    unsigned arg0;          /* first argument                   */  
    unsigned arg1;          /* second argument                  */  
    unsigned data_length;   /* length of payload (0 is allowed) */  
    unsigned data_crc32;    /* crc32 of data payload            */  
    unsigned magic;         /* command ^ 0xffffffff             */
    
First argument, second argument? Okay, I guess there is no point in having tons of fields with `null` values for half depending on the type of command we're sending.  

But where is the data you ask?  I don't even know. 

Since the CONNECT message is supposed to have a format of `CONNECT(version, maxdata, 
"system-identity-string")`, you'd think it's safe to assume that since the packet 
has the `data_length` and `data_crc32`, that you can just append the actual data 
to the end of your node buffer / horrible C array.  
But no, you can't. `¯\_(ツ)_/¯`  

See the packet capture images in the next section to see how you have to send 
data over USB to have ADB not die on you.  

**NOTE:** You might be able to do the whole append *"data to the end of the packet 
as usual"* thing if you're using the ADB protocol over TCP.  As an 
[example](https://github.com/sidorares/node-adbhost), Andrey seems to be able to
 do this just fine over TCP.

##Packet Captures
To prove to you that I'm not lying here's some hexdumps of the packet capture I 
did to actually figure out how this thing works. 

First, the whole packet + data structure that totally makes sense and should work: 
![adb](https://github.com/cstyan/adbDocumentation/raw/master/images/cnxnHost.png)  
Here you can see both the `CNXN` command as well as the `host::` string for the 
"system-identity" portion of our connection request.  But when you send this you 
never get a response from the device you sent to.  

---

Here's what Googles own implementation of ADB does:  
 
1. The CNXN command !
[cnxn](https://github.com/cstyan/adbDocumentation/raw/master/images/googleCNXN.jpg) 
Notice that the 8 bytes are the same as the bytes previous to the `host::` bytes 
in the last hex dump.  This is from the `data_length` and `data_crc32` fields 
being set based on wanting to send `host::` as our data.    
2. The `host::` system-identity string !
[host](https://github.com/cstyan/adbDocumentation/raw/master/images/googleHost.png)  
^ there's our actual data payload.  So you have to send twice for every command.  
Such overhead, many packets!

To go from a state of `no connection established` to `you have an adb shell into 
your device` is about 40 USB packets. Efficiency!

## Handshake
Most protocols that are connection oriented have a handshake and actual 
documentation of their handshake process, such as 
[TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment).  Unfortunately ADB does 
not, thanks Google.  

But really, are you surprised?

According to the documnetation in that wonderful `protocol.txt` file:
> Both sides send a CONNECT message when the connection between them is
established.  Until a CONNECT message is received no other messages may
be sent.  Any messages received before a CONNECT message MUST be ignored.

That's really useful, it's clear who needs to initiate the connection by sending 
a CONNECT message first right?  It's also clear that the device's response to our 
CONNECT message is going to be an AUTH message if the device is running Android 
4.4 or higher, because that's clearly documented as part of the protocol. :angry: 

So the actual handshake to get connected to a device, so you're in a state where 
you could run `adb shell` if you wanted, is as follows:

1. We send a CONNECT message to the device
2. We send our system-information string to the device
3. The device sends you an AUTH messagea
4. The device sends you the token you can sign
5. Now you have two options
  - Sign the token with your private key and send it back with an AUTH type 2 **OR**
  - Send the device your public key with an AUTH type 3, this option will open 
  the "trust this computer" prompt on the device.
6. The device accepts your signed token or public key and sends back it's own CONNECT message
7. Device sends some information about itself

You can now send normal messages to your device.

## Sending Messages
After that handshake business we can start doing other things, like opening a 
shell on the device.

Command type OPEN lets you open another stream into the device, in this case we're opening a shell stream.  
Command type WRTE is for sending any old data across as part of that stream.

The documentation doesn't like to be consistent, and defines a message type of 
READY, but the string you send as the COMMAND field of the packet is OKAY.
I'm going to refer to this as an OKAY message.  If we're sending OPEN/WRTE then 
we must wait for a response of OKAY before it can send again, otherwise the 
device will disconnect from us because this protocol handles the receiving of 
out-of-order data gracefully.

So, if we want to open a shell after the handshake we would have another flow 
that's something like:

1. We send OPEN message to device
2. We send shell string to device
3. Device sends OKAY message back to us
4. Device sends WRTE message to us
5. Device sends as string that's the shells terminal prompt
6. We send OKAY to the device

Note that the end of whatever string you're sending the device as part of an OPEN
message seems to require a `.` at the end of the string.  As an example, if we
wanted to send a `shell ls` command, the data payload as part of our OPEN message
needs to be `shell:ls.`.

## Undocumented Commands
Besides the 7 command types listed in the documentation for ADB there are a number of
undocumented command types.  You can think of these as sub commands, as they come as
the data payload for another command.  These sub commands are used to signal the device 
about the next thing we want to do, or information we want it to send us.

The sub commands include: SEND, RECV, STAT, QUIT

For example say we want to use the `adb push` command, the protocol nests STAT
and SEND within WRTE commands during the transfer of the data, and our host machine
will nest a QUIT inside a final WRTE in order to signal the end of the transfer.
The `adb pull` command works similar except that there is a RECV nested inside a WRITE
rather than a SEND, we also recv a DATA + file data inside of another WRTE.

The STAT sub command is used to get file attributes.  More info about the next
section is available [here](http://blogs.kgsoft.co.uk/2013_03_15_prg.htm).
```
typedef struct _rf_stat__ 
{
    unsigned id;    // ID_STAT('S' 'T' 'A' 'T')
    unsigned mode;
    unsigned size;
    unsigned time;
} FILE_STAT;
```

## ADB Push
1. We send OPEN message to device
2. We send sync: to the device `sync: is related flushing on the device`
3. Device sends us OKAY
4. We send WRTE message to device
5. We send STAT to the device
6. Device sends us OKAY
7. We send WRTE to device
8. We send the path of where we want to push a file to, `sdcard/testFile.txt`
9. Device sends us OKAY
10. We send WRTE to the device
11. We send STAT to check if the file already exists at the path from step 8
12. Device sends us OKAY
13. We send WRTE to device
14. We send SEND to device
15. Device sends us OKAY
16. We send WRTE to device
17. We send string about the file we're sending `sdcard/testFile.txt,XXXXX,DATAnnnnTheFileData`  
    - This string can be confusing at first glance, to clarify, the format is:
    full file path, the mode of the file in decimal (0644 becomes 33188),
    `DATAnnnnTheFileData` where nnnn is the size of the file sending, each n is one 
    byte.  
    - If your file is larger than 64k bits you just need to keep sending WRTE 
    followed by another `DATA nnnnFileData` until you've sent all the file data. 
    - When we're sending the the packet containing the last of the file data we append 
    `DONEnnnn` to the end of the packet, where `nnnn` is the creation time we want 
    the file to have on the device. 
18. Device sends us OKAY ` NOTE: here were assuming we just sent a data packet 
that also contained DONE, indicating we've sent all the data.`
19. Device sends us WRTE
20. Device sends us OKAY
21. We send OKAY to device
22. We send WRTE to device
23. We send QUIT to device
24. Device sends us OKAY
25. We send CLSE to device
26. Device sends us CLSE

## ADB Pull
The flow of an ADB pull starts of like the pull flow up until step 8:  

8. We send full path of file we want to pull * sdcard/someFile.txt *
9. Device sends us OKAY
10. We send WRTE to device
11. We send STAT to the device, this contains some more data but I'm not sure
what the data actually is.  The link above seems to suggest the data would be
the size of the filename among other things, but it doesn't make sense to send
the size of the filename after we've already sent the filename.
12. Device sends us OKAY
13. We send WRTE message to device
14. We send RECV message to device
15. Device sends us OKAY
16. We send WRTE to the device
17. We send the path of the file we want again * sdcard/someFile.txt *
18. Device sends us OKAY
19. Device sends us WRTE
20. Device sends us DATA + data length + the file data, if the file is more than 
65k there will be multiple DATA messages.  The last file data transfer will be 
terminated with DONE after the file data portion of the payload.
21. We send the device OKAY after each DATA message
22. At the end of the data transfer we send WRTE to the device
23. We send QUIT message to device
24. Device sends us OKAY
25. We send CLSE message to device
26. Device sends us CLSE
