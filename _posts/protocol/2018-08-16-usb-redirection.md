---
title: USB重定向协议
description: usb网络重定向协议描述,版本0.6(2012-12-13)
categories: protocol
tags:
  - usbredir
  - protocol
---

## 概述

本文档描述的协议是单个usb设备的隧道化传输，不是整个集线器。最典型的使用场景是usb设备插在client端，使usb设备在guest虚拟机内部显示和使用，就好象usb设备直接插在虚拟机上一样。所描述的协议假定有可靠的双向传输通信，例如tcp套接字。协议中的所有整数都是通过管道以最低有效字节优先的顺序发送的。通过管道发送的所有结构都是打包的（没有填充字节）。  
定义：

+ usb-device：usb设备，通过隧道传输。
+ usb-guest：连接到usb设备并使用它的实体（guest），就像直接连接到它一样。 
+ usb-host：使usb-device可供usb-guest使用的实体（client）。

## 基本包结构/通信

在usb-guest和usb-host之间交换的每个数据包都以`usb_redir_header`开头，后跟可选的数据包类型指定的头部，加上可选的附加数据。  
每个包都以`usb_redir_header`开始，结构如下：

```c
struct usb_redir_header {
    uint32_t type;
    uint32_t length;
    uint32_t id;
}
//或者如果两端都有usb_redir_cap_64bits_ids 能力：
struct usb_redir_header {
    uint32_t type;
    uint32_t length;
    uint64_t id;
}
```
>type： 数据包的类型id，来自于type的枚举类型  
length： 可选的数据包类型指定的头部加上可选的数据的长度。  
id： 一个唯一的值，usb-guest发送包时产生，usb-host响应包返回。usb-guest使用id来匹配请求和响应。

有两种类型的数据包

1. 控制包 control packets
2. 数据包 data packets

控制包在usb-host内同步处理，它会将请求传递给host os，然后等待响应。在这期间，usb-host将停止处理更多的包。 对于数据包，usb-host将其和request一起传递给主机操作系统，让usb-host进程知道何时有来自usb-device的响应。

请注意，只有在受控制包影响的设备/接口/端点上没有数据包待处理时，才应将控制包发送到usb-host。 任何待处理的数据包都将被丢弃，任何活动的iso流/分配的bulk流将被停止/free-ed。

## Packet type list

```c
//control packets
usb_redir_hello
usb_redir_device_connect
usb_redir_device_disconnect
usb_redir_reset
usb_redir_interface_info
usb_redir_ep_info
usb_redir_set_configuration
usb_redir_get_configuration
usb_redir_configuration_status
usb_redir_set_alt_setting
usb_redir_get_alt_setting
usb_redir_alt_setting_status
usb_redir_start_iso_stream
usb_redir_stop_iso_stream
usb_redir_iso_stream_status
usb_redir_start_interrupt_receiving
usb_redir_stop_interrupt_receiving
usb_redir_interrupt_receiving_status
usb_redir_alloc_bulk_streams
usb_redir_free_bulk_streams
usb_redir_bulk_streams_status
usb_redir_cancel_data_packet
usb_redir_filter_reject
usb_redir_filter_filter
usb_redir_device_disconnect_ack
usb_redir_start_bulk_receiving
usb_redir_stop_bulk_receiving
usb_redir_bulk_receiving_status

//data packets
usb_redir_control_packet
usb_redir_bulk_packet
usb_redir_iso_packet
usb_redir_interrupt_packet
usb_redir_buffered_bulk_packet
```

## Status code list

许多usb-host回复都有一个状态字段，该字段可以具有以下值：

```c
enum {
    usb_redir_success,
    usb_redir_cancelled,    /* The transfer was cancelled */
    usb_redir_inval,        /* Invalid packet type / length / ep, etc. */
    usb_redir_ioerror,      /* IO error */
    usb_redir_stall,        /* Stalled */
    usb_redir_timeout,      /* Request timed out */
    usb_redir_babble,       /* The device has "babbled" */
};
```
请注意，在将来的版本中，可能还有其他状态代码来指示新的/其他错误条件。 因此，任何未知状态值都应该被解释为错误

