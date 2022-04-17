<template>
  <v-app class="">
    <div class="pa-3">
      <v-sheet class="">
        <h1 class="">Цифры</h1>
        <vue-drawing-canvas
          ref="VueCanvasDrawing"
          :image.sync="image"
          canvasId="VueCanvasDrawing"
          :width="width"
          :height="height"
          :lineWidth="line"
          saveAs="png"
          :styles="{
            border: 'solid 3px #000',
          }"
        />
      </v-sheet>

      <v-sheet class="my-2">
        <div class="mr-4">
          <v-btn
            color="primary"
            @click="$refs.VueCanvasDrawing.reset()"
            class="mr-2"
            ><v-icon>mdi-eraser</v-icon></v-btn
          >
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="recognize">Угадать</v-btn>
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="saveWeights()">Сохранить веса</v-btn>
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputWeights.click()"
            >Загрузить веса</v-btn
          >
          <input
            v-show="false"
            accept=""
            ref="inputWeights"
            type="file"
            @change="(e) => loadWeights(e)"
          />
        </div>
        <div class="mt-2">
          <v-btn color="primary" @click="$refs.inputUpload.click()"
            >Загрузить датасет</v-btn
          >
          <input
            multiple
            accept=".png"
            v-show="false"
            ref="inputUpload"
            type="file"
            @change="(e) => loadDataset(e)"
          />
        </div>
      </v-sheet>
    </div>
  </v-app>
</template>

