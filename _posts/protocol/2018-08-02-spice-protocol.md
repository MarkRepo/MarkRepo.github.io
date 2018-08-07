---
title: Spice 协议
description: Spice协议对全局和各通道的通信定义
categories: protocol
tags:
  - Spice
---

## 介绍

Spice协议定义了一组协议消息，用于跨网络**访问，控制和接收**来自远程计算设备（例如，键盘，视频，鼠标）的输入，并向它们发送输出。 受控设备可以驻留在客户端或服务器上任一侧，。 此外，该协议定义了一组调用，用于支持远程服务器从一个网络地址迁移到另一个网络地址。 除了一个例外，传输数据的加密被禁止在协议之外，以便在选择加密方法时获得最大的灵活性。 Spice使用简单的消息传递，不依赖于任何RPC标准或特定的传输层。

Spice通信会话被分成多个通信通道（例如，每个通道是远程设备），以便能够在运行时根据通道类型控制消息的通信和执行（例如QoS加密），以及添加和删除通信通道（由spice协议定义支持）。 以下通信通道在当前协议定义中定义：

+ main 通道作为主spice会话连接
+ display 通道用于接收远程显示更新
+ inputs 通道用于发送鼠标和键盘事件
+ cursor 通道用于接收指针形状和位置
+ playback 通道用于接收音频流
+ record 通道发送捕获的音频

随着协议的发展，将添加更多的通道类型。 Spice还定义了一组协议定义，用于同步远端的通道执行。

## 通用协议定义

### 字节序

除非另有说明，否则所有数据结构都是打包的，字节和位顺序是小端格式。

### 数据类型

```c
UINT8 //8 bits unsigned integer
INT16  //16 bits signed integer
UINT16 //16 bits unsigned integer
UINT32 //32 bits unsigned integer
INT32 //32 bits signed integer
UINT64 //64 bits unsigned integer
ADDRESS //64 bits unsigned integer, value是已寻址数据的偏移量,从spice协议消息体算起 (例如RedDataHeader后面的数据，或者RedSubMessage）
FIXED28_4 //32 bits fixed point number. 28位高位是有符号整数。 低4位是分母为16的分数的无符号整数分子。

POINT
INT32 x
INT32 y

POINT16
INT16 x
INT16 y

RECT
INT32 top
INT32 left
INT32 bottom
INT32 right

POINTFIX
FIXED28_4 x
FIXED28_4 y
```

### 协议魔数 UINT8[4]

RED_MAGIC = {0x52, 0x45,0x44, 0x51}

### 协议版本

协议版本定义为两个UINT32值，主协议版本和次协议版本。 具有相同主版本的服务器和客户端必须保持兼容性而不管次要版本（即，增加主要版本会打破兼容性）。 具有*巨大*值的主协议版本保留用于开发目的，并且被认为是不受支持且不可靠的。 *巨大*值被定义为第31位为1。 每次协议更改都会增加次要协议版本，但不会破坏兼容性。 主协议版本增加时此协议版本为0。

当前协议版本

```c
RED_VERSION_MAJOR = 1
RED_VERSION_MINOR = 0
```

### 兼容性 UINT32[]

为了在客户端和服务器实现中允许一定程度的灵活性并且为了提高兼容性，spice协议支持通道兼容性的双向交换。 兼容性用UINT32 vector表示，该向量分为两组：通用兼容性和通道兼容性。 通用兼容性代表所有通道共享的兼容性，通道兼容性代表通道特定兼容性。 将矢量拆分为两种类型允许我们独立地添加通道兼容性。 使用兼容性向量中的一个或多个位表示每个兼容性。

### 通道类型 UINT8

```c
RED_CHANNEL_MAIN = 1
RED_CHANNEL_DISPLAY = 2
RED_CHANNEL_INPUTS = 3
RED_CHANNEL_CURSOR = 4
RED_CHANNEL_PLAYBACK = 5
RED_CHANNEL_RECORD = 6
```

### 错误码 UINT32

```c
RED_ERROR_OK = 0
RED_ERROR_ERROR = 1
RED_ERROR_INVALID_MAGIC = 2
RED_ERROR_INVALID_DATA = 3
RED_ERROR_VERSION_MISMATCH = 4
RED_ERROR_NEED_SECURED = 5
RED_ERROR_NEED_UNSECURED = 6
RED_ERROR_PERMISSION_DENIED = 7
RED_ERROR_BAD_CONNECTION_ID = 8
RED_ERROR_CHANNEL_NOT_AVAILABLE = 9
```

### 警告码

`RED_WARN_GENERAL = 0`

### 信息码

`RED_INFO_GENERAL = 0`

### 公钥缓冲区大小

`RED_TICKET_PUBKEY_BYTES = 162`   
size needed for holding 1024 bit RSA public key in X.509 SubjectPublicKeyInfo format

### 通道链接： 建立一个通道连接

+ 连接过程  
通道连接过程由客户端启动。 客户端发送RedLinkMess，作为响应，服务器发送RedLinkReply。 当客户端收到RedLinkReply时，它会检查错误代码，如果没有错误，它会使用RedLinkReply中收到的公钥加密其密码并将其发送到服务器。 服务器接收密码后将链接结果发送给客户端。 客户端检查链接结果，如果结果等于`RED_ERROR_OK`，则建立有效连接。   
只有在客户端具有活动的`RED_CHANNEL_MAIN`通道连接后，才允许使用`RED_CHANNEL_MAIN`以外的通道类型的通道连接。 只允许一个`RED_CHANNEL_MAIN`连接，此通道连接与远程服务器建立spice会话。
+ Ticketing  
Ticketing是spice中实现的一种机制，以确保仅从已认证源打开连接。 要启用此机制，将在spice服务器中设置一个由密码和时间有效性组成的ticket。 时间有效期过后，ticket就过期。 ticket已加密。 要进行加密，服务器会生成1024位RSA密钥，并将公共部分发送到客户端（通过RedLinkInfo）。 客户端使用此密钥加密密码并将其发送回服务器（在RedLinkMess之后）。 服务器解密密码，将其与ticket进行比较，并确保在允许的时间范围内收到密码。
+ RedLinkMess definition.

```c
UINT32 magic //value of this fields must be equal to RED_MAGIC
UINT32 major_version //value of this fields must be equal to RED_VERSION_MAJOR.
UINT32 minor_version //value of this fields must be equal to RED_VERSION_MINOR.
UINT32 size //number of bytes following this field to the end of this message.
UINT32 connection_id //In case of a new session (i.e., channel type is RED_CHANNEL_MAIN) this field is set to zero, 
//and in response the server will allocate session id and will send it via the RedLinkReply message. In case of all 
//other channel types, this field will be equal to the allocated session id.
UINT8 channel_type //one of RED_CHANNEL_?
UINT8 channel_id //channel id to connect to. This enables having multiple channels of the same type.
UINT32 num_common_caps //number of common client channel capabilities words
UINT32 num_channel_caps //number of specific client channel capabilities words
UINT32 caps_offset //location of the start of the capabilities vector given by the 
//bytes offset from the “ size” member (i.e., from the address of the “connection_id” member).
```
+ RedLinkReply definition

```c
UINT32 magic //value of this field must be equal to RED_MAGIC
UINT32 major_version //server major protocol version.
UINT32 minor_version //server minor protocol version.
UINT32 size //number of bytes following this field to the end of this message.
UINT32 error //Error codes (i.e., RED_ERROR_?)
UINT8[RED_TICKET_PUBKEY_BYTES] pub_key //1024 bit RSA public key in X.509 SubjectPublicKeyInfo format.
UINT32 num_common_caps //number of common server channel capabilities words
UINT32 num_channel_caps //number of specific server channel capabilities words
UINT32 caps_offset //location of the start of the capabilities vector given by the bytes offset 
//from the “ size” member (i.e., from the address of the “connection_id” member) .
```
+ Encrypted Password  
客户端发送RSA加密密码，使用从服务器接收的公钥（在RedLinkReply中）。 格式为EME-OAEP，如PKCS＃1 v2.0中所述，带有SHA-1，MGF1和空编码参数.
+ Link Result UINT32  
服务器发送链接结果错误码 (i.e.`RED_ERROR_?`)

### 协议消息定义

链接阶段之后发送的所有消息都具有通用消息布局。 它以RedDataHeader开头，它描述了一条主要消息和一个可选的子消息列表

+ RedDataHeader

