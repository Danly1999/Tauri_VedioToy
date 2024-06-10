<template>
  <div>
    <div style="display: flex; justify-content: center;">
      <img src="./FunctionCardIconPNG.png" alt="" style="width:30% ;height: auto;opacity: 0.1;position:absolute;z-index:-10;margin-left: 100px;">
    </div>
    
    路径选择：
    <div style="margin-bottom: 5px;">
      <input disabled type="text" :value="selectedFilePath"  style="margin-right: 20px;"/>
      <button @click="selectFile" style="margin-right: 5px;">选源视频</button>
      源比特率:{{videoBitrate}}kb/s, 源帧率:{{ frameRate }}fps
    </div>

    <div>
      <input disabled type="text" :value="SavedFilePath"  style="margin-right: 20px;"/>
      <button @click="selectOutputPath">输出路径</button>
    </div>


    <br>
    参数设置：
    <div v-if="selectedFormat != 'gif'" style="margin-bottom: 5px;">
      目标比特率: <input  type="number" v-model="OutputVideoBitrate" style="width: 60px;"/>
      目标帧率: <input  type="number" v-model="OutputFrameRate" style="width: 60px;"/>
    </div>
    <div v-if="selectedFormat === 'gif'" style="margin-bottom: 5px;">
      时长(秒): <input  type="number" v-model="gifTime" style="width: 60px;"/>
    </div>
    <select v-model="selectedFormat" style="margin-right: 20px;" @change="Format">
      <option value="mp4">MP4</option>
      <option value="webm">WEBM</option>
      <option value="avi">AVI</option>
      <option value="gif">GIF</option>
    </select>
    <!-- <progress :value="progress" max="100"></progress>
    <div v-if="transcoding">正在转码...</div>
    <div v-else>转码完成</div> -->
    
    <button @click="transcodeVideo">开始转换</button>
    <br>
    <progress :value="progress" max="100" style="width: 350px;"></progress>
    <br>
    <div v-if="transcoding">正在转码...</div>
    <div v-else-if="progress === 100">转码完成</div>
  </div>
</template>

<script>
import { Command } from '@tauri-apps/api/shell';
import { open, save  } from '@tauri-apps/api/dialog';
import { exists, removeFile } from '@tauri-apps/api/fs';

export default {
  data() {
    return {
      progress: 0,
      transcoding: false,
      selectedFilePath: '',
      SavedFilePath: '',
      outputFileName: 'output',
      selectedFormat: 'mp4',
      OutputVideoBitrate: 0,
      videoBitrate: 0,
      OutputFrameRate: 0,
      frameRate: 0,
      gifTime: 1
    };
  },

  methods: {
    Format()
    {
      if (this.SavedFilePath) {
        const extension = this.SavedFilePath.match(/\.[^.]+$/)[0];
        this.SavedFilePath = this.SavedFilePath.replace(extension,`.${this.selectedFormat}`);
      }
    },
    async selectFile() {
      const selected = await open({
        filters: [
          {
            name: 'Video Files',
            extensions: ['avi', 'mp4', 'webm']
          }
        ]
      });
      if (selected) {
        // 获取文件路径
          this.selectedFilePath = selected;
          const command = Command.sidecar('bin/ffmpeg', [
              '-i',
              this.selectedFilePath,
          ]);
          const output = await command.execute();
          var bitrateMatch = output.stderr.match(/(\d+)\s+kb\/s,/g);
          this.videoBitrate = parseInt(bitrateMatch[0]);
          bitrateMatch = output.stderr.match(/(\d+)\s+fps,/g);
          this.frameRate = parseInt(bitrateMatch[0]);
        }
      },
      async selectOutputPath()
      {
        const savedPath = await save({
          defaultPath: this.outputFileName,
          filters: [
            {
              name: 'Video Files',
              extensions: [this.selectedFormat]
            }
          ]
        });
        this.SavedFilePath = savedPath;
        this.OutputVideoBitrate = parseInt(this.videoBitrate);
        this.OutputFrameRate = parseInt(this.frameRate);
        console.log(this.OutputVideoBitrate);
        console.log(this.OutputFrameRate);
      },
      async transcodeVideo() {
        if(this.SavedFilePath === null || this.SavedFilePath === '')
        {
          alert(`请选择输出路径`);
          return;
        }
        if(this.selectedFilePath === null || this.selectedFilePath === '')
        {
          alert(`请选择需要转换的文件`);
          return;
        }
        try {
          if (await exists(this.SavedFilePath)) 
          {
            await removeFile(this.SavedFilePath);
          }
        } catch (error) {
          alert("删了目标同名文件再来,本程序拿不到删除的权限");
        }
        var command;
        if(this.selectedFormat === "gif")
        {
           command = Command.sidecar
          ('bin/ffmpeg', [
            '-i', this.selectedFilePath,
            '-t', `${this.gifTime}`,
            this.SavedFilePath,
          ]);
        }else
        {
           command = Command.sidecar
          ('bin/ffmpeg', [
            '-i', this.selectedFilePath,
            '-b:v', `${this.OutputVideoBitrate}k`,
            '-r', `${this.OutputFrameRate}`,
            this.SavedFilePath,
          ]);
        }
        console.log(command);
      // 监听命令的输出
      command.stderr.on('data', (data) => {
        // 从输出中解析进度信息
        const progressMatch = data.match(/frame=\s*(\d+)/);
        if (progressMatch) {
          // 解析当前帧数并更新进度条
          const frame = parseInt(progressMatch[1], 10);
          const totalFrames = 1000; // 假设总帧数为1000
          this.progress = (frame / totalFrames) * 100;
        }
      });

      // 等待命令执行完成
      this.transcoding = true;
      await command.execute();

      // 转码完成，更新状态
      this.transcoding = false;
      this.progress = 100;
    }
  }
};
</script>
