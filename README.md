
## 一 、 概述


          PeerJS 是一个基于浏览器WebRTC功能实现的js功能包,简化了WebrRTC的开发过程，对底层的细节做了封装，直接调用API即可,再配合surging 协议组件化从而做到稳定，高效可扩展的微服务，再利用RtmpToWebrtc 引擎组件可以做到不仅可以利用httpflv 观看rtmp推流直播，还可以采用基于 Webrtc的peerjs 进行观看，那么今天要讲的是如何结合开发语音视频通话功能。放到手机和电脑上都可以实现语音视频通话。


    一键运行打包成品下载：[https://pan.baidu.com/s/1MVsjKtVYpUonauAz9ZXtPg?pwd\=1q2g](https://github.com)


    测试用户：fanly


    测试密码：123456


      为了让大家节约时间，能尽快运行产品看到效果，上面有 一键运行打包成品可以进行下载测试运行。


## 二、如何测试运行


以下是目录结构,


IDE:consul 注册中心


kayak.client: 网关


kayak.server:微服务


apache\-skywalking\-apm：skywalking链路跟踪


![](https://img2024.cnblogs.com/blog/192878/202408/192878-20240830090826551-1721729040.png)


 


 以上是目录结构， 不需要进入管理界面配置网络组件，启动后自带端口96的ws协议主机，只要打开video文件夹，里面有两个语音通话的html测试文件，在同一一个局域网只要输入对方的name就可以进行语音通话


![](https://img2024.cnblogs.com/blog/192878/202408/192878-20240830091054410-1514136363.png)


 打开界面如图


![](https://img2024.cnblogs.com/blog/192878/202408/192878-20240830091139629-551324421.png)


 


## 三、基于surging如何开发


以上是没有开发环境的进行下载进行下载测试，那么正在使用surging 的如何开发此功能呢？


1\. 创建服务接口，继承于IServiceKey




```
 [ServiceBundle("Device/{Service}")]
 public  interface IChatService : IServiceKey
 {
 }
```


2\. 创建中间服务，继承于WSBehavior, IChatService




```
  internal class ChatService : WSBehavior, IChatService
  {
      private static readonly ConcurrentDictionary<string, string> _users = new ConcurrentDictionary<string, string>();
      private static readonly ConcurrentDictionary<string, string> _clients = new ConcurrentDictionary<string, string>();

      protected override void OnOpen()
      {
         var _name = Context.QueryString["name"]; 
          if (!string.IsNullOrEmpty(_name))
          {
              _clients[ID] = _name;
              _users[_name] = ID;
          }
      }

      protected override void OnError( WebSocketCore.ErrorEventArgs e)
      {
        var msg = e.Message;
      }

      protected override void OnMessage(MessageEventArgs e)
      {
          if (_clients.ContainsKey(ID))
          {

              var message = JsonConvert.DeserializeObjectstring, object>>(e.Data);
              //消息类型
             message.TryGetValue("type",out object @type);
              message.TryGetValue("toUser", out object toUser);
              message.TryGetValue("fromUser", out object fromUser);
              message.TryGetValue("msg", out object msg);
              message.TryGetValue("sdp", out object sdp);
              message.TryGetValue("iceCandidate", out object iceCandidate);
              

              Dictionary result = new Dictionary();
              result.Add("type", @type);

              //呼叫的用户不在线
              if (!_users.ContainsKey(toUser?.ToString()))
              {
                  result["type"]= "call_back";
                  result.Add("fromUser", "系统消息");
                  result.Add("msg", "Sorry，呼叫的用户不在线！");

                  this.Client().SendTo(JsonConvert.SerializeObject(result), ID);
                  return;
              }

              //对方挂断
              if ("hangup".Equals(@type))
              {
                  result.Add("fromUser", fromUser);
                  result.Add("msg", "对方挂断！");
              }

              //视频通话请求
              if ("call_start".Equals(@type))
              {
                  result.Add("fromUser", fromUser);
                  result.Add("msg", "1");
              }

              //视频通话请求回应
              if ("call_back".Equals(type))
              {
                  result.Add("fromUser", toUser);
                  result.Add("msg", msg);
              }

              //offer
              if ("offer".Equals(type))
              {
                  result.Add("fromUser", toUser); 
                  result.Add("sdp", sdp);
              }

              //answer
              if ("answer".Equals(type))
              {
                  result.Add("fromUser", toUser);
                  result.Add("sdp", sdp);
              }

              //ice
              if ("_ice".Equals(type))
              {
                  result.Add("fromUser", toUser);
                  result.Add("iceCandidate", iceCandidate);
              }

              this.Client().SendTo(JsonConvert.SerializeObject(result), _users.GetValueOrDefault(toUser?.ToString()));
          }
      }

      protected override void OnClose(CloseEventArgs e)
      {
         if( _clients.TryRemove(ID, out string name))
          _users.TryRemove (name, out string value);
      }

  }
```


3\.设置surgingSettings的WSPort端口配置，默认96


以上就是利用websocket协议中转消息，下面是页面如何编号，代码如下：




```
DOCTYPE>


<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>WebRTC + WebSockettitle>
    <meta name="viewport" content="width=device-width,initial-scale=1.0,user-scalable=no">
    <style>
        html,body{
            margin: 0;
            padding: 0;
        }
        #main{
            position: absolute;
            width: 370px;
            height: 550px;
        }
        #localVideo{
            position: absolute;
            background: #757474;
            top: 10px;
            right: 10px;
            width: 100px;
            height: 150px;
            z-index: 2;
        }
        #remoteVideo{
            position: absolute;
            top: 0px;
            left: 0px;
            width: 100%;
            height: 100%;
            background: #222;
        }
        #buttons{
            z-index: 3;
            bottom: 20px;
            left: 90px;
            position: absolute;
        }
        #toUser{
            border: 1px solid #ccc;
            padding: 7px 0px;
            border-radius: 5px;
            padding-left: 5px;
            margin-bottom: 5px;
        }
        #toUser:focus{
            border-color: #66afe9;
            outline: 0;
            -webkit-box-shadow: inset 0 1px 1px rgba(0,0,0,.075),0 0 8px rgba(102,175,233,.6);
            box-shadow: inset 0 1px 1px rgba(0,0,0,.075),0 0 8px rgba(102,175,233,.6)
        }
        #call{
            width: 70px;
            height: 35px;
            background-color: #00BB00;
            border: none;
            margin-right: 25px;
            color: white;
            border-radius: 5px;
        }
        #hangup{
            width:70px;
            height:35px;
            background-color:#FF5151;
            border:none;
            color:white;
            border-radius: 5px;
        }
    style>
head>
<body>
    <div id="main">
        <video id="remoteVideo" playsinline autoplay>video>
        <video id="localVideo" playsinline autoplay muted>video>

        <div id="buttons">
            <input id="toUser" placeholder="输入在线好友账号"/><br/>
            <button id="call">视频通话button>
            <button id="hangup">挂断button>
        div>
    div>
body>


<script type="text/javascript" th:inline="javascript">
    let username = "fanly";
    let localVideo = document.getElementById('localVideo');
    let remoteVideo = document.getElementById('remoteVideo');
    let websocket = null;
    let peer = null;

    WebSocketInit();
    ButtonFunInit();

    /* WebSocket */
    function WebSocketInit(){
        //判断当前浏览器是否支持WebSocket
        if ('WebSocket' in window) {
            websocket = new WebSocket("ws://127.0.0.1:961/device/chat?name="+username);
        } else {
            alert("当前浏览器不支持WebSocket！");
        }

        //连接发生错误的回调方法
        websocket.onerror = function (e) {
            alert("WebSocket连接发生错误！");
        };

        //连接关闭的回调方法
        websocket.onclose = function () {
            console.error("WebSocket连接关闭");
        };

        //连接成功建立的回调方法
        websocket.onopen = function () {
            console.log("WebSocket连接成功");
        };

        //接收到消息的回调方法
        websocket.onmessage = async function (event) {
            let { type, fromUser, msg, sdp, iceCandidate } = JSON.parse(event.data.replace(/\n/g,"\\n").replace(/\r/g,"\\r"));

            console.log(type);

            if (type === 'hangup') {
                console.log(msg);
                document.getElementById('hangup').click();
                return;
            }

            if (type === 'call_start') {
                let msg = "0"
                if(confirm(fromUser + "发起视频通话，确定接听吗")==true){
                    document.getElementById('toUser').value = fromUser;
                    WebRTCInit();
                    msg = "1"
                }

                websocket.send(JSON.stringify({
                    type:"call_back",
                    toUser:fromUser,
                    fromUser:username,
                    msg:msg
                }));

                return;
            }

            if (type === 'call_back') {
                if(msg === "1"){
                    console.log(document.getElementById('toUser').value + "同意视频通话");

                    //创建本地视频并发送offer
                    let stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true })
                    localVideo.srcObject = stream;
                    stream.getTracks().forEach(track => {
                        peer.addTrack(track, stream);
                    });

                    let offer = await peer.createOffer();
                    await peer.setLocalDescription(offer); 
                    let newOffer = offer;
   
                    newOffer["fromUser"] = username;
                    newOffer["toUser"] = document.getElementById('toUser').value;
                    websocket.send(JSON.stringify(newOffer));
                }else if(msg === "0"){
                    alert(document.getElementById('toUser').value + "拒绝视频通话");
                    document.getElementById('hangup').click();
                }else{
                    alert(msg);
                    document.getElementById('hangup').click();
                }

                return;
            }

            if (type === 'offer') {
                let stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
                localVideo.srcObject = stream;
                stream.getTracks().forEach(track => {
                    peer.addTrack(track, stream);
                });

                await peer.setRemoteDescription(new RTCSessionDescription({ type, sdp }));
                let answer = await peer.createAnswer();
                let newAnswer = answer;

                newAnswer["fromUser"] = username;
                newAnswer["toUser"] = document.getElementById('toUser').value;
                websocket.send(JSON.stringify(newAnswer));

                await peer.setLocalDescription(answer);
                return;
            }

            if (type === 'answer') {
                peer.setRemoteDescription(new RTCSessionDescription({ type, sdp }));
                return;
            }

            if (type === '_ice') {
                peer.addIceCandidate(iceCandidate);
                return;
            }

        }
    }

    /* WebRTC */
    function WebRTCInit(){
        peer = new RTCPeerConnection();

        //ice
        peer.onicecandidate = function (e) {
            if (e.candidate) {
                websocket.send(JSON.stringify({
                    type: '_ice',
                    toUser:document.getElementById('toUser').value,
                    fromUser:username,
                    iceCandidate: e.candidate
                }));
            }
        };

        //track
        peer.ontrack = function (e) {
            if (e && e.streams) {
                remoteVideo.srcObject = e.streams[0];
            }
        };
    }

    /* 按钮事件 */
    function ButtonFunInit(){
        //视频通话
        document.getElementById('call').onclick = function (e){
            document.getElementById('toUser').style.visibility = 'hidden';

            let toUser = document.getElementById('toUser').value;
            if(!toUser){
                alert("请先指定好友账号，再发起视频通话！");
                return;
            }

            if(peer == null){
                WebRTCInit();
            }

            websocket.send(JSON.stringify({
                type:"call_start",
                fromUser:username,
                toUser:toUser,
            }));
        }

        //挂断
        document.getElementById('hangup').onclick = function (e){
            document.getElementById('toUser').style.visibility = 'unset';

            if(localVideo.srcObject){
                const videoTracks = localVideo.srcObject.getVideoTracks();
                videoTracks.forEach(videoTrack => {
                    videoTrack.stop();
                    localVideo.srcObject.removeTrack(videoTrack);
                });
            }

            if(remoteVideo.srcObject){
                const videoTracks = remoteVideo.srcObject.getVideoTracks();
                videoTracks.forEach(videoTrack => {
                    videoTrack.stop();
                    remoteVideo.srcObject.removeTrack(videoTrack);
                });

                //挂断同时，通知对方
                websocket.send(JSON.stringify({
                    type:"hangup",
                    fromUser:username,
                    toUser:document.getElementById('toUser').value,
                }));
            }

            if(peer){
                peer.ontrack = null;
                peer.onremovetrack = null;
                peer.onremovestream = null;
                peer.onicecandidate = null;
                peer.oniceconnectionstatechange = null;
                peer.onsignalingstatechange = null;
                peer.onicegatheringstatechange = null;
                peer.onnegotiationneeded = null;

                peer.close();
                peer = null;
            }

            localVideo.srcObject = null;
            remoteVideo.srcObject = null;
        }
    }
script>
html>
```


以上是页面的代码，如需要添加其它账号测试只要更改username ，或者ws地址也可以更改标记红色的区域。


 


## 三、总结


本人正在开发平台，如有疑问可以联系作者，QQ群：744677125


 本博客参考[楚门加速器p](https://tianchuang88.com)。转载请注明出处！
