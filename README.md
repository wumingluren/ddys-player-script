# 低端影视替换默认播放器为 DPlayer 播放器 tampermonkey 油猴脚本 

新建一个油猴脚本把代码粘贴进去即可使用
```
// ==UserScript==
// @name         低端影视 DPlayer 播放器
// @namespace    http://your-namespace.example.com
// @version      1.0
// @description  低端影视默认播放器替换为 DPlayer 导航页地址 https://ddys.info/ 备用地址 https://ddys.love/
// @match        https://ddys.pro/*
// @match        https://ddys.mov/*
// @match        https://ddys.tv/*
// @match        https://ddrk.me/*
// @require      https://cdn.bootcdn.net/ajax/libs/vue/2.7.9/vue.min.js
// @require      https://cdn.bootcdn.net/ajax/libs/dplayer/1.27.1/DPlayer.min.js
// @run-at       document-end
// @grant        unsafeWindow
// @grant        GM_xmlhttpRequest
// @grant        GM_registerMenuCommand
// ==/UserScript==
// 低端影视需要设置 @grant 为  unsafeWindow 否则代码不是执行

(function () {
  ("use strict");

  function registerMenu() {
    if (typeof GM_registerMenuCommand !== "function") return;
    GM_registerMenuCommand("💬 给作者留言", function () {
      window.open("https://github.com/wumingluren/ddys-player-script", "_blank");
    });
  }

  registerMenu();

  // 注释
  // 字幕与视频部分
  const container = document.getElementsByClassName(
    "wp-playlist wp-video-playlist"
  )[0];
  if (!container) {
    return;
  }

  // 找到 vjsp 元素
  let vjspElement = document.getElementById("vjsp");

  // 隐藏 vjsp 元素
  vjspElement.style.display = "none";

  // 创建一个新的 <div> 元素作为 dplayer-box
  let dplayerBoxElement = document.createElement("div");
  dplayerBoxElement.id = "dplayer-box";
  dplayerBoxElement.style.width = "100%";
  dplayerBoxElement.style.height = "auto";

  // 创建一个新的 <div> 元素作为 dplayer
  let dplayerElement = document.createElement("div");
  dplayerElement.id = "dplayer";

  // 将 dplayer 元素插入到 dplayer-box 元素中
  dplayerBoxElement.appendChild(dplayerElement);

  // 将 dplayer-box 元素插入到 vjsp 元素的父级的第一个子元素之前
  vjspElement.parentNode.insertBefore(dplayerBoxElement, vjspElement);

  // 在这里编写你的油猴脚本代码，并使用Vue.js和DPlayer进行界面处理
  // ...

  new Vue({
    el: "dplayer-box",
    data: {
      url: "https://v.ddys.pro",
      videoList: [],
      videoInfo: {
        videoUrl: "",
        sub: "",
      },
      origin: "",
      dp: null,
    },
    async created() {
      this.origin = window.location.origin;
      await this.getVideoList();
      await this.getVideoInfo();
      this.initDPlayer();
      this.handleListenLink();
      this.handleListenVjsp();
    },
    methods: {
      // 获取视频列表
      getVideoList() {
        const videoList = JSON.parse(
          document.querySelector(".wp-playlist-script").innerText
        );
        // console.clear();
        console.log("视频信息列表playlist: ", videoList);
        this.videoList = videoList;
      },
      // 获取当前集数
      async getVideoInfo() {
        const search = window.location.search || "?ep=1";
        const index = parseInt(search.split("?ep=")[1].split("&")[0]) - 1;
        const track = this.videoList.tracks[index];

        console.log(search, "===", index, "===", track);

        await this.fetchVideoUrl(track);
      },
      // 视频链接请求主体
      fetchVideoUrl(track) {
        const ddrUrl = `${this.origin}/subddr${track.subsrc}`;
        let videoUrl = "";

        if (!track.src1) {
          videoUrl = `${this.url}${track.src0}`;
        } else {
          return new Promise((resolve, reject) => {
            const endpoint = track.srctype === "1" ? "getvddr2" : "getvddr3";
            // 没有执行不知道为啥
            // GM_xmlhttpRequest({
            //   method: "GET",
            //   url: `${this.origin}/${endpoint}/video?id=${track.src1}&type=json`,
            //   onload: (response) => {
            //     const json = JSON.parse(response.responseText);
            //     videoUrl = json.url || `${this.url}${track.src0}`;
            //     const result = {
            //       videoUrl: videoUrl,
            //       sub: ddrUrl,
            //     };
            //     this.videoInfo = result;

            //     console.log(json, "视频信息: ", result);
            //     resolve(result);
            //   },
            //   onerror: (error) => {
            //     console.log("请求出错", error);
            //     reject(error);
            //   },
            // });
            fetch(`${this.origin}/${endpoint}/video?id=${track.src1}&type=json`)
              .then((response) => response.json())
              .then((json) => {
                videoUrl = json.url || `${this.url}${track.src0}`;
                const result = {
                  videoUrl: videoUrl,
                  sub: ddrUrl,
                };
                this.videoInfo = result;
                console.log(json, "视频信息: ", result);
                resolve(result);
              })
              .catch((error) => {
                reject(error);
              });
          });
        }

        const result = {
          videoUrl: videoUrl,
          sub: ddrUrl,
        };

        this.videoInfo = result;
        return Promise.resolve(result);
      },

      initDPlayer() {
        console.log("初始化");
        const dp = new DPlayer({
          container: document.getElementById("dplayer"),
          autoplay: true,
          screenshot: true,
          hotkey: true,
          volume: 1, // 音量
          playbackSpeed: [0.1, 0.5, 0.75, 1, 1.25, 1.5, 2, 2.5, 3], // 播放速度
          video: {
            // url: "https://lf3-static.bytednsdoc.com/obj/eden-cn/nupenuvpxnuvo/xgplayer_doc/xgplayer-demo-720p.mp4",
            url: this.videoInfo.videoUrl,
            pic: "",
          },
          // subtitle: {
          //   url: "",
          //   type: "webvtt",
          //   fontSize: "25px",
          //   bottom: "10%",
          //   color: "#b7daff",
          // },
        });
        dp.on("error", (e) => {
          console.log("播放器报错", e);
        });
        this.dp = dp;
      },
      handleSwitchVideo() {
        console.log("切换视频", this.videoInfo.videoUrl);
        // 切换视频会播放器界面卡死
        // this.dp.switchVideo({
        //   url: this.videoInfo.videoUrl,
        //   pic: "",
        //   thumbnails: "",
        // });

        // this.dp.play();

        // 销毁也是
        // this.dp.destroy();

        document.getElementById("dplayer")?.remove();

        let dplayerElement = document.getElementById("dplayer");
        if (!dplayerElement) {
          document.getElementById("dplayer-box").innerHTML =
            '<div id="dplayer"></div>';
          this.initDPlayer();
        }
      },
      // 监听url变化
      // 直接监听不到变化所以监听切换集数事件
      async handleListenLink() {
        // 获取所有具有 wp-playlist-caption 类名的元素
        const elements = document.getElementsByClassName("wp-playlist-caption");

        // 遍历元素列表，并为每个元素添加点击事件监听器
        Array.from(elements).forEach((element) => {
          element.addEventListener("click", async () => {
            // 在元素被点击时执行的操作
            console.log("点击集数:", element);

            await this.getVideoInfo();
            this.handleSwitchVideo();
            this.handleListenLink();

            // 为了快速屏蔽老播放器所以应该在被重新创建的时候进行屏蔽
            // document.getElementById("vjsp").style.display = "none";
          });
        });
      },
      // 监听默认播放器是否被重新创建
      handleListenVjsp() {
        // 目标元素的id
        const targetId = "vjsp";

        // 创建一个MutationObserver实例
        const observer = new MutationObserver((mutationsList, observer) => {
          // 在回调函数中检查元素的删除和重新创建
          mutationsList.forEach((mutation) => {
            if (mutation.removedNodes.length > 0) {
              // 检查是否有节点被删除
              for (let i = 0; i < mutation.removedNodes.length; i++) {
                const removedNode = mutation.removedNodes[i];
                if (removedNode.id === targetId) {
                  console.log("元素被删除");
                }
              }
            }
            if (mutation.addedNodes.length > 0) {
              // 检查是否有节点被添加
              for (let i = 0; i < mutation.addedNodes.length; i++) {
                const addedNode = mutation.addedNodes[i];
                if (addedNode.id === targetId) {
                  console.log("元素被重新创建");
                  document.getElementById(targetId).style.display = "none";
                }
              }
            }
          });
        });

        // 配置观察选项
        const config = { childList: true, subtree: true };

        // 获取目标元素
        const targetElement = document.getElementById(targetId);

        // 获取目标元素的父节点
        const parentElement = targetElement.parentNode;

        // 开始观察目标元素的删除和重新创建
        observer.observe(parentElement, config);

        console.log("开始监听", parentElement);
      },
    },
  });
})();

```