```c
/*serial number of the message within the channel. 
Serial numbers start with a value of 1 and are incremented on every message transmitted*/
UINT64 serial 
/*message type can be one that is accepted by all channel (e.g.,RED_MIGRATE), 
or specific to a channel type (e.g., RED_DISPLAY_MODE for display channel).*/
UINT16 type 
/*size of the message body in bytes. In case sub_list (see below) is not zero
then the actual main message size is sub_list. The message body follows RedDataHeader.*/
UINT32 size 
/*optional sub-messages list. If this field is not zero then sub_list is
the offset in bytes to RedSubMessageList from the end of RedDataHeader. All submessages
need to be executed before the main message, and in the order they appear in
the sub-messageslist.*/
UINT32 sub_list 
```
+ RedSubMessageList

```c
UINT16 size //number of sub-messages in this list.
UINT32[] sub_messages //array of offsets to sub message, offset is number of bytes from the end of RedDataHeader to start of RedSubMessage.
```
+ RedSubMessage

```c
UINT16 type  //message type can be one that is accepted by all channel (e.g., RED_MIGRATE), 
//or specific to a channel type (e.g., RED_DISPLAY_MODE for display channel).
UINT32 size //size of the message body in bytes. The message body follows RedSubMessage.
```

### 通用消息和消息命名惯例

根据消息的来源添加type和structure前缀。 从服务器发送到客户端的消息的type前缀为RED，structure前缀为Red。 对于从客户端发送的消息，前缀是REDC和Redc。

### 服务器端通用消息

```c
RED_MIGRATE = 1
RED_MIGRATE_DATA = 2
RED_SET_ACK = 3
RED_PING = 4
RED_WAIT_FOR_CHANNELS = 5
RED_DISCONNECTING = 6
RED_NOTIFY = 7
RED_FIRST_AVAIL_MESSAGE = 101
```
特定通道服务器消息从`RED_FIRST_AVAIL_MESSAGE`开始。 从`RED_NOTIFY + 1`到`RED_FIRST_AVAIL_MESSAGE - 1`的所有消息类型都保留供进一步使用

### 客户端通用消息

```c
REDC_ACK_SYNC = 1
REDC_ACK = 2
REDC_PONG = 3
REDC_MIGRATE_FLUSH_MARK = 4
REDC_MIGRATE_DATA = 5
REDC_DISCONNECTING = 6
REDC_FIRST_AVAIL_MESSAGE = 101
```
特定通道客户端消息从`REDC_FIRST_AVAIL_MESSAGE`开始。 从`REDC_ACK_SYNC + 1`到`REDC_FIRST_AVAIL_MESSAGE - 1`的所有消息类型都保留供进一步使用。

### 消息确认

Spice提供一组消息用于请求客户端在消耗每条或多条消息后发送一个确认。 为了请求确认消息，服务器发送带有*请求确认频率*的`RED_SET_ACK`消息，*请求确认频率*指在客户端每次接收到多少条消息后发送确认。作为响应，客户端发送`REDC_ACK_SYNC`。 从这一点来看，客户端每次收到指定数量的消息后，它将发送REDC_ACK消息。

+ `RED_SET_ACK`, `RedSetAck`

```c
/*the generation of the acknowledgment sequence. This value will
be sent back by REDC_ACK_SYNC. It is used for acknowledgment accounting
synchronization.*/
UINT32 generation 
/*the window size. Spice client will send acknowledgment for every
“window” messages. Zero window size will disable messages acknowledgment*/
UINT32 window 
```
+ `REDC_ACK_SYNC`, UINT32

```c
UINT32 //Spice client sends RedSetAck.generation in response to RED_SET_ACK
```
+ REDC_ACK,VOID  
Spice client sends REDC_ACK message for every RedSetAck.window messages it consumes.

### Ping

Spice协议提供ping消息以进行调试。 Spice服务器发送`RED_PING`,客户端使用`REDC_PONG`响应。 服务器可以通过用`REDC_PONG`消息中返回的时间与当前时间做减法来测量往返时间。

+ RED_PING, RedPing

```c
UINT32 id 		//the id of this message
UINT64 time 	//time stamp of this message
```
+ REDC_PONG, RedPong

```c
UINT32 id 		//Spice client copies it from RedPing.id
UINT64 time 	//Spice client copies it from RedPing.time
```

### 通道迁移

Spice支持Spice server的迁移。 以下通用消息与主通道特定消息一起用于迁移spice server之间的通道连接。 我们将这些server称为源server和目的server。 主通道用于初始化和控制迁移过程。 以下描述了实际的通道迁移过程。  
通道迁移过程从服务器发送RED_MIGRATE消息开始。 客户端收到消息，检查附加的标志和：

+ 如果服务器要求flush消息（即，`RED_MIGRATE_NEED_FLUSH`标志打开），则客户端向服务器发送`REDC_MIGRATE_FLUSH_MARK`消息。 此过程可用于确保在执行迁移操作之前安全传递所有半空中消息
+ 如果服务器请求数据传输（即，`RED_MIGRATE_NEED_DATA_TRANSFER`标志打开），则客户端希望在迁移到目的server之前从源server接收最后一条消息。 此消息类型必须为`RED_MIGRATE_DATA`类型。 接收消息的内容将在连接交换时传输到目的server。

之后，客户端交换通信通道（即，开始使用与目的server的连接）。 只有在所有其他通道也完成迁移过程后，客户端才能关闭与源服务器的连接。 如果源服务器端已请求数据传输，则客户端首先发送`RED_MIGRATE_DATA`消息到目的server，其包含从源server收到的`REDC_MIGRATE_DATA`消息的内容。

+ Migration flags

```c
RED_MIGRATE_NEED_FLUSH = 1
RED_MIGRATE_NEED_DATA_TRANSFER = 2
```
+ RED_MIGRATE, RedMigrate

```c
UINT32 flags  //combination of red migration flags.
```
+ `RED_MIGRATE_DATA, UINT8[]`  
服务器迁移数据，此消息的主体是可变长度的原始数据，由每个通道类型独立确定
+ `REDC_MIGRATE_FLUSH_MARK`, VOID  
此消息标记客户端通信通道flush的完成。
+ `REDC_MIGRATE_DATA`, UINT8[]  
发送迁移数据，由client发送到目的server，包含源server使用`RED_MIGRATE_DATA`消息发送的数据.

### 通道同步

Spice提供了在client端同步通道消息执行的机制。 服务器发送`RED_WAIT_FOR_CHANNELS`消息，该消息包含要等待的通道消息列表（即，RedWaitForChannels）。 在执行任何更多消息之前，Spice客户端将等待该列表中的所有消息的完成

+ RedWaitForChannel

```c
UINT8 type 		//channel type (e.g., RED_CHANNEL_INPUTS)
UINT8 id 		//channel id.
UINT64 serial 	//message serial id (i.e, RedDataHeader.serial) to wait for
```
+ `RED_WAIT_FOR_CHANNELS`, RedWaitForChannels

```c
UINT8 wait_count 					//number of items in wait_list
RedWaitForChannel[] wait_list 		//list of channels to wait for.
```

### 断开连接原因

以下消息用于有关服务器或客户端的有序断开连接的通知

+ RED_DISCONNECTING, RedDisconnect

```c
UINT64 time_stamp  		//time stamp of disconnect action on the server.
UINT32 reason 			//disconnect reason, RED_ERROR_?
```
+ REDC_DISCONNECTING, RedcDisconnect
	
```c
UINT64 time_stamp 		//time stamp of disconnect action on the client.
UINT32 reason 			//disconnect reason, RED_ERROR_?
```

### 服务器通知

Spice协议定义了使用RED_NOTIFY消息向客户端发送通知的消息。 消息按严重性和可见性分类。 后者可用作向用户显示消息的方式的提示。 例如，高可见性通知将触发消息框，低可见性通知将定向到日志

+ RED_NOTIFY, RedNotify

```c
UINT64 time_stamp 			//server side time stamp of this message.
UINT32 severity 			//one of RED_NOTIFY_SEVERITY_?
UINT32 visibility 			//one of RED_NOTIFY_VISIBILITY_?
UINT32 what 				//one of RED_ERROR_?, RED_WARN_? Or RED_INFO_?, depending on severity.
UINT32 message_len 			// size of message
UINT8[] message 			//message string in UTF8.
UINT8 0 					//string zero termination
```

## Main 通道定义

### Server messages

```c
RED_MAIN_MIGRATE_BEGIN = 101
RED_MAIN_MIGRATE_CANCEL = 102
RED_MAIN_INIT = 103
RED_MAIN_CHANNELS_LIST = 104
RED_MAIN_MOUSE_MODE = 105
RED_MAIN_MULTI_MEDIA_TIME = 106
RED_MAIN_AGENT_CONNECTED = 107
RED_MAIN_AGENT_DISCONNECTED = 108
RED_MAIN_AGENT_DATA = 109
RED_MAIN_AGENT_TOKEN = 110
```

