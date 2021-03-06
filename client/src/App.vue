<template>
  <div id="app">
    <h1>{{ data.msg }}</h1>
    <h2> Socket connected: {{ data.connected }}</h2>
    <h2 v-if="data.err">Socket error: {{ data.err }}</h2>
    <div
      id="chart-container"
      align="center"
    >
      <canvas ref="chartCanvas"/>
    </div>
  </div>
</template>

<script>
const Chart = require('chart.js');
const { initializeSocket } = require('./socket');
const { buildModel } = require('./model');

const data = {
  msg: 'Waiting for messages from server...',
  err: undefined,
  connected: false
};

function drawChart(el) {
  const ctx = el.getContext('2d');

  const pointDataSet = {
    label: 'Received data',
    data: [],
    pointRadius: 5,
    pointBackgroundColor: '#0000ff',
    showLine: false,
    xAxisID: 'x-axis',
    yAxisID: 'y-axis'
  };

  const predictionDataSet = {
    label: 'Predicted data',
    data: [],
    fill: false,
    borderColor: '#ff0000',
    xAxisID: 'x-axis',
    yAxisID: 'y-axis'
  };

  const lineData = {
    datasets: [
      pointDataSet,
      predictionDataSet
    ]
  };

  const options = {
    maintainAspectRatio: false,
    scales: {
      xAxes: [{
        id: 'x-axis',
        display: true,
        type: 'linear'
      }],
      yAxes: [{
        id: 'y-axis',
        display: true,
        type: 'linear',
      }],
    }
  };

  return new Chart(ctx, {
    type: 'line',
    data: lineData,
    options
  });
}

const onConnect = () => {
  data.connected = true;
};

const onError = (err) => {
  console.err('Received socket error', err);
  data.err = err.message;
};

const onDisconnect = () => {
  console.log('Socket disconnected');
  data.connected = false;
};

let chart;

const model = buildModel();

export default {
  name: 'App',
  data() {
    return {
      data,
      chart
    };
  },
  created() {
    initializeSocket({
      onDataPoint: this.onDataPointReceived,
      onConnect,
      onDisconnect,
      onError
    });
  },
  mounted() {
    this.chart = drawChart(this.$refs.chartCanvas);
  },
  methods: {
    async onDataPointReceived(dataPoint) {
      console.log(`Received data point ${JSON.stringify(dataPoint)}`);
      const { x, y } = dataPoint;
      this.addData(x, y);
      const { a, b } = model.fit({ x, y });
      // console.log(`a: ${a}, b: ${b}`);
      this.updatePrediction(a, b);
    },
    addData(x, y) {
      this.chart.data.datasets[0].data.push({ x, y });
      this.chart.update();
    },
    updatePrediction(a, b) {
      const xBound = 5;
      const x1 = -xBound;
      const x2 = xBound;
      const toY = x => a * x + b;
      this.chart.data.datasets[1].data = [{ x: x1, y: toY(x1) },
        { x: x2, y: toY(x2) }];
      this.chart.update();
    }
  }
};
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

#chart-container {
  width: 60%;
  height: 500px;
  text-align: center;
  margin-left: auto;
  margin-right: auto;
}

h1, h2 {
  font-weight: normal;
}

</style>
