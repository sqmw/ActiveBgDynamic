<template>
  <video volume="0" id="active-bg-video" class="full" loop="loop" autoplay="autoplay" style="width: 100%;height:100%;z-index: -1" fullscreen="true" :src="nowUrl"></video>
</template>

<script>
  /**
   *  backend response
   *  regular activeDynamicBg communication format as follows
   *  {
   *    action: changeBg | executeScript | getImgFromVideo | mouseAction  ----> {changeBg: 0,executeScript:1, getImgFromVideo: 2}
   *    data of each action
   *      changeBg: url,
   *      executeScript: script,
   *      getImgFromVideo: ["url1","url2",...]
   *      mouseAction: {
   *        type: click | over | move,
   *        // 对 domId 进行了处理，如果没有找到对应的id的 dom 就直接对首个canvas进行操作
   *        domId: xxx,
   *        point: {
   *          x: 0,
   *          y: 0
   *        }
   *      }
   *
   *
   *  }
   */
  /**
   * fronted reqData
   * changeBg: none
   * executeScript: none
   * getImgFromVideo: return the img char stream encoded base64
   * {
   *   type: -1 -> rest  0 -> imgBase64
   *   data: ""
   * }
   *
   * data: imgBase64 -> data = {index:?, base64}
   */
  import { fetch, Body } from '@tauri-apps/api/http';
  import {ref} from "vue";
  export default {
    name: "ActiveTauriBg",
    setup(){
      // scriptList, 用来存储保存的script id，new Date().getTime()
      /**
       * {
       *   index:url,
       * }
       */
      let videoUrlMap = {}
      // 为了有更好的效果，图片应该一张一张地传输
      // let videoResList = []
      const req_typeRest = -1, req_typeImgBase64 = 0
      const action_rest = -1, action_changeBg = 0,action_executeScript = 1, action_getImgFromVideo = 2, action_mouse = 3
      const type_mouseActionClick = 1, type_mouseActionOver = 2, type_mouseActionMove = 4
      /// 设置样式
      const canvas = document.createElement("canvas");
      const ctx = canvas.getContext("2d");
      const video = document.createElement("video");
      video.setAttribute('crossorigin', 'anonymous');
      video.setAttribute('controls', 'controls');
      video.currentTime = 1
      let nowUrl = ref("https://img-baofun.zhhainiao.com/pcwallpaper_ugc/scene/b70351c2fe86b597c9265cfe7259ea4e_preview.mp4");
      return {
        nowUrl,
        action_rest,
        action_changeBg,
        action_getImgFromVideo,
        action_executeScript,
        action_mouse,
        video,
        canvas,
        videoUrlMap,
        req_typeRest,
        req_typeImgBase64,
        ctx,
        type_mouseActionClick,
        type_mouseActionOver,
        type_mouseActionMove
      }
    },

    methods:{
      change(receive){
        this.nowUrl = receive
      },
      getBase64ImgCodeFromVideo(url,width=400,height=240) {
        /// 返回一个 Promise
        return new Promise((resolve, reject)=> {
          /// 清除原来的画像
          this.ctx.clearRect(0,0, width, height)
          let dataURL = '';
          this.video.setAttribute('src', url);
          /// 这个是必须的，否则就是白屏
          this.video.currentTime = 1
          this.video.addEventListener('loadeddata', (e)=>{
            this.canvas.width = width;
            this.canvas.height = height;
            this.canvas.getContext("2d").drawImage(this.video, 0, 0, width, height);
            dataURL = this.canvas.toDataURL('image/png');
            /// 返回的直接是 base64 编码
            // console.log("base64编码",dataURL)
            resolve(dataURL.split(",")[1]);
          });
        })
      },
      // type = 0(imgBase64), data: imgBase64 -> data = {index:?, base64}
      // 这个连接仅仅是为了post msg
      async postImgBase64(data = {}) {
          /// 必须设置同步，一个一个处理
        await fetch("http://localhost:4444/", {
          timeout: 30,
          method: "POST",
          body: Body.json(data)
        })
      },
      triggerMouseClickEvent(_dom,x = 0, y = 0){
        _dom.dispatchEvent(new MouseEvent("click",{
          view: window,
          screenX: x,
          screenY: y,
        }))
      },
      triggerMouseOverEvent(_dom, x = 0, y = 0){
        _dom.dispatchEvent(new MouseEvent("mouseover",{
          screenX: x,
          screenY: y,
        }))
      },
      triggerMouseMoveEvent(_dom,x = 0, y = 0){
        _dom.dispatchEvent(new MouseEvent("mousemove",{
          screenX: x,
          screenY: y,
        }))
      },
    },
    async mounted() {
      // 只有用来承接返回的数据
      let resObj = {};
      /// 返回给 backend 的data
      let reqData = {
        type: this.req_typeRest,
        data: ""
      }
      setInterval(async () => {
        let script
        let scriptId
        let _dom
        let _clientX
        let _clientY
        fetch("http://localhost:4444/",{
          timeout: 30,
          method: "POST",
          body: Body.json(reqData)
          /// then 之后的是连接一次结束之后的处理
        }).then(async (res) => {
          resObj = res.data;
          switch (resObj["action"]) {
            /// 修改背景
            case this.action_changeBg: {
              if (this.nowUrl !== resObj["data"]) {
                this.nowUrl = resObj["data"]
                this.change(this.nowUrl);
                reqData["type"] = this.req_typeRest
              }
              break
            }
            /// 执行脚本
            /// 重新定义这里的规范
            /// backend 会告诉前台需要的script的name XXX
            case this.action_executeScript: {
              /// scriptName
              script = document.createElement("script")
              script.setAttribute("type","text/javascript")
              script.setAttribute("src",`http://localhost:4444/script?name=${resObj["data"]}`)
              scriptId = "scriptId" + new Date().getTime().toString()
              script.setAttribute("id", scriptId)
              document.body.appendChild(script)
              /// 定时清理script标签
              setTimeout(()=>{
                document.getElementById(`${scriptId}`).remove();
              },10000)
              /// 添加scriptId
              reqData["type"] = this.req_typeRest
              reqData["data"] = ""
              break
            }
            /// 获取base64图片
            case this.action_getImgFromVideo: {
              /** 每次传输来的是一页的图片的链接
               * {
               *    index1:url1,
               *    index2:url2,
               * }
               */
              this.videoUrlMap = resObj["data"]
              /// js的 obj/map 的 key 是 str
              // console.log(this.videoUrlMap);

              for (const videoUrlMapKey in this.videoUrlMap) {
                /// 这个下面的 data 也是和上面的 reqData 对应的 this.videoUrlMap[index]
                // console.log(`index:${videoUrlMapKey} ${this.videoUrlMap[`${videoUrlMapKey}`]}`)
                await this.postImgBase64({
                  "type": this.req_typeImgBase64,
                  "data": {
                    "index": videoUrlMapKey,
                    "base64": await this.getBase64ImgCodeFromVideo(this.videoUrlMap[videoUrlMapKey])
                  }
                })
              }
              reqData["type"] = this.req_typeRest
              reqData["data"] = ""
              break
            }
            /// 因为各种原因，鼠标事件未能实现，后期需要根据钩子来完善
            case this.action_mouse: {
              _dom = document.getElementById(resObj["data"]["domId"])??document.querySelector("canvas")
              _clientX = resObj["data"]["point"]["x"]
              _clientY = resObj["data"]["point"]["y"]
              switch (resObj["data"]["type"]){
                case this.type_mouseActionClick:{
                  this.triggerMouseClickEvent(_dom,_clientX,_clientY)
                  break;
                }
                case this.type_mouseActionOver:{
                  this.triggerMouseOverEvent(_dom,_clientX,_clientY)
                  break;
                }
                case this.type_mouseActionMove:{
                  this.triggerMouseMoveEvent(_dom,_clientX,_clientY)
                  break;
                }
                /// 表示两种情况
                case this.type_mouseActionMove | this.type_mouseActionClick:{
                  this.triggerMouseMoveEvent(_dom,_clientX,_clientY)
                  this.triggerMouseClickEvent(_dom,_clientX,_clientY)
                  break;
                }
                case this.type_mouseActionMove | this.type_mouseActionOver:{
                  this.triggerMouseMoveEvent(_dom,_clientX,_clientY)
                  this.triggerMouseOverEvent(_dom,_clientX,_clientY)
                  break;
                }
                case this.type_mouseActionOver | this.type_mouseActionClick:{
                  this.triggerMouseOverEvent(_dom,_clientX,_clientY)
                  this.triggerMouseClickEvent(_dom,_clientX,_clientY)
                  break;
                }
                /// 三种
                case this.type_mouseActionMove | this.type_mouseActionOver | this.type_mouseActionClick:{
                  this.triggerMouseMoveEvent(_dom,_clientX,_clientY)
                  this.triggerMouseOverEvent(_dom,_clientX,_clientY)
                  this.triggerMouseClickEvent(_dom,_clientX,_clientY)
                  break;
                }
              }
              /// 这里需要处理鼠标的点击事件，通过win32捕获之后发送过来
              break;
            }
            /// 表示的是 rest
            case this.action_rest:{
              /// 重置 resData
              reqData["type"] = this.req_typeRest
              reqData["data"] = ""
              break
            }
          }

        }).catch((err)=>{
          //console.log(err)
        })
      },100)
    }
  }
  /// npm run tauri dev
</script>

<style>
body{
  overflow: hidden;
}

.full{
  object-fit: fill;
  z-index: 1;
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
}
</style>