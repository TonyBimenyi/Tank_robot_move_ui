<template>
  <div class="telemetry-view">
    <MissionHeader
      title="ROV-1"
      subtitle="Mission Control · Telemetry"
      :robotIP="robotIP"
      :batteryPercent="batteryLevel"
    />

    <MissionTabs active="telemetry" @switch="$emit('switch', $event)" />

    <div class="toolbar">
      <button class="tool-btn" type="button" @click="showSettings = !showSettings">
        <span class="tool-icon" aria-hidden="true">
          <svg viewBox="0 0 24 24" fill="none">
            <path
              d="M12 15.25a3.25 3.25 0 1 0 0-6.5 3.25 3.25 0 0 0 0 6.5Z"
              stroke="currentColor"
              stroke-width="1.8"
            />
            <path
              d="M19.4 13.03c.04-.34.06-.69.06-1.03s-.02-.69-.06-1.03l1.53-1.19a.8.8 0 0 0 .19-1.03l-1.45-2.51a.8.8 0 0 0-.96-.35l-1.8.72a7.93 7.93 0 0 0-1.78-1.03l-.28-1.92A.8.8 0 0 0 13.06 2h-2.9a.8.8 0 0 0-.79.67l-.28 1.92c-.63.25-1.22.6-1.78 1.03l-1.8-.72a.8.8 0 0 0-.96.35L2.1 7.76a.8.8 0 0 0 .19 1.03l1.53 1.19c-.04.34-.06.69-.06 1.03s.02.69.06 1.03L2.29 14.2a.8.8 0 0 0-.19 1.03l1.45 2.51c.2.35.62.5.96.35l1.8-.72c.56.43 1.15.78 1.78 1.03l.28 1.92c.06.39.4.68.79.68h2.9c.39 0 .73-.29.79-.68l.28-1.92c.63-.25 1.22-.6 1.78-1.03l1.8.72c.34.15.76 0 .96-.35l1.45-2.51a.8.8 0 0 0-.19-1.03l-1.53-1.19Z"
              stroke="currentColor"
              stroke-width="1.4"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </span>
        <span>Settings</span>
      </button>

      <div class="link-pill" :class="linkStateClass">
        <span class="dot" aria-hidden="true"></span>
        <span>{{ linkText }}</span>
      </div>
    </div>

    <div v-if="showSettings" class="settings-card">
      <div class="settings-grid">
        <label class="field">
          <span class="field-label">Robot IP</span>
          <input v-model.trim="robotIP" class="field-input" type="text" inputmode="url" />
        </label>

        <label class="field">
          <span class="field-label">Telemetry</span>
          <select v-model="mode" class="field-input">
            <option value="sim">Simulation</option>
            <option value="live">Live (/telemetry)</option>
          </select>
        </label>

        <label class="field">
          <span class="field-label">Update (ms)</span>
          <input v-model.number="pollIntervalMs" class="field-input" type="number" min="200" step="100" />
        </label>

        <label class="field">
          <span class="field-label">Max points</span>
          <input v-model.number="maxPoints" class="field-input" type="number" min="10" max="600" step="10" />
        </label>

        <label class="field">
          <span class="field-label">Threshold (cm)</span>
          <input v-model.number="distanceThreshold" class="field-input" type="number" min="1" max="400" step="1" />
        </label>

        <button class="tool-btn secondary" type="button" @click="paused = !paused">
          <span class="tool-icon" aria-hidden="true">
            <svg v-if="paused" viewBox="0 0 24 24" fill="none">
              <path d="M8.5 6.5v11l9-5.5-9-5.5Z" stroke="currentColor" stroke-width="2" stroke-linejoin="round"/>
            </svg>
            <svg v-else viewBox="0 0 24 24" fill="none">
              <path d="M8 6.5v11M16 6.5v11" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
            </svg>
          </span>
          <span>{{ paused ? "Resume plotting" : "Pause plotting" }}</span>
        </button>

        <button class="tool-btn secondary" type="button" @click="clearSeries">
          <span class="tool-icon" aria-hidden="true">
            <svg viewBox="0 0 24 24" fill="none">
              <path d="M6.5 7.5h11M10 7.5v-2h4v2" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
              <path d="M9 10v7M12 10v7M15 10v7" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
              <path d="M7.5 7.5l.8 13a1.2 1.2 0 0 0 1.2 1.1h5a1.2 1.2 0 0 0 1.2-1.1l.8-13" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
            </svg>
          </span>
          <span>Clear</span>
        </button>
      </div>
    </div>

    <div class="stats">
      <div class="stat-card">
        <div class="stat-label">PWM</div>
        <div class="stat-value">{{ Math.round(pwm) }}</div>
        <div class="stat-meta">Current</div>
      </div>

      <div class="stat-card">
        <div class="stat-label">Target</div>
        <div class="stat-value">{{ Math.round(pwmTarget) }}</div>
        <div class="stat-meta">Requested</div>
      </div>

      <div class="stat-card">
        <div class="stat-label">Obstacle</div>
        <div class="stat-value">{{ latestDistanceDisplay }}</div>
        <div class="stat-meta">cm · {{ modeLabel }}</div>
      </div>

      <div class="stat-card wide">
        <div class="stat-label">Distance covered</div>
        <div class="stat-value">{{ latestOdometerDisplay }}</div>
        <div class="stat-meta">m · parcours</div>
      </div>
    </div>

    <!-- PWM CHART -->
    <div class="chart-card">
      <div class="chart-head">
        <div class="chart-title">PWM Speed</div>
        <div class="badge">{{ paused ? "PAUSED" : "LIVE" }}</div>
      </div>

      <div class="chart-body">
        <Line
          ref="pwmChart"
          chart-id="pwmChart"
          :data="pwmChartData"
          :options="pwmChartOptions"
        />
      </div>
    </div>

    <!-- DISTANCE CHART -->
    <div class="chart-card">
      <div class="chart-head">
        <div class="chart-title">Ultrasonic Distance</div>
        <div class="badge warn">{{ distanceThreshold }}cm threshold</div>
      </div>

      <div class="chart-body">
        <Line
          ref="distanceChart"
          chart-id="distanceChart"
          :data="distanceChartData"
          :options="distanceChartOptions"
        />
      </div>

      <div class="chart-note">
        Red dashed line = obstacle alert threshold ({{ distanceThreshold }}cm)
      </div>
    </div>

    <div class="chart-card">
      <div class="chart-head">
        <div class="chart-title">Distance Covered (Parcours)</div>
        <div class="badge">{{ modeLabel.toUpperCase() }}</div>
      </div>

      <div class="chart-body">
        <Line
          ref="odometerChart"
          chart-id="odometerChart"
          :data="odometerChartData"
          :options="odometerChartOptions"
        />
      </div>
    </div>

    <div class="footer-bar">
      ↑↓←→ Drive · SPACE Stop
    </div>
  </div>
