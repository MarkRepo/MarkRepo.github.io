---
title: 虚拟桌面网页版实时音视频直播方案
description: 总结h5实时视频直播方案
categoriess: avcodec
tags:
  - h264
  - opus
  - codec
  - 直播
---

## MSE for h264 living

[完整demo](https://github.com/MarkRepo/wfs.js)

### 各种方案介绍

常见的直播方案有`RTMP RTSP HLS` 等等， 由于这些流都需要先传输到服务器，然后进行推流，延时比较大，RTMP可以优化到1s，hls延时最高，大概10s左右。
虚拟桌面要求延时能在100ms以内。经过google查找资料发现有以下几种方案可以实现：

1. 用 `websocket` 传输h264编码数据，在浏览器中使用broadway开源库进行解码，调用html5 canvas绘制图像。在github上有一个demo，经过测试，broadway解码效率不高。（测试环境 chrome book）
参考<https://github.com/131/h264-live-player>
2. 使用webRTC   进行点对点直播，找了一个demo，搭建了一个聊天室测试，延时效果大概在500ms左右，应该可以优化。webRTC的接口封装的很好，只有三个接口。
[demo](https://github.com/LingyuCoder/SkyRTC-demo) 参考<https://segmentfault.com/a/1190000000436544>
上面的demo有一个地方需要注意： 使用http服务无法获取到视频流，浏览器报错，提示需要https服务。改成https服务之后，测试成功。
这个方案可行，但是需要自己去改webRTC的源码，工作量比较大，所以没有采用。
3. 使用MSE（`Media Source Extension`, 具体参考W3C标准）扩展实现 `HTML5 video tag`的流式直播。（最终采用的方案）  
方案描述： 使用websocket 从服务端传输h264编码数据到浏览器， 在浏览器端使用JS 解析h264数据 ， 封装成`fMP4 fragment`, 喂给media source 中的sourceBuffer， 浏览器video tag自动获取sourceBuffer中的数据进行解码渲染。   
最后实现的demo体验效果良好，延时能达到100ms以内，使用笔记本软解、硬解， chrome book 软解表现都很完美，唯独chrome book 硬解会缓冲一帧数据，是一个瑕疵， 不过这个缺点可以在服务器端多发一帧数据解决。（见后文）

### 重要问题记录

下面主要记录预研过程中出现的重要问题和解决方案：  
解析h264数据，封装`fMP4 fragment`:  
这一步比较复杂，由于之前没有JS开发经验，没有选择自己写，在github找了一个开源实现。参考<https://github.com/ChihChengYang/wfs.js> 根据wfs.js搭建的直播方案，主要出现三个问题（只有第一个延时是wfs.js库的问题，其余是自己的问题）：

+ 第一个是延时问题，延时很大，在3~5s左右，原因有两个： 
	1. wfs.js库中做了缓存，收到一定的数据之后才执行fMP4 fragment的封装。
	2. chrome浏览器的解码器默认不是以直播流的模式解码视频帧，所以会在解码的时候缓存4帧数据。  
+ 解决方法：    
	1. 把wfs.js库中的缓存去掉，每来一帧数据都执行fMP4 fragment的封装
	2. 设置`mvhd.duration = 0`，如果有mehd的话，设置`mehd.fragmentDuration = 0`，这样chrome 会进入“low delay mode”， 不会缓存数据。
具体参考： <https://stackoverflow.com/questions/36364943/frame-by-frame-decode-using-media-source-extension>
　　　　　 <https://bugs.chromium.org/p/chromium/issues/detail?id=465324>
+ 第二个就是解码问题，解码花屏  
原因： 虚拟机spice服务端使用了websokify代理（python写的）。首先，这个代理服务器是流式的（出现数据帧被分割和合并的现象），浏览器端js没有进行数据帧边界的解析； 第二，代理缓冲区过小，导致数据帧被分割传输。  
解决方法：
	1. 修改websokify代理的接收缓冲区大小。
	2. 在wfs.js库中对收到的数据进行解析，一帧一帧的提交数据，封装fMP4 fragment。
+ 第三是屏幕倒转问题  
原因： spice服务端发过来的h264数据就是倒的，在终端平台，是由终端处理的。  
解决方法： 利用css的画面旋转功能，以x轴为旋转轴， 旋转180度。如：  

```
<style type="text/css" media="screen">
video.rotate180{
　　width:100%;
　　height:100%;
　　transform:rotateX(180deg);
　　-moz-transform:rotateX(180deg);
　　-webkit-transform:rotateX(180deg);
　　-o-transform:rotateX(180deg);
　　-ms-transform:rotateX(180deg);
　　}
</style>
```
+ 关于chrome book硬解码缓存一帧问题解决办法  
通过看chrome源码 decoder部分，发现decoder处理几个数据类型会直接flush缓冲区，所以可以在wfs.js每收到一帧数据，
就构造一帧这种类型的空数据，喂给video tag， 把缓冲的一帧flush出来，同时把播放时间缩短一半即可（否则会帧堵塞）。示例：

```js
var copy2 = new Uint8Array(4);
copy2[0] = 0, copy2[1] = 0, copy2[2] = 1, copy2[3] = 10; //类型10,11 都可以，但是10可以兼容软解　　　　　
this.wfs.trigger(Event.H264_DATA_PARSING, {data: copy2});
```

## web audio living

总结网页音频直播的方案和遇到的问题。

代码：（github，待整理）  
结果： 使用opus音频编码，web audio api 播放，可以达到100ms以内延时，高质量，低流量的音频直播。  
背景： VDI（虚拟桌面） h264网页版预研，继h264视频直播方案解决之后的又一个对延时有高要求的音频直播方案（交互性，音视频同步）。
前提：flexVDI开源项目对音频的支持只实现了对未编码压缩的PCM音频数据。并且效果不好，要么卡顿，要么延时，流量在2~3Mbps（根据缓冲的大小）。 
解决方案： 在spice server端对音频采用opus进行编码，flexVDI playback通道拿到opus packet数据后，调用opus js解码库解码成PCM数据，喂给audioContext进行播放。  
流程简介：flexVDI palyback通道接收opus音频数据，调用libopus.js解码得到PCM数据，保存到buffer。创建scriptProcessorNode， 在onaudioprocess函数中从buffer里面拿到PCM数据，按声道填充outputBuffer， 把scriptProcessorNode连接到audioContext.destination进行播放。具体代码见后文或者github。
opus编解码接口介绍：参考 <http://opus-codec.org/docs/opus_api-1.2/index.html>

### libopus 实例

下面是我用opus c库解码opus音频，再用ffplay播放PCM数据的一个demo，可以看看opus解码接口是怎么使用的：

```c
#include <stdio.h>                                         
#include <stdlib.h>
#include <string.h>
#include "opus.h"
/*
static void int_to_char(opus_uint32 i, unsigned char ch[4])
{
    ch[0] = i>>24;
    ch[1] = (i>>16)&0xFF;
    ch[2] = (i>>8)&0xFF;
    ch[3] = i&0xFF;
}*/
static opus_uint32 char_to_int(unsigned char ch[4]){
    return ((opus_uint32)ch[0]<<24) | ((opus_uint32)ch[1]<<16)
         | ((opus_uint32)ch[2]<< 8) |  (opus_uint32)ch[3];
}

int main(int argc, char** argv){
    opus_int32 sampleRate = 0;
    int channels = 0, err = 0, len = 0;
    int max_payload_bytes = 1500;
    int max_frame_size = 48000*2;
    OpusDecoder*  dec = NULL;
    sampleRate = (opus_int32)atol(argv[1]);
    channels = atoi(argv[2]);
    FILE*  fin = fopen(argv[3], "rb");
    FILE*  fout = fopen(argv[4], "wb+");
 
    short *out;
    unsigned char* fbytes, *data;
    //in = (short*)malloc(max_frame_size*channels*sizeof(short));
    out = (short*)malloc(max_frame_size*channels*sizeof(short));
    /* We need to allocate for 16-bit PCM data, but we store it as unsigned char. */
    fbytes = (unsigned char*)malloc(max_frame_size*channels*sizeof(short));
    data   = (unsigned char*)calloc(max_payload_bytes, sizeof(unsigned char));
    dec = opus_decoder_create(sampleRate, channels, &err);
    int nBytesRead = 0;
    opus_uint64 tot_out = 0;
    while(1){
 　　　　unsigned char ch[4] = {0};
        nBytesRead = fread(ch, 1, 4, fin);
        if(nBytesRead != 4)
            break;
        len = char_to_int(ch);
        nBytesRead = fread(data, 1, len, fin);
        if(nBytesRead != len)
            break;
        
        opus_int32 output_samples = max_frame_size;
        output_samples = opus_decode(dec, data, len, out, output_samples, 0);
        int i;
        for(i=0; i < output_samples*channels; i++)
        {
            short s;
            s=out[i];
            fbytes[2*i]=s&0xFF;
            fbytes[2*i+1]=(s>>8)&0xFF;
        }
        if (fwrite(fbytes, sizeof(short)*channels, output_samples, fout) != (unsigned)output_samples){
            fprintf(stderr, "Error writing.\n");
            return EXIT_FAILURE;
        }
        tot_out += output_samples;
    }
     
    printf("tot_out: %llu \n", tot_out);
     
    return 0;
}    
```
这个程序对opus packets组成的文件（简单的length+packet格式）解码后得到PCM数据，再用ffplay播放PCM数据，看能否正常播放：

```shell
ffplay -f f32le -ac 1 -ar 48000 input_audio     #播放float32型PCM数据
ffplay -f s16le -ac 1 -ar 48000 input_audio    #播放short16型PCM数据
```
ac表示声道数， ar表示采样率， input_audio是PCM音频文件。

### 浏览器保存二进制文件

要获取PCM数据文件，首先要得到opus packet二进制文件， 所以这里涉及到浏览器如何保存二进制文件到本地的问题，
参考代码：

```js
var saveFile = (function(){
        var a  = document.createElement("a");
        document.body.appendChild(a);
        a.style = "display:none";
        return function(data, name){
                var blob = new Blob([data]);
                var url = window.URL.createObjectURL(blob);
                a.href = url;
                a.download = name;
                a.click();
                window.URL.revokeObjectURL(url);
        };
}());
saveFile(data, 'test.pcm');
```
说明：首先把二进制数据写到typedArray中，然后用这个buffer构造Blob对象，生成URL， 再使用a标签把这个blob下载到本地。

### 播放PCM数据方案

利用audioContext播放PCM音频数据的两种方案：

flexVDI的实现 参考<https://github.com/flexVDI/spice-web-client>

```js
function play(buffer, dataTimestamp) {
    // Each data packet is 16 bits, the first being left channel data and the second being right channel data (LR-LR-LR-LR...)
    //var audio = new Int16Array(buffer);
    var audio = new Float32Array(buffer);
    // We split the audio buffer in two channels. Float32Array is the type required by Web Audio API
    var left = new Float32Array(audio.length / 2);
    var right = new Float32Array(audio.length / 2);
    var channelCounter = 0;
    var audioContext = this.audioContext;
    var len = audio.length;
    for (var i = 0; i < len; ) {
      //because the audio data spice gives us is 16 bits signed int (32768) and we wont to get a float out of it (between -1.0 and 1.0)
      left[channelCounter] = audio[i++] / 32768;
      right[channelCounter] = audio[i++] / 32768;
      channelCounter++;
    }
    var source = audioContext['createBufferSource'](); // creates a sound source
    var audioBuffer = audioContext['createBuffer'](2, channelCounter, this.frequency);
    audioBuffer['getChannelData'](0)['set'](left);
    audioBuffer['getChannelData'](1)['set'](right);
    source['buffer'] = audioBuffer;
    source['connect'](this.audioContext['destination']);
    source['start'](0);
}
```
>注： buffer中保存的是short 型PCM数据，这里为了简单，去掉了对时间戳的处理，因为source.start(0)表示立即播放。如果是float型数据，不需要除以32768.

ws-audio-api的实现,参考<https://github.com/Ivan-Feofanov/ws-audio-api>

```js
var bufL = new Float32Array(this.config.codec.bufferSize);
var bufR = new Float32Array(this.config.codec.bufferSize);
this.scriptNode = audioContext.createScriptProcessor(this.config.codec.bufferSize, 0, 2);
if (typeof AudioBuffer.prototype.copyToChannel === "function") {
    this.scriptNode.onaudioprocess = function(e) {
        var buf = e.outputBuffer;
        _this.process(bufL, bufR);　　//获取PCM数据到bufL， bufR
        buf.copyToChannel(bufL, 0);
        buf.copyToChannel(bufR, 1);
    };
} else {
    this.scriptNode.onaudioprocess = function(e) {
		var buf = e.outputBuffer;
		_this.process(bufL, bufR);
		buf.getChannelData(0).set(bufL);
		buf.getChannelData(1).set(bufR);
    };
}
this.scriptNode.connect(audioContext.destination);
```
>延时卡顿的问题：audioContext有的浏览器默认是48000采样率，有的浏览器默认是44100的采样率，如果喂给audioContext的PCM数据的采样率不匹配，就会产生延时和卡顿的问题。