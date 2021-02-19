# ADB Protocol Documentation
ADB (Android Debug Bridge) and its protocol is what your computer uses to 
communicate with Android devices. The protocol itself is an 
[application layer](https://en.wikipedia.org/wiki/Application_layer) protocol, 
which can sit inside TCP or USB. The AOSP (Android Open Source Project) 
documentation for the protocol and processes that use it can be found in their 
ADB code repository:
* [protocol](https://android.googlesource.com/platform/packages/modules/adb/+/master/protocol.txt)
* [ADB overview](https://android.googlesource.com/platform/packages/modules/adb/+/master/OVERVIEW.TXT)
* [sync](https://android.googlesource.com/platform/packages/modules/adb/+/master/SYNC.TXT)
* [services](https://android.googlesource.com/platform/packages/modules/adb/+/master/SERVICES.TXT)

The issue with the documentation provided by the AOSP is that it's severely 
lacking in details around the implementation of the protocol. These details are 
things anyone familiar with common communication protocols would expect to see in the 
documentation of a protocol.

For example: The documentation regarding making a connection to the device consists
of:
> Both sides send a CONNECT message when the connection between them is
  established. Until a CONNECT message is received no other messages may
  be sent. Any messages received before a CONNECT message MUST be ignored.

This is not enough information for us to perform the handshake required to
connect to a device. From this it sounds like each side just sends CONNECT endlessly
until it receives a CONNECT from the other side. This is not at all how the 
actual handshake takes place.

Hopefully this document will clear up ambiguous documentation and 
details regarding the implementation of the ADB protocol for those who are interested.
The packet capture I refer to that was captured during a connection sequence 
using Googles ADB implementation is in this repo under `adbCapture.pcapng`.

My aim is for this document to eventually be a suitable replacement for other
documentation out there about the ADB protocol

NOTE: The majority of the information presented is for use of the ADB protocol
with USB. If you have information about it's use with TCP please submit a
pull request. :smile:

## Disclaimer
I like protocols, and reverse engineering ADB was a lot of fun. My comments are representations of 
my immediate reactions to certain aspects of ADB. I'm sure there were actual rational decisions behind 
most of the things I poke fun at in this doc.

## Packet Format
ADB packets:
> unsigned command;       /* command identifier constant        */  
  unsigned arg1;          /* first argument                   */  
  unsigned arg2;          /* second argument                  */  
  unsigned data_length;   /* length of payload (0 is allowed) */  
  unsigned data_crc32;    /* crc32 of data payload            */  
  unsigned magic;         /* command ^ 0xffffffff             */

Having different possible meanings for the value of `arg1` and `arg2` based on what type of 
`command` we're sending can be confusing. It also means that any error messages are passed 
back as strings and we would have to parse strings in order to react to those errors.

But where is the data you ask?  
![shark](https://github.com/cstyan/adbDocumentation/raw/master/images/shark.jpg)

ADB over USB expects to receive an ADB packet with the fields listed above, followed 
by another USB packet with any data payload associated with that ADB packet. Again, 
there is no mention of this in any of the AOSP's ADB documentation.

Since the CONNECT message is supposed to have a format of `CONNECT(version, maxdata, 
"system-identity-string")`, you'd think it's safe to assume that since the packet 
has the `data_length` and `data_crc32`, that you can just append the actual data 
to the end of your node buffer / C array.  
But no, you can't. `¯\_(ツ)_/¯` 

Keep in mind that when you're sending packets in ADB, most fields need to be 
written in little endian byte order as well as read back from the device in
little endian. This can be confusing initially if you're looking at packet capture
hex dumps to determine the sequence of some ADB command.

**NOTE:** You might be able to do the whole *"append data to the end of the packet 
as usual"* thing if you're using the ADB protocol over TCP. As an 
[example](https://github.com/sidorares/node-adbhost), Andrey seems to be able to
 do this just fine over TCP.

## Packet Captures
To prove to you that I'm not lying here's some hexdumps of the packet capture I 
did to actually figure out how this thing works. 

First, the whole packet + data structure that totally makes sense and should work: 
![adb](https://github.com/cstyan/adbDocumentation/raw/master/images/cnxnHost.png)  
Here you can see both the `CNXN` command as well as the `host::` string for the 
"system-identity" portion of our connection request. But when you send this you 
never get a response from the device you sent to.  

---

Here's what the AOSP's own implementation of ADB does:  
 
1. The CNXN command  
![cnxn](https://github.com/cstyan/adbDocumentation/raw/master/images/googleCNXN.jpg)  
Notice that the 8 bytes are the same as the bytes previous to the `host::` bytes 
in the last hex dump. This is from the `data_length` and `data_crc32` fields 
being set based on wanting to send `host::` as our data.    
2. The `host::` system-identity string
![host](https://github.com/cstyan/adbDocumentation/raw/master/images/googleHost.png)  
^ there's our actual data payload. So you have to send twice for every command.  

To go from a state of `no connection established` to `you have an adb shell into 
your device` is about 40 USB packets. Efficiency :thumbsup:

## Handshake
Most protocols that are connection oriented have a handshake and actual 
documentation of their handshake process, such as 
[TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment).  
Unfortunately ADB does not.  

But really, are you surprised?

According to the documentation in that wonderful `protocol.txt` file:
> Both sides send a CONNECT message when the connection between them is
established. Until a CONNECT message is received no other messages may
be sent. Any messages received before a CONNECT message MUST be ignored.

That's really useful, it's clear who needs to initiate the connection by sending 
a CONNECT message first right? It's also clear that the device's response to our 
CONNECT message is going to be an AUTH message if the device is running Android 
4.4 or higher, because that's clearly documented as part of the protocol. :angry: 

So the actual handshake to get connected to a device, so you're in a state where 
you could run `adb shell` if you wanted, is as follows:

1. We send a CONNECT message to the device
2. We send our system-information string to the device
3. The device sends you an AUTH message
4. The device sends you the token you can sign
5. Now you have two options
  - Sign the token with your private key and send it back with an AUTH type 2 **OR**
  - Send the device your public key with an AUTH type 3, this option will open 
  the "trust this computer" prompt on the device.
6. The device accepts your signed token or public key and sends back it's own 
CONNECT message
NOTE: if for any reason you send back the AUTH packet with your public key and the
device does not respond, a retransmission of the AUTH/public key does not appear
to work, and you must restart the connection process from step 1.
7. Device sends some information about itself

You can now send normal messages to your device.

## Sending Messages & ADB Shell
After that handshake business we can start doing other things, like opening a 
shell on the device.

Command type OPEN lets you open another stream into the device, in this case 
we're opening a shell stream.  
Command type WRTE is for sending any old data across as part of that stream.

The documentation doesn't like to be consistent, and defines a message type of 
READY, but the string you send as the COMMAND field of the packet is OKAY.
I'm going to refer to this as an OKAY message. If we're sending OPEN/WRTE then 
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
message requires a `:` between the type of stream we want to open and the
rest of the commands string. As an example, if we wanted to send a `shell ls -al` 
command, the data payload as part of our OPEN message needs to be `shell:ls -al`.

So this means that any stream we want to open to the device has the following 
format: `streamType:options.`. For example `adb reboot` would be `reboot:`, 
`adb shell rm -rf /` would be `shell:rm -rf /`, etc.

## Sync Commands
Besides the 7 command types listed in the documentation for ADB there are a number 
of 'undocumented' sync command types. You can think of these as sub commands, as they 
come as the data payload for another command. These sub commands are used to signal 
the device about the next thing we want to do, or information we want it to send us.

There is some information about these commands in the 
[sync](https://android.googlesource.com/platform/system/core/+/master/adb/SYNC.TXT) 
documentation but as usual the documentation is incomplete.

The sub commands include: SEND, RECV, DATA, STAT, and QUIT. There may be others.

For example say we want to use the `adb push` command, the protocol nests STAT
and SEND within WRTE commands during the transfer of the data, and our host machine
will nest a QUIT inside a final WRTE in order to signal the end of the transfer.
The `adb pull` command works similarly except that there is a RECV nested inside a 
WRITE rather than a SEND, we also recv a DATA + file data inside of another WRTE.

DATA is the subcommand that is the exception to the `command first, then another
packet with the data payload` rule, though technically the DATA command is part of
the data payload for another command.

The STAT sub command is used to get file attributes.
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
2. We send sync: to the device `sync: starts a SYNC service `
3. Device sends us OKAY
4. We send WRTE message to device
5. We send STAT to the device
6. Device sends us OKAY
7. We send WRTE to device
8. We send the destination of where we want to push a file to, `sdcard/`
9. Device sends us OKAY
10. Device sends us WRTE
11. Device sends us STAT + some info about the destination we're sending to 
`sdcard/` 
12. We send OKAY to device
13. We send WRTE to device
14. We send SEND to device, note that there is another 4 bytes in the data payload 
of this packet which is the length of the file destination + name in characters plus
the ',mode' portion from 17. So if we're sending `sdcard//testFile.txt,XXXXX` we
write SEND26.
The ADB protocol is full of inconsistencies, in case you hadn't already noticed.
15. Device sends us OKAY
16. We send WRTE to device
17. We send string about the file we're sending `sdcard//testFile.txt,XXXXX,DATAnnnnTheFileData`  
    - This string can be confusing at first glance, to clarify, the format is:
    full file path, the mode of the file in decimal (0644 becomes 33188),
    `DATAnnnnTheFileData` where nnnn is the size of the file sending, each n is one 
    byte.  
    - If your file is larger than 64k bits you just need to keep sending WRTE with
    file data followed by another `DATA nnnnFileData` until you've sent all the file data. 
    - When we're sending the packet containing the last of the file data we append 
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
The flow of an ADB pull starts of like the push flow up until step 8:  

8. We send full path of file we want to pull `sdcard/someFile.txt`
9. Device sends us OKAY
10. Device sends us WRTE
11. Device sends us STAT with stats about the file we want to pull
12. We send OKAY to device
13. We send WRTE to device
14. We send RECV to device
15. Device sends us OKAY
16. We send WRTE to the device
17. We send the path of the file we want again `sdcard/someFile.txt`
18. Device sends us OKAY
19. Device sends us WRTE
20. Device sends us DATA + data length + the file data, if the file is more than 
64k there will be multiple DATA messages.  The last file data transfer will be 
terminated with DONE after the file data portion of the payload.
21. We send the device OKAY after each DATA message
22. At the end of the data transfer we send WRTE to the device
23. We send QUIT message to device
24. Device sends us OKAY
25. We send CLSE message to device
26. Device sends us CLSE

## ADB List

This command is not available through the `adb` command. It lists files in a folder. It is more reliable than parsing the results of `adb shell 'ls -al <remote dir>'` - see http://mywiki.wooledge.org/ParsingLs.

NOTE: I've been in contact with someone at Google who passed on this message from an engineer who works on Android:  
> "adb list" is available through `adb`, it's just called `ls`: `adb ls/`, for example (as opposed to `adb shell ls /`).

1. We send OPEN message to device
2. We send sync: to the device `sync: starts a SYNC service `
3. Device sends us OKAY
4. We send WRTE to device
5. We send LIST to device and the length (in characters) of the remote path we want to list
6. Device sends OKAY
7. We send WRTE to device
8. We send the remote path we want to list e.g. `sdcard/`
9. Device sends OKAY
10. We send WRTE to device
11. We send RECV to device
12. Device sends a series of "dents" (directory entries), across an arbitrary number of packets of arbitrary length - sometimes the packets are only 1 byte long. It will send OKAY when the next packet is ready. We keep returning to 10. (send WRTE, RECV, read DENT) until we receive DONE. "dents" can be split across several packets, but are the following structure: 
    1. A four-byte response id DENT (or DONE, if there are no more files to list. DONE is sent immediately if the remote path does not exist or if it is not a folder)
    2. A four-byte integer representing file mode - first 9 bits of this mode represent the file permissions, as with chmod mode. Bits 14 to 16 seem to represent the file type, one of `0b100` (file), `0b010` (directory), `0b101` (symlink)
    3. A four-byte integer representing file size.
    4. A four-byte integer representing last modified time in seconds since Unix Epoch.
    5. A four-byte integer representing file name length.
    6. A utf-8 string representing the file name.
    7. Back to 1.
18. Device sends DONE. DONE can appear anywhere within the packet, either at the start, middle, or end. Sometimes the rest of the packet after DONE is padded with 0s.
19. We send OKAY to device
20. We send WRTE to device
21. We send QUIT to device
22. Device sends us OKAY
23. We send CLSE to device
24. Device sends us CLSE

## ADB Install
Install is just a combination of pushing an APK to /data/local/tmp and then running
the shell command `pm install /data/local/temp/apkName.apk`.

## ADB Reboot
Just open a stream with the command `reboot:`.

## Authentication
The AOSP thought it was cool to use their own crypto library in order to sign tokens 
used in the authentication process, even though they're using standard RSA keys. 
If you want to be able to authenticate with a device instead of sending it your public 
key everytime you make a connection, you'll need to use their library. Or reimplement it
yourself.

The library is called libmincrypt, and can be found [here](https://android.googlesource.com/platform/system/core/+/master/libmincrypt/).