## `usb_redir_hello`

```c
usb_redir_header.type = usb_redir_hello
usb_redir_header.length = <see description>
usb_redir_header.id =0 //(always as this is an unsolicited(主动提供) packet)

struct usb_redir_hello_header {
    char     version[64];
    uint32_t capabilities[0];
}
```
没有包类型特定的附加数据。  
一旦建立连接，双方就发送这种类型的包。 该数据包必须是双方发送的第一个数据包！ 该包包含：

+ version：一个以0结尾的自由格式的版本字符串，对于日志记录非常有用，不应该被解析！ 建议格式：“qemu 0.13”，“usb-redir-daemon 0.1”等
+ 功能：用于通知功能的可变长度数组。

请注意，由于在收到`usb_redir_hello`数据包之前不知道对方的能力，因此hello数据包始终具有32位id字段！  
length字段的值取决于capabilities数组的大小。 值为64 + capabilities-array-size * sizeof（uint32_t）。  
Currently the following capabilities are defined:

```c
enum {
    /* Supports USB 3 bulk streams */
    usb_redir_cap_bulk_streams, 
    /* The device_connect packet has the device_version_bcd field */
    usb_redir_cap_connect_device_version,
    /* Supports usb_redir_filter_reject and usb_redir_filter_filter pkts */
    usb_redir_cap_filter,
    /* Supports the usb_redir_device_disconnect_ack packet */
    usb_redir_cap_device_disconnect_ack,
    /* The ep_info packet has the max_packet_size field */
    usb_redir_cap_ep_info_max_packet_size,
    /* Supports 64 bits ids in usb_redir_header */
    usb_redir_cap_64bits_ids,
    /* Supports 32 bits length in usb_redir_bulk_packet_header */
    usb_redir_cap_32bits_bulk_length,
    /* Supports bulk receiving / buffered bulk input */
    usb_redir_cap_bulk_receiving,
};
```

## `usb_redir_device_connect`

```c
usb_redir_header.type = usb_redir_device_connect
usb_redir_header.length = sizeof(usb_redir_device_connect_header)
usb_redir_header.id  = 0 // (always as this is an unsolicited packet)

enum {
    usb_redir_speed_low,
    usb_redir_speed_full,
    usb_redir_speed_high,
    usb_redir_speed_super,
    usb_redir_speed_unknown = 255
}

struct usb_redir_device_connect_header {
    uint8_t speed;
    uint8_t device_class;
    uint8_t device_subclass;
    uint8_t device_protocol;
    uint16_t vendor_id;
    uint16_t product_id;
    uint16_t device_version_bcd;
}
```
No packet type specific additional data.

当设备可用时，该数据包由usb-host发送（usb-host可能等待设备插入）。  
只有当双方都具有`usb_redir_cap_connect_device_version`功能时，才应发送（并在接收时预期）`device_version_bcd`字段。 如果不是这种情况，数据包的长度将减少2个字节！  
请注意，usb-host可以重新使用现有连接用于新的/重新插入的设备，在这种情况下，可以在`usb_redir_device_disconnect`消息之后发送该数据包以通知usb-guest新设备可用。  
注意usbredir-host 在发送`usb_redir_device_connect_info`之前，必须在发送`usb_redir_interface_info`之后接着发送`usb_redir_ep_info`。

## `usb_redir_device_disconnect`

```c
usb_redir_header.type =  usb_redir_device_disconnect
usb_redir_header.length = 0
usb_redir_header.id = 0// (always as this is an unsolicited packet)
```
No packet type specific header.  
No packet type specific additional data.  
该数据包可以由usb-host发送，以指示设备已断开连接（拔掉插头）。 请注意，在某些平台上，usb-host可能不会意识到设备已断开连接，直到一个usb包发送给设备。

## `usb_redir_reset`

```c
usb_redir_header.type  = usb_redir_reset
usb_redir_header.length = 0
```
No packet type specific header.
No packet type specific additional data.

