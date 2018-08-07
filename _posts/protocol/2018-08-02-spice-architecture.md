---
title: Spice 架构与特性
description: 介绍Spice的基本架构与数据流
categories: protocol
tags:
  - Spice
---

## 介绍

Spice是一种开放的远程计算解决方案，为远程机器的显示器和设备（例如键盘，鼠标，音频）提供客户端访问。
Spice实现了与本地计算机交互类似的用户体验，同时尝试将大部分密集型CPU和GPU任务从客户端卸载。
Spice适用于LAN和WAN使用，不会影响用户体验。

## 基础架构

Spice基本构建块是Spice protocol，Spice server和Spice client。 与Spice相关的组件包括QXL device和 guest QXL driver。

### 图形命令流

![graphic-commands-flow](/assets/images/spice/graphic-command-flow.png)

上图显示了使用 libspice 和 QEMU 时图形命令的基本Spice架构和 guest 到 client 的数据流。 libspice 也可以被任何其他 VDI 
兼容的主机应用程序使用。图形命令数据流由用户程序请求OS图形引擎（X或GDI）执行渲染操作开始。图形引擎将命令传递给QXL driver，QXL 
driver将OS命令转换为QXL命令并将它们推入命令环。命令环驻留在设备内存中。 libspice 
从环中提取命令并将它们添加到图形命令树中。图形命令树包含一组命令，其执行将重新生成显示内容。 
libspice使用树来优化到客户端的命令传输，方法是删除被其他命令隐藏的命令。命令树还用于视频流检测。 libspice还维护一个要发送到客户端的
命令队列，以更新其显示。当从队列中拉出命令以传输到客户端时，它被转换为Spice协议消息。从树中删除的命令也会从发送队列中删除。当libspic
e不再需要一个命令时，它会被推入设备释放环。驱动程序使用此环来释放命令资源。当客户端收到图形命令时，它使用该命令更新显示。

### Agent 命令流

![agent-command-flow](/assets/images/spice/agent-command-flow.png)

Spice agent是在guest中执行的软件模块。 Spice server和client将agent用于需要在guest上下文中执行的tasks，
例如配置guest显示设置。 上图显示了使用VDI Port device和VDI Port driver的Spice client和server与agent的通信。 消息可以由client（例如，guest显示设置的配置）、server（例如，鼠标移动）和agent（例如，配置确认）生成。 driver使用device的输入和输出环与device通信。 client和server生成的消息将写入server中的同一写入队列，然后写入device输出环。 server从device输入环读取消息到读缓冲区。 由message port确定消息是由server处理还是转发到client。

### Spice Client

Spice跨平台（Linux和Windows）客户端是最终用户的界面。

#### Client 基础架构

![client-basic-architecture](/assets/images/spice/client-basic-architecture.png)

#### Client类

以下是Spice Client关键类的介绍。 为了拥有一个干净的跨平台结构，Spice定义了通用接口，将其特定于平台的实现保留在并行目录中。 一个这样的通用接口是Platform类，它定义了许多低级服务，例如定时器和游标操作。

Application是主类，它包含和控制client，monitors和screens。 它处理一般应用程序功能，例如解析命令行参数，运行主消息循环，处理事件（连接，断开连接，错误等），将鼠标事件重定向到输入处理程序，切换全屏模式等。

##### Channels

client和server通过channels进行通信。 每种channel类型专用于特定类型的数据。 每个channel都使用专用的TCP套接字，可以是安全（使用SSL）或不安全。 在client端，每个通道都有一个专用线程，因此可以通过区分其线程优先级为每个通道提供不同的QoS。

RedClient是main channel。它拥有所有其他实例化channel并控制它们（使用factory创建channel，连接，断开连接等），并处理控制、配置和迁移（使用Migrate类）。

所有channel的祖先是：

