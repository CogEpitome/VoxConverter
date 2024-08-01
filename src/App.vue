<template>
  <img alt="Vue logo" src="./assets/logo.png">
  <h1>Jonas fulhackade vox-till-mp3-konverterare</h1>

  <div class="wrapper">
    <div class="fileupload">
      <input type="file" @change="onFileChange" />
      <p v-if="selectedFile">{{ selectedFile.name }}</p>      
      <p v-else>Select a .vox file</p>      
    </div>
    <div class="fileconversion">
      <button @click="convertFile" v-if="selectedFile">Convert</button>
    </div>
    <div class="filedownload">
      <audio ref="audioPlayer" v-if="audioSrc" :src="audioSrc" @canPlay="canPlay" controls></audio> 
      <button @click="downloadFile" v-if="audioSrc">Download</button>
    </div>
  </div>
</template>

<script>
import { FFmpeg } from '@ffmpeg/ffmpeg'
// toBlobURL avoids some CORS-issues that may arise using URL.[...]
import { fetchFile, toBlobURL } from '@ffmpeg/util'

export default {
  name: 'App',
  data() {
    return {      
      ffmpeg: new FFmpeg(), // Create instance of FFmpeg
      selectedFile: null, // Store uploaded .vox file
      audioSrc: null, // Save result
    }
  },
  methods: {
    // Used by audio player to preview mp3
    playAudio() {
      this.$refs.audioPlayer.play();
    },
    // Used by audio player to preview mp3
    canPlay() {
      return this.audioSrc != null;
    },

    // Called when a .vox is uploaded
    async onFileChange(event) {
      this.selectedFile = event.target.files[0];
    },
    // Does the conversion by loading the vox into ffmpeg's makeshift file system, converting, and reading the new file back.
    async convertFile() {
      // URL stuff loads some dependencies. Could probably install these via npm and rely on local code only.
      const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/umd';         
      await this.ffmpeg.load(
        {
            coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
            wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
        }
      );

      const fileName = this.selectedFile.name; 
      // Replace extension
      const outputFileName = this.selectedFile.name.replace(`.${this.selectedFile.name.split('.').pop()}`, '.mp3');           

      // Writes vox file to ffmpeg local file system
      // ffmpeg creates a fake file system in the clientside in order to be able to do conversions, very nice
      await this.ffmpeg.writeFile(fileName, await fetchFile(this.selectedFile));

      // Long list of arguments here, most of which I had to guess at.
      // Since vox files don't have headers, we need to give ffmpeg all the required info.
      // You may experiment with better settings here to make the resulting mp3 sound nicer, I think I'm slightly off somewhere.
      await this.ffmpeg.exec(['-f', 'u8', '-c', 'adpcm_ima_oki', '-ar', '8.0k', '-ac', '1', '-i', fileName, outputFileName]);
      
      // Load the mp3
      const data = await this.ffmpeg.readFile(outputFileName);

      // Create a url for download, and audio player preview
      const url = URL.createObjectURL(new Blob([data.buffer], { type: 'audio/mp3'}));
      this.audioSrc = url;        
    },
    // Does what it says
    downloadFile() {
      const outputFileName = this.selectedFile.name.replace(`.${this.selectedFile.name.split('.').pop()}`, '.mp3');
      const link = document.createElement('a');
      link.href = this.audioSrc;
      link.download = outputFileName;
      document.body.appendChild(link);
      link.click();
      link.remove();
    },
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;  
}

.fileupload {
  width: 300px;
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: rgb(233, 233, 233);  
  padding: 12px;
  border-radius: 12px;
}

.fileconversion {
  width: 300px;
  display: flex;
  flex-direction: column;
  align-items: center;  
  background-color: rgb(233, 233, 233);  
  padding: 12px;
  border-radius: 12px;
}

.filedownload {
  width: 300px;
  display: flex;
  flex-direction: column;
  align-items: center;  
  background-color: rgb(233, 233, 233);  
  padding: 12px;
  border-radius: 12px;
}

::file-selector-button,
button {
  background-color: #04AA6C;
  border: none;
  color: white;
  padding: 8px;
  width :100%;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 16px;
  transition-duration: 1s;
}
::file-selector-button:hover,
button:hover {
  background-color: #016d45;
  color: white;
}
::file-selector-button:disabled,
button:disabled {
  background-color: #92a8a0;
}

p {
  font-size: 16px;
}

audio,
input {
  width: 100%;

}

audio {
  padding-bottom: 10px;
}
</style>
