码率的英文名为**bps(bit per second)**，就是用平均每秒多少bit来衡量一个视频大小。



- 100X60S=6000s
- 1GB=1024MB= 1024X1024KB=1024X1024X1024Byte=1024X1024X1024X8bit=8589934592bit

那么这个视频的码率大概就是1.4Mbit/s(8589934592/6000),这个比特率在在线视频中已经是非常高的了，一般主流视频平台的最高码率在1Mbit左右，比如直播网站斗鱼的高清选项实际播放的视频码率是900Kbit/s(0.9Mbit)。



先采集视频音频，后创建peer connection



加入会议消息、会议状态变化、sdp交换等都是通过socket信令服务器，但sdp创建是peerconnection进行创建。

通过信令服务sdp交换后，两个peer都知道了两端所需要的编码等信息，准备开始网络传输。

开始网络传输前需要收集cadidace，这个收集过程在调用setLocalDescription后即会在SdpObserver的onSetSuccess方法中进行收集。



onIceCandidate()的调用时机？？？（好像是在setLocalDescription后就开始调用了）



收集后通过信令服务发送给对端（边收集边交换）




