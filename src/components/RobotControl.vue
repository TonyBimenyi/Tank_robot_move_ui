<template>
  <div class="mission-control">
    <MissionHeader
      title="ROV-1"
      subtitle="Mission Control · Control"
      :robotIP="robotIP"
      :batteryPercent="batteryLevel"
    />

    <MissionTabs active="control" @switch="$emit('switch', $event)" />

    <div class="status-box" :class="stateClass">
      {{ controlState }}
    </div>

    <div class="quick-bar">
      <div class="quick-left">
        <div class="chip">
          <span class="chip-label">Speed</span>
          <span class="chip-value">{{ pwmPercent }}%</span>
        </div>
        <div class="chip">
          <span class="chip-label">Direction</span>
          <span class="chip-value">{{ directionLabel }}</span>
        </div>
      </div>

      <button class="stop-btn" type="button" @click="stop">
        <span class="btn-icon" aria-hidden="true">
          <svg viewBox="0 0 24 24" fill="none">
            <path d="M7 7h10v10H7z" stroke="currentColor" stroke-width="2" stroke-linejoin="round"/>
          </svg>
        </span>
        <span>STOP</span>
      </button>
    </div>

    <div class="grid">
      <div class="card joystick-card">
        <div class="card-head">
          <div class="card-title">Drive</div>
          <div class="badge">{{ dragging ? "DRIVING" : "JOYSTICK" }}</div>
        </div>

        <div class="joystick-container">
          <div
            class="joystick-base"
            ref="joystickBase"
            @pointerdown="startDrag"
            @pointerup="stop"
            @pointercancel="stop"
            @pointerleave="stop"
            @pointermove="onDrag"
          >
            <div class="center"></div>
            <div class="stick" :style="stickStyle"></div>
          </div>
        </div>

        <div class="drive-hints">
          <div class="hint">
            <span class="hint-key">↑↓←→</span>
            <span class="hint-text">Keyboard drive</span>
          </div>
          <div class="hint">
            <span class="hint-key">SPACE</span>
            <span class="hint-text">Stop</span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-head">
          <div class="card-title">PWM Speed</div>
          <div class="badge">0–255</div>
        </div>

        <div class="pwm-value">{{ pwm }}</div>
        <input type="range" min="0" max="255" v-model.number="pwm" class="pwm-slider" />
        <div class="pwm-scale">
          <span>0</span>
          <span>255</span>
        </div>

        <div class="preset-row">
          <button class="preset" type="button" :class="{ active: pwm === 64 }" @click="setPwm(64)">25%</button>
          <button class="preset" type="button" :class="{ active: pwm === 128 }" @click="setPwm(128)">50%</button>
          <button class="preset" type="button" :class="{ active: pwm === 192 }" @click="setPwm(192)">75%</button>
          <button class="preset" type="button" :class="{ active: pwm === 255 }" @click="setPwm(255)">MAX</button>
        </div>

        <div class="nudge-row">
          <button class="nudge" type="button" @click="setPwm(pwm - 10)">
            <span class="btn-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24" fill="none">
                <path d="M8.5 12h7" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </span>
            <span>-10</span>
          </button>
          <button class="nudge" type="button" @click="setPwm(pwm + 10)">
            <span>+10</span>
            <span class="btn-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24" fill="none">
                <path d="M12 8.5v7M8.5 12h7" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </span>
          </button>
        </div>

        <div class="arrow-buttons">
          <button class="arrow-btn" @pointerdown="setDirectionManual('left')" @pointerup="stop">
            <span class="btn-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24" fill="none">
                <path d="M14.5 6.5 9 12l5.5 5.5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </span>
            <span>LEFT</span>
          </button>
          <button class="arrow-btn" @pointerdown="setDirectionManual('right')" @pointerup="stop">
            <span>RIGHT</span>
            <span class="btn-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24" fill="none">
                <path d="M9.5 6.5 15 12l-5.5 5.5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </span>
          </button>
        </div>

        <button class="emergency" @click="emergencyStop">
          <span class="btn-icon" aria-hidden="true">
            <svg viewBox="0 0 24 24" fill="none">
              <path d="M7 7h10v10H7z" stroke="currentColor" stroke-width="2" stroke-linejoin="round"/>
            </svg>
          </span>
          <span>EMERGENCY STOP</span>
        </button>
      </div>
    </div>

    <div class="footer-bar">
      <span>ti-Drive · SPACE Stop</span>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import MissionHeader from "./MissionHeader.vue"