### Client messages

```c
REDC_MAIN_RESERVED = 101
REDC_MAIN_MIGRATE_READY = 102
REDC_MAIN_MIGRATE_ERROR = 103
REDC_MAIN_ATTACH_CHANNELS = 104
REDC_MAIN_MOUSE_MODE_REQUEST = 105
REDC_MAIN_AGENT_START = 106
REDC_MAIN_AGENT_DATA = 107
REDC_MAIN_AGENT_TOKEN = 108
```

### 迁移控制

使用主通道消息执行Spice迁移控制。 Spice服务器通过发送`RED_MAIN_MIGRATE_BEGIN`消息来启动迁移过程。 客户端完成pre-migrate的过程后，会通过发送`REDC_MAIN_MIGRATE_READY`消息通知服务器。 在pre-migrate过程错误的情况下，客户端发送`REDC_MAIN_MIGRATE_ERROR`。 一旦服务器收到`REDC_MAIN_MIGRATE_READY`，他就可以开始迁移过程。 服务器可以发送`RED_MAIN_MIGRATE_CANCEL`以指示客户端取消迁移过程。

+ `RED_MAIN_MIGRATE_BEGIN`, RedMigrationBegin

```c
UINT16 port 			//port of destination server
UINT16 sport 			//secure port of destination server
UINT8[] host_name 		//host name of destination server
```
+ `RED_MAIN_MIGRATE_CANCEL`, VOID  
Instruct the client to cancel migration process
+ `REDC_MAIN_MIGRATE_READY`, VOID  
Notify the server of successful completion of the pre-migrate stage
+ `REDC_MAIN_MIGRATE_ERROR`, VOID  
Notify the server of pre-migrate stage error

### 鼠标模式

Spice协议指定两种鼠标模式，客户端模式和服务器模式。   
在客户端模式下，实际鼠标是客户端鼠标：客户端发送在显示内的鼠标位置，服务器发送鼠标形状消息。   
在服务器模式下，客户端发送鼠标相对移动，服务器发送位置和形状命令。 Spice主通道用于鼠标模式控制。

+ Modes  
`RED_MOUSE_MODE_SERVER` = 1  
`RED_MOUSE_MODE_CLIENT` = 2  
+ `RED_MAIN_MOUSE_MODE`, RedMouseMode 

```c 
//Spice server sends this message on every mouse mode change
UINT32 supported_modes 		//current supported mouse mode, this is any combination of RED_MOUSE_MODE_?
UINT32 current_mode 		//the current mouse mode. Can be one of RED_MOUSE_MODE_?
```
+ `REDC_MAIN_MOUSE_MODE_REQUEST`, UINT32

```c
/*Spice client sends this message to request specific mouse mode. It is not guarantied that
the server will accept the request. Only on receiving RED_MOUSE_MODE message,
the client can know of actual mouse mode change.*/
UINT32 		//requested mode, one of RED_MOUSE_MODE_?
```

### 主通道初始化消息

Spice服务器必须发送RedInit作为第一个发送的消息，并且不允许在任何其他点发送它。

+ `RED_MAIN_INIT`, RedInit

```c
//session id is generated by the server. This id will be send on every
//new channel connection within this session (i.e., in RedLinkMess.connection_id).
UINT32 session_id  
//optional hint of expected number of display channels.Zero is defined as an invalid value
UINT32 display_channels_hint 
//supported mouse modes. This is any combination of RED_MOUSE_MODE_?
UINT32 supported_mouse_modes
//the current mouse mode, one of RED_MOUSE_MODE_?
UINT32 current_mouse_mode
//current state of Spice agent (see Section 3.8), 0 and 1 stand for disconnected and connected state respectively.
UINT32 agent_connected
//number of available tokens for sending messages to Spice agent
UINT32 agent_tokens
//current server multimedia time. The multimedia time is used for synchronizing video 
UINT32 multi_media_time
//optional hint for help in determining global LZ compression
//dictionary size (for more information see section “Spice Image” in “Display Channel”).
UINT32 ram_hint 
```

### 服务端通道通知

为了能够动态连接到服务器端通道，Spice协议包括`RED_MAIN_CHANNELS_LIST`消息。 此消息通知客户端服务器端的可用通道。
为响应该消息，客户端可以决定链接新的可用通道。 在发送任何`RED_MAIN_CHANNELS_LIST`消息之前，
服务器必须接收`REDC_MAIN_ATTACH_CHANNELS`。

+ `RED_MAIN_CHANNELS_LIST`, RedChannels

```c
UINT32 num_of_channels 			//number of channels in this list
RedChanneID[] channels 			//vector of “num_of_channels” channel ids
```
+ RedChanneID

```c
UINT8 type 			//channel type, one of RED_CHANNEL_? channel types, except for RED_CHANNEL_MAIN
UINT8 id 			//channel id
```

### Multimedia time

Spice定义了用于设置多媒体时间以便同步视频和音频流的消息。 支持两种更新多媒体时间的方法。 第一种方法使用到达playback 通道的数据的时间戳。第二种方法使用主通道`RED_MAIN_MULTI_MEDIA_TIME`消息。 当不存在活动playback 通道时使用后一种方法。

+ `RED_MAIN_MULTI_MEDIA_TIME`, UINT32  
UINT32 – multimedia time

### Spice agent

Spice协议为Spice client和远程服务器上的spice client agent之间的双向通信通道定义了一组消息。 Spice仅提供通信通道，实际传输的数据内容对协议不透明。 此通道可用于各种目的，例如，client-guest 剪贴板共享，身份验证和显示配置。

Spice client 接收远端agent连接的通知，该通知可以包含在`RED_MAIN_INIT`消息中，或特定的`RED_MAIN_AGENT_CONNECTED`
消息。远程代理断开连接通知由`RED_MAIN_AGENT_DISCONNECTED`消息提供。使用双向令牌机制以防止代理消息阻塞主通道（例如，在代理停止消费数据的情况下）。不允许每一方发送的消息多于另一方分配给它的令牌。`RED_MAIN_INIT`消息初始化为客户端分配的令牌数，并使用`RED_MAIN_AGENT_TOKEN`完成令牌的进一步分配。服务器令牌初始计数在`REDC_MAIN_AGENT_START`消息中提供。此消息必须是客户端发送到服务器的第一个与代理相关的消息。使用`REDC_MAIN_AGENT_TOKEN`完成服务器令牌的进一步分配。使用`RED_MAIN_AGENT_DATA`和`REDC_MAIN_AGENT_DATA`传送实际数据包。

+ 尽管代理消息对于协议是不透明的，但是代理数据流由Spice协议定义以便描绘消息。
仍然，客户端与服务器的通信独立于代理通道，例如，代理协议冲突不影响其余通道。 代理流被定义为具有以下格式的一组消息

```c
UINT32 protocol //unique protocol of this message. The protocol id must be registered in order to prevent conflicts.
UINT32 type 	//protocol dependent message type.
UINT64 opaque 	//protocol dependent opaque data.
UINT32 size 	//size of data in bytes.
UINT8 data[0] 	//data of this message.
```
客户端和服务器必须继续处理未知协议消息或具有未知类型的消息（即接收和转储）。

+ `RED_MAIN_AGENT_CONNECTED`, VOID
+ `RED_MAIN_AGENT_DISCONNECTED`, UINT32  
UINT32 – disconnect error code `RED_ERROR_?`
+ `RED_AGENT_MAX_DATA_SIZE` = 2048
+ `RED_MAIN_AGENT_DATA`, UINT8[]  
Agent packet is the entire message body (i.e. RedDataHeader.size).   
The maximum packet size is `RED_AGENT_MAX_DATA_SIZE`.
+ `RED_MAIN_AGENT_TOKEN`, UINT32  
UINT32 – allocated tokens count for the client
+ `REDC_MAIN_AGENT_START`, UINT32  
UINT32 – allocated tokens count for the server
+ `REDC_MAIN_AGENT_DATA`, UINT8[]  
Agent packet is the entire message body (i.e. RedDataHeader.size). 
The maximum packet size is `RED_AGENT_MAX_DATA_SIZE`.
+ `REDC_MAIN_AGENT_TOKEN`, UINT32   
UINT32 – allocated tokens count for the server

## Inputs 通道定义

Spice Inputs channel controls the server mouse and the keyboard.

### Client messages

```c
REDC_INPUTS_KEY_DOWN = 101
REDC_INPUTS_KEY_UP = 102
REDC_INPUTS_KEY_MODIFAIERS = 103
REDC_INPUTS_MOUSE_MOTION = 111
REDC_INPUTS_MOUSE_POSITION = 112
REDC_INPUTS_MOUSE_PRESS = 113
REDC_INPUTS_MOUSE_RELEASE = 114
```

