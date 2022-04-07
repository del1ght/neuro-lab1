<template>
  <v-app class="">
    <div class="pa-3">

      <v-sheet class="">
        <h1 class="">Рисовалка</h1>
        <vue-drawing-canvas
          ref="VueCanvasDrawing"
          :image.sync="image"
          canvasId="VueCanvasDrawing"
          :width="width"
          :height="height"
          backgroundColor="#fff0"
          :background-image="backgroundImage"
          :lineWidth="line"
          saveAs="png"
          :styles="{
            border: 'solid 3px #000',
          }"
        />
      </v-sheet>

      <v-sheet class="my-2">
        <div class="mr-4">
          <v-btn color="primary" @click="$refs.VueCanvasDrawing.reset();" class="mr-2"><v-icon>mdi-eraser</v-icon></v-btn>
        </div>
        <!-- <div class="mr-4 my-2">
          <v-btn color="primary" @click="setCross(); recalculateWeights()" class="mr-2">Крестик </v-btn>
          <v-btn color="primary" @click="setCircle(); recalculateWeights()">Нолик</v-btn>
        </div> -->
        <!-- <div class="">
          <v-btn color="primary" @click="save(weightMatrix)">Сохранить веса</v-btn>
        </div> -->
        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()">Загрузить датасет</v-btn> 
          <input multiple accept=".png" v-show="false" ref="inputUpload" type="file" id="load" @change="(e) => loadDataset(e)">
        </div>
      </v-sheet>

    </div>
  </v-app>
</template>

<script>
import VueDrawingCanvas from 'vue-drawing-canvas';

export default {
  name: 'App',
  components: {
    VueDrawingCanvas
  },

  data: () => ({
    image: "",
    width: 150,
    height: 150,
    line: 25,
    imageArray: null,
    backgroundImage: null,
  }),

  methods: {
    shuffle(arr = []) {
    let newArr = [];

      for (let i = arr.length - 1; i >= 0; i--) {
        let id = Math.floor(Math.random() * arr.length);
        newArr.push(arr[id]);
        arr.splice(id, 1);
      }

      return newArr;
    },

    async loadDataset(e) {
      let ctx = document.getElementById('VueCanvasDrawing').getContext('2d')
      let files = e.target.files;
      files = Object.values(files);
      files = this.shuffle(files);
      console.log(files)

      const promise = files.map(
      (file) =>
      new Promise((resolve) => {
        const reader = new FileReader();
        let fileName = file.name;

        reader.onload = (e) => {
          let img = new Image();
          img.onload = () => {
            this.width = img.width;
            this.height = img.height;
            ctx.drawImage(img, 0, 0);

            resolve();
          };
          img.src = e.target.result;
        };
        reader.readAsDataURL(file);
      })
  );
      await Promise.all(promise);
      console.log()

    },

    // async setImage(event, numberOfFiles) {
    //   for (let i = 0; i < numberOfFiles.length; i++) {
    //     this.backgroundImage = URL.createObjectURL(event.target.files[i]);
    //     console.log(this.backgroundImage)
    //     await this.$refs.VueCanvasDrawing.redraw();
    //   }
    // },

    imageData(width, height){
      let context = document.getElementById('VueCanvasDrawing').getContext('2d').getImageData(0, 0, width, height).data
      let array = Array.from(context)
      array.splice(0,3)
      let newArray = array.filter((_,i) => i % 4 == 0)
      for (let i = 0; i < newArray.length; i++) {
        if (newArray[i] > 0){
          newArray[i] = 1
        }
      }
      console.log(newArray)
      return newArray
    },

    async resetBg(){
      this.backgroundImage = null
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
}
</style>