import MissionTabs from "./MissionTabs.vue"

export default {
  name: "RobotControl",
  components: { MissionHeader, MissionTabs },
  props: {
    updatePwmTelemetry: { type: Function, required: true }, // send PWM to Telemetry
    addLog: { type: Function, default: () => {} },
  },
  data() {
    return {
      controlState: "IDLE",
      pwm: 120,
      stickPos: { x: 0, y: 0 },
      dragging: false,
      robotIP: "http://192.168.4.1",
      currentDir: "Z",
      batteryLevel: 0,
      batteryInterval: null,
    };
  },
  computed: {
    stateClass() {
      return this.controlState === "IDLE" ? "idle" : "active";
    },
    stickStyle() {
      return {
        transform: `translate(calc(-50% + ${this.stickPos.x}px), calc(-50% + ${this.stickPos.y}px))`,
      };
    },
    pwmPercent() {
      return Math.round((Math.max(0, Math.min(255, this.pwm)) / 255) * 100)
    },
    directionLabel() {
      const map = {
        A: "FWD-R",
        B: "FWD",
        C: "RIGHT",
        D: "BWD-R",
        E: "BWD",
        F: "BWD-L",
        G: "LEFT",
        H: "FWD-L",
        Z: "IDLE",
        left: "LEFT",
        right: "RIGHT",
      }
      return map[this.currentDir] ?? "—"
    },
  },
  watch: {
    pwm(newVal) {
      // Update Telemetry component with actual and desired PWM
      this.updatePwmTelemetry(newVal, this.pwm);
    },
  },
  methods: {
    setPwm(value) {
      const next = Math.max(0, Math.min(255, Math.round(value)))
      this.pwm = next
    },
    handleKeydown(e) {
      const tag = document.activeElement?.tagName?.toLowerCase()
      if (tag === "input" || tag === "textarea" || tag === "select") return
      if (e.repeat) return

      if (e.code === "Space") {
        e.preventDefault()
        this.stop()
        return
      }

      if (e.key === "ArrowLeft") {
        e.preventDefault()
        this.setDirectionManual("left")
        return
      }

      if (e.key === "ArrowRight") {
        e.preventDefault()
        this.setDirectionManual("right")
        return
      }

      if (e.key === "ArrowUp") {
        e.preventDefault()
        this.controlState = "MOVING forward"
        axios.get(`${this.robotIP}/forward`, { params: { pwm: this.pwm } }).catch(() => {})
        return
      }

      if (e.key === "ArrowDown") {
        e.preventDefault()
        this.controlState = "MOVING backward"
        axios.get(`${this.robotIP}/backward`, { params: { pwm: this.pwm } }).catch(() => {})
      }
    },
    handleKeyup(e) {
      if (e.key === "ArrowLeft" || e.key === "ArrowRight" || e.key === "ArrowUp" || e.key === "ArrowDown") {
        e.preventDefault()
        this.stop()
      }
    },
    startDrag(e) {
      this.dragging = true;
      this.updateStick(e);
    },
    onDrag(e) {
      if (this.dragging) this.updateStick(e);
    },
    updateStick(e) {
      const rect = this.$refs.joystickBase.getBoundingClientRect();
      const centerX = rect.width / 2;
      const centerY = rect.height / 2;

      let dx = e.clientX - rect.left - centerX;
      let dy = e.clientY - rect.top - centerY;

      const radius = rect.width / 2 - 24;
      const distance = Math.sqrt(dx * dx + dy * dy);

      if (distance > radius) {
        const angle = Math.atan2(dy, dx);
        dx = radius * Math.cos(angle);
        dy = radius * Math.sin(angle);
      }

      this.stickPos = { x: dx, y: dy };
      this.calculateDirection(Math.atan2(dy, dx), distance);
    },
    calculateDirection(angle, distance) {
      if (distance < 5) {
        this.currentDir = "Z";
        this.controlState = "IDLE";
        axios.get(`${this.robotIP}/stop`).catch(() => {});
        this.addLog({ level: "info", source: "control", message: "stop" })
        return;
      }

      const deg = (angle * 180) / Math.PI;
      let dir = "Z";

      if (deg >= -22.5 && deg < 22.5) dir = "G";
      else if (deg >= 22.5 && deg < 67.5) dir = "A";
      else if (deg >= 67.5 && deg < 112.5) dir = "B";
      else if (deg >= 112.5 && deg < 157.5) dir = "H";
      else if (deg >= 157.5 || deg < -157.5) dir = "C";
      else if (deg >= -157.5 && deg < -112.5) dir = "F";
      else if (deg >= -112.5 && deg < -67.5) dir = "E";
      else if (deg >= -67.5 && deg < -22.5) dir = "D";

      this.currentDir = dir;
      this.controlState = `MOVING ${dir}`;

      const map = {
        A: "forward-right",
        B: "forward",
        C: "right",
        D: "backward-right",
        E: "backward",
        F: "backward-left",
        G: "left",
        H: "forward-left",
      };

      axios.get(`${this.robotIP}/${map[dir]}`, { params: { pwm: this.pwm } }).catch(() => {});
      this.addLog({ level: "info", source: "control", message: `move ${map[dir]} (pwm=${this.pwm})` })
    },
    stop() {
      this.dragging = false;
      this.stickPos = { x: 0, y: 0 };
      this.currentDir = "Z";
      this.controlState = "IDLE";
      axios.get(`${this.robotIP}/stop`).catch(() => {});
      this.addLog({ level: "info", source: "control", message: "stop" })
    },
    emergencyStop() {
      this.stop();
      this.pwm = 0;
      this.controlState = "EMERGENCY STOP";
      console.log("!!! EMERGENCY STOP !!!");
      this.addLog({ level: "warn", source: "control", message: "EMERGENCY STOP" })
    },
    setDirectionManual(dir) {
      this.currentDir = dir;
      this.controlState = `MOVING ${dir}`;
      this.stickPos = { x: dir === "right" ? 40 : dir === "left" ? -40 : 0, y: 0 };
      const map = { left: "left", right: "right" };
      axios.get(`${this.robotIP}/${map[dir]}`, { params: { pwm: this.pwm } }).catch(() => {});
      this.addLog({ level: "info", source: "control", message: `move ${map[dir]} (pwm=${this.pwm})` })
    },
    fetchBatteryStatus() {
      axios.get(`${this.robotIP}/battery`)
        .then((res) => {
          this.batteryLevel = Math.round(res.data.battery * 100 / 3.3);
        })
        .catch(() => {});
    },
  },
  mounted() {
    this.batteryInterval = setInterval(this.fetchBatteryStatus, 1000);
    window.addEventListener("keydown", this.handleKeydown, { passive: false })
    window.addEventListener("keyup", this.handleKeyup, { passive: false })
  },
  beforeUnmount() {
    clearInterval(this.batteryInterval)
    window.removeEventListener("keydown", this.handleKeydown)
    window.removeEventListener("keyup", this.handleKeyup)
  },
};
</script>
<style scoped>
.mission-control {
  font-family: var(--mc-font);
  background: var(--mc-bg-gradient);
  color: var(--mc-text);
  min-height: 100vh;
  padding: 14px 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
  touch-action: manipulation;
}