### Server Messages

```c
RED_INPUTS_INIT = 101
RED_INPUTS_KEY_MODIFAIERS = 102
RED_INPUTS_MOUSE_MOTION_ACK = 111
```

### Keyboard messages

Spice支持发送键盘键事件和键盘LED同步。 客户端使用REDC_INPUTS_KEY_DOWN和REDC_INPUTS_KEY_UP消息发送密钥事件。 键值使用PC AT扫描码表示（参见KeyCode）。 键盘LED同步是通过服务器发送RED_INPUTS_KEY_MODIFAIERS消息或通过客户端发送REDC_INPUTS_KEY_MODIFAIERS来完成的，这些消息包含键盘指示灯状态。 键盘修饰符也由服务器使用RED_INPUTS_INIT发送，此消息必须作为第一个服务器消息发送，服务器不得在任何其他点发送

+ Keyboard led bits

```c
RED_SCROLL_LOCK_MODIFIER = 1
RED_NUM_LOCK_MODIFIER = 2
RED_CAPS_LOCK_MODIFIER = 4
```
+ `RED_INPUTS_INIT`, UINT32  
UINT32 – any combination of keyboard led bits. If bit is set then the led is on.
+ `RED_INPUTS_KEY_MODIFAIERS`, UINT32  
UINT32 – any combination of keyboard led bits. If bit is set then the led is on.
+ `REDC_INPUTS_KEY_MODIFAIERS`, UINT32  
UINT32 – any combination of keyboard led bits. If bit is set then the led is on.
+ KeyCode  
UINT8[4] - the value of key code is a PC AT scan code. The code is composed by up to
four bytes for supporting extended codes. A code is terminated by a zero byte.
+ `REDC_INPUTS_KEY_DOWN`, KeyCode  
KeyCode – client sends this message to notify of key press event.
+ `REDC_INPUTS_KEY_UP`, KeyCode  
KeyCode – client sends this message to notify of key release event.

### Mouse messages

Spice支持两种鼠标操作模式：客户端鼠标和服务器鼠标（有关详细信息，请参阅“鼠标模式”）。 在服务器鼠标模式中，客户端发送鼠标移动消息（即，`REDC_INPUTS_MOUSE_MOTION`），并且在客户端鼠标模式中，它发送位置消息（即，`REDC_INPUTS_MOUSE_POSITION`）。 位置消息保存客户端鼠标在显示器上的位置以及显示通道的ID，该ID来自`RedLinkMess.channel_id`。 为了防止鼠标移动和位置事件的泛滥，服务器在它接收的每个`RED_MOTION_ACK_BUNCH`消息上发送`RED_INPUTS_MOUSE_MOTION_ACK`消息。 此机制允许客户端跟踪服务器的消息消耗率并根据它更改事件推送策略。 鼠标按钮事件使用`REDC_INPUTS_MOUSE_PRESS`和`REDC_INPUTS_MOUSE_RELEASE`消息发送到服务器。

+ Red Button ID

```c
REDC_MOUSE_LBUTTON = 1, left button
REDC_MOUSE_MBUTTON = 2, middle button
REDC_MOUSE_RBUTTON = 3, right button
REDC_MOUSE_UBUTTON = 4, scroll up button
REDC_MOUSE_DBUTTON = 5, scroll down button
```
+ Buttons masks

```c
REDC_LBUTTON_MASK = 1, left button mask
REDC_MBUTTON_MASK = 2, middle button mask
REDC_RBUTTON_MASK = 4, right button mask
```
+ `RED_MOTION_ACK_BUNCH` = 4
+ `REDC_INPUTS_MOUSE_MOTION`, RedcMouseMotion

```c
INT32 dx 	//number of pixels the mouse had moved on x axis
INT32 dy 	//number of pixels the mouse had moved on y axis
UINT32 buttons_state 	//any combination of buttons mask. Set bit describe pressed
//button and clear bit describe unpressed button.
```
+ `REDC_INPUTS_MOUSE_POSITION`, RedcMousePosition

```c
UINT32 x 		//position on x axis
UINT32 y 		//position on y axis
UINT32 buttons_state   //any combination of buttons mask. Set bit describe pressed
//button and clear bit describe unpressed button.
UINT8 display_id 	//id of the display that client mouse is on.
```
+ `REDC_INPUTS_MOUSE_PRESS`, RedcMousePress

```c
UINT32 button_id 	//one of REDC_MOUSE_?BUTTON
UINT32 buttons_state //any combination of buttons masks. Set bit describes pressed
//button, and clear bit describes unpressed button.
```
+ `REDC_INPUTS_MOUSE_RELEASE`, RedcMouseRelease

```c
UINT32 button_id 		//one of REDC_MOUSE_?BUTTON
UINT32 buttons_state 	//any combination of buttons mask. Set bit describes pressed
//button and clear bit describes unpressed button.
```

## display 通道定义

Spice协议定义了一组消息，用于支持在客户端显示器上呈现远程显示区域。 该协议支持渲染图形基元（例如，线，图像）和视频流。 该协议还支持在客户端缓存图像和调色板。 Spice显示通道支持多种图像压缩方法，以减少网络流量

### Server messages

```c
RED_DISPLAY_MODE = 101
RED_DISPLAY_MARK = 102
RED_DISPLAY_RESET = 103
RED_DISPLAY_COPY_BITS = 104
RED_DISPLAY_INVAL_LIST = 105
RED_DISPLAY_INVAL_ALL_IMAGES = 106
RED_DISPLAY_INVAL_PALETTE = 107
RED_DISPLAY_INVAL_ALL_PALETTES = 108
RED_DISPLAY_STREAM_CREATE = 122
RED_DISPLAY_STREAM_DATA = 123
RED_DISPLAY_STREAM_CLIP = 124
RED_DISPLAY_STREAM_DESTROY = 125
RED_DISPLAY_STREAM_DESTROY_ALL = 126
RED_DISPLAY_DRAW_FILL = 302
RED_DISPLAY_DRAW_OPAQUE = 303
RED_DISPLAY_DRAW_COPY = 304
RED_DISPLAY_DRAW_BLEND = 305
RED_DISPLAY_DRAW_BLACKNESS = 306
RED_DISPLAY_DRAW_WHITENESS = 307
RED_DISPLAY_DRAW_INVERS = 308
RED_DISPLAY_DRAW_ROP3 = 309
RED_DISPLAY_DRAW_STROKE = 310
RED_DISPLAY_DRAW_TEXT = 311
RED_DISPLAY_DRAW_TRANSPARENT = 312
RED_DISPLAY_DRAW_ALPHA_BLEND = 313
```

### Client messages

```
REDC_DISPLAY_INIT = 101
```

### Operation flow

Spice服务器使用`RED_DISPLAY_MODE`向客户端发送模式消息，以指定当前绘制区域大小和格式。 作为响应，客户端创建一个绘制区域，用于渲染服务器发送的所有后续渲染命令。 只有在从服务器接收到标记命令（即`RED_DISPLAY_MARK`）之后，客户机才会公开新的远程显示区域内容（即，在模式命令之后）。 服务器可以使用`RED_DISPLAY_RESET`发送重置命令，以指示客户端删除其绘制区域和调色板缓存。 仅当客户端上不存在活动绘图区域时，才允许发送模式消息。 仅在客户端存在活动绘图区域时才允许发送重置消息。 在模式和重置消息之间只允许发送一次标记消息。 仅当客户端具有活动显示区域（即，在`RED_DISPLAY_MODE`到`RED_DISPLAY_RESET`之间）时，才允许绘制命令，复制位命令和流命令。

在通道连接上，客户端可选地使用`REDC_DISPLAY_INIT`发送init消息，以便启用图像缓存和全局字典压缩。 该消息包括缓存ID及其大小和字典压缩窗口的大小。 这些尺寸和ID由client确定。 不允许发送多个init消息。

调色板缓存由服务器管理。 items缓存插入命令作为渲染命令的一部分发送。 服务器发送`RED_DISPLAY_INVAL_LIST`或`RED_DISPLAY_INVAL_LIST`消息显式删除缓存items，发送`RED_DISPLAY_INVAL_ALL_IMAGES`或`RED_DISPLAY_INVAL_ALL_PALETTES`服务器消息来重置客户端缓存。

### Draw area control

+ `RED_DISPLAY_MODE`, RedMode

```c
UINT32 width 	//width of the display area
UINT32 height 	//height of the display area
UINT32 depth 	//color depth of the display area. Valid values are 16bpp or 32bpp.
```
+ `RED_DISPLAY_MARK`, VOID  
Mark the beginning of the display area visibility
c) `RED_DISPLAY_RESET`, VOID  
Drop current display area of the channel and reset palette cache