这个数据包可以由usb-guest发送，以便重置usb设备。 请注意，出现问题后，usb-host可能无法在重置后重新连接到设备！ 如果发生这种情况，usb-host将发送`usb_redir_device_disconnect`数据包。

## `usb_redir_interface_info`

```c
usb_redir_header.type  =   usb_redir_interface_info
usb_redir_header.length = sizeof(usb_redir_interface_info_header)
usb_redir_header.id  =  0 //(always as this is an unsolicited packet)

struct usb_redir_interface_info_header {
    uint32_t interface_count;
    uint8_t interface[32];
    uint8_t interface_class[32];
    uint8_t interface_subclass[32];
    uint8_t interface_protocol[32];
}
```
No packet type specific additional data.  
该数据包由usb-host发送，以通知usb-guest有关设备接口的信息。 它包含`interface_count`个接口的接口编号，类和协议信息。 这在（成功的）初始连接，`set_config和set_alt_setting`时发送。

## `usb_redir_ep_info`

```c
usb_redir_header.type    =   usb_redir_ep_info
usb_redir_header.length =  sizeof(usb_redir_ep_info_header)
usb_redir_header.id =     0 //(always as this is an unsolicited packet)

enum {
    /* Note these 4 match the usb spec! */
    usb_redir_type_control,
    usb_redir_type_iso,
    usb_redir_type_bulk,
    usb_redir_type_interrupt,
    usb_redir_type_invalid = 255
}

struct usb_redir_ep_info_header {
    uint8_t type[32];
    uint8_t interval[32];
    uint8_t interface[32];
    uint16_t max_packet_size[32];
}
```
No packet type specific additional data.

这个数据包由usb-host发送，让usb-guest知道所有可能端点所属的端点类型，间隔和接口，先是0-15 out，然后是0-15 in。这会在 初始化连接，`set_config`和`set_alt_setting`时发送  
只有当双方都具有`usb_redir_cap_ep_info_max_packet_size`功能时，才应发送`max_packet_size`字段（并在接收时预期）。 如果不是这种情况，数据包的长度将减少64个字节！

## `usb_redir_set_configuration`

```c
usb_redir_header.type =   usb_redir_set_configuration
usb_redir_header.length= sizeof(usb_redir_set_configuration_header)

struct usb_redir_set_configuration_header {
    uint8_t configuration;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包以设置（更改）usb设备的活动配置。

## `usb_redir_get_configuration`

```c
usb_redir_header.type:    usb_redir_get_configuration
usb_redir_header.length:  0
```
No packet type specific header.  
No packet type specific additional data.  
usb-guest可以发送此数据包以获取（查询）usb设备的活动配置。

## `usb_redir_configuration_status`

```c
usb_redir_header.type:    usb_redir_configuration_status
usb_redir_header.length:  sizeof(usb_redir_configuration_status_header)

struct usb_redir_configuration_status_header {
    uint8_t status;
    uint8_t configuration;
}
```
No packet type specific additional data.  
这是由usb-host发送的，用于响应`usb_redir_set_configuration`/`usb_redir_get_configuration`数据包。 它会报告状态代码，如果命令成功，还会返回 resulting/active 配置。
请注意，在成功的`usb_redir_set_configuration`命令之后，usbredir-host必须首先发送`usb_redir_ep_info`，然后发送`usb_redir_interface_info`，然后再发送`usb_redir_configuration_status`，以确保usb-guest在开始使用新配置时具有新信息。

## `usb_redir_set_alt_setting`

```c
usb_redir_header.type:    usb_redir_set_alt_setting
usb_redir_header.length:  sizeof(usb_redir_set_alt_setting_header)

