---
title: "Electron+vue从零开始打造一个本地播放器"
date: 2021-03-21T22:38:07+08:00
draft: false
---

![BCM9HI.png](https://s1.ax1x.com/2020/10/21/BCM9HI.png)
## 为什么要做？
女朋友工作是音频后期，平常会收集一些音频音乐，需要看音频的频谱波形，每次用au这种大型软件播放音乐看波形，很不方便，看到她这么辛苦，身为程序猿的我痛心疾首，于是，就有了这么一个小软件，软件涉及到的技术主要为electron,vue,node,波形的展示主要通过[wavesurfer](https://wavesurfer-js.org/ "wavesurfer")生成。

## 从零开始-搭建项目
项目通过vue脚手架搭建的，所以需要安装cli工具,如果已经装了，可以跳过这一步.

    npm install -g @vue/cli
    # OR
    yarn global add @vue/cli

装好后,通过脚手架搭建项目

    vue create music

vue需要与electron集成，这里社区已经有比较成熟的vue插件了,[Vue CLI Plugin Electron Builder](https://nklayman.github.io/vue-cli-plugin-electron-builder/guide/#installation "Vue CLI Plugin Electron Builder")。

    vue add electron-builder

懒人可以直接去clone我的搭建好得架子直接开发， [戳这里](https://github.com/Kerinlin/simple-electron-vue-template "戳这里")。

## 从零开始-项目开发
首先先明确下这个播放器的功能需求，主要有这几个
- 不添加文件目录，加载任意的本地文件系统内的音频文件，直接调用播放器播放
- 前一首后一首功能
- 声音音量控制
- 自定义软件窗口

### 如何关联播放
如何实现关联播放？因为对electron不是很熟，查了很久electron的资料，终于找到了配置项，需要配置fileAssociations

            fileAssociations: [
              {
                ext: ["mp3", "wav", "flac", "ogg", "m4a"],
                name: "music",
                role: "Editor"
              }
            ],

配置好后，通过electron的[open-file事件](https://www.electronjs.org/docs/api/app#%E4%BA%8B%E4%BB%B6-open-file-macos "open-file事件")，获取打开的音频文件的本地路径。对于windows,需要通过process.argv，来获取文件路径。

    const filePath = process.argv[1];

### 如何加载本地音频文件
上一步通过配置拿到文件的本地路径后，下一步就是通过路径读取音频文件的信息。由于音频的插件无法解析绝对路径，所以需要通过node的文件系统，通过[fs.readFileSync](http://nodejs.cn/api/fs.html#fs_fs_readfilesync_path_options "fs.readFileSync")读取到文件的buffer信息。

    let buffer = fs.readFileSync(diskPath); //读取文件，并将缓存区进行转换

读取后需要将buffer转换成node可读流

    const stream = this.bufferToStream(buffer);//将buffer数据转换成node 可读流

转换方法 **bufferToStream**

        bufferToStream(binary) {
          const readableInstanceStream = new Readable({
            read() {
              this.push(binary);
              this.push(null);
            }
          });
          return readableInstanceStream;
        }


转换成流后需要将音频流转换成blob对象来加载，实现[方法](https://github.com/feross/stream-to-blob/blob/master/index.js "方法")

    module.exports = streamToBlob

    function streamToBlob (stream, mimeType) {
      if (mimeType != null && typeof mimeType !== 'string') {
        throw new Error('Invalid mimetype, expected string.')
      }
      return new Promise((resolve, reject) => {
        const chunks = []
        stream
          .on('data', chunk => chunks.push(chunk))
          .once('end', () => {
            const blob = mimeType != null
              ? new Blob(chunks, { type: mimeType })
              : new Blob(chunks)
            resolve(blob)
          })
          .once('error', reject)
      })
    }

转blob

      let fileUrl; // blob对象
      streamToBlob(stream)
        .then(res => {
          fileUrl = res;
          // console.log(fileUrl);

          //将blob对象转成blob链接
          let filePath = window.URL.createObjectURL(fileUrl);
          // console.log(filePath);
          this.wavesurfer.load(filePath);

          // 自动播放
          this.wavesurfer.play();
          this.playing = true;
        })
        .catch(err => {
          console.log(err);
        });

这样就实现了加载本地文件播放了

### 上一首下一首功能
这里的上一首下一首的功能是基于上面获取到的文件的绝对路径，通过node的path模块,[path.dirname](http://nodejs.cn/api/path.html#path_path_dirname_path "path.dirname")获取到文件的父级目录。

    const dirPath = path.dirname(diskPath);

然后通过[fs.readdir](http://nodejs.cn/api/fs.html#fs_fs_readdir_path_options_callback "fs.readdir")读取目录下所有文件，会返回一个文件名数组，找到该目录下正在播放的文件的下标，通过数组下标判断前一首和后一首歌曲的名称，然后再组装成绝对路径，读取资源播放

        playFileList(diskPath, pos) {
          let isInFiles;
          let fileIndex;
          let preIndex;
          let nextIndex;
          let fullPath;
          let dirPath = path.dirname(diskPath);
          let basename = path.basename(diskPath);
          fs.readdir(dirPath, (err, files) => {
            isInFiles = files.includes(basename);

            if (isInFiles && pos === "pre") {
              fileIndex = files.indexOf(basename);
              preIndex = fileIndex - 1;
              fullPath = path.resolve(dirPath, files[preIndex]);

              this.loadMusic(fullPath);
            }
            if (isInFiles && pos === "next") {
              fileIndex = files.indexOf(basename);
              nextIndex = fileIndex + 1;
              fullPath = path.resolve(dirPath, files[nextIndex]);
              this.loadMusic(fullPath);
            }
          });
        },

### 声音音量控制
音量控制需要通过监听input的键入事件，获取到range的值，然后通过设置样式[background-image](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-image "background-image"),动态计算百分比，然后调用wavesurfer的setVolume方法调节音量

    :style="`background-image:linear-gradient( to right, ${fillColor}, ${fillColor} ${percent}, ${emptyColor} ${percent})`"

改变音量changeVol事件

        changeVol(e) {
          let val = e.target.value;
          let min = e.target.min;
          let max = e.target.max;
          let rate = (val - min) / (max - min);
          this.percent = 100 * rate + "%";
          console.log(this.percent, rate);
          this.wavesurfer.setVolume(Number(rate));
        },

### 自定义标题栏
个人觉得系统自带的菜单栏太丑了，就给设置了无边框再自己加上最小化，关闭的功能。最小化，关闭是通过ipc通信，渲染进程监听到有点击操作后，通知主进程进行相应的操作。

**渲染进程**

        close() {
          ipcRenderer.send("close");
        },
        minimize() {
          ipcRenderer.send("minimize");
        }

**主进程**

    ipcMain.on("close", () => {
      win.close();
      app.quit();
    });

    ipcMain.on("minimize", () => {
      win.minimize();
    });

### 打开多个实例的问题
在实际测试的过程中发现会出现，打开一首新的音乐播放，就会出现重新开一个实例的现象，不能实现覆盖播放，后面查阅资料发现electron有一个[second-instance](https://www.electronjs.org/docs/api/app#%E4%BA%8B%E4%BB%B6-second-instance "second-instance")事件，可以监听是否打开了第二个实例。当第二个实例被执行并且调用 [app.requestSingleInstanceLock()](https://www.electronjs.org/docs/api/app#apprequestsingleinstancelock "app.requestSingleInstanceLock()") 时，这个事件将在应用程序的首个实例中触发，并且会返回第二个实例的相关信息，然后通过主进程通知渲染进程，告知渲染进程第二个实例的本地绝对路径，渲染进程接收到信息后，立马加载第二个实例的资源。app.requestSingleInstanceLock()，表示应用程序实例是否成功取得了锁。 如果它取得锁失败，可以假设另一个应用实例已经取得了锁并且仍旧在运行，所以可以直接关闭掉，这样就避免了打开多个实例的问题

**主进程**

    const gotTheLock = app.requestSingleInstanceLock();
    if (gotTheLock) {
      app.on("second-instance", (event, commandLine, workingDirectory) => {
        // 监听是否有第二个实例，向渲染进程发送第二个实例的本地路径
        win.webContents.send("path", `${commandLine[commandLine.length - 1]}`);
        if (win) {
          if (win.isMinimized()) win.restore();
          win.focus();
        }
      });

      app.on("ready", async () => {
        createWindow();
      });
    } else {
      app.quit();
    }

**渲染进程**

      ipcRenderer.on("path", (event, arg) => {
        const newOriginPath = arg;

        // console.log(newOriginPath);
        this.loadMusic(newOriginPath);
      });

### 自动更新
需求的起因是，在很兴奋的给女朋友成品的时候，尴尬的被女朋友试出很多bug（捂脸ing）,然后频繁的修改打包，然后通过私发传给她。特别麻烦，所以这个需求很急迫。最后查了资料，通过electron-updater实现了这个需求.

**安装electron-updater**

    yarn add electron-updater

**发布设置**

        electronBuilder: {
          builderOptions: {
            publish: ['github']
          }
        }
**主进程监听**

    autoUpdater.on("checking-for-update", () => {});
    autoUpdater.on("update-available", info => {
      dialog.showMessageBox({
        title: "新版本发布",
        message: "有新内容更新，稍后将重新为您安装",
        buttons: ["确定"],
        type: "info",
        noLink: true
      });
    });

    autoUpdater.on("update-downloaded", info => {
      autoUpdater.quitAndInstall();
    });

**生成Github Access Token**
因为是用github作为更新站，所以本地需要相应的操作权限，去这里生成token，[戳这](https://github.com/settings/tokens "戳这")，生成后，在powershell中设置

    [Environment]::SetEnvironmentVariable("GH_TOKEN","<YOUR_TOKEN_HERE>","User")
    # 例如 [Environment]::SetEnvironmentVariable("GH_TOKEN","sdfdsfgsdg14463232","User")

**打包上传Github**

    yarn electron:build -p always

完成上面步骤后软件会自动上传打包后的文件到release,然后编辑下release就可以直接发布了，软件是基于版本号更新的，所以记得一定要改版本号

## 从零开始-结束
作为程序猿最开心的事莫过于得到女朋友的夸奖，虽然这是一个小程序，实现难度也不高，但是最后做出最小可用的版本呈现在女朋友面前的时候，看到女盆友感动的眼神，我想，这应该是我作为程序猿唯一感到欣慰的时候。软件还有很多可以改进的地方，源码在此，[戳这里Github](https://github.com/Kerinlin/localMusicPlayer "戳这里Github")