### Raster operation descriptor

下面定义了一组标志，用于描述可以在渲染操作期间应用于源图像，源画笔，目标和结果的光栅操作。 这些标志的组合定义了在渲染操作期间需要执行的必要步骤。 在以下渲染命令的定义中，这种组合由'rop_descriptor'引用。

```c
ROPD_INVERS_SRC = 1 //Source Image need to be inverted before rendering
ROPD_INVERS_BRUSH = 2 //Brush need to be inverted before rendering
ROPD_INVERS_DEST = 4  //Destination area need to be inverted before rendering
ROPD_OP_PUT = 8  //Copy operation should be used.
ROPD_OP_OR = 16 //OR operation should be used.
ROPD_OP_AND = 32 //AND operation should be used.
ROPD_OP_XOR =64 //XOR operation should be used.
ROPD_OP_BLACKNESS = 128 //Destination pixel should be replaced by black
ROPD_OP_WHITENESS = 256  //Destination pixel should be replaced by white
ROPD_OP_INVERS = 512  //Destination pixel should be inverted
ROPD_INVERS_RES = 1024  //Result of the operation needs to be inverted

OP_PUT, OP_OR, OP_AND, OP_XOR, OP_BLACKNESS, OP_WHITENESS, and OP_INVERS are mutually exclusive
OP_BLACKNESS, OP_WHITENESS, and OP_INVERS are exclusive
```

### Raw raster image

以下部分描述了Spice原始光栅图像（Pixmap）。 Pixmap是使用Spice协议传输图像的几种方法之一（有关更多信息，请参阅“Spice图像”）。

+ Pixmap format types

```c
PIXMAP_FORMAT_1BIT_LE = 1
//1 bit per pixel and bits order is little endian. Each pixel value is an index in a color table. 
//The color table size is 2.
PIXMAP_FORMAT_1BIT_BE = 2
//1 bit per pixel and bits order is big endian. Each pixel value is index in a color table. 
//The color table size is 2.
PIXMAP_FORMAT_4BIT_LE = 3
//4 bits per pixel and nibble order inside a byte is little endian. Each pixel value is an
//index in a color table. The color table size is 16.
PIXMAP_FORMAT_4BIT_BE = 4
//4 bits per pixel and nibble order inside a byte is big endian. Each pixel value is an index
//in a color table. The color table size is 16.
PIXMAP_FORMAT_8BIT = 5
//8 bits per pixel. Each pixel value is an index in a color table. The color table size is 256.
PIXMAP_FORMAT_16BIT = 6
//pixel format is 16 bits RGB555.
PIXMAP_FORMAT_24BIT = 7
//pixel format is 24 bits RGB888.
PIXMAP_FORMAT_32BIT = 8
//pixel format is 32 bits RGB888.
PIXMAP_FORMAT_RGBA = 9
//pixel format is 32 bits ARGB8888.
```
+ Palette

```c
UINT64 id 	//unique id of the palette
UINT16 table_size 	//number of entries in the color table
UINT32[] color_table //each entry is RGB555 or RGB888 color depending on the
//current display area mode. If display area mode color depth is 32, the effective format is
//RGB888. If display area mode color depth is 16 the effective format is RGB555.
```
+ Pixmap flags

```c
PIXMAP_FLAG_PAL_CACHE_ME = 1
//Instruct the client to add the palette to cache
PIXMAP_FLAG_PAL_FROM_CACHE = 2
//Instruct the client to retrieve palette from cache.
PIXMAP_FLAG_TOP_DOWN = 4
//Pixmap lines are ordered from top to bottom (i.e., line 0 is the highest line).
```
+ Pixmap

```c
UINT8 format 	//one of PIXMAP_FORMAT_?
UINT8 flags 	//combination of PIXMAP_FLAG_?
UINT32 width 	//width of the pixmap
UINT32 height 	//height of the pixmap
UINT32 stride 	//number of bytes to add for moving from the beginning of line n to the beginning of line n+1
union {
ADDRESS palette 	//address of the color palette. Must be zero if no color table is required for format.
UINT64 palette_id 	//id of the palette, valid if FLAG_PAL_FROM_CACHE is set
}
ADDRESS data 		//address of line 0 of the pixmap.
```

### LZ with palette

本节描述了一种数据结构，它是调色板和压缩像素图数据的组合。 使用我们的LZSS算法实现压缩pixmap（参见下一节）。 每个解码的像素值是调色板中的索引

+ LZPalette Flags

```c
LZPALETTE_FLAG_PAL_CACHE_ME = 1
//Instruct the client to add the palette to the cache
LZPALETTE_FLAG_PAL_FROM_CACHE = 2
//Instruct the client to retrieve palette from the cache.
LZPALETTE_FLAG_TOP_DOWN = 4
//pixmap lines are ordered from top to bottom (i.e. line 0 is the highest line).
```
+ LZPalette

```c
UINT8 flags 		//combination of LZPALETTE_FLAG_?
UINT32 data_size 	//size of compressed data
union {
ADDRESS palette 	//address of the color palette (see Palette section in “Raw raster image”). Zero value is disallowed.
UINT64 palette_id 	//id of the palette, valid if FLAG_PAL_FROM_CACHE is set.
}
UINT8[] data 		//compressed pixmap
```

### Spice Image

以下部分描述了Spice图像。 Spice图像用于各种命令和数据结构中以表示光栅图像。 除原始模式外，Spice图像还支持多种压缩类型：Quic，LZ和GLZ。 Quic是一种预测编码算法。 它是SFALIC的概括，从灰度到彩色图像，加上RLE编码。 通过LZ，我们参考了我们对LZSS算法的实现，该算法针对不同格式的图像进行了调整。 通过GLZ我们引用LZ的扩展，允许它使用基于一组图像而不仅仅是被压缩图像的字典。

+ Image types

```c
IMAGE_TYPE_PIXMAP = 0
IMAGE_TYPE_QUIC = 1
IMAGE_TYPE_LZ_PLT = 100
IMAGE_TYPE_LZ_RGB = 101
IMAGE_TYPE_GLZ_RGB = 102
IMAGE_TYPE_FROM_CACHE = 103
```
+ Image flags
`IMAGE_FLAG_CACHE_ME` = 1, this flag instruct the client to add the image to image
cache, cache key is ImageDescriptor.id (see below).
+ ImageDescriptor

```c
UINT64 id 		//unique id of the image
UINT8 type 		//type of the image. One of IMAGE_TYPE_?
UINT8 flags 	//any combination of IMAGE_FLAG_?
UINT32 width 	//width of the image
UINT32 height 	//height of the image
```
+ Image data  
Image data follows ImageDescriptor and its content depends on ImageDescriptor.type:
	+ In case of PIXMAP – content is Pixmap.
	+ In case of QUIC – content is Quic compressed image. Data begins with the size of the 
	compressed data, represented by UINT32, followed by the compressed data.
	+ In case of LZ_PLT – content is LZPalette.
	+ In case of `LZ_RGB` – content is LZ_RGB – LZ encoding of an RGB image. Data
	begins with the size of the compressed data, represented by UINT32, followed by the compressed data.
	+ In case of `GLZ_RGB` – content is GLZ_RGB – GLZ encoding of an RGB image. Data
	begins with the size of the compressed data, represented by UINT32, , followed by the compressed data.
	+ In case of FROM_CACHE – No image data. The client should use ImageDescriptor.id 
	to retrieve the relevant image from cache.

### Glyph String

字形字符串定义一个用于渲染的字形数组。 字符串中的字形可以是A1，A4或A8格式（即1bpp，4bpp或8bpp alpha掩码）。 每个字形都包含目标绘制区域上的渲染位置。

+ RasterGlyph

```c
POINT render_pos //location of the glyph on the draw area
POINT glyph_origin //origin of the glyph. The origin is relative to the upper left corner
//of the draw area. Positive value on x axis advances leftward and positive value on yaxis advances upward.
UINT16 width //glyph's width
UINT16 height //glyph's height
UINT8[] data //alpha mask of the glyph. Actual mask data depends on the glyph string's
//flags. If the format is A1 then the line stride is ALIGN(width, 8) / 8. If the format is A4,
//the line stride is ALIGN(width, 2) / 2. If the format is A8, the line stride is width.
```
+ Glyph String flags

