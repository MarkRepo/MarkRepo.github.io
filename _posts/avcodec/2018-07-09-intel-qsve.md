---
title: Intel Quick Sync Video Encoder
description: intel集成显卡硬编码h264的总结
categories: avcodec
tags:
  - codec
  - video
  - h264
  - intel
  - qsve
--- 

[github项目地址](github: https://github.com/MarkRepo/qsve)

## Intel Quick Sync Video Encoder

记录Intel E3 1275处理器集成显卡的硬编码预研过程。  
步骤如下：

1. 环境搭建
2. demo编译，测试
3. 研究demo源码，Media SDK API使用
4. 编写so动态库封装RGB，YUV的编码接口

下面记录每个过程的主要事项以及遇到的一些重要问题。

### 环境搭建

1. 首先在intel官网下载 MediaServerStudioEssentials2017.tar.gz, 主要考虑这个版本的SDK最适合 CentOS7.2.1511， 也可以下载R2版本。
2. 安装CentOS7.2.1511。
3. 安装Media SDK，具体步骤参考`media_server_studio_getting_started_guide.pdf`文档。这里要说明的是media sdk安装过程中会依赖很多基础库，比如libdrm， libva，X11， libGL等等，但是在`install_sdk_script.sh`脚本中安装的rpm包依赖的这些基础库的版本可能比CentOS7.2.1511中的低，这时候会安装失败，提示依赖错误。解决办法是在`install_sdk_script.sh`脚本中的yum命令参数加上`--oldpackage`参数，强制安装旧版本，使依赖关系通过。  
第二个问题是依赖冲突，出现过一个问题是kernel-tools安装了两个不同版本rmp包，这时候会依赖冲突，导致安装失败，解决方法是删掉一个rpm包。对于yum如何处理两个不同版本依赖库的问题后面再考虑。（这里我是对比一个安装正确的环境，删掉了一个高版本的库）

### demo编译，测试

环境搭建好之后，可以根据文档中的说明验证`media sdk`安装的正确性。然后根据`Sample_guide.pdf`文档，使用cmake程序编译，再根据每一个demo目录下的文档中参数的说明测试demo。这一步没什么好说的，如果环境安装正确的话，一切顺利。需要提一下的是，用`sample_encode` 把yuv数据编码成h264后不能用VLC直接播放，需要在命令行加`--demux h264`参数启动vlc才能播放。

### 源码分析， SDK API使用

对照着demo和media_sdk.pdf文档看，基本上没啥问题。需要关注的几个点是：

1. VPP，Encoder参数初始化
2. 输入输出缓冲区是如何分配与管理的（有一个”异步深度”的概念，主要关系到分配的输入缓冲区个数）
3. VPP， Encoder的异步处理流程（编码RGB数据源需要用到VPP模块）
4. 编码结果数据的获取。
5. RGBA的编码处理： 把VPP的输入格式改成RGB4， 输出改成NV12，把rgb视频数据拷贝`pData.R`， 然后赋值`pData.G = pData.R+1`, `pData.B=pData.R+2`, `pData.A = pData.R+3`，利用VPP完成RGB到yuv颜色空间的转换。

### so动态库

这里把源码直接拿过来写makefile编译的，需要注意的几个问题：

1. makefile里面需要定义几个与libVA有关的宏，使硬编码生效。（ `-DLIBVA_SUPPORT  -DLIBVA_DRM_SUPPORT  -DLIBVA_X11_SUPPORT`）
2. so链接的静态库不是`libmfx.a`， 而是`libdispatch_shared.a`, 这个静态库需要自己在`/opt/intel/mediasdk/opensource/mfx_dispatch`目录下用cmake编译出来，具体过程看dispatch的相关文档。如果使用libmfx.a链接，so会产生符号冲突，导致崩溃。

### 性能质量相关的关键参数设置

VPP处理过程伪代码：

```c
MFXVideoVPP_QueryIOSurf(session, &init_param, response); 
allocate_pool_of_surfaces(in_pool, response[0].NumFrameSuggested); 
allocate_pool_of_surfaces(out_pool, response[1].NumFrameSuggested); 
MFXVideoVPP_Init(session, &init_param); 
in=find_unlocked_surface_and_fill_content(in_pool); 
out=find_unlocked_surface_from_the_pool(out_pool); 
for (;;) { 
　　sts=MFXVideoVPP_RunFrameVPPAsync(session,in,out,aux,&syncp); 
　　if (sts==MFX_ERR_MORE_SURFACE || sts==MFX_ERR_NONE) {
　　　　MFXVideoCore_SyncOperation(session,syncp,INFINITE); 
　　　　process_output_frame(out); 
　　　　out=find_unlocked_surface_from_the_pool(out_pool); 
　　} 
　　if (sts==MFX_ERR_MORE_DATA && in==NULL) break; 
　　if (sts==MFX_ERR_NONE || sts==MFX_ERR_MORE_DATA) { 
　　　　in=find_unlocked_surface(in_pool); 
　　　　fill_content_for_video_processing(in); 
　　　　if (end_of_input_sequence()) in=NULL; 
　　} 
} 
MFXVideoVPP_Close(session); 
free_pool_of_surfaces(in_pool); 
free_pool_of_surfaces(out_pool);
```

Encoder处理过程伪代码：

```c
MFXVideoENCODE_QueryIOSurf(session, &init_param, &request);
allocate_pool_of_frame_surfaces(request.NumFrameSuggested);
MFXVideoENCODE_Init(session, &init_param);
sts=MFX_ERR_MORE_DATA;
for (;;) {
　　if (sts==MFX_ERR_MORE_DATA && !end_of_stream()) {
　　　　find_unlocked_surface_from_the_pool(&surface);
　　　　fill_content_for_encoding(surface);
　　}
　　surface2=end_of_stream()?NULL:surface;
　　sts=MFXVideoENCODE_EncodeFrameAsync(session,NULL,surface2,bits,&syncp);
　　if (end_of_stream() && sts==MFX_ERR_MORE_DATA) break;
　　… // other error handling
　　if (sts==MFX_ERR_NONE) {
　　　　MFXVideoCORE_SyncOperation(session, syncp, INFINITE);
　　　　do_something_with_encoded_bits(bits);
　　}
}
MFXVideoENCODE_Close();
free_pool_of_frame_surfaces();
```
Lowlatency 低延时参数设置:

```c
//Encoder参数设置：
m_mfxEncParams.mfx.GopRefDist = 1;　　
m_mfxEncParams.AsyncDepth = 1;
m_mfxEncParams.mfx.NumRefFrame = 1;
//Vpp参数设置：
m_mfxVppParams.AsyncDepth = 1;
```

Quality 编码质量相关参数:

```c
m_mfxEncParams.mfx.TargetKbps    //  码率越高，质量越好， 流量越大
m_mfxEncParams.mfx.TargetUsage   //  1~7 质量从高到低， 流量几乎不变，质量变化不明显
```

SPS PPS信息（开始一个新的编码序列）

```c
//获取当前参数设置　　
mfxVideoParam par;
memset(&par, 0, sizeof(p ar));
sts = m_pMfxEnc->GetVideoParam(&par);
MSDK_CHECK_RESULT(sts, MFX_ERR_NONE, sts);

//设置编码器扩展选项，开始一个新序列
mfxExtEncoderResetOption resetOption;
memset(&resetOption, 0, sizeof(resetOption));
resetOption.Header.BufferId = MFX_EXTBUFF_ENCODER_RESET_OPTION;
resetOption.Header.BufferSz = sizeof(resetOption);
resetOption.StartNewSequence = MFX_CODINGOPTION_ON;
mfxExtBuffer* extendedBuffers[1];
extendedBuffers[0] = (mfxExtBuffer*) & resetOption;
par.NumExtParam = 1;
par.ExtParam = extendedBuffers;
sts = m_pMfxEnc->Reset(&par);
MSDK_CHECK_RESULT(sts,MFX_ERR_NONE,sts);
//手动设置编码参数
mfxEncodeCtrl curEncCtrl;
memset(&curEncCtrl, 0, sizeof(curEncCtrl));
curEncCtrl.FrameType = MFX_FRAMETYPE_I | MFX_FRAMETYPE_REF | MFX_FRAMETYPE_IDR;
sts = m_pMfxEnc->EncodeFrameAsync(&curEncCtrl, &m_pVPPSurfacesVPPOutEnc[nEncSurfIdx], &m_mfxBS, &syncpEnc);
```
运行环境依赖的rpm:  
libdrm, libdrm-devel, libva, intel-linux-media, kmod-ukmd(内核模块), libippcc.so, libippcore.so,(libippcc.so 会根据cpu型号依赖不同的动态库，如E3 1275 依赖libippccl9.so, i5 6400 依赖libippccy8.so)
剩下的细节参考github上的源代码。

## Intel IPP 图像空间转换

### 背景

用QuickSync VPP模块做RGBA到NV12的颜色空间转换导致文字显示蒙上一层颜色的问题， 暂时怀疑是VPP自身的问题，因为参数设置都是按官方demo设置的。所以尝试使用IPP来做RGBA到NV12的转化。

### IPP 探索历程

1. 下载IPP安装包， google “IPP”，即可找到下载链接。
2. 安装， 执行 `install.sh`。 如果有问题，看看文档，安装环境是否满足安装需求。
3. 编写测试程序（可在官网找到），需要关注头文件和库的目录，在安装目录下都可以找到。

### RGBA到NV12的转换

1. 头文件ippcc.h, 库文件 libippcc.so
2. 实例：

```c
void Rgb2NV12(const unsigned char I[], const int image_width, 
              const int image_height,unsigned char J[]){
    //memcpy(J, I, image_width*image_height*3);
    IppStatus ipp_status;
    int srcStep = image_width*3;
    int dstYStep = image_width;
    int dstCbCrStep = image_width;
    IppiSize roiSize = {image_width, image_height};

    const Ipp8u* pSrc = (Ipp8u*)I;
    Ipp8u *pDstY    = (Ipp8u*)J;   //Y color plane is the first image_width*image_height pixels of J.
    Ipp8u *pDstCbCr    = (Ipp8u*)&J[image_width*image_height];  //In NV12 format, UV plane starts below Y.
    ipp_status = ippiRGBToYCbCr420_8u_C3P2R(pSrc, srcStep, pDstY, dstYStep, pDstCbCr, dstCbCrStep, roiSize);
    if (ipp_status != ippStsNoErr){
        memset(J, 128, image_width*image_height*3/2);
    }
}
```
上面实例是转RGB24到NV12的接口

```c
IPPAPI(IppStatus, ippiRGBToYCbCr420_8u_C3P2R,( const Ipp8u* pRGB, int rgbStep,  Ipp8u* pY,
		int YStep,Ipp8u* pCbCr, int CbCrStep, IppiSize roiSize ))//  RGB24-->NV12
IPPAPI(IppStatus, ippiRGBToYCbCr420_8u_C4P2R,( const Ipp8u* pRGB, int rgbStep,  Ipp8u* pY, 
	    int YStep,Ipp8u* pCbCr, int CbCrStep, IppiSize roiSize ))//  ARGB-->NV12
IPPAPI(IppStatus, ippiBGRToYCbCr420_8u_C3P2R,( const Ipp8u* pRGB, int rgbStep,  Ipp8u* pY, 
		int YStep,Ipp8u* pCbCr, int CbCrStep, IppiSize roiSize ))//  BGR24-->NV12
IPPAPI(IppStatus, ippiBGRToYCbCr420_8u_AC4P2R,( const Ipp8u* pRGB, int rgbStep,  Ipp8u* pY, 
		int YStep,Ipp8u* pCbCr, int CbCrStep, IppiSize roiSize ))// ABGR-->NV12
```
实际结果以测试结果为准。
有一些分辨率在把NV12编码成h264时需要对齐行像素，此时可以更改YStep，CbCrStep进行对齐。例如：

```c
IppiSize roiSize;
roiSize.width  = m_mfxEncParams.mfx.FrameInfo.CropW;
roiSize.height = m_mfxEncParams.mfx.FrameInfo.CropH;
mfxU16 pitch   = m_pVPPSurfacesVPPOutEnc[nEncSurfIdx].Data.Pitch;
ippiBGRToYCbCr420_8u_AC4P2R( (Ipp8u*)pInputBuffer, roiSize.width*4,  (Ipp8u*)m_pVPPSurfacesVPPOutEnc[nEncSurfIdx].Data.Y, 
	pitch, (Ipp8u*)m_pVPPSurfacesVPPOutEnc[nEncSurfIdx].Data.UV,pitch, roiSize);
```
上面的`Data.Pitch`就是编码器对特定分辨率下的行对齐的字节宽度。