+ RedPeer - 用于安全和不安全通信的socket封装类，提供基础功能，例如connect，disconnect，close，send，receive和用于迁移的套接字交换。 它定义了通用消息类：InMessages，CompoundInMessage和OutMessage。 所有消息都包括type，size和data。
+ RedChannelBase - 继承RedPeer，提供与server建立通道连接的基本功能，并支持与服务器的通道功能交换。
+ RedChannel - 继承RedChannelBase。 此类是所有实例化channel的父级。 处理发送传出消息和分派传入消息。 RedChannel线程运行具有各种事件源的事件循环（例如，发送和中止触发器）。 通道套接字被添加为事件源，用于触发Spice消息的发送和接收。

可用的channel有：

+ Main - 由RedClient实现（见上文）。
+ DisplayChannel - 处理图形命令，图像和视频流。
+ InputsChannel  - 键盘和鼠标输入。
+ CursorChannel  - 指针设备位置，可见性和光标形状。
+ PlaybackChannel - 从server接收的音频，由client播放。
+ RecordChannel - 在client捕获的音频。

ChannelFactory是所有channel factories的基类。 每个channel都注册其特定factory，以使RedClient能够按channel类型创建channel。

##### Screens and Windows

+ ScreenLayer - screen layer被附加到特定screen，提供矩形区域的操作（set，clear，update，invalidate等）。 layers是z-ordered的（例如，光标在显示之上）。
+ RedScreen - 使用screen layers（例如，显示，光标）实现screen逻辑并控制窗口以显示其内容。
+ RedDrawable - basic pixmap的特定于平台的实现。 它支持基本渲染操作（例如，复制(copy)，混合(blend)，组合(combine）。
+ RedWindow_p - 特定于平台的窗口数据和方法。
+ RedWindow - 继承RedDrawable和RedWindow_p。基本窗口状态和功能的跨平台实现（例如，显示，隐藏，移动，最小化，设置标题，设置光标等）。

### Spice Server

Spice服务器在libspice中实现，libspice是一个虚拟设备接口（VDI）可插拔库。 VDI提供了一种通过软件组件发布虚拟设备接口的标准方法。这使其他软件组件能够与这些设备进行交互。有关更多信息，请参阅[2]。一方面，server使用Spice协议与远程client通信。另一方面，它与VDI host 应用程序（例如，QEMU）交互。为了远程显示的目的，server维护一个命令队列和一颗树，用于管理当前对象的依赖关系(dependencies)和覆盖关系(hidings)。处理QXL命令并将其转换为发送到client的Spice协议命令。 Spice总是试图将渲染任务传递给client，从而利用其硬件加速能力。通过软件或GPU在host端进行渲染是最后的结果。 Spice server保留组成当前图像的guest图形命令。它仅在完全被其他命令覆盖并且没有依赖性时，或者当我们需要将图形命令渲染到帧缓冲区时才释放命令。触发绘制到帧缓冲区的两个主要原因是（1）资源耗尽; （2）guest需要从帧缓冲区中读取。

#### Server Structure

![server-structure](/assets/images/spice/server-structure.png)

服务器通过通道与client通信。 每种通道类型专用于特定类型的数据。 每个通道都使用专用的TCP套接字，可以是安全（使用SSL）或不安全的。 服务器通道类似于client通道：Main，Inputs，Display，Cursor，Playback和Record（更多信息请参见通道2.3.2.1）。主通道和输入通道由处理函数（handler functions）控制（在reds.c中实现）。 显示和光标通道由每一个显示的red work thread处理。 音频播放和录制通道有自己的处理器（handler）（snd_worker.c）。 Libspice和VDI host应用程序（例如QEMU）通过为每个功能（例如，QXL，代理，键盘，鼠标，平板电脑，回放，记录）定义的接口进行通信，如[2]中所详述。 如上图所示，spice服务器包含以下主要组件：

##### Red Server（reds.c）

服务器监听并接受客户端连接，与它们通信。
Reds 负责：

+ Channels
	+ 拥有并且管理channels（注册，注销，关闭）
	+ 通知client有关活动通道的信息，以便client可以创建它们
	+ 主通道和输入通道的处理
	+ 建立连接（主通道和其他通道）
	+ 套接字操作和连接管理
	+ 处理SSL和ticket
+ 添加和删除VDI接口（例如，core，migration，键盘，鼠标，平板电脑，agent）
+ 迁移过程协调
+ 处理用户命令（例如，来自Qemu监视器）
+ 与guset agent沟通
+ 统计

##### Graphics subsystem

![graphics-subsystem](/assets/images/spice/graphics-subsystem.png)

与Spice服务器中的其他子系统不同，图形子系统在专用线程（即red worker）上与server并行运行。
这种结构实现了QEMU流的处理和渲染传入的图形命令之间的独立性，这可能消耗大量CPU资源。 上图显示了Spice服务器图形子系统结构。 
Red server在新的QXL接口（即VDI）上初始化一个调度程序。 调度程序为该接口创建red worker。 worker处理的命令可以来自三个来源：
（1）同步QXL设备命令，（2）Red server命令，调度程序使用套接字（即套接字对）传送（1和2），（3） 异步QXL设备命令，worker 使用接口从QXL设备环拉取。

**Red Worker(red_worker.c)**

Spice服务器为每个QXL设备实例保存一个red worker thread的不同实例。 red_worker的职责是：

+ 处理QXL设备命令（例如绘制，更新，光标）
+ 处理从调度程序收到的消息
+ channel pipes（管道）和 pipe（输送）items
+ 显示和光标通道
+ 图像压缩（使用quic，lz和glzencoding）
+ 视频流 - 识别，编码和流创建
+ 缓存 - 客户端共享像素图（pixmap）缓存，光标缓存，调色板缓存
+ Graphic items删除优化 - 使用item树，容器，阴影，排除区域，不透明items
+ Cairo和OpenGL（pbuf和pixmap）渲染器 - canvas，surfaces等。
+ 环操作

**Red Dispatcher (red_dispatcher.c)**

+ Dispatcher，每个QXL设备实例一个
+ 封装QXL设备和reds中worker的内部
+ 为QXL设备实例初始化一个worker并创建一个worker thread
+ 使用socketpair channel调度worker
+ QXL设备使用QXL Worker接口，由red dispatcher实现并附加，该接口将设备调用转换为通过red worker pipe传输的消息。 这种方式使两者分开并在逻辑上独立。
+ Reds使用`red_dispatcher.h`中定义的接口来执行调度程序功能，例如调度程序初始化，图像压缩更改，视频流状态更改，鼠标模式设置和渲染器添加。

### Spice Protocol

Spice协议用于客户端 - 服务器通信，即用于传输图形对象，键盘和鼠标事件，光标信息，音频播放和record chunks以及控制命令。 有关Spice协议的详细文档可以在[1]中找到。

### QXL Device

Spice服务器支持QXL VDI接口。 当libspice与QEMU一起使用时，可以使用特定的QEMU QXL PCI设备来改善远程显示性能并增强guest图形系统的图形功能。 QXL device需要guest QXL driver才能实现全部功能。 但是，如果不存在driver，则支持标准VGA。 此模式还允许从虚拟机（VM）引导阶段进行活动显示。 device使用（图像）命令和光标环，显示和光标事件中断以及I/O ports与driver交互。 该设备的其他职责包括：

+ 初始化设备ROM，RAM和VRAM并将其映射到物理内存
+ 映射I/O ports并处理读写操作用于管理：区域更新，命令和光标通知，IRQ更新，模式设置，设备重置，日志记录等。
+ Rings - 初始化和维护（图形）命令和光标环，从环中获取命令和光标命令并等待通知。维护资源环。
+ 使用QXL Worker接口与相应的red worker进行通信，由red dispatcher实现并附加。red dispatcher将设备调用转换为从red worker pipe读取或写入的消息。
+ 注册QXLInterface以使worker能够与device通信。该接口包括PCI信息和附加worker的函数，从环中获取display和光标命令，display和光标通知，模式更改通知等。
+ 定义支持的qxl模式并允许修改当前模式，包括vga模式，其中所有monitor 镜像(mirror)到单个设备（vga clients）
+ 在VGA模式下处理display初始化，更新，调整大小和刷新

### QXL Guest Drivers

特定于平台的guest driver用于启用QXL设备并与之通信。 Windows驱动程序包括一个与图形设备接口（GDI）调用和结构一起使用的显示驱动程序，以及一个处理内存映射，端口和中断的微型端口驱动程序。

### Spice Agent

Spice agent是一个可选组件，用于增强用户体验和执行面向guest的任务。 例如，agent在使用**客户端鼠标模式**时将鼠标位置和状态注入到guest虚拟机。 此外，它还用于配置guest显示设置。 未来的功能包括从/向客户复制和粘贴对象。 Windows agent由系统服务和用户进程组成

### VDIPort Device and Driver

Spice协议支持客户端与服务器端agent之间的通信通道。 使用QEMU时，Spice agent驻留在guest虚拟机上。 VDI port是用于与agent进行通信的QEMU PCI device。 采用特定的代理协议进行通信。 Windows guest driver已经实现.

## 特性

### Graphic Commands

Spice支持2D图形命令的传输和处理（3D支持即将推出），而不是帧缓冲区更新，帧缓冲区更新在许多其他远程桌面解决方案中使用。 QXL设备命令是通用的，与平台无关，因此Windows和X驱动程序本机都使用它们。

### Hardware Acceleration

基本的Spice client渲染是使用Cairo执行的，Cairo是一个跨平台的，设备独立的库。 Cairo为二维绘图提供矢量图形基元。 硬件加速是一种额外的渲染模式，其中渲染由client GPU在硬件上执行，而不是由软件使用CPU执行。 硬件加速是使用Linux中的OpenGL（实验性）和Windows中的GDI实现的。硬件加速优势是：

+ 高性能渲染 - 使用OpenGL，Spice client能够比以前更快地渲染。 拉伸（由视频流使用）等繁重的软件操作在由硬件执行时比通过软件执行要快得多。 因此，Spice实现了更加流畅的用户体验。
+ 减少客户端CPU使用率 - 客户端享有更多CPU时间，可用于其他任务，如音频

与Cairo是一个独立的软件库不同，OpenGL是一个依赖于驱动程序和硬件实现的硬件库。 因此，Spice可能会遭受渲染错误，大量CPU占用，甚至是client或host崩溃。 此外，虽然OpenGL是一个全球标准，但硬件和驱动程序的实现在供应商之间发生了巨大变化。 因此，在不同的GPU上，Spice可能会显示不同的渲染输出，并且可能会检测到不同的性能。 此外，有些设备根本不支持OpenGL。  
服务器还使用OpenGL进行硬件加速，与Linux客户端使用相同的代码。

### Image Compression

Spice提供了多种图像压缩算法，可以在服务器启动时选择，也可以在运行时动态选择。 Quic是Spice专有的图像压缩实用程序，它基于SFALIC算法[3]。 调整到图像的LZ（LZSS）[4]算法是另一种选择。 Quic和LZ都是局部算法，即它们独立地编码每个图像。 Global LZ（GLZ）是另一种Spice专有的，它使用LZ和基于历史的全局字典。 GLZ利用图像之间的重复模式来缩小流量并节省带宽，这在WAN环境中至关重要。 Spice还提供了一种自动模式，用于每个图像的压缩选择，其中LZ/GLZ和Quic之间的选择是启发式的，基于图像属性。 从概念上讲，LZ/GLZ可以更好地压缩人工图像，Quic可以更好地压缩真实图像。

### Video Compression

Spice对发送到client的图像使用无损压缩，而不是有损压缩，以避免重要显示对象的中断。 然而，由于（1）视频流可能是带宽的主要消费者，因为每个视频帧是独立的图像，并且（2）它们的内容大多是不加批判的，Spice对这样的流使用有损视频压缩：Spice服务器通过识别以高速率更新的区域来启发式地识别视频区域。 使用易丢失的Motion JPEG算法（M-JPEG）编码的视频流作为这些区域的更新被发送到客户端。 这种机制可以节省大量流量，从而提高Spice性能，尤其是在WAN中。然而，在某些情况下，启发式行为可能导致低质量图像（例如，当将更新的文本区域识别为视频流时）。 可以在服务器启动时选择视频流，并可以在运行时动态更改。

### Caching

Spice实现client端图像缓存，以避免向client的冗余传输。缓存适用于发送到客户端的任何类型的图像数据，包括像素图，调色板和光标。每个图像都来自驱动程序，具有唯一ID和缓存提示。不相同的图像具有不同的ID，而相同的图像具有相同的id。缓存提示建议服务器缓存映像。 Pixmap缓存在所有displays之间共享。缓存在每个连接中定义并在服务器和客户端之间同步，即，在每个时刻服务器确切地知道client缓存了哪些图像。此外，由服务器决定是否应该从缓存添加或删除项目。客户端缓存大小由客户端设置，并通过display通道初始化消息传送到服务器。服务器监视当前缓存容量，当它缺少空间时，它会删除最近最少使用的缓存项，直到有足够的可用缓存空间。服务器发送带有这些项的invalidate命令，客户端将其删除。

### Mouse Modes

Spice支持server和client两种鼠标模式。 模式可以动态更改，并在客户端和服务器之间进行协商。

+ Server mouse - 使用QEMU ps/2鼠标仿真以使鼠标在guest中可用。用户在Spice client窗口内单击后，将捕获client鼠标并将其设置为不可见。client将鼠标移动作为delta坐标发送到服务器。因此，每次移动后，client鼠标都会返回到窗口中心。在此模式下，服务器控制显示器上的鼠标位置，因此它始终在client和guest之间同步。但是，它可能在WAN或加载的服务器上存在问题，其中鼠标光标可能具有一些延迟或无响应。
+ Client mouse - client mouse用作实际的指针设备，它不会被捕获，并且guest光标设置为不可见。
client将鼠标移动作为绝对坐标发送到服务器。Guest agent为guest虚拟桌面缩放坐标并注入适当的光标位置。对于单个监视器，如果VDI host应用程序注册绝对定位设备（例如，QEMU中的USB平板电脑），则即使没有agent也可以使用客户端鼠标。在这种情况下。 Spice服务器缩放坐标。客户端模式适用于WAN或高负载的服务器，因为光标具有平滑的运动和响应性。但是，光标可能会丢失同步（位置和形状）一段时间。client mouse光标根据guest mouse光标进行更新。

### Multiple Monitors

Spice支持任意数量的监视器，仅受guest，client和server限制的约束。 启动VM时会设置监视器数量及其RAM大小。 Spice支持根据client 终端设置自动配置guest 监视器分辨率和显示设置。 这是通过client发送命令到guest agent实现的

### 2-way Audio and Lip-sync

Spice支持音频播放和录制。 使用CELT [5]算法压缩音频数据。 视频和音频之间的Lip-sync是通过在QXL设备中对视频帧加时间戳并在客户端注入时间戳来实现与独立的音频同步。

### Hardware Cursor

QXL设备支持光标硬件加速。 将cursor与display分开可以确定cursor的优先级，以便更好地响应。 此外，它还可以减少网络流量。

### Live Migration

服务器之间的VM迁移对连接的客户端是无缝的。 完整的连接状态（包括打开的通道和光标）将保存在源服务器上并在目标服务器上恢复。
