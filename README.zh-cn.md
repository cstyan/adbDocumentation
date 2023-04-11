# ADB协议文档

ADB（Android 调试桥）及其协议是您的计算机与 Android 设备通信时使用的内容。该协议本身是一个[应用层](https://en.wikipedia.org/wiki/Application_layer)协议，可嵌入 TCP 或 USB 中。AOSP（Android 开放源代码项目）有关该协议及使用它的过程的文档可以在他们的 ADB 代码存储库中找到：

- [协议](https://android.googlesource.com/platform/packages/modules/adb/+/master/protocol.txt)
- [ADB 概述](https://android.googlesource.com/platform/packages/modules/adb/+/master/OVERVIEW.TXT)
- [同步](https://android.googlesource.com/platform/packages/modules/adb/+/master/SYNC.TXT)
- [服务](https://android.googlesource.com/platform/packages/modules/adb/+/master/SERVICES.TXT)

AOSP提供的文档存在问题，就是在协议实现方面缺乏详细说明。这些细节是任何熟悉常见通信协议的人都希望在协议的文档中看到的。

有关如何连接设备的文档包括：

> 当它们之间建立连接时，双方都会发送CONNECT消息。在接收到CONNECT消息之前，不能发送任何其他消息。在收到CONNECT消息之前接收到的所有消息都必须被忽略。
> 

这段信息不足以进行与设备连接所需的握手。从中可以得出，每一方仅不断地发送CONNECT消息，直到接收到另一方的CONNECT消息。但这并不是实际的握手过程。

希望本文档能够澄清ADB协议实现方面的模糊文档和细节，以供有兴趣的人参考。我提到的数据包捕获是在使用Google ADB实现进行连接序列期间捕获的，可在此存储库中的`adbCapture.pcapng`下找到。

我的目标是最终将本文档作为ADB协议的其他文档的合适替代品

> **warning** : 大部分所呈现的信息是关于使用USB的ADB协议。如果您有关于使用TCP的信息，请提交一个pull request。😊
> 

## 免责声明

我喜欢协议，反向工程ADB非常有趣。我的评论代表了我对ADB某些方面的直接反应。我相信在本文中我嘲笑的大多数事情背后都有实际的合理决策。

## 数据包格式

ADB packets: 

```
unsigned command; /* command identifier constant */
unsigned arg1; /* first argument */
unsigned arg2; /* second argument */
unsigned data_length; /* length of payload (0 is allowed) */
unsigned data_crc32; /* crc32 of data payload */
unsigned magic; /* command ^ 0xffffffff */
``` 

根据我们发送的命令类型，对于`arg1`和`arg2`的值有不同的可能含义，这可能会令人困惑。这也意味着任何错误消息都以字符串形式传递，我们必须解析字符串才能对这些错误做出反应。

但你问数据在哪里？

![https://github.com/cstyan/adbDocumentation/raw/master/images/shark.jpg](https://github.com/cstyan/adbDocumentation/raw/master/images/shark.jpg)

ADB通过USB接收列在上面的ADB数据包，后跟任何与该ADB数据包相关联的数据负载的另一个USB数据包。再次强调，AOSP的ADB文档中没有提到这一点。

由于CONNECT消息的格式应为`CONNECT(version，maxdata，“system-identity-string”)`，因此您可能认为可以安全地假设，由于数据包具有`data_length`和`data_crc32`，因此可以将实际数据附加到节点缓冲区/ C数组的末尾。但不，你不能。 `¯\\_(ツ)_/¯`

请记住，在ADB中发送数据包时，大多数字段都需要按照**小端字节顺序**编写，同时需要从设备以小端字节顺序读取。如果您正在查看数据包捕获十六进制转储以确定某些ADB命令的顺序，则这可能会令人困惑。

> **warning:** 如果您正在使用TCP的ADB协议，则可以像往常一样将数据附加到数据包的末尾。例如，Andrey似乎可以在TCP上很好地完成这项任务（[https://github.com/sidorares/node-adbhost）。](https://github.com/sidorares/node-adbhost%EF%BC%89%E3%80%82)
> 

## Packet Captures

为了向您证明我没有撒谎，这里是一些十六进制转储，这是我进行的数据包捕获，以实际了解这个东西的工作原理。

首先，整个数据包+数据结构是完全合理的，应该可以工作：

![https://github.com/cstyan/adbDocumentation/raw/master/images/cnxnHost.png](https://github.com/cstyan/adbDocumentation/raw/master/images/cnxnHost.png)

在这里，您可以看到以下内容：

```
CNXN
```

以及命令

```
host::
```

在我们的连接请求中，“system-identity”部分的字符串是必需的。但是，当您发送此字符串时，您从您发送的设备中永远不会收到响应。

---

以下是AOSP自己的ADB实现方式：

1. CNXN 命令
请注意，这 8 个字节与上一个十六进制转储中 `host::` 字节之前的字节相同。这是因为 `data_length` 和 `data_crc32` 字段是基于想要将 `host::` 作为我们的数据发送而设置的。
    
    ![https://github.com/cstyan/adbDocumentation/raw/master/images/googleCNXN.jpg](https://github.com/cstyan/adbDocumentation/raw/master/images/googleCNXN.jpg)
    
2. `host::` 是系统标识字符串，这是我们实际的数据负载。所以每个命令都需要发送两次。 @`
    
    ![https://github.com/cstyan/adbDocumentation/raw/master/images/googleHost.png](https://github.com/cstyan/adbDocumentation/raw/master/images/googleHost.png)
    

从“未建立连接”状态到“你已经进入设备的adb shell”状态大约需要40个USB数据包。效率: 👍

## 握手

大多数面向连接的协议都有握手和握手过程的实际文档，例如[TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment).
不幸的是，ADB没有。

但你真的感到惊讶吗？

根据那个美妙的`protocol.txt`文件中的文档：

> 当它们之间的连接建立时，双方都会发送CONNECT消息。在收到CONNECT消息之前，不能发送任何其他消息。必须忽略任何在CONNECT消息之前收到的消息。
> 

这真的很有用，很清楚谁需要通过首先发送CONNECT消息来启动连接，对吧？如果设备运行Android 4.4或更高版本，则很明显设备的响应是一个AUTH消息，因为它在协议中被清楚地记录下来。 😡

因此，获取连接到设备的实际握手，以便在需要时运行`adb shell`，如下所示：

1. 我们向设备发送CONNECT消息
2. 我们向设备发送我们的系统信息字符串
3. 设备向您发送AUTH消息
4. 设备向您发送您可以签名的令牌
5. 现在您有两个选择
- 使用私钥签名令牌并使用AUTH类型2将其发送回**OR**
- 使用AUTH类型3将您的公钥发送给设备，此选项将在设备上打开“信任此计算机”提示。
1. 设备接受您的签名令牌或公钥，并发送回自己的CONNECT消息。注意：如果由于任何原因您发送带有公钥的AUTH数据包并且设备没有响应，则AUTH/公钥的重传似乎不起作用，您必须从步骤1重新启动连接过程。
2. 设备发送有关自身的一些信息

现在，您可以向设备发送普通消息。

## 发送消息和ADB Shell

完成握手后，我们可以开始做其他事情，比如在设备上打开一个 shell。

通过 OPEN 命令类型，您可以打开设备中的另一个流，这种情况下我们正在打开一个 shell 流。
WRTE 命令类型用于发送任何旧数据作为该流的一部分。

文档不喜欢保持一致，并定义了一个 READY 消息类型，但您作为数据包 COMMAND 字段发送的字符串为 OKAY。我将称之为 OKAY 消息。如果我们发送 OPEN/WRTE，则必须在发送之前等待 OKAY 响应，否则设备将与我们断开连接，因为该协议能够优雅地处理乱序数据的接收。

因此，如果我们想在握手后打开 shell，则会有另一个流程，如下所示：

1. 我们向设备发送 OPEN 消息
2. 我们向设备发送 shell 字符串
3. 设备向我们发送 OKAY 消息
4. 设备向我们发送 WRTE 消息
5. 设备发送作为 shell 终端提示符的字符串
6. 我们向设备发送 OKAY

请注意，在作为 OPEN 消息的一部分向设备发送的任何字符串末尾，需要在所需打开的流类型和命令字符串之间加上 `:`。例如，如果我们想发送 `shell ls -al` 命令，则作为 OPEN 消息的数据有效负载需要是 `shell:ls -al`。

因此，我们要打开到设备的任何流具有以下格式：`streamType:options.`。例如，`adb reboot` 将是 `reboot:`，`adb shell rm -rf /` 将是 `shell:rm -rf /`，等等。

同步命令

除了 ADB 文档中列出的 7 种命令类型外，还有许多“未记录”的同步命令类型。您可以将这些视为子命令，因为它们作为另一个命令的数据有效负载而来。这些子命令用于向设备发信号，告诉它我们要做下一件事情或者请求它发送信息给我们。

[同步](https://android.googlesource.com/platform/system/core/+/master/adb/SYNC.TXT)文档中有关于这些命令的一些信息，但通常文档是不完整的。

子命令包括：SEND、RECV、DATA、STAT 和 QUIT。可能还有其他类型的子命令。

例如，假设我们想使用 `adb push` 命令，该协议在数据传输期间将 STAT 和 SEND 嵌套在 WRTE 命令中，我们的主机将在最后的 WRTE 中嵌套 QUIT，以表示传输的结束。`adb pull` 命令类似地工作，只是在 WRITE 中嵌套了 RECV，我们还在另一个 WRTE 中接收了 DATA + 文件数据。

DATA 是与“首先发送命令，然后是带有数据有效负载的另一个数据包”的规则不同的子命令，尽管从技术上讲，DATA 命令是另一个命令的数据有效负载的一部分。

STAT 子命令用于获取文件属性。

```
typedef struct _rf_stat__
{
    unsigned id;    // ID_STAT('S' 'T' 'A' 'T')
    unsigned mode;
    unsigned size;
    unsigned time;
} FILE_STAT;
```

## ADB PUSH

1. 我们向设备发送OPEN消息
2. 我们向设备发送sync：`sync：启动SYNC服务`
3. 设备向我们发送OKAY
4. 我们向设备发送WRTE消息
5. 我们向设备发送STAT
6. 设备向我们发送OKAY
7. 我们向设备发送WRTE
8. 我们发送我们要将文件推送到的目标位置，`sdcard/`
9. 设备向我们发送OKAY
10. 设备向我们发送WRTE
11. 设备向我们发送STAT +有关我们正在发送到的目标位置`sdcard/`的一些信息
12. 我们向设备发送OKAY
13. 我们向设备发送WRTE
14. 我们向设备发送SEND，请注意，此数据包的数据负载中还有另外4个字节，即文件目标+名称的字符长度加上来自17的“，mode”部分。如果我们发送`sdcards//testFile.txt，XXXXX`，则写入SEND26。如果您还没有注意到，ADB协议充满了不一致之处。
15. 设备向我们发送OKAY
16. 我们向设备发送WRTE
17. 我们发送关于我们正在发送的文件的字符串 `sdcard//testFile.txt,XXXXX,DATAnnnnTheFileData`
    - 这个字符串可能一开始看起来有些困惑，为了澄清，格式为: 完整的文件路径，文件的模式用十进制表示 (0644 变成 33188)，`DATAnnnnTheFileData`，其中 nnnn 是发送的文件大小，每个 n 是一个字节。
    - 如果您的文件大于 64k bits，您只需要继续发送带有文件数据的 WRTE，后跟另一个 `DATA nnnnFileData`，直到您发送了所有文件数据。
    - 当我们发送包含最后文件数据的数据包时，我们在数据包的末尾追加 `DONEnnnn`，其中 `nnnn` 是我们希望文件在设备上拥有的创建时间。
18. 设备向我们发送 OKAY `注意: 这里我们假设我们刚刚发送了一个包含 DONE 的数据包，表明我们已经发送了所有数据。`
19. 设备向我们发送 WRTE
20. 设备向我们发送 OKAY
21. 我们向设备发送 OKAY
22. 我们向设备发送 WRTE
23. 我们向设备发送 QUIT
24. 设备向我们发送 OKAY
25. 我们向设备发送 CLSE
26. 设备向我们发送 CLSE

## ADB Pull

ADB pull 的流程与 push 流程类似，直到第 8 步：

1. 我们发送要拉取的文件的完整路径 `sdcard/someFile.txt`
2. 设备发送 OKAY 给我们
3. 设备发送 WRTE 给我们
4. 设备发送包含我们要拉取的文件信息的 STAT 给我们
5. 我们向设备发送 OKAY
6. 我们向设备发送 WRTE
7. 我们向设备发送 RECV
8. 设备向我们发送 OKAY
9. 我们向设备发送 WRTE
10. 我们再次发送要拉取的文件路径 `sdcard/someFile.txt`
11. 设备发送 OKAY 给我们
12. 设备发送 WRTE 给我们
13. 设备发送 DATA + 数据长度 + 文件数据，如果文件大于 64k，则会有多个 DATA 消息。最后一个文件数据传输在载荷的文件数据部分后用 DONE 终止。
14. 我们在每个 DATA 消息后向设备发送 OKAY
15. 在数据传输结束时，我们向设备发送 WRTE
16. 我们向设备发送 QUIT 消息
17. 设备向我们发送 OKAY
18. 我们向设备发送 CLSE 消息
19. 设备向我们发送 CLSE

## ADB List

这个命令不能通过`adb`命令使用。它可以列出文件夹中的文件。相比解析`adb shell 'ls -al <remote dir>'`的结果更加可靠，参考链接http://mywiki.wooledge.org/ParsingLs

> **warning：** 我已经联系了谷歌的一位工程师并得到了回复：`adb list`可以通过`adb`使用，只要使用`ls`即可，例如：`adb ls/`（而不是`adb shell ls /`）。
> 
1. 我们发送OPEN消息到设备
2. 我们发送sync到设备，`sync：`开始一个SYNC服务
3. 设备发送OKAY到我们
4. 我们发送WRTE到设备
5. 我们发送LIST和远程路径的字符长度到设备
6. 设备发送OKAY到我们
7. 我们发送WRTE到设备
8. 我们发送我们想要列出的远程路径，例如`sdcard/`
9. 设备发送OKAY到我们
10. 我们发送WRTE到设备
11. 我们发送RECV到设备
12. 设备通过任意数量的任意长度的数据包发送一系列“dents”（目录条目）-有时数据包只有1个字节长度。当下一个数据包准备好时，它会发送OKAY。我们将继续返回10（发送WRTE，RECV，读取DENT），直到接收到DONE。 “dents”可以跨越多个数据包，但是具有以下结构：
    1. 一个四字节的响应ID DENT（如果没有更多要列出的文件，则为DONE。如果远程路径不存在或不是文件夹，则立即发送DONE）
    2. 一个四字节整数表示文件模式-这个模式的前9位表示文件权限，就像chmod模式一样。位14到16似乎表示文件类型之一：`0b100`（文件），`0b010`（目录），`0b101`（符号链接）
    3. 一个四字节整数表示文件大小。
    4. 一个四字节整数表示自Unix时代以来的最后修改时间。
    5. 一个四字节整数表示文件名长度。
    6. 一个utf-8字符串表示文件名。
    7. 回到1。
13. 设备发送DONE。 DONE可以出现在数据包的任何位置，无论是在开头、中间还是结尾。有时，在DONE之后的数据包的其余部分会用0填充。
14. 我们向设备发送OKAY
15. 我们向设备发送WRTE
16. 我们向设备发送QUIT
17. 设备向我们发送OKAY
18. 我们向设备发送CLSE
19. 设备向我们发送CLSE

## ADB Install

安装只是将APK推送到/data/local/tmp，然后运行shell命令`pm install /data/local/temp/apkName.apk`的组合。

## ADB Reboot

只需使用命令`reboot:`打开流。

## Authentication

AOSP认为使用自己的加密库来签署身份验证过程中使用的令牌很酷，即使他们使用标准的RSA密钥。如果您想能够通过设备进行身份验证，而不是每次建立连接时都发送公钥，则需要使用他们的库。或者自己重新实现它。

该库称为libmincrypt，可以在[此处](https://android.googlesource.com/platform/system/core/+/master/libmincrypt/)找到。