```c
GLYPH_STRING_FLAG_RASTER_A1 = 1
//Glyphs type is 1bpp alpha value (i.e., 0 is transparent 1 is opaque)
GLYPH_STRING_FLAG_RASTER_A4 = 2
//Glyphs type is 4bpp alpha value (i.e., 0 is transparent 16 is opaque)
GLYPH_STRING_FLAG_RASTER_A8 = 4
//Glyphs type is 4bpp alpha value (i.e., 0 is transparent 256 is opaque)
GLYPH_STRING_FLAG_RASTER_TOP_DOWN = 8
//Line 0 is the top line of the mask
```
+ GlyphString

```c
UINT16 length  	//number of glyphs
UINT16 flags 	//combination of GLYPH_STRING_FLAG_?
UINT8[] data 	//glyphs
```

### Data Types

+ RectList

```c
UINT32 count 	//number of RECT items in rects
RECT[] rects 	//array of <count> RECT
```
+ Path segment flags

```c
PATH_SEGMENT_FLAG_BEGIN = 1 //this segment begins a new path
PATH_SEGMENT_FLAG_END = 2 //this segment ends the current path
PATH_SEGMENT_FLAG_CLOSE = 8 //this segment closes the path and is invalid if PATH_SEGMENT_FLAG_END is not set
PATH_SEGMENT_FLAG_BEZIER = 16 //this segment content is a Bezier curve
```
+ PathSeg

```c
UINT32 flags 		//any combination of PATH_SEGMENT_FLAG_?
UINT32 count 		//number of points in the segment
POINTFIX[] points 	//segment points
```
+ PathSegList

```c
//List of PathSeg items. End of the list is reached if the sum of all previous PathSegs'
//sizes is equal to list_size. Address of next segment is the address of
//PathSeg.points[PathSeg.count]
UINT32 list_size 		//total size of in bytes of all PathSegs in the list,
PathSeg seg0 			//first path segment.
```
+ Clip types

```c
CLIP_TYPE_NONE = 0  //no clipping
CLIP_TYPE_RECTS = 1 //data is RectList and union of all rectangles in RectList is the effective clip
CLIP_TYPE_PATH = 2  //data is PathSegList and the figure described by PathSegList is the effective clip
```
+ Clip

```c
UIN32 type 			//one of CLIP_TYPE_?
ADDRESS data 		//address of clip data. The content depends on <type>
```
+ Mask flags  
`MASK_FLAG_INVERS` = 1, the effective mask is the inverse of the mask
+ Mask

```c
UINT8 flags 		// flags of the mask, combination of MASK_FLAG_?
POINT position 		//- origin of the mask in bitmap coordinates
ADDRESS bitmap 		//– address of the mask's image, the format of the image must be 1bpp.
//If the bitmap is zero then no masking operation needs to be preformed.
```
In all rendering commands, the mask must be big enough to cover the destination rectangle
+ Brush types

```c
BRUSH_TYPE_NONE = 0, the brush is invalid.
BRUSH_TYPE_SOLID = 1, the brush is solid RGB color
BRUSH_TYPE_PATTERN = 2, the brush is a pattern.
```
+ Pattern

```c
ADDRESS image 		// address of the pattern's Image
POINT position 		// origin coordinates of the pattern in the image
```
+ Brush

```c
UINT32 type 		//one of BRUSH_TYPE_?
Union {
UINT32 color 		//– RGB color. The format of the color depends on current draw area mode.
Pattern pattern;
}
```
+ Image scale mode

```c
//The following defines the method for scaling image
IMAGE_SCALE_INTERPOLATE = 0 //The client is allowed to INTERPOLATE pixel color.
IMAGE_SCALE_NEAREST = 1 //The client must use the nearest pixel.
```
+ LineAtrr flags

```c
LINE_ATTR_FLAG_START_WITH_GAP = 4 //first style segment if gap (i.e., foreground)
LINE_ATTR_FLAG_STYLED = 8 //style member of LineAtrr is valid and contains effective line style for the rendering operation.
```
+ LineAtrr join style

```c
LINE_ATTR_JOIN_ROUND = 0
LINE_ATTR_JOIN_BEVEL = 1
LINE_ATTR_JOIN_MITER = 2
```
+ LineAtrr cap style

```c
LINE_ATTR_CAP_ROUND = 0
LINE_ATTR_CAP_SQUARE = 1
LINE_ATTR_CAP_BUTT = 2
```
+ LineAttr

```c
UINT8 flags 	//combination of LINE_ATTR_?
UINT8 join_style 	//one of LINE_ATTR_JOIN_?
UINT8 cap_style 	//one of LINE_ATTR_CAP_?
UINT8 style_num_segments 	//number of style segments in line style
FIXED28_4 width 	//width of the line in pixels
FIXED28_4 miter_limit 	//miter limit in pixels
ADDRESS style 	//address of line style
/*line style is array of FIXED28_4. The array defines segments that each represents length
of foreground or background pixels in the style. If FLAG_START_WITH_GAP is
defined then the first segment in the style is background, otherwise it is foreground.
Renderer uses this array of segments repeatedly during rendering operation.*/
```

### Rendering command

+ RedDrawBase  
Common field to all rendering command

```c
RECT bounding_box //the affected area on the display area
Clip clip 	//the effective clip to set before rendering a command
```
+ `RED_DISPLAY_COPY_BITS`

```c
RedDrawBase
POINT source_position
/*Copy bits from the draw area to bounding_box on the draw area. Source area left top
corner is source_position and its height and width is equal to bounding_box height and
width. Source and destination rectangles can overlap.*/
```
+ `RED_DISPLAY_DRAW_FILL`

```c
RedDrawBase
Brush brush
UINT16 rop_descriptor
Mask mask
/*Fill bounding_box using brush as the fill pattern and rop_descriptor instructions. If the
mask is valid, it will limit the modified area (i.e., only pixels on the destination area that
their corresponding bits are set will be affected).*/
```
+ `RED_DISPLAY_DRAW_OPAQUE`

```c
RedDrawBase
ADDRESS source_image
RECT source_area
Brush brush
UINT16 rop_descriptor
UINT8 scale_mode
Mask mask
/*Combine pixels from source_area in source_image with the brush's pattern using
rop_descriptor instructions. The result image will be rendered into bounding_box. In
case scaling of source image is required it will be performed according to scale_mode
and before the combination with brush pixels. If mask is valid it will limit the modified
area.*/
```
+ `RED_DISPLAY_DRAW_COPY`

```c
RedDrawBase
ADDRESS source_image
RECT source_area
UINT16 rop_descriptor
UINT8 scale_mode
Mask mask
/*Copy pixels from source_area in source_image to bounding_box using rop_descriptor
instructions. In case scaling of source image is required it will be performed according
to scale_mode and before the copying to the draw area. If mask is valid it will limit the
modified area.*/
```
+ `RED_DISPLAY_DRAW_BLEND`

```c
RedDrawBase
ADDRESS source_image
RECT source_area
UINT16 rop_descriptor
UINT8 scale_mode
Mask mask
/*Mixing pixels from source_area in source_image with bounding_box pixels on the draw
area using rop_descriptor instructions. In case scaling of source image is required it will
be performed according to scale_mode and before the mixing with the draw area. If
mask is valid it will limit the modified area.*/
```
+ `RED_DISPLAY_DRAW_BLACKNESS`

```c
RedDrawBase
Mask mask
//Fill bounding_box with black pixels. If mask is valid it will limit the modified area.
```
+ `RED_DISPLAY_DRAW_WHITENESS`

```c
RedDrawBase
Mask mask
//Fill bounding_box with white pixels. If mask is valid it will limit the modified area.
```
+ `RED_DISPLAY_DRAW_INVERS`

```c
RedDrawBase
Mask mask
//Inverse all pixels in bounding_box. If mask is valid it will limit the modified area.
```
+ `RED_DISPLAY_DRAW_ROP3`

```c
RedDrawBase
ADDRESS source_image
RECT source_area
Brush brush
UINT8 rop3
UINT8 scale_mode
Mask mask
/*Mix pixels from source_area in source_image, bounding_box pixels in the draw area,
and the brush pattern. The method for mixing three pixels into the destination area (i.e.,
bounding_box) is defined by rop3 (i.e., ternary raster operations). In case scaling of
source image is required it will be performed according to scale_mode and before the
mixing. If mask is valid it will limit the modified area.*/
```
+ `RED_DISPLAY_DRAW_TRANSPARENT`

```c
RedDrawBase
ADDRESS source_image
RECT source_area
UINT32 transparent_color
UINT32 transparent _true_color
/*Copy pixels from source_area on source_image to bounding_box on the draw area. In
case scaling of source image is required it will use IMAGE_SCALE_NEAREST. Pixels
with value equal to the transparent color will be masked out. Transparent color is
provided in two forms: true color (i.e., RGB888) and the color in the original format
(i.e., before compression) .*/
```
+ `RED_DISPLAY_DRAW_ALPHA_BLEND`