<script>
import VueDrawingCanvas from "vue-drawing-canvas";
var _ = require("lodash");
export default {
  name: "App",
  components: {
    VueDrawingCanvas,
  },

  data: () => ({
    image: "",
    width: 100,
    height: 100,
    line: 8,
    imageArray: null,
    weightMatrix: [],
    numberOfNeurons: 10,
    learnSpeed: 0.4,
    errors: 0,
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
      let t0 = performance.now();
      let ctx = document.getElementById("VueCanvasDrawing").getContext("2d");
      let files = e.target.files;
      files = Object.values(files);
      files = this.shuffle(files);
      let vectorsAndAnswer = [];
      // console.log(files);

      const promise = files.map(
        (file) =>
          new Promise((resolve) => {
            const reader = new FileReader();
            let fileName = file.name;

            reader.onload = (e) => {
              let img = new Image();
              img.onload = () => {
                // this.width = img.width;
                // this.height = img.height;
                ctx.drawImage(img, 0, 0);
                const indexCorrect = Number(fileName[0]);
                let correctAnswers = Array(this.numberOfNeurons).fill(0);
                // this.imageData(this.width, this.height)
                correctAnswers[indexCorrect] = 1;
                vectorsAndAnswer.push({
                  answer: correctAnswers,
                  vector: this.imageData(this.width, this.height),
                });
                resolve();
              };
              img.src = e.target.result;
            };
            reader.readAsDataURL(file);
          })
      );
      await Promise.all(promise);

      let percentCorrect = 0;

      let epoch = 0;
      do {
        percentCorrect = this.TrainDataset(_.shuffle(vectorsAndAnswer));

        epoch++;

        console.log(percentCorrect);
      } while (percentCorrect < 100);

      const getSeconds = () => {
        let res = (performance.now() - t0) / 1000;
        return Number(res.toFixed(3));
      };

      console.log("Обучение завершено");
      this.$refs.VueCanvasDrawing.reset();
      console.table([
        ["Прошло эпох", epoch],
        ["Всего образов", vectorsAndAnswer.length],
        ["Всего ошибок", this.errors],
        ["Время обучения", getSeconds()],
      ]);
    },

    TrainDataset(vectorsAndAnswers) {
      for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
        this.Train(vectorsAndAnswers[i].vector, vectorsAndAnswers[i].answer);
      }
      let correctCount = 0;

      for (let i = 0, arrLen = vectorsAndAnswers.length; i < arrLen; i++) {
        let neuronOutput = this.Predict(vectorsAndAnswers[i].vector, true);

        let sumError = 0;
        for (let j = 0, outputLen = neuronOutput.length; j < outputLen; j++) {
          const error = neuronOutput[j] - vectorsAndAnswers[i].answer[j];
          sumError += error;
        }
        if (sumError == 0) correctCount++;
      }
      console.log(vectorsAndAnswers.length - correctCount + " ошибок");
      this.errors += vectorsAndAnswers.length - correctCount;
      let correctPercent = (correctCount / vectorsAndAnswers.length) * 100;
      return correctPercent;
    },

    // Расчет дельты и корректировка весов
    Train(vector, correctAnswers) {
      let answer = this.Predict(vector, true);

      for (
        let i = 0, neuronLen = this.weightMatrix.length;
        i < neuronLen;
        i++
      ) {
        let weightsDelta = correctAnswers[i] - answer[i];

        for (
          let j = 0, weightsLen = this.weightMatrix[i].length;
          j < weightsLen;
          j++
        ) {
          if (vector[j] === 1 && weightsDelta != 0) {
            this.weightMatrix[i][j] += this.learnSpeed * weightsDelta;
          }
        }
      }
    },

    // Сумматор + функция активации
    Predict(vector, isLearning) {
      let neuronSums = [];

      for (
        let i = 0, neuronLen = this.weightMatrix.length;
        i < neuronLen;
        i++
      ) {
        let sum = 0;
        for (
          let j = 0, weightsLen = this.weightMatrix[i].length;
          j < weightsLen;
          j++
        ) {
          sum += vector[j] * this.weightMatrix[i][j];
        }
        if (isLearning) {
          neuronSums.push(this.threshold(sum));
        } else neuronSums.push(sum);
      }
      return neuronSums;
    },

    threshold(sum) {
      return sum >= 0 ? 1 : 0;
    },

    imageData(width, height) {
      let context = document
        .getElementById("VueCanvasDrawing")
        .getContext("2d")
        .getImageData(0, 0, width, height).data;
      let array = Array.from(context);
      let newArray = array.filter((_, i) => i % 4 == 0);
      for (let i = 0; i < newArray.length; i++) {
        if (newArray[i] == 255) {
          newArray[i] = 0;
        } else newArray[i] = 1;
      }
      return newArray;
    },

    initWeights(canvasSize, numberOfNeurons, max, min) {
      let weightRes = [];
      for (let i = 0; i < numberOfNeurons; i++) {
        let weights = [];
        for (let j = 0; j < canvasSize ** 2; j++) {
          let randomNumber = Math.random() * (max - min) + min;
          weights.push(randomNumber);
        }
        weightRes.push(weights);
      }
      this.weightMatrix = weightRes;
    },

    recognize() {
      let imageVector = this.Predict(
        this.imageData(this.width, this.height),
        false
      );
      if (!imageVector.some((x) => x > 0)) return alert("Не распознано");
      console.log(imageVector);
      let max = Math.max(...imageVector);
      alert(imageVector.indexOf(max));
    },

    saveWeights() {
      const a = document.createElement("a");
      let data = JSON.stringify(this.weightMatrix);
      const file = new Blob([data], { type: "application/json" });

      a.href = URL.createObjectURL(file);
      a.download = "weights.json";
      a.click();

      URL.revokeObjectURL(a.href);
    },

    loadWeights(e) {
      const file = e.target.files[0];
      let reader = new FileReader();
      reader.readAsText(file);
      let context = this;

      reader.onload = function () {
        let res = reader.result;
        context.weightMatrix = JSON.parse(res);
        console.log("Веса загружены");
        console.log(context.weightMatrix);
      };

      reader.onerror = function () {
        console.log(reader.error);
        return;
      };
    },
  },

  mounted() {
    this.initWeights(this.width, this.numberOfNeurons, 0.3, -0.3);
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
}
</style>