</template>

<script>
import MissionHeader from "./MissionHeader.vue"
import MissionTabs from "./MissionTabs.vue"
import { Line } from "vue-chartjs"
import axios from "axios"

import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineController,
  LineElement,
  Filler,
  Tooltip,
  Legend
} from "chart.js"

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineController,
  LineElement,
  Filler,
  Tooltip,
  Legend
)

export default {

  name: "Telemetry",
  components: { Line, MissionHeader, MissionTabs },
  props: {
    pwm: { type: Number, default: 0 },
    pwmTarget: { type: Number, default: 0 },
    addLog: { type: Function, default: () => {} },
  },

  data() {

    return {
      robotIP: "http://192.168.4.1",
      batteryLevel: null,
      showSettings: false,
      mode: "live",
      pollIntervalMs: 1000,
      paused: false,
      distanceThreshold: 30,
      lastTelemetryAt: null,
      telemetryInFlight: false,
      maxPoints: 60,
      timeLabels: [],
      pwmActualValues: [],
      pwmTargetValues: [],
      distanceValues: [],
      latestDistance: null,
      odometerValues: [],
      latestOdometer: null,

      pwmChartData: {
        labels: [],
        datasets: [
          {
            label: "PWM",
            data: [],
            borderColor: "#00aaff",
            backgroundColor: "rgba(0,170,255,0.14)",
            borderWidth: 3,
            pointRadius: 3,
            tension: 0.32,
            fill: true
          },
          {
            label: "Target",
            data: [],
            borderColor: "#a78bfa",
            backgroundColor: "rgba(167,139,250,0.08)",
            borderWidth: 2,
            pointRadius: 0,
            tension: 0.32,
            fill: false
          }
        ]
      },

      pwmChartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        animation: false,

        plugins: {
          legend: {
            display: true,
            labels: { color: "#8a96a8", usePointStyle: true, boxWidth: 8, boxHeight: 8 }
          }
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

      odometerChartData: {
        labels: [],
        datasets: [
          {
            label: "Distance covered",
            data: [],
            borderColor: "#34c759",
            backgroundColor: "rgba(52,199,89,0.12)",
            borderWidth: 3,
            pointRadius: 0,
            tension: 0.32,
            fill: true
          }
        ]
      },

      odometerChartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        animation: false,
        plugins: {
          legend: {
            display: true,
            labels: { color: "#8a96a8", usePointStyle: true, boxWidth: 8, boxHeight: 8 }
          }
        },
        scales: {
          x:{
            grid:{ color:"#2a354d"},
            ticks:{ color:"#8a96a8"}
          },
          y:{
            min:0,
            grid:{ color:"#2a354d"},
            ticks:{ color:"#8a96a8"}
          }
        }
      },

      distanceChartOptions:{
        responsive:true,
        maintainAspectRatio:false,
        animation: false,

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
  computed: {
    latestDistanceDisplay() {
      return Number.isFinite(this.latestDistance) ? Math.round(this.latestDistance) : "—"
    },
    latestOdometerDisplay() {
      return Number.isFinite(this.latestOdometer) ? this.latestOdometer.toFixed(2) : "—"
    },
    modeLabel() {
      return this.mode === "live" ? "live" : "sim"
    },
    linkStateClass() {
      if (this.mode !== "live") return "off"
      if (!this.lastTelemetryAt) return "bad"
      return Date.now() - this.lastTelemetryAt < Math.max(2500, this.pollIntervalMs * 3) ? "ok" : "bad"
    },
    linkText() {
      if (this.mode !== "live") return "SIM"
      return this.linkStateClass === "ok" ? "ONLINE" : "OFFLINE"
    },
  },
  watch: {
    pollIntervalMs() {
      this.restartTelemetryLoop()
    },
    mode() {
      this.lastTelemetryAt = null
      this.restartTelemetryLoop()
    },
    distanceThreshold() {
      this.rebuildThresholdLine()
    },
    robotIP() {
      this.lastTelemetryAt = null
    },
    maxPoints() {
      this.trimSeries()
      this.rebuildThresholdLine()
    },
  },

  mounted(){
    this.restartTelemetryLoop()
    this.fetchBatteryStatus()
    this.batteryInterval = setInterval(this.fetchBatteryStatus, 1000)
  },

  beforeUnmount(){
    clearInterval(this.simInterval)
    clearInterval(this.batteryInterval)
  },

  methods:{
    fetchBatteryStatus() {
      axios.get(`${this.robotIP}/battery`)
        .then((res) => {
          const pct = Math.round((res.data.battery * 100) / 3.3)
          this.batteryLevel = pct
        })
        .catch(() => {})
    },
    restartTelemetryLoop() {
      clearInterval(this.simInterval)
      const interval = Number.isFinite(this.pollIntervalMs) ? Math.max(200, this.pollIntervalMs) : 1000
      this.simInterval = setInterval(() => {
        this.tickTelemetry()
      }, interval)
    },
    tickTelemetry() {
      if (this.paused) return
      if (this.mode === "live") {
        this.fetchLiveTelemetry()
        return
      }
      const now = new Date()
      const mm = now.getMinutes().toString().padStart(2,"0")
      const ss = now.getSeconds().toString().padStart(2,"0")
      const time = `${mm}:${ss}`

      const pwm = Number.isFinite(this.pwm) ? this.pwm : Math.floor(Math.random() * 255)
      const pwmTarget = Number.isFinite(this.pwmTarget) ? this.pwmTarget : pwm
      const dist = Math.floor(Math.random() * 200)

      const metersPerSec = (Math.max(0, Math.min(255, pwmTarget)) / 255) * 0.9
      const intervalMs = Number.isFinite(this.pollIntervalMs) ? Math.max(200, this.pollIntervalMs) : 1000
      const dt = intervalMs / 1000
      const nextOdo = (Number.isFinite(this.latestOdometer) ? this.latestOdometer : 0) + metersPerSec * dt
      this.addDataPoint(time, pwm, pwmTarget, dist, nextOdo)
    },
    fetchLiveTelemetry() {
      if (this.telemetryInFlight) return
      const now = new Date()
      const mm = now.getMinutes().toString().padStart(2,"0")
      const ss = now.getSeconds().toString().padStart(2,"0")
      const time = `${mm}:${ss}`

      this.telemetryInFlight = true
      axios.get(`${this.robotIP}/telemetry`, { timeout: 900 })
        .then((res) => {
          const data = res?.data ?? {}
          const pwm = Number.isFinite(data.pwm) ? data.pwm : (Number.isFinite(data.pwmActual) ? data.pwmActual : this.pwm)
          const pwmTarget = Number.isFinite(data.pwmTarget) ? data.pwmTarget : (Number.isFinite(data.target) ? data.target : this.pwmTarget)
          const dist = Number.isFinite(data.distance) ? data.distance : (Number.isFinite(data.dist) ? data.dist : (Number.isFinite(data.ultrasonic) ? data.ultrasonic : null))
          const odo = Number.isFinite(data.odometer) ? data.odometer
            : (Number.isFinite(data.distanceCovered) ? data.distanceCovered
              : (Number.isFinite(data.parcours) ? data.parcours
                : (Number.isFinite(data.traveled) ? data.traveled
                  : (Number.isFinite(data.distanceTravelled) ? data.distanceTravelled : null))))

          if (!Number.isFinite(dist) && !Number.isFinite(pwm) && !Number.isFinite(pwmTarget)) return
          this.lastTelemetryAt = Date.now()
          const pwmN = Number(pwm) || 0
          const targetN = Number(pwmTarget) || 0
          const distN = Number(dist) || 0
          const fallbackOdo = this.estimateOdometer(targetN)
          const odoN = Number.isFinite(odo) ? Number(odo) : fallbackOdo
          this.addDataPoint(time, pwmN, targetN, distN, odoN)
          this.addLog({ level: "info", source: "telemetry", message: `telemetry ok (pwm=${pwmN}, dist=${distN})` })
        })
        .catch(() => {
          this.addLog({ level: "warn", source: "telemetry", message: "telemetry request failed" })
        })
        .finally(() => {
          this.telemetryInFlight = false
        })
    },
    estimateOdometer(pwmTarget) {
      const now = Date.now()
      const last = Number.isFinite(this.lastTelemetryAt) ? this.lastTelemetryAt : now
      const dt = Math.max(0, Math.min(5, (now - last) / 1000))
      const metersPerSec = (Math.max(0, Math.min(255, pwmTarget)) / 255) * 0.9
      return (Number.isFinite(this.latestOdometer) ? this.latestOdometer : 0) + metersPerSec * dt
    },
    clearSeries() {
      this.timeLabels = []
      this.pwmActualValues = []
      this.pwmTargetValues = []
      this.distanceValues = []
      this.latestDistance = null
      this.odometerValues = []
      this.latestOdometer = null
      this.pwmChartData = {
        labels: [],
        datasets: [
          { ...this.pwmChartData.datasets[0], data: [] },
          { ...this.pwmChartData.datasets[1], data: [] },
        ]
      }
      this.distanceChartData = {
        labels: [],
        datasets: [
          { ...this.distanceChartData.datasets[0], data: [] },
          { ...this.distanceChartData.datasets[1], data: [] },
        ]
      }
      this.odometerChartData = {
        labels: [],
        datasets: [
          { ...this.odometerChartData.datasets[0], data: [] },
        ]
      }
    },
    trimSeries() {
      const max = Number.isFinite(this.maxPoints) ? Math.max(10, this.maxPoints) : 60
      while (this.timeLabels.length > max) {
        this.timeLabels.shift()
        this.pwmActualValues.shift()
        this.pwmTargetValues.shift()
        this.distanceValues.shift()
        this.odometerValues.shift()
      }
    },
    rebuildThresholdLine() {
      this.distanceChartData = {
        labels: [...this.timeLabels],
        datasets: [
          {
            ...this.distanceChartData.datasets[0],
            data: [...this.distanceValues]
          },
          {
            ...this.distanceChartData.datasets[1],
            data: new Array(this.timeLabels.length).fill(this.distanceThreshold)
          }
        ]
      }
    },

   addDataPoint(time, pwm, pwmTarget, dist, odometer) {

  this.timeLabels.push(time)
  this.pwmActualValues.push(pwm)
  this.pwmTargetValues.push(pwmTarget)
  this.distanceValues.push(dist)
  this.latestDistance = dist
  this.odometerValues.push(odometer)
  this.latestOdometer = odometer

  // keep only last MAX_POINTS
  this.trimSeries()

  // update charts
  this.pwmChartData = {
    labels: [...this.timeLabels],
    datasets: [
      {
        ...this.pwmChartData.datasets[0],
        data: [...this.pwmActualValues]
      },
      {
        ...this.pwmChartData.datasets[1],
        data: [...this.pwmTargetValues]
      }
    ]
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
        data: new Array(this.timeLabels.length).fill(this.distanceThreshold)
      }
    ]
  }

  this.odometerChartData = {
    labels: [...this.timeLabels],
    datasets: [
      {
        ...this.odometerChartData.datasets[0],
        data: [...this.odometerValues]
      }
    ]
  }

},

  }

}
</script>

<style scoped>

.telemetry-view {
  font-family: var(--mc-font);
  background: var(--mc-bg-gradient);
  color: var(--mc-text);
  min-height: 100vh;
  padding: 14px 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.tool-btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 14px;
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  color: var(--mc-text);
  font-weight: 800;
  box-shadow: var(--mc-shadow);
}

.tool-btn.secondary {
  box-shadow: none;
  background: rgba(20, 27, 43, 0.45);
}

.tool-btn:active {
  transform: translateY(1px);
}

.tool-icon {
  width: 18px;
  height: 18px;
  display: inline-flex;
}

.tool-icon svg {
  width: 18px;
  height: 18px;
  display: block;
}

.link-pill {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 8px 10px;
  border-radius: 999px;
  border: 1px solid var(--mc-border);
  background: rgba(20, 27, 43, 0.45);
  color: var(--mc-muted);
  font-weight: 900;
  letter-spacing: 0.2px;
}

.link-pill .dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: var(--mc-muted);
}