```c
RedDrawBase
UINT8 alpha
ADDRESS source_image
RECT source_area
/*Alpha blend source_area of source_image on bounding_box of draw area using alpha
value or alternatively per pixel alpha value. In case scaling of source image is required,
it will use IMAGE_SCALE_INTERPOLATE mode. Alpha value is defined as 0 is full
transparency and 255 is full opacity. Format of source image can be pre-multiplied
ARGB8888 for per pixel alpha value.
New RGB color is defined as:
color' = (source_color * alpha) / 255
alpha' = (source_alpha * alpha) / 255
new_color = color' + ((255 - alpha' ) * destination_color) / 255
*/
```
+ `RED_DISPLAY_DRAW_STROKE`

```c
RedDrawBase
ADDRESS path 		//address of the PathSegList that defines the path to render
LineAttr attr
Bush brush
UINT16 fore_mode 	//foreground rop_descriptor
UINT16 back_mode 	//background rop_descriptor

/*Render path using brush line attribute and rop descriptors. If the line is styled (i.e.,
LINE_ATTR_FLAG_STYLED is set in attr.falgs) then background (i.e., inverse of the
style) is drawn using back_mode and the foreground is drawn using fore_mode. If the
line is not styled, the entire path is rendered using fore_mode.*/
```
+ `RED_DISPLAY_DRAW_TEXT`

```c
RedDrawBase
ADDRESS string – address of GlyphString
RECT back_area
Brush fore_brush
Brush back_brush
UINT16 fore_mode
UINT16 back_mode
/*Render string of glyph on the display area using brush fore_brush and the rop_descriptor
fore_mode. If back_area is not empty the renderer fill back_area on the display area
prior to rendering the glyph string. back_area is filled using back_brush and the
rop_descriptor back_mode.*/
```

### Video streaming commands

Spice支持服务器创建视频流，以在客户端显示区域上呈现视频内容。与其他渲染命令不同，可以使用有损或视频特定的压缩算法来压缩流数据。在视频帧到达时不需要渲染视频帧，也允许丢弃视频帧。这使得能够使用视频帧缓冲以实现更流畅的播放和音频同步。通过使用附加到音频和视频流的时间戳来实现音频同步。通过使用视频流，可以大大减少网络流量。创建流时，服务器使用`RED_DISPLAY_STREAM_CREATE`发送创建消息。在服务器创建流之后，他可以使用`RED_DISPLAY_STREAM_DATA`发送数据，或者通过使用`RED_DISPLAY_STREAM_CLIP`发送剪辑消息来设置新的流剪辑。一旦服务器不再需要流，他就可以使用`RED_DISPLAY_STREAM_DESTROY`发送destroy命令。服务器还可以通过使用`RED_DISPLAY_STREAM_DESTROY_ALL`消息来销毁所有活动流。

+ Stream flags  
`STREAM_FLAG_TOP_DOWN = 1`, stream frame line order is from top to bottom
+ Codec types  
`STREAM_CODEC_TYPE_MJPEG = 1`, this stream uses motion JPEG codec
+ `RED_DISPLAY_STREAM_CREATE`, RedStreamCreate

```c
UINT32 id 			//id of the new stream. It is the server's responsibility to manage stream ids
UINT32 flags  		//flags of the stream, any combination of STREAM_FLAG_?
UINT32 codec_type 	//type of codec used for this stream, one of STREAM_CODEC_TYPE_?
UINT64 reserved 	//must be zero
UINT32 stream_width 	//width of the source frame.
UINT32 stream_height 	//height of the source frame
UINT32 source_width 	//actual frame width to use, must be less or equal to stream_width.
UINT32 source_height 	//actual frame height to use, must be less or equal to stream_height.
RECT destination 		//area to render into on the client display area
Clip clip 				//clipping of the stream
```
+ `RED_DISPLAY_STREAM_DATA`, RedStreamData

```c
UINT32 id 	//stream id (i.e., RedStreamCreate.id)
UINT32 multimedia_time 	//frame time stamp
UINT32 data_size 		//stream data size to consume in bytes
UINT32 pad_size 		//additional data padding in bytes
UINT8[] data 			//stream data depending on RedStreamCreate.codec_type. Size of data is ( data_size + pad_size)
```
+ `RED_DISPLAY_STREAM_CLIP`, RedStreamClip

```c
UINT32 id 		//stream id (i.e., RedStreamCreate.id)
Clip clip 		//new clipping of the stream
```
+ `RED_DISPLAY_STREAM_DESTROY`, UINT32  
UINT32 – id of stream to destroy
+ `RED_DISPLAY_STREAM_DESTROY_ALL`, VOID  
Destroy all active streams

### Cache control

+ Resource type  
`RED_RES_TYPE_IMAGE = 1`
+ RedResourceID

```c
UINT8 type 		//type of the resource, one of RED_RES_TYPE_?
UINT64 id 		//id of the resource
```
+ RedResourceList

```c
UINT16 count 		//number of items in resources
RedResourceID[] resources 		//list of resources id
```
+ `RED_DISPLAY_INVAL_LIST`, RedResourceList  
RedResourceList – list of resources to remove from cache
+ `RED_DISPLAY_INVAL_ALL_IMAGES`, RedWaitForChannels  
Remove all images from the image cache. The client must use RedWaitForChannels (for
more info see 2.19 Channel synchronization) to synchronize with other channels before
clearing the cache.
+ `RED_DISPLAY_INVAL_PALETTE`, UINT64  
UINT64 – id of palette, client needs to remove palette with that id from the cache
+ `RED_DISPLAY_INVAL_ALL_PALETTES`, VOID  
Remove all palettes from palette cache

## Cursor 通道定义

Spice协议定义了一组用于控制远程显示区域上光标形状和位置的消息，光标位置消息与客户端鼠标模式无关（参见“鼠标模式”）。 Spice协议还定义了一组用于管理客户端站点上的游标形状缓存的消息。 客户端必须严格遵守所有此类指令。 服务器发送`RED_CURSOR_INIT`以设置当前指针状态（即，形状，位置，可见性等）并清除形状缓存。 在服务器发送init消息之后，它可以发送除`RED_CURSOR_INIT`之外的任何其他游标命令。 服务器可以发送`RED_CURSOR_RESET`消息,以禁用游标并重置游标缓存。 在此消息之后，服务器可以发送的唯一有效消息是`RED_CURSOR_INIT`。 用于光标通道的相关远程显示区域是具有相同通道ID的显示通道之一（即，`RedLinkMess.channel_id`）。

### 服务端消息

```c
RED_CURSOR_INIT = 101
RED_CURSOR_RESET = 102
RED_CURSOR_SET = 103
RED_CURSOR_MOVE = 104
RED_CURSOR_HIDE = 105
RED_CURSOR_TRAIL = 106
RED_CURSOR_INVAL_ONE = 107
RED_CURSOR_INVAL_ALL = 108
```
+ Cursors types

```c
CURSOR_TYPE_ALPHA = 0
CURSOR_TYPE_MONO = 1
CURSOR_TYPE_COLOR4 = 2
CURSOR_TYPE_COLOR8 = 3
CURSOR_TYPE_COLOR16 = 4
CURSOR_TYPE_COLOR24 = 5
CURSOR_TYPE_COLOR32 = 6
```
+ CursorHeader

```c
UINT64 unique 	//unique identifier of the corresponding cursor shape. 
//It is used for storing and retrieving cursors from the cursor cache.
UINT16 type 	//type of the shape, one of CURSOR_TYPE_?
UINT16 width 	//width of the shape
UINT16 height 	//height of the shape
UINT16 hot_spot_x 	//position of hot spot on x axis
UINT16 hot_spot_y 	//position of hot spot on y axis
```
+ Cursor flags

```c
CURSOR_FLAGS_NONE = 1 //set when RedCursor (see below) is invalid
CURSOR_CURSOR_FLAGS _CACHE_ME = 2 //set when the client should add this shape to the shapes cache. 
//The client will use CursorHeader.unique as cache key.
CURSOR_FLAGS_FROM_CACHE = 4
//set when the client should retrieve the cursor shape, using CursorHeader.unique as key,
//from the shapes cache. In this case all fields of CursorHeader except for 'unique' are invalid.
```
+ RedCursor

