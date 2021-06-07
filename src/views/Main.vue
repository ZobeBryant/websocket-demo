<template>
  <div id="container">
    <div id="receive">
      <p>服务端返回的消息</p>
      <div id="receiveBox" ref="receiveBox">
        <div v-for="(msg,index) in receiveMsg" :key="index">{{msg}}</div>
      </div>
    </div>

    <div id="send">
      <textarea type="text" id="msg-need-send" v-model="sendMsg"></textarea>
      <p>
        <button id="send-btn" @click="send">点击发消息给服务端</button>
      </p>
    </div>
    <div id="exit" @click="exit">关闭连接</div>
  </div>
</template>

<script>
export default {
  name: "Main",
  data() {
    return {
      receiveMsg: [],
      sendMsg: "",
      ws: null, // 建立的连接
      timeout: 30 * 1000, // 30秒一次心跳
      timeoutObj: null, // 心跳倒计时
      serverTimeoutObj: null, // 心跳倒计时
      timeoutnum: null, // 重连倒计时
      scrollHeight: ""
    };
  },
  methods: {
    init() {
      console.log("websocket初始化");
      var _this = this;
      try {
        this.ws = new WebSocket("ws://127.0.0.1:3000/websocket/test");
        this.ws.onopen = this.onopen;
        this.ws.onmessage = this.onmessage;
        this.ws.onclose = this.onclose;
      } catch (error) {
        console.log(error);
        this.reconnect();
      }
    },
    // 重新连接
    reconnect() {
      console.log("重新连接");
      var _this = this;
      if (this.lockReconnect) {
        return;
      }
      this.lockReconnect = true;
      this.timeoutnum && clearTimeout(this.timeoutnum);
      // 没有连接上会一直重连，设置延迟避免请求过多
      this.timeoutnum = setTimeout(function() {
        // 新连接
        _this.init();
        _this.lockReconnect = false;
      }, 5000);
    },
    // 开启心跳
    start() {
      console.log("心跳检测");
      var _this = this;
      this.timeoutObj && clearTimeout(this.timeoutObj);
      this.serverTimeoutObj && clearTimeout(this.serverTimeoutObj);
      this.timeoutObj = setTimeout(function() {
        // 这里发送一个心跳，后端收到后，返回一个心跳消息
        if (_this.ws.readyState == 1) {
          // 如果连接正常
          _this.ws.send("heartCheck");
        }
        _this.serverTimeoutObj = setTimeout(function() {
          // 超时关闭
          _this.ws.close();
        }, _this.timeout);
      }, this.timeout);
    },
    onopen() {
      console.log("onopen");
      // 心跳检测
      this.start();
    },
    // 如果用户没有发送或者接收消息，onmessage也会间隔一段时间执行start方法进行心跳检测。因为在初始化时，客户端发送了"heartcheck"消息，服务器会回复消息，onmessage监听到服务器发送到的消息，就再次进行心跳检测，一直循环下去。这样就维持了心跳检测。
    onmessage(data) {
      console.log("onmessage");
      var _this = this;
      _this.receiveMsg.push(data.data);
      this.scrollHeight = _this.$refs.receiveBox.scrollHeight;
      // scrollHeight: 文档内容实际高度，包括超出视窗的溢出部分
      // scrollTop: 滚动条滚动距离
      _this.$refs.receiveBox.scrollTo({
        top: _this.scrollHeight,
        behavior: "smooth"
      });
      // 心跳检测
      this.start();
    },
    onclose() {
      console.log("连接关闭");
      // 重连
      this.reconnect();
    },
    onerror() {
      console.log("出现错误");
      // 重连
      this.reconnect();
    },
    exit() {
      this.ws.close();
    },
    send() {
      this.ws.send(this.sendMsg);
      this.sendMsg = "";
    }
  },
  created() {
    this.init();
  }
};
</script>

<style scoped>
#container {
  width: 800px;
  height: 600px;
  border: 1px solid #ccc;
  margin: 100px auto 0 auto;
  text-align: center;
}
#receiveBox {
  width: 300px;
  height: 200px;
  overflow: auto;
  margin: 0 auto 24px auto;
}
#msg-need-send {
  width: 300px;
  height: 100px;
  resize: none;
  border-radius: 10px;
}
#send-btn {
  border: none;
  border-radius: 5px;
  background: #25d1ff;
  padding: 5px 10px;
  color: #fff;
  cursor: pointer;
}
#exit {
  border: 1px solid #ccc;
  border-radius: 5px;
  background: none;
  padding: 5px 10px;
  cursor: pointer;
}
</style>