.link-pill.ok {
  color: rgba(208, 216, 224, 0.92);
}

.link-pill.ok .dot {
  background: var(--mc-success);
  box-shadow: 0 0 0 4px rgba(52, 199, 89, 0.12);
}

.link-pill.bad {
  color: rgba(208, 216, 224, 0.92);
}

.link-pill.bad .dot {
  background: var(--mc-danger);
  box-shadow: 0 0 0 4px rgba(255, 59, 48, 0.12);
}

.settings-card {
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  border-radius: 16px;
  padding: 12px;
  box-shadow: var(--mc-shadow);
}

.settings-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  align-items: end;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.field-label {
  font-size: 0.84rem;
  color: var(--mc-muted);
  font-weight: 800;
}

.field-input {
  width: 100%;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.95);
  background: rgba(20, 27, 43, 0.45);
  color: var(--mc-text);
  outline: none;
}

.field-input:focus {
  border-color: rgba(0, 170, 255, 0.55);
}

.chart-body {
  height: 240px;
}

.stats {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
}

.stat-card.wide {
  grid-column: 1 / -1;
}

.stat-card {
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  border-radius: 14px;
  padding: 12px 12px;
  backdrop-filter: blur(10px);
  box-shadow: var(--mc-shadow);
}

.stat-label {
  font-size: 0.84rem;
  color: var(--mc-muted);
}