```c
UINT32 flags 	//any valid combination of RED_CURSOR_? 
CursorHeader header
UINT8[] data 	//actual cursor shape data, the size is determine by width, height and type from CursorHeader. 
//Next we will describe in detail the shape data format according to  cursor type:
/*
1. ALPHA, alpha shape – data contains pre-multiplied ARGB8888 pixmap. Line stride is is <width * 4>.
2. MONO, monochrome shape - data contains two bitmaps with size <width> * <height>. The first bitmap is AND mask and the second is XOR mask. Line stride is ALIGN(<width>, 8) / 8. Bits order within every byte is big endian.
3. COLOR4, 4 bits per pixel shape - First data region is pixmap: the stride of the pixmap is ALIGN(width , 2) / 2; every nibble is translated to a color usingthe
color palette; Nibble order is big endian. Second data region contain 16 colors
palette: each entry is 32 bit RGB color. Third region is a bitmap mask: line stride is ALIGN(<width>, 8) / 8; bits order within every byte is big endian.
4. COLOR4, 8 bits per pixel shape - First data region is pixmap: the stride of the
pixmap is <width>; every byte is translated to color using the color palette.
Second data region contain 256 colors palette: each entry is 32 bit RGB color.
Third region is a bitmap mask: line stride is ALIGN(<width>, 8) / 8; bits order
within every byte is big endian.
5. COLOR16, 16 bits per pixel shape - First data region is pixmap: the stride of the
pixmap is <width * 2>; every UINT16 is RGB_555. Second region is a bitmap
mask: line stride is ALIGN(<width>, 8) / 8; bits order within every byte is big
endian.
6. COLOR24, 24 bits per pixel shape - First data region is pixmap: the stride of the
pixmap is <width * 3>; every UINT8[3] is RGB_888. Second region is a bitmap
mask: line stride is ALIGN(<width>, 8) / 8; bits order within every byte is big
endian.
7. COLOR32, 32 bits per pixel shape - First data region is pixmap: the stride of the
pixmap is <width * 4>,;every UINT32 is RGB_888. Second region is a bitmap
mask: line stride is ALIGN(<width>, 8) / 8; bits order within every byte is big
endian.
For more deatails on drawing the cursor shape see Section 6.2.*/
```

+ `RED_CURSOR_INIT`, RedCursorInit

```c
POINT16 position 		//position of mouse pointer on the relevant display area. Not relevant in client mode.
UINT16 trail_length 	//number of cursors in the trail excluding main cursor.
UINT16 trail_frequency  //millisecond interval between trail updates.
UIN8 visible 			//if 1, the cursor is visible. If 0, the cursor is invisible.
RedCursor cursor 		//current cursor shape
```
+ `RED_CURSOR_RESET`, VOID
+ `RED_CURSOR_SET`, RedCursorSet

```c
POINT16 position //position of mouse pointer on the relevant display area. not relevant in client mode.
UINT8 visible 	//if 1, the cursor is visible. If 0, the cursor is invisible.
RedCursor cursor //current cursor shape
```
+ `RED_CURSOR_MOVE`, POINT16  
POINT16 – new mouse position. Not relevant in client mode. This message also
implicitly sets cursor visibility to 1.
+ `RED_CURSOR_HIDE`, VOID  
Hide pointer on the relevant display area.
+ `RED_CURSOR_TRAIL`  
UINT16 length - number of cursors in the trail excluding main cursor.
UINT16 frequency - millisecond interval between trail updates
+ `RED_CURSOR_INVAL_ONE`, UINT64  
UINT64 – id of cursor shape to remove from the cursor cache
+ `RED_CURSOR_INVAL_ALL`, VOLD  
Clear cursor cache

### Drawing the cursor shape according to the cursor type

本节仅适用于服务器鼠标模式。 通过将光标热点放在当前光标位置来完成显示区域上的光标形状定位。

+ Alpha  
no spacial handling, just bland the shape on the display area.
+ Monochrome  
	+ For each cleared bit in the AND mask clear the corresponding bits in the relevant 
	pixel on the display area
	+ For each set bit in the XOR mask reverse the corresponding bits in the relevant pixel 
	on the display area
+ Color  
	+ If the source color is black and mask bit is set, NOP.
	+ Else, if the source color is white and the mask bit is set, reverse all bits in the 
	relevant pixel on the display area.
	+ Else, put source color.

## Playback 通道定义

Spice支持发送音频流以便在客户端进行播放。 服务器使用`RED_PLAYBACK_DATA`消息在音频数据包中发送音频流。 音频数据包的内容由服务器使用`RED_PLAYBACK_MODE`消息发送的playback模式控制。 服务器可以使用`RED_PLAYBACK_START`和`RED_PLAYBACK_STOP`消息来启动和停止流。 仅在开始和停止消息之间允许发送音频数据包。 仅在停止状态和发送至少一个模式消息后才允许发送开始消息。 仅在启动状态期间允许发送停止消息。

### Server messages

```c
RED_PLAYBACK_DATA = 101
RED_PLAYBACK_MODE = 102
RED_PLAYBACK_START = 103
RED_PLAYBACK_STOP = 104
```

### Audio format

`RED_PLAYBACK_FMT_S16` = 1, each channel sample is a 16 bit signed integer

### Playback data mode

```c
//Two types of data mode are available: (1) raw PCM data and (2) compressed data in CELT 0_5_1 format.
RED_PLAYBACK_DATA_MODE_RAW = 1
RED_PLAYBACK_DATA_MODE_CELT_0_5_1= 2
```

### Playback channel capabilities

`RED_PLAYBACK_CAP_CELT_0_5_1` = 0  
Spice client needs to declare support of `CELT_5_1` in channel capabilities in order to allow
the server to send playback packets in `CELT_0_5_1` format.

### `RED_PLAYBACK_MODE`, RedPlaybackMode

```c
UINT32 time 		//– server time stamp
UINT32 mode 		//– one of RED_PLAYBACK_DATA_MODE_?
UINT8[] data 		//– specific data, content depend on mode
```

### `RED_PLAYBACK_START`, RedRecordStart

```c
UINT32 channels 	//– number of audio channels
UINT32 format 		//– one of RED_PLAYBACK_FMT_?
UINT32 frequency 	//– channel samples per second
```

### `RED_PLAYBACK_DATA`, RedPlaybackPacket

```c
UINT32 time 		//- server time stamp
UINT8[] data 		//– playback data , content depend on mode
```

### `RED_PLAYBACK_STOP`, VOID

Stop current audio playback

## Record 通道定义

Spice支持将客户端捕获的音频流传输到服务器。 Spice服务器使用`RED_RECORD_START`消息启动音频捕获。此消息指示客户端开始传输捕获的音频。作为响应，客户端使用`REDC_RECORD_START_MARK`发送流开始的时间戳。客户端发送开始标记后，它可以使用`REDC_RECORD_DATA`开始传输音频流数据。在使用`REDC_RECORD_MODE`的任何其他消息之前，客户端必须发送一条模式消息。这是为了通知服务器将传输什么类型的数据。模式消息也可以在任何其他时间传输，以便切换`REDC_RECORD_DATA`传递的数据类型。服务器可以发送`RED_RECORD_STOP`以停止捕获的音频流。仅当流处于停止状态时才允许发送开始消息。仅当流处于启动状态时才允许发送停止消息和数据消息。仅在开始消息和第一个数据消息之间允许发送标记消息。

### Server messages

```c
RED_RECORD_START = 101
RED_RECORD_STOP = 102
```

### Client messages

```c
REDC_RECORD_DATA = 101
REDC_RECORD_MODE = 102
REDC_RECORD_START_MARK = 103
```

### Audio format

`RED_RECORD_FMT_S16 = 1`, each channel sample is a 16 bit signed integer

### Record data mode

```c
//Two types of data mode are available: (1) raw PCM data (2) compressed data in CELT 0.5.1 format.
RED_RECORD_DATA_MODE_RAW = 1
RED_RECORD_DATA_MODE_CELT_0_5_1 = 2
```

### Record channel capabilities

`RED_PLAYBACK_CAP_CELT_0_5_1 = 0`  
Spice server needs to declare support of `CELT_5_1` in channel capabilities in order to allow
the client to send recorded packets in `CELT_0_5_1` format.

### `REDC_RECORD_MODE`, RedcRecordMode

```c
UINT32 time 		//– client time stamp
UINT32 mode 		//– one of RED_RECORD_DATA_MODE_?
UINT8[] data 		//– specific data, content depend on mode
```

### `RED_RECORD_START`, RedRecordStart

```c
UINT32 channels 	//– number of audio channels
UINT32 format 		//– one of RED_AUDIO_FMT_?
UINT32 frequency 	//– channel samples per second
```

### `REDC_RECORD_START_MARK`, UINT32

UINT32 – client time stamp of stream start

### `REDC_RECORD_DATA`, RedcRecordPacket

```c
UINT32 time 		//- client time stamp
UINT8[] data 		//– recorded data , content depend on mode
```

### `RED_RECORD_STOP`, VOID

Stop current audio capture