struct usb_redir_set_alt_setting_header {
    uint8_t interface;
    uint8_t alt;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包，以将接口`<interface>`的alt_setting设置（更改）为`<alt>`。

## `usb_redir_get_alt_setting`

```c
usb_redir_header.type:    usb_redir_get_alt_setting
usb_redir_header.length:  sizeof(usb_redir_get_alt_setting_header)

struct usb_redir_get_alt_setting_header {
    uint8_t interface;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包以获取（查询）usb设备接口的活动alt_setting。

## `usb_redir_alt_setting_status`

```c
usb_redir_header.type:    usb_redir_alt_setting_status
usb_redir_header.length:  sizeof(usb_redir_alt_setting_status_header)

struct usb_redir_alt_setting_status_header {
    uint8_t status;
    uint8_t interface;
    uint8_t alt;
}
```
No packet type specific additional data.  
这是由usb-host发送的，用于响应`usb_redir_set_alt_setting` / `usb_redir_get_alt_setting`数据包。 它会报告状态代码，受影响的接口以及如果成功还包括该接口的结果/活动alt_setting。

请注意，在成功的`usb_redir_set_alt_setting`命令之后，usbredir-host必须首先发送`usb_redir_ep_info`，然后发送`usb_redir_interface_info`，然后再发送`usb_redir_alt_setting_statu`s，以确保usb-guest在开始使用新的alt设置时具有新信息

## `usb_redir_start_iso_stream`

```c
usb_redir_header.type:    usb_redir_start_iso_stream
usb_redir_header.length:  sizeof(usb_redir_start_iso_stream_header)

struct usb_redir_start_iso_stream_header {
    uint8_t endpoint;
    uint8_t pkts_per_urb;
    uint8_t no_urbs;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包，以便在usb-device的指定端点上启动iso流。  
此功能分配 `no_urbs` 个urb，每个urb有`pkts_per_urb`个iso数据包/帧。   对于iso输入端点，这些urb将立即提交给设备，对于iso输出端点，usb-host将等待它收到（`pkts_per_urb` * no_urbs / 2）数据包以填充其缓冲区，然后再提交第一个urb。

## `usb_redir_stop_iso_stream`

```c
usb_redir_header.type:    usb_redir_stop_iso_stream
usb_redir_header.length:  sizeof(struct usb_redir_start_iso_stream_header)

struct usb_redir_stop_iso_stream_header {
    uint8_t endpoint;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包以停止指定端点上的iso流。 这将取消所有待处理的urb，刷新usb-host的缓冲区并释放所有相关资源。 请注意，usb-guest仍然可以在发送之后从端点中的isoc接收isoc数据包，因为某些数据包可能已经在传输管道内。

## `usb_redir_iso_stream_status`

```c
usb_redir_header.type:    usb_redir_iso_stream_status
usb_redir_header.length:  sizeof(usb_redir_iso_stream_status_header)

struct usb_redir_iso_stream_status_header {
    uint8_t status;
    uint8_t endpoint;
}
```
No packet type specific additional data.  
该数据包由usb-host发送，以响应`usb_redir_start_iso_stream`或`usb_redir_stop_iso_stream`数据包。   请注意，对于输出iso流的启动，成功状态仅指示已成功分配所有缓冲区，在缓冲足够的数据包之前不会启动实际流。  
请注意，如果iso输出流发生错误，也可以通过usb-host主动发送，请参阅`usb_redir_iso_packet`。  
为了允许usb-guest检测到流是否被阻止，如果流由于`usb_redir_stop_iso_stream`之外的任何原因而被停止，usb-host将总是报告为`usb_redir_stall`状态，

## `usb_redir_start_interrupt_receiving`

```c
usb_redir_header.type:    usb_redir_start_interrupt_receiving
usb_redir_header.length:  sizeof(usb_redir_start_interrupt_receiving_header)

struct usb_redir_start_interrupt_receiving_header {
    uint8_t endpoint;
}
```
usb-guest可以发送此数据包以开始从usb设备的指定端点接收中断。  
此功能仅用于input中断端点。 需要及时轮询输入中断端点，否则数据可能会丢失。 因此，对于输入中断端点，usb-host负责提交和重新提交urb。  
收到此数据包后，usb-host将使用描述符中的interval和maxPacketSize启动到端点的中断传输。 当此传输完成时，usb-host将向usb-guest发送`usb_redir_interrupt_packet`，并将重新提交urb。

## `usb_redir_stop_interrupt_receiving`

```c
usb_redir_header.type:    usb_redir_stop_interrupt_receiving
usb_redir_header.length:  sizeof(struct usb_redir_start_interrupt_receiving_header)

struct usb_redir_stop_interrupt_receiving_header {
    uint8_t endpoint;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包以停止指定端点上的中断接收。 这将取消待处理的urb。 请注意，usb-guest在发送之后仍然可以接收`usb_redir_interrupt_packet-s`，因为一些数据包可能已经在传输管道内。

## `usb_redir_interrupt_receiving_status`

```c
usb_redir_header.type:    usb_redir_interrupt_receiving_status
usb_redir_header.length:  sizeof(usb_redir_interrupt_receiving_status_header)

struct usb_redir_interrupt_receiving_status_header {
    uint8_t status;
    uint8_t endpoint;
}
```
No packet type specific additional data.   
该数据包由usb-host发送，以响应`usb_redir_start_interrupt_receiving`或`usb_redir_stop_interrupt_receiving`数据包。  
请注意，如果重新提交中断urb时出错，也可以由usb-host主动发送。  
为了允许usb-guest检测到流是否被阻止，如果流由于`usb_redir_stop_interrupt_receiving`之外的任何原因而被停止，则usb-host将始终报告为`usb_redir_stall`状态。

## `usb_redir_alloc_bulk_streams`

```c
usb_redir_header.type:    usb_redir_alloc_bulk_streams
usb_redir_header.length:  sizeof(usb_redir_alloc_bulk_streams_header)

struct usb_redir_alloc_bulk_streams_header {
    uint8_t endpoint;
    uint8_t no_streams;
}
```
No packet type specific additional data.   
这个数据包可以由usb-guest发送到usb-host，以请求usb-host分配ID，这样usb-guest就可以使用最多no_streams流ID。

## `usb_redir_free_bulk_streams`

```c
usb_redir_header.type:    usb_redir_free_bulk_streams
usb_redir_header.length:  sizeof(usb_redir_free_bulk_streams_header)

struct usb_redir_free_bulk_streams_header {
    uint8_t endpoint;
}
```
No packet type specific additional data.   
这个数据包可以由usb-guest发送到usb-host，以释放在端点上之前分配的任何bulk流。

## `usb_redir_bulk_streams_status`

```c
usb_redir_header.type:    usb_redir_bulk_streams_status
usb_redir_header.length:  sizeof(usb_redir_bulk_streams_status_header)

struct usb_redir_bulk_streams_status_header {
    uint8_t status;
    uint8_t endpoint;
    uint8_t no_streams;
}
```
No packet type specific additional data.

该数据包由usb-host发送，以响应`usb_redir_alloc_bulk_streams`或`usb_redir_free_bulk_streams`数据包。 请注意，响应`usb_redir_alloc_bulk_streams`是成功状态时，由于主机控制器/设备限制，`no_streams`可能会少于请求。 响应`usb_redir_alloc_bulk_streams`是成功状态时，usb-guest可以使用流ID 1到no_streams。

## `usb_redir_start_bulk_receiving`

```c
usb_redir_header.type:    usb_redir_start_bulk_receiving
usb_redir_header.length:  sizeof(usb_redir_start_bulk_receiving_header)

struct usb_redir_start_bulk_receiving_header {
    uint32_t stream_id;
    uint32_t bytes_per_transfer;
    uint8_t endpoint;  
    uint8_t no_transfers;
}
```
No packet type specific additional data.  
usb-guest发送这个包启动从bulk端点读取缓存的数据。  
收到此数据包后，usb-host将提交`no_transfers`个transfer中的bulk（每一个transfer中含有`bytes_per_transfer`字节）到usb-device的指定端点。 一旦一个transfer完成，usb-host将向usb-guest发送带有接收到的数据的`usb_redir_buffered_bulk_packet`，并立即重新提交已完成的transfer。  
注意`bytes_per_transfer`必须是端点`max_packet_size`的倍数。  
请注意，此数据包只应发送到具有`usb_redir_cap_bulk_receiving`功能的usb-hosts。

## `usb_redir_stop_bulk_receiving`

```c
usb_redir_header.type:    usb_redir_stop_bulk_receiving
usb_redir_header.length:  sizeof(usb_redir_stop_bulk_receiving_header)

struct usb_redir_stop_bulk_receiving_header {
    uint32_t stream_id;
    uint8_t endpoint;
}
```
No packet type specific additional data.  
usb-guest可以发送此数据包以停止指定端点上的bulk接收。 这将取消所有待处理的transfer。 请注意，usb-guest在发送之后仍然可以接收`usb_redir_bulk_packet-s`，因为一些数据包可能已经在传输管道内。  
请注意，此数据包只应发送到具有`usb_redir_cap_bulk_receiving`功能的usb-hosts。

## `usb_redir_bulk_receiving_status`

```c
usb_redir_header.type:    usb_redir_bulk_receiving_status
usb_redir_header.length:  sizeof(usb_redir_bulk_receiving_status_header)

struct usb_redir_bulk_receiving_status_header {
    uint32_t stream_id;
    uint8_t endpoint;
    uint8_t status;
}
```
No packet type specific additional data.  
该数据包由usb-host发送，以响应`usb_redir_start_bulk_receiving`或`usb_redir_stop_bulk_receiving`数据包。  
请注意，如果重新提交bulk transfer时出错，也可以由usb-host主动发送。  
为了允许usb-guest检测到流是否被阻止，如果流由于`usb_redir_stop_interrupt_receiving`之外的任何原因而被停止，则usb-host将始终报告为`usb_redir_stall`状态。  
请注意，此数据包只应发送给具有`usb_redir_cap_bulk_receiving`功能的usb-guests。

## `usb_redir_cancel_data_packet`

```c
usb_redir_header.type:    usb_redir_cancel_data_packet
usb_redir_header.id       <id of packet to cancel>
usb_redir_header.length:  0
```
No packet type specific header.  
No packet type specific additional data.  
usb-guest可以发送此数据包以取消先前发送的数据包，id应设置为usb-guest希望取消的数据包使用的id。  
请注意，usb-guest将总是收到一个具有相同类型相同ID的数据包，usb-guest可以通过查看返回数据包的状态字段检查数据包是否正常完成（在取消数据包由usb-host处理之前）或者是被取消。

## `usb_redir_filter_reject`

```c
usb_redir_header.type:    usb_redir_filter_reject
usb_redir_header.length:  0
usb_redir_header.id:      0 (always as this is an unsolicited packet)
```
No packet type specific header.  
No packet type specific additional data.  
这个数据包是在usb-guest收到`usb-redir_device_connect`或`usb_redir_interface_info`数据包并被设备过滤器拒绝后由usb-guest发送的。 此数据包只应发送到具有`usb_redir_cap_filter`功能的usb-hosts。

## `usb_redir_filter_filter`

```c
usb_redir_header.type:    usb_redir_filter_filter
usb_redir_header.length:  string-length + 1 (for 0 termination)
usb_redir_header.id:      0 (always as this is an unsolicited packet)
```
No packet type specific header.  
附加数据包含一个C风格的usredirfilter字符串。  
可以在hello数据包之后直接发送此数据包，以通知对方过滤器已就位且某些设备可能被拒绝。  
usredirfilter由一个或多个规则组成，其中字符串形式的每个规则都具有以下格式： <类>，<厂商>，<产品>，<版本>，<允许>  
值可以是十进制格式，也可以是以0x预先固定的十六进制格式，值-1可以用于允许任何值。  
过滤器的所有规则都是连续的，用“|”分隔 形成单个usredirfilter字符串的字符： <规则1> | <规则2> | <规则3>  
如果设备不匹配任何规则，则过滤的结果是拒绝，设备将被拒绝。  
有关过滤的更多信息，请参阅usbredirfilter.h  
该数据包只应发送给具有`usb_redir_cap_filter`功能的对等体。

## `usb_redir_device_disconnect_ack`

```c
usb_redir_header.type:    usb_redir_device_disconnect_ack
usb_redir_header.length:  0
usb_redir_header.id:      0 (as the id of the device_disconnect is always 0)
```
No packet type specific header.  
No packet type specific additional data.  
在处理usb-host发送的`usb_redir_device_disconnect`数据包之后，usb-guest会发送此数据包。 这让想要重新使用现有连接的usb-host知道usb-guest已经看到断开连接并且不会再发送用于已断开连接设备的数据包。 如果没有这个，那么usb-host可能会有一个可用的新设备，但它仍在接收旧设备的数据包，因为usb-guest尚未看到断开连接。  
请注意，只有双方都具有`usb_redir_cap_device_disconnect_ack`功能时才会发送此数据包。

## `usb_redir_control_packet`

```c
usb_redir_header.type:    usb_redir_control_packet
usb_redir_header.length:  sizeof(usb_redir_control_packet_header) [+ length]

struct usb_redir_control_packet_header {
    uint8_t endpoint;
    uint8_t request;
    uint8_t requesttype;
    uint8_t status;
    uint16_t value;
    uint16_t index;
    uint16_t length;
}
```
附加数据包含要发送/接收的控制消息数据。  
这种类型的数据包可以由usb-guest发送到usb-host，以启动usb设备上的控制传输。 端点，请求，请求类型，值和索引对于USB控制消息具有其标准含义。 status字段仅用于usb-host的响应。  
length是usb-guest发送/期望读取的数据量（在`USB_DIR_IN`情况下）。 请注意，长度应仅在一个方向上添加到`usb_redir_header`.length（并且实际数据包长度应匹配）。  
当usb-device处理了控制消息时，usb-host将`usb_redir_control_packet`发送回usb-guest，所有字段都保持不变，除了状态字段和长度更新以匹配实际结果。

## `usb_redir_bulk_packet`

```c
usb_redir_header.type:    usb_redir_bulk_packet
usb_redir_header.length:  sizeof(usb_redir_bulk_packet_header) [+ length]

struct usb_redir_bulk_packet_header {
    uint8_t endpoint;
    uint8_t status;
    uint16_t length;
    uint32_t stream_id;
    uint16_t length_high; /* High 16 bits of the packet length */
}
```
附加数据包含要发送/接收的bulk msg数据。  
这种类型的数据包可以由usb-guest发送到usb-host，以便在usb设备上启动一个bulk传输。 endpoint和stream_id对usb bulk消息有其标准含义。 status字段仅用于usb-host的响应。 length是usb-guest发送/期望读取的数据量（取决于端点的方向）。  
`length_high`包含16位高位长度以允许大于65535字节的数据包，仅在双方都具有`usb_redir_cap_32bits_bulk_length`能力时发送/接收。  
当usb-device处理bulk消息时，usb-host将`usb_redir_bulk_packet`发送回usb-guest，更新状态字段和长度以匹配实际结果。  
请注意，就像`usb_redir_control_packet`一样，此数据包只在一个方向上有额外的数据，具体取决于端点的方向。  
注意请参阅`usb_redir_buffered_bulk_packet`以获取从bulk端点接收数据的替代方法。  

## `usb_redir_iso_packet`

```c
usb_redir_header.type:    usb_redir_iso_packet
usb_redir_header.length:  sizeof(usb_redir_iso_packet_header) + length

struct usb_redir_iso_packet_header {
    uint8_t endpoint;
    uint8_t status;
    uint16_t length;
}
```
附加数据包含要发送/接收的iso msg数据。  
一旦使用`usb_redir_start_iso_stream`启动iso流，就应该持续发送此类型的数据包（以端点间隔速度），它们发送的方向取决于端点方向。  
状态字段仅对从usb-host发送到usb-guest的数据包有意义（对于iso输入端点）。由于缓冲，不可能及时通知usb-guest关于iso输出包的传输错误。 usb-host自身将尝试清除任何错误条件。如果它失败了，它将向usb-guest发送`usb_redir_iso_stream_status`，表明iso流存在问题。  
由于usb-host会在iso输入端点上启动流后连续发送`usb_redir_iso_packet`，因此usb-host无法将`usb_redir_header`.id设置为相应接收的数据包的id。所以对于`usb_redir_iso_packet`来说，usb-host只是以id为0开始，并且每个数据包都会递增。请注意，当usb-host从stall状态中恢复时，id将从0重新开始！

## `usb_redir_interrupt_packet`

```c
usb_redir_header.type:    usb_redir_interrupt_packet
usb_redir_header.length:  sizeof(usb_redir_interrupt_packet_header) [+ length]

struct usb_redir_interrupt_packet_header {
    uint8_t endpoint;
    uint8_t status;
    uint16_t length;
}
```
附加数据包含要发送/接收的中断msg数据。  
中断端点的处理根据端点是输入还是输出端点而显著不同。  

### 输入端点

需要及时轮询输入中断端点，否则数据可能会丢失。 因此，对于输入中断端点，usb-host负责提交和重新提交urb，usb-guest可以使用`usb_redir_start_interrupt_receiving` / `usb_redir_stop_interrupt_receiving`数据包启动/停止接收中断数据包。 请注意，对于输入中断端点，`usb_redir_interrupt_packet-s`仅在一个方向上发送，从usb-host发送到usb-guest！  
由于`usb_redir_interrupt_packet`是在中断接收开始后由usb-host主动发送的，所以usb-host不能将`usb_redir_header`.id设置为相应接收数据包的id。 因此，对于`usb_redir_interrupt_packet`，usb-host只是以id为0开始，并将每个数据包递增。 请注意，当usb-host从停顿中恢复时，id将从0重新开始！

### 输出端点

对于中断输出端点，使用同样用于control和bulk transfers的普通异步机制：  
usb-guest将`usb_redir_interrupt_packet`发送到usb-host。 当usb-device处理了中断消息时，usb-host将`usb_redir_interrupt_packet`发送回usb-guest，更新状态字段和长度以匹配实际结果。 当从usb-guest发送到usb-host时，此数据包仅包含其他数据（要输出的数据）。  
请注意，由于与iso数据不同，通常中断数据没有流的概念，缓冲对输出中断数据包没有意义，而是尽快交付。  
尽管是尽快交付，但很可能无法满足应用于中断输出传输的时间限制。 其后果因设备而异。

## `usb_redir_buffered_bulk_packet`

```c
usb_redir_header.type:    usb_redir_bulk_packet
usb_redir_header.length:  sizeof(usb_redir_bulk_packet_header) + length
usb_redir_header.id:      starts at 0, incremented by 1 per send packet

struct usb_redir_buffered_bulk_packet_header {
    uint32_t stream_id;
    uint32_t length;
    uint8_t endpoint;
    uint8_t status;
}
```
附加数据包含收到的bulk msg数据。  
buffered bulk模式适用于bulk输入端点，其中数据具有流式特性（不是命令响应协议的一部分）。如果数据读取速度不够快，这些端点的输入缓冲区可能会溢出。因此，在buffered bulk模式下，usb-host负责提交和重新提交bulk传输。 usb-guest可以使用`usb_redir_start_bulk_receiving` / `usb_redir_stop_bulk_receiving`数据包开始/停止接收buffered bulk数据。  
请注意，`usb_redir_buffered_bulk_packet-s`仅从一个方向发送，从usb-host发送到usb-guest！  
由于`usb-redir_buffered_bulk_packet-s`在bulk 接收开始后由usb-host主动发送，因此usb-host无法将`usb_redir_header`.id设置为相应接收的数据包的id。因此，对于`usb_redir_buffered_bulk_packet-s`，usb-host只需以id为0开始，并将每个数据包递增。请注意，当usb-host从stall状态中恢复时，id将从0重新开始！  
应该使用buffered bulk模式的典型示例是 usb 端点到串行转换器中的bulk。  
注意buffered bulk模式只能在双方都具有`usb_redir_cap_bulk_receiving`功能时使用。