.stat-value {
  font-size: 1.6rem;
  font-weight: 800;
  letter-spacing: 0.3px;
  margin-top: 2px;
}

.stat-meta {
  font-size: 0.82rem;
  color: var(--mc-muted);
  margin-top: 2px;
}

.chart-card {
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  border-radius: 16px;
  padding: 14px 14px 10px;
  backdrop-filter: blur(10px);
}

.chart-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  gap: 10px;
}

.chart-title {
  font-size: 1.02rem;
  font-weight: 800;
  letter-spacing: 0.3px;
}

.badge {
  padding: 6px 9px;
  border-radius: 999px;
  font-size: 0.78rem;
  font-weight: 700;
  color: #08111f;
  background: linear-gradient(135deg, var(--mc-accent), var(--mc-accent-2));
}

.badge.warn {
  color: #1a0b09;
  background: linear-gradient(135deg, var(--mc-warn), var(--mc-danger));
}

.chart-note {
  margin-top: 6px;
  font-size: 0.85rem;
  color: var(--mc-muted);
  text-align: center;
}

.footer-bar {
  margin-top: auto;
  padding: 12px 10px;
  text-align: center;
  color: #7b8aa6;
  border-top: 1px solid var(--mc-border);
}

@media (max-width: 520px) {
  .settings-grid {
    grid-template-columns: 1fr;
  }
  .stats {
    grid-template-columns: 1fr;
  }
}

</style>
