<template>
  <view>
    <view class="title-box" @click="NFClistener">
      {{ title }}
    </view>
    <view style="margin: 30upx">
      写入的：
      <view class="form-box">
        <view class="form-box-time"> ID: </view>
        <input class="form-box-input" type="text" v-model="id" />
      </view>
      <view class="form-box">
        <view class="form-box-time"> PAYLOAD: </view>
        <input class="form-box-input" type="text" v-model="payload" />
      </view>
      <view class="form-box">
        <view class="form-box-time"> TYPE: </view>
        <input class="form-box-input" type="text" v-model="type" />
      </view>
    </view>
    <view style="margin: 30upx">
      读取的：
      <view class="form-box">
        <view class="form-box-time"> ID: </view>
        <view class="form-box-input">
          {{ read.id }}
        </view>
      </view>
      <view class="form-box">
        <view class="form-box-time"> PAYLOAD: </view>
        <view class="form-box-input">
          {{ read.payload }}
        </view>
      </view>
      <view class="form-box">
        <view class="form-box-time"> TYPE: </view>
        <view class="form-box-input">
          {{ read.type }}
        </view>
      </view>
    </view>
    <view style="margin: 30upx">
      {{ content }}
    </view>
    <view class="fixed-btn" style="z-index: 1">
      <view v-if="isWrite" class="my-button-primary" @click="writeData"
        >写入数据</view
      >
    </view>
  </view>
</template>

<!-- <script src="./index.js"></script> -->
<script>
/**
 * 字节对象转字符串
 * @param {Object} arr
 */
const byteToString = function (arr) {
  if (typeof arr === "string") {
    return arr;
  }

  var str = "",
    _arr = arr;

  for (var i = 0; i < _arr.length; i++) {
    var one = _arr[i].toString(2),
      v = one.match(/^1+?(?=0)/);

    if (v && one.length == 8) {
      var bytesLength = v[0].length;

      var store = _arr[i].toString(2).slice(7 - bytesLength);

      for (var st = 1; st < bytesLength; st++) {
        store += _arr[st + i].toString(2).slice(2);
      }

      str += String.fromCharCode(parseInt(store, 2));

      i += bytesLength - 1;
    } else {
      str += String.fromCharCode(_arr[i]);
    }
  }
  return str;
};

/**
 * 字符串转字节
 * @param {Object} str
 */
const string2ArrayBuffer = function (str) {
  // 首先将字符串转为16进制
  let val = "";
  for (let i = 0; i < str.length; i++) {
    if (val === "") {
      val = str.charCodeAt(i).toString(16);
    } else {
      val += "," + str.charCodeAt(i).toString(16);
    }
  }
  // 将16进制转化为ArrayBuffer
  return new Uint8Array(
    val.match(/[\da-f]{2}/gi).map(function (h) {
      return parseInt(h, 16);
    })
  ).buffer;
};
/**
 * 格式化得到aid值
 * @param {Object} buffer
 */
const ab2hex = function (buffer) {
  var hexArr = Array.prototype.map.call(
    new Uint8Array(buffer),

    function (bit) {
      return ("00" + bit.toString(16)).slice(-2);
    }
  );
  return hexArr.join("");
};
let tab = {};
// aid
let aid = [];
// NFC实例对象
let NFCAdapter = null;
// NFC标签对象
let NFCTab = null;
export default {
  onLoad() {
    this.initDevice();
  },
  onUnload() {
    this.closeNFC();
  },
  onHide() {},
  data() {
    return {
      // 视图提示
      title: "初始化...",
      // 视图内容
      content: "",
      // 支持写入
      isWrite: false,
      // 数据
      id: "1001",
      payload: "zou",
      type: "zou",
      read: {
        id: "",
        payload: "",
        type: "",
      },
    };
  },
  methods: {
    // 初始化 NFC 模块。获取实例
    initDevice() {
      console.log("initDevice");
      NFCAdapter = wx.getNFCAdapter();
      console.log(NFCAdapter);
      tab = {
        "ISO-DEP": NFCAdapter.getIsoDep(),
        "MIFARE Classic": NFCAdapter.getMifareClassic(),
        "MIFARE Ultraligh": NFCAdapter.getMifareUltralight(),
        NDEF: NFCAdapter.getNdef(),
        "NFC-A": NFCAdapter.getNfcA(),
        "NFC-B": NFCAdapter.getNfcB(),
        "NFC-F": NFCAdapter.getNfcF(),
        "NFC-V": NFCAdapter.getNfcV(),
      };
      this.NFClistener();
    },
    //  触发监听NFG事件
    NFClistener() {
      console.log("NFClistener");
      NFCAdapter.startDiscovery({
        success: (res) => {
          this.title = "请将设备放入识别区NFC";
          console.log(res);
        },
        fail: (error) => {
          this.title = "点击重试";
          console.error(error);
        },
        complete: (res) => {
          console.log(res);
        },
      });
      // 监听 NFC 标签
      NFCAdapter.onDiscovered((callback) => {
        console.log("onDiscovered callback=>", callback);
        let aid = parseInt(ab2hex(callback.id), 16);
        console.log(aid);
        if (callback.messages) {
          let cordsArray = callback.messages[0].records;
          cordsArray.find((item) => {
            this.read.payload = byteToString(new Uint8Array(item.payload));
            this.read.id = byteToString(new Uint8Array(item.id));
            this.read.type = byteToString(new Uint8Array(item.type));
          });
        }

        if (callback.techs.length != 0) {
          this.title = "识别成功！";
          this.content = "可支持标签：";
          callback.techs.forEach((res, index) => {
            console.log(res);
            if (index != 0) {
              this.content += "、";
            }
            this.content += res;
            // 支持写入
            if (res == "NDEF") {
              this.isWrite = true;
            }
          });
        } else {
          this.title = "无效设备";
          console.log("无效设备");
        }
      });
    },
    /* 设备标签 */
    initTab(item) {
      let NFCTab = tab[item];
      NFCTab.connect({
        success: (res) => {
          this.title = "连接设备成功";
          console.log(res);
        },
        fail: (error) => {
          this.title = "连接设备失败";
          console.error(error);
        },
        complete: (res) => {
          console.log(res);
        },
      });
      return NFCTab;
    },
    /* 获取ATQA信息 */
    getAtqa() {
      NFCTab.getAtqa({
        success: (res) => {
          console.log(res);
        },
        fail: (error) => {
          console.error(error);
        },
        complete: (res) => {
          console.log(res);
        },
      });
    },
    /* 获取最大传输长度 */
    getMaxTransceiveLength() {
      NFCTab.getMaxTransceiveLength({
        success: (res) => {
          console.log(res);
        },
        fail: (error) => {
          console.error(error);
        },
        complete: (res) => {
          console.log(res);
        },
      });
    },
    /* 获取SAK信息 */
    getSak() {
      NFCTab.getSak({
        success: (res) => {
          console.log(res);
        },
        fail: (error) => {
          console.error(error);
        },
        complete: (res) => {
          console.log(res);
        },
      });
    },
    /**
     * 写入数据
     */
    async writeData() {
      // 获取初始化标签对象——连接设备
      NFCTab = await this.initTab("NDEF");
      // 准备写入的数据
      // let data:Array<string> = new Array('maker');
      const records = [
        {
          id: string2ArrayBuffer(this.id),
          payload: string2ArrayBuffer(this.payload),
          type: string2ArrayBuffer(this.type),
          tnf: 2,
        },
      ];
      // 执行写入
      NFCTab.writeNdefMessage({
        records: records,
        success: (res) => {
          this.title = "数据写入成功";
          console.log(res);
        },
        fail: (error) => {
          this.title = "数据写入失败";
          console.error(error);
        },
        complete: (res) => {
          this.closeConnect(NFCTab);
          this.isWrite = false;
          this.title = "请将设备放入识别区NFC";
          console.log(res);
        },
      });
    },
    // 关闭 连接
    closeConnect(NFCTab) {
      NFCTab.close({
        success: (res) => {
          this.title = "清除标签连接成功";
          console.log("清除标签连接成功");
        },
        fail: (error) => {
          this.title = "清除标签连接失败";
          console.error("清除标签连接失败");
        },
        complete: (res) => {
          console.log(res);
        },
      });
    },
    /* 取消取消监听 NFC Tag */
    closeNFC() {
      NFCAdapter.offDiscovered((callback) => {});
      NFCAdapter.stopDiscovery();
    },
  },
};
</script>
<style lang="scss">
.title-box {
  height: 100upx;
  line-height: 100upx;
  padding-left: 30upx;
  text-align: start;
  border-bottom: 1upx solid #d1d1d1;
}
.fixed-btn {
  position: fixed;
  left: 0;
  bottom: 0;
  z-index: 99;
  width: 100%;
  padding: 24upx;
  box-sizing: border-box;
  background-color: #ffffff;
}
.my-button-primary {
  height: 80upx;
  background: linear-gradient(270deg, #5063b4 2%, #457fff 97%);
  border-radius: 8rpx;
  font-size: 34rpx;
  color: #ffffff;
  line-height: 80upx;
  margin: 10upx 0upx;
  text-align: center;
}
.my-button-primary:active {
  background: linear-gradient(270deg, #4b5eaa 2%, #427af4 97%);
}
.form-box {
  display: flex;
  justify-content: space-between;
  align-items: center;
  min-height: 80upx;

  .form-box-input {
    font-size: 28upx;
    flex: 1;
    margin-left: 30upx;
    color: #3e3e3e;
    text-align: start;
  }
  .form-box-time {
    font-size: 28upx;
    margin-left: 30upx;
    width: 200upx;
    color: #3e3e3e;
    text-align: start;
  }
}
</style>
