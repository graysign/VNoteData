#### 1 MTH323Client支持的协议族  
声明支持H323协议族的多媒体终端，需要支持基本的H225协议，H245协议和音频编解码器单元。视频编解码器单元和用户数据应用是可选的。基本的H323终端包含：用户设备接口，视频编解码器，音频编解码器，远程信息处理设备，H.225.0层，系统控制功能和基于分组的网络的接口。  

image-1 一个典型的H323终端构成  


#### 2 MTH323Client的开源库选型  
##### 2.1 H323协议族开源库的选型  

H323Plus项目：该项目实现了H323协议族，为基于IP网络的多媒体通信奠定了坚实基础，还扩展了323功能，增加了NAT / FW穿越解决方案。H323Plus(V1.26.8)项目依赖PTLib(V2.12.8)项目。  
项目地址：https://github.com/willamowius/h323plus  
API文档：https://www.h323plus.org/api/v1_26_0/index.html  


##### 2.2 H323的MCU服务的选型  

OpenMCU-ru项目:OpenMCU-ru是全功能的H.323，SIP和RTSP的开源多点控制单元，具有以下特性：支持通过端口1420(http://localhost:1420)上的Web界面进行配置和控制;支持多国语言;支持的协议:H.323，SIP;支持的视频编解码器：H.261，H.263，H.263 +，H.264，VP8;支持的音频编解码器:G.711，G.722，G723.1，G.726，G.729，iLBC，Speex，SILK，OPUS;支持多个不同的会议可以同时进行，使用不同的“房间”;支持实时显示呼叫统计;支持从MCU发起呼叫到远程终端;支持会议翻译为网络流。  
项目地址：https://github.com/muggot/openmcu  
项目文档：https://wiki.videoswitch.ru/en/start  

#### 3 MTH323Client的设计  
Coming soon…  



#### 4 MTH323Client的编译  
Coming soon…  



#### 参考文献  

* [The Standard in Open Source H.323](https://www.h323plus.org/)
* [Web Real-Time Communication(WebRTC)](https://www.webrtc.org/)
* [a cross-platform solution to stream audio and video(ffmpeg)](http://www.ffmpeg.org/)
* [packetizer-gatekeepers](https://www.packetizer.com/ipmc/h323/papers/primer/#gatekeepers)