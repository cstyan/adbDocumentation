# ADB Protocol Documentation
ADB (Android Debug Bridge) and it's protocol is what you're computer uses to communicate with Android devices.  The protocol itself is an [application layer](https://en.wikipedia.org/wiki/Application_layer) protocol, which can sit inside TCP or USB.  Googles documentation for the protocol and processes that use it can be found in their ADB code repository as **text** files:
* [protocol](https://android.googlesource.com/platform/system/core/+/master/adb/protocol.txt)
* [ADB overview](https://android.googlesource.com/platform/system/core/+/master/adb/OVERVIEW.TXT)
* [sync](https://android.googlesource.com/platform/system/core/+/master/adb/SYNC.TXT)

Hopefully this document will clear up the lack of actual documentation and details regarding the implementation of the ADB protocol.

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

Since the CONNECT message is supposed to have a format of `CONNECT(version, maxdata, "system-identity-string")`, you'd think it's safe to assume that since the packet has the `data_length` and `data_crc32`, that you can just append the actual data to the end of your node buffer / horrible C array, but no.  You can't.
`¯\_(ツ)_/¯`
See the packet capture images in the next section to see how you have to send data over USB to have ADB not die on you.

**NOTE:** You might be able to do the whole append *"data to the end of the packet as usual"* thing if you're using the ADB protocol over TCP.  As an [example](https://github.com/sidorares/node-adbhost), Andrey seems to be able to do this just fine over TCP.

##Packet Captures
To prove to you that I'm not lying here's some hexdumps of the packet capture I did to actually figure out how this thing works.

First, the whole packet + data structure that totally makes sense and should work:
![adb](https://github.com/cstyan/adbDocumentation/raw/master/images/cnxnHost.png)
Here you can see both the `CNXN` command as well as the `host::` string for the "system-identity" portion of our connection request.  But when you send this you never get a response from the device you sent to.

Here's what Googles own implementation of ADB does:
1. The CNXN command ![cnxn](https://github.com/cstyan/adbDocumentation/raw/master/images/googleCNXN.jpg) Notice that the 8 bytes are the same as the bytes previous to the `host::` bytes in the last hex dump.  This is from the `data_length` and `data_crc32` fields being set based on wanting to send `host::` as our data.
2. The "host::" system-identity string ![host](https://github.com/cstyan/adbDocumentation/raw/master/images/googleHost.png) ^ there's our actual data payload.  So you have to send twice for every command.  Such overhead, many packets!

To go from a state of `no connection established` to `you have an adb shell into your device` is about 40 USB packets. Efficiency!

## Handshake
Most protocols that are connection oriented have a handshake and actual documentation of their handshake process, such as [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment).  Unfortunately ADB does not, thanks Google.

But really, are you surprised?

According to the documnetation in that wonderful `protocol.txt` file:
> Both sides send a CONNECT message when the connection between them is
established.  Until a CONNECT message is received no other messages may
be sent.  Any messages received before a CONNECT message MUST be ignored.

That's really useful, we now totally know who needs to initiate the connection by sending a CONNECT message first right?  We also know that the devices response to our CONNECT message is going to be an AUTH message if the device is running Android 4.4 or higher, because that's totally documented.

So the actual handshake to get connected to a device, so you're in a state where you could run `adb shell` if you wanted, is as follows:
1. We send a CONNECT message to the device
2. We send our system-information string to the device
3. The device sends you an AUTH messagea
4. The device sends you the token you can sign
5. Now you have two options
  * Sign the token with your private key and send it back with an AUTH type 2 **OR**
  * Send the device your public key with an AUTH type 3, this option will open the "trust this computer" prompt on the device.
6. The device accepts your signed token or public key and sends back it's own CONNECT message

**HOORAY YOU CONNECTED TO A DEVICE** ![shark](https://github.com/cstyan/adbDocumentation/raw/master/images/shark.jpg)

