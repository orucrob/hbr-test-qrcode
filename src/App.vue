<template>
  <div id="app">
    <video autoplay ref="vid"></video>
    <canvas v-show="showImage" ref="can" ></canvas>
    <div class="content">
      <select id="videoSource" @change="videoChange"></select>
      <div>{{qrData}}</div>
      <div>{{info}}</div>
      <button @click="start">Start</button>
      <button @click="stop">Stop</button>
    </div>
  </div>
</template>

<script>
import jsQR from 'jsqr'
/*global navigator*/

export default {
  name: 'app',
  data: () => {
    return {
      info: "no info",
      stream: undefined,//undefined when not streaming
      qrData: undefined,
      showImage: false,
      stopped: true, //false when snapping
      deviceIds : []
    }
  },
  mounted: async function(){
    //fill list of devices
    await this.updateDevicesList()
    //make first device active
    if(this.deviceIds.length>0){
      this.startStream(this.deviceIds[0])
    }
  },
  methods: {
    /**
     * Start trying to get qr code from camera input
     */
    async snap(){
      //only if stream is working
      if(this.stopped) return;
      
      //facts
      let me = this
      let w = this.$refs['vid'].videoWidth
      let h = this.$refs['vid'].videoHeight
      this.info = w+'x'+h
      let can = this.$refs['can'] //document.createElement('canvas');
      let ctx = can.getContext('2d')

      //update canvas size to get the size of video stream
      can.width = w
      can.height = h
      
      //draw current frame
      ctx.drawImage(this.$refs['vid'], 0, 0, w, h)
      
      //get image from canvas (TODO: can we do it directly from vid stream?)
      let imageData = ctx.getImageData(0, 0, w, h)

      //try to find the code in image snapshot
      const code = jsQR(imageData.data, w, h)

      if (code) {
        //well done, we found code -> freeze image (=show canvas) for 2s and display info
        this.showImage = true
        this.qrData = code.data
        setTimeout(()=>{
          me.showImage = false
          me.snap() //continue
        }, 2000)
      }else{
        //unable to detect qr code in image
        this.qrData = 'no code'
        setTimeout(()=>{
          me.snap() //try again in .5 sec
        }, 500)
      }
    },
    /**
     * start reading snapping (reading) qr code  
     */
    async start(){
      //start stream if stopped
      if(!this.stream && this.deviceIds.length>0){
        this.startStream(this.deviceIds[0])
      }
      //start snapping
      if(this.stopped){
        this.stopped = false
        this.snap()
      }
    },
    /**
     * stop streaming camera and snapping the code
     */
    async stop(){
      //stop snapping
      this.stopped = true
      
      //stop streaming
      if(this.stream){
        this.stream.getTracks().forEach(track => track.stop())
      } 
      this.stream = undefined

    },
    /**
     * handle video source change. Start the stream (not snapping) when video changed. 
     */
    async videoChange(event){
      let deviceId = event.target.value
      this.startStream(deviceId)
    },
    /**
     * start streaming from device 
     * KEY FEATURE: here is the "new browsers magic": navigator.mediaDevices.getUserMedia()
     */
    async startStream(deviceId){
      this.stop()
      this.stream = await navigator.mediaDevices.getUserMedia({
            audio: false,
            video: {deviceId:{exact: deviceId }}
        })
      this.$refs['vid'].srcObject = this.stream
    },
    /**
     * fill the list of videoinputs of this device
     * KEY FEATURE: here is the "new browsers magic": navigator.mediaDevices.enumerateDevices()
     */
    async updateDevicesList(){
      const videoSelect = document.querySelector('select#videoSource')
      console.log('videoSelect', videoSelect)

      let deviceInfos = await navigator.mediaDevices.enumerateDevices()
      for (let i = 0; i !== deviceInfos.length; ++i) {
        const deviceInfo = deviceInfos[i];
        if (deviceInfo.kind === 'audioinput') {
          //skipping - we do not care about audio
        } else if (deviceInfo.kind === 'videoinput') {
          const option = document.createElement('option')
          option.value = deviceInfo.deviceId
          option.text = deviceInfo.label || 'camera ' + (videoSelect.length + 1)
          videoSelect.appendChild(option)
          this.deviceIds.push(deviceInfo.deviceId)
        } else {
          console.log('Found another kind of device: ', deviceInfo);
        }
      }
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  border:1px solid black;
}

.content {
  position: fixed;
  /*bottom: 0;*/
  background: rgba(0, 0, 0, 0.5);
  color: #f1f1f1;
  /*width: 100%;*/
  padding: 20px;
}

video, canvas {
  position: fixed;
  left: 0;
  top: 0;
}
</style>
