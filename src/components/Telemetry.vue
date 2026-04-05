<template>
  <div class="telemetry-view">
    <div class="header">
      <span class="status-dot">●</span> ROV-1 | MISSION CTRL
      <span class="battery">□ 98%</span>
    </div>

    <div class="tabs">
      <button class="tab" @click="$emit('switch','control')">
        <span class="icon">🎮</span> CONTROL
      </button>

      <button class="tab active">
        <span class="icon">📊</span> TELEMETRY
      </button>
    </div>

    <!-- PWM CHART -->
    <div class="chart-card">
      <div class="chart-title">PWM SPEED – LIVE</div>

      <Line
        ref="pwmChart"
        :data="pwmChartData"
        :options="pwmChartOptions"
        height="220"
      />
    </div>

    <!-- DISTANCE CHART -->
    <div class="chart-card">
      <div class="chart-title">ULTRASONIC DISTANCE (CM)</div>

      <Line
        ref="distanceChart"
        :data="distanceChartData"
        :options="distanceChartOptions"
        height="220"
      />

      <div class="chart-note">
        Red dashed line = obstacle alert threshold (30cm)
      </div>
    </div>

    <div class="footer-bar">
      ↑↓←→ Drive · SPACE Stop
    </div>
  </div>
</template>

<script>
import { Line } from "vue-chartjs"

import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Tooltip,
  Legend
} from "chart.js"

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Tooltip,
  Legend
)

export default {

  name: "Telemetry",
  components: { Line },

  data() {

    return {
        maxPoints: 60,
      timeLabels: [],
      pwmValues: [],
      distanceValues: [],

      pwmChartData: {
        labels: [],
        datasets: [{
          label: "PWM",
          data: [],
          borderColor: "#00aaff",
          backgroundColor: "rgba(0,170,255,0.18)",
          borderWidth: 3,
          pointRadius: 4,
          tension: 0.3,
          fill: true
        }]
      },

      pwmChartOptions: {
        responsive: true,
        maintainAspectRatio: false,

        plugins: {
          legend: { display:false }
        },

        scales: {

          x:{
            grid:{ color:"#2a354d"},
            ticks:{ color:"#8a96a8"}
          },

          y:{
            min:0,
            max:255,
            grid:{ color:"#2a354d"},
            ticks:{ color:"#8a96a8"}
          }

        }
      },

      distanceChartData:{
        labels:[],
        datasets:[

          {
            label:"Distance",
            data:[],
            borderColor:"#ff3b30",
            backgroundColor:"rgba(255,59,48,0.15)",
            borderWidth:3,
            pointRadius:4,
            tension:0.3,
            fill:true
          },

          {
            label:"Threshold",
            data:[],
            borderColor:"#ff3b30",
            borderDash:[6,4],
            borderWidth:2,
            pointRadius:0,
            fill:false
          }

        ]
      },

      distanceChartOptions:{
        responsive:true,
        maintainAspectRatio:false,

        plugins:{
          legend:{display:false}
        },

        scales:{

          x:{
            grid:{color:"#2a354d"},
            ticks:{color:"#8a96a8"}
          },

          y:{
            min:0,
            max:400,
            grid:{color:"#2a354d"},
            ticks:{color:"#8a96a8"}
          }

        }
      }

    }

  },

  mounted(){
    this.startSimulation()
  },

  beforeUnmount(){
    clearInterval(this.simInterval)
  },

  methods:{

   addDataPoint(time, pwm, dist) {

  this.timeLabels.push(time)
  this.pwmValues.push(pwm)
  this.distanceValues.push(dist)

  // keep only last MAX_POINTS
  if (this.timeLabels.length > this.maxPoints) {
    this.timeLabels.shift()
    this.pwmValues.shift()
    this.distanceValues.shift()
  }

  // update charts
  this.pwmChartData = {
    labels: [...this.timeLabels],
    datasets: [{
      ...this.pwmChartData.datasets[0],
      data: [...this.pwmValues]
    }]
  }

  this.distanceChartData = {
    labels: [...this.timeLabels],
    datasets: [
      {
        ...this.distanceChartData.datasets[0],
        data: [...this.distanceValues]
      },
      {
        ...this.distanceChartData.datasets[1],
        data: new Array(this.timeLabels.length).fill(30)
      }
    ]
  }

},


   startSimulation() {

  this.simInterval = setInterval(() => {

    const now = new Date()

    const mm = now.getMinutes().toString().padStart(2,"0")
    const ss = now.getSeconds().toString().padStart(2,"0")

    const time = `${mm}:${ss}`

    // Example values (replace later with ESP32 data)
    const pwm = Math.floor(Math.random() * 255)
    const dist = Math.floor(Math.random() * 200)

    this.addDataPoint(time, pwm, dist)

  }, 2000) // every 2 seconds
}

  }

}
</script>

<style scoped>

.telemetry-view{
font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;
background:#0a0e17;
color:#d0d8e0;
min-height:100vh;
padding:12px 16px;
display:flex;
flex-direction:column;
}

.header{
display:flex;
justify-content:space-between;
align-items:center;
font-size:1.1rem;
font-weight:600;
margin-bottom:8px;
}

.status-dot{
color:#ff3b30;
margin-right:6px;
}

.battery{
color:#34c759;
}

.tabs{
display:flex;
gap:8px;
margin:12px 0;
}

.tab{
flex:1;
background:#1c2333;
border:none;
color:#8a96a8;
padding:10px 16px;
border-radius:8px;
font-weight:500;
display:flex;
align-items:center;
justify-content:center;
gap:6px;
}

.tab.active{
background:#0066ff;
color:white;
}

.chart-card{
background:#141b2b;
border-radius:12px;
padding:16px;
margin-bottom:20px;
height:240px;
}

.chart-title{
font-size:1.1rem;
font-weight:600;
margin-bottom:12px;
}

.chart-note{
margin-top:8px;
font-size:0.85rem;
color:#8a96a8;
text-align:center;
}

.footer-bar{
margin-top:auto;
padding:12px;
text-align:center;
color:#5a6b8c;
border-top:1px solid #2a354d;
}

</style>