.grid {
  display: grid;
  grid-template-columns: 1.05fr 0.95fr;
  gap: 12px;
  align-items: start;
}

.card {
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  border-radius: 16px;
  padding: 14px 14px 12px;
  backdrop-filter: blur(10px);
  box-shadow: var(--mc-shadow);
}

.card-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  gap: 10px;
}

.card-title {
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

.joystick-card {
  display: flex;
  flex-direction: column;
  align-items: stretch;
}

.status-box {
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  text-align: center;
  padding: 14px 12px;
  border-radius: 14px;
  font-size: 1.25rem;
  font-weight: 800;
  letter-spacing: 1px;
}

.status-box.idle {
  color: var(--mc-muted);
}

.status-box.active {
  color: #00ff9d;
  background: rgba(0, 255, 157, 0.12);
  border-color: rgba(0, 255, 157, 0.25);
}

.quick-bar {
  display: flex;
  gap: 10px;
  align-items: center;
  justify-content: space-between;
}

.quick-left {
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}

.chip {
  display: inline-flex;
  align-items: baseline;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 14px;
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  box-shadow: var(--mc-shadow);
}

.chip-label {
  color: var(--mc-muted);
  font-size: 0.84rem;
  font-weight: 800;
}

.chip-value {
  font-weight: 900;
  letter-spacing: 0.2px;
}

.stop-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 10px 14px;
  border-radius: 14px;
  background: rgba(20, 27, 43, 0.45);
  border: 1px solid rgba(42, 53, 77, 0.95);
  color: var(--mc-text);
  font-weight: 900;
}

.stop-btn:active {
  transform: translateY(1px);
}

.joystick-container {
  position: relative;
  width: 100%;
  max-width: 320px;
  margin: 6px auto 8px;
  aspect-ratio: 1 / 1;
}

.joystick-base {
  width: 100%;
  height: 100%;
  background: radial-gradient(circle at 50% 50%, rgba(28, 35, 51, 0.9), rgba(15, 22, 37, 0.9));
  border-radius: 50%;
  border: 1px solid rgba(42, 53, 77, 0.95);
  position: relative;
  overflow: hidden;
  touch-action: none;
}

.center {
  position: absolute;
  inset: 0;
  margin: auto;
  width: 66px;
  height: 66px;
  background: radial-gradient(circle, rgba(42, 53, 77, 0.9), rgba(20, 27, 43, 0.9));
  border-radius: 50%;
  border: 1px solid rgba(58, 71, 95, 0.9);
  pointer-events: none;
}

.stick {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 54px;
  height: 54px;
  background: linear-gradient(135deg, #0066ff, #00aaff);
  border-radius: 50%;
  box-shadow: 0 0 20px rgba(0, 102, 255, 0.6);
  transform: translate(-50%, -50%);
  transition: transform 0.05s ease-out;
  pointer-events: none;
}

.drive-hints {
  display: flex;
  gap: 10px;
  justify-content: space-between;
  margin-top: 10px;
  flex-wrap: wrap;
}

.hint {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 14px;
  border: 1px solid rgba(42, 53, 77, 0.75);
  background: rgba(20, 27, 43, 0.45);
  color: rgba(208, 216, 224, 0.92);
  font-weight: 800;
}

.hint-key {
  color: var(--mc-accent);
  font-weight: 900;
  letter-spacing: 0.2px;
}

.hint-text {
  color: var(--mc-muted);
  font-weight: 800;
}

.pwm-value {
  font-size: 2rem;
  font-weight: 900;
  color: var(--mc-accent);
  text-align: center;
  margin: 8px 0 10px;
}

.pwm-slider {
  width: 100%;
  height: 12px;
  -webkit-appearance: none;
  background: rgba(42, 53, 77, 0.95);
  border-radius: 6px;
  outline: none;
}

.pwm-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 24px;
  height: 24px;
  background: var(--mc-accent);
  border-radius: 50%;
  box-shadow: 0 0 12px rgba(0, 170, 255, 0.6);
  cursor: pointer;
}

.pwm-scale {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  color: #6b7c96;
  margin-top: 6px;
}

.preset-row {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 8px;
  margin-top: 12px;
}

.preset {
  padding: 10px 10px;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.95);
  background: rgba(20, 27, 43, 0.45);
  color: var(--mc-text);
  font-weight: 900;
}

.preset.active {
  background: var(--mc-accent-gradient);
  border-color: rgba(0, 170, 255, 0.55);
}

.preset:active {
  transform: translateY(1px);
}

.nudge-row {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
  margin-top: 10px;
}

.nudge {
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.95);
  background: rgba(20, 27, 43, 0.45);
  color: var(--mc-text);
  font-weight: 900;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.nudge:active {
  transform: translateY(1px);
}

.arrow-buttons {
  display: flex;
  gap: 12px;
  margin: 14px 0 10px;
}

.arrow-btn {
  flex: 1;
  background: rgba(20, 27, 43, 0.6);
  border: 1px solid rgba(42, 53, 77, 0.95);
  color: var(--mc-text);
  padding: 12px 12px;
  border-radius: 12px;
  font-weight: 900;
  font-size: 0.98rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: transform 0.12s ease, background 0.12s ease;
}

.arrow-btn:active {
  transform: translateY(1px);
}

.btn-icon {
  width: 18px;
  height: 18px;
  display: inline-flex;
}

.btn-icon svg {
  width: 18px;
  height: 18px;
  display: block;
}

.emergency {
  width: 100%;
  background: linear-gradient(135deg, var(--mc-danger-2), var(--mc-danger));
  color: white;
  border: none;
  padding: 16px 14px;
  font-size: 1.08rem;
  font-weight: 900;
  border-radius: 14px;
  margin-top: 8px;
  box-shadow: 0 10px 22px -14px rgba(255, 45, 85, 0.6);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.footer-bar {
  margin-top: auto;
  padding: 12px;
  text-align: center;
  color: #5a6b8c;
  font-size: 0.85rem;
  border-top: 1px solid var(--mc-border);
}

@media (max-width: 860px) {
  .grid {
    grid-template-columns: 1fr;
  }
  .preset-row {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

</style>
