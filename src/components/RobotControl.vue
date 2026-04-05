<template>
  <div class="mission-control">
    <div class="header">
      <span class="status-dot">●</span> ROV-1 | MISSION CTRL
      <span class="battery">⏻ {{ batteryLevel }}%</span>
    </div>

    <div class="tabs">
      <button class="tab active">CONTROL</button>
      <button class="tab" @click="$emit('switch','telemetry')">TELEMETRY</button>
    </div>

    <div class="status-box" :class="stateClass">
      {{ controlState }}
    </div>

    <!-- Joystick -->
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

    <!-- PWM -->
    <div class="pwm-section">
      <div class="pwm-label">PWM SPEED</div>
      <div class="pwm-value">{{ pwm }}</div>
      <input type="range" min="0" max="255" v-model.number="pwm" class="pwm-slider" />
      <div class="pwm-scale">
        <span>0</span>
        <span>255</span>
      </div>
    </div>

    <!-- Arrow buttons -->
    <div class="arrow-buttons">
      <button class="arrow-btn" @pointerdown="setDirectionManual('left')" @pointerup="stop">← LEFT</button>
      <button class="arrow-btn" @pointerdown="setDirectionManual('right')" @pointerup="stop">RIGHT →</button>
    </div>

    <button class="emergency" @click="emergencyStop">○ EMERGENCY STOP</button>

    <div class="robot-ip">
      <span class="icon">✦</span> ROBOT IP
      <span class="ip-value">{{ robotIP }}</span>
    </div>

    <div class="footer-bar">
      <span>ti-Drive · SPACE Stop</span>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  name: "RobotControl",
  props: {
    updatePwmTelemetry: { type: Function, required: true } // send PWM to Telemetry
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
  },
  watch: {
    pwm(newVal) {
      // Update Telemetry component with actual and desired PWM
      this.updatePwmTelemetry(newVal, this.pwm);
    },
  },
  methods: {
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
    },
    stop() {
      this.dragging = false;
      this.stickPos = { x: 0, y: 0 };
      this.currentDir = "Z";
      this.controlState = "IDLE";
      axios.get(`${this.robotIP}/stop`).catch(() => {});
    },
    emergencyStop() {
      this.stop();
      this.pwm = 0;
      this.controlState = "EMERGENCY STOP";
      console.log("!!! EMERGENCY STOP !!!");
    },
    setDirectionManual(dir) {
      this.currentDir = dir;
      this.controlState = `MOVING ${dir}`;
      this.stickPos = { x: dir === "right" ? 40 : dir === "left" ? -40 : 0, y: 0 };
      const map = { left: "left", right: "right" };
      axios.get(`${this.robotIP}/${map[dir]}`, { params: { pwm: this.pwm } }).catch(() => {});
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
    setInterval(this.fetchBatteryStatus, 1000);
  },
};
</script>
<style scoped>
.mission-control {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #0a0e17;
  color: #d0d8e0;
  min-height: 100vh;
  padding: 12px 16px;
  display: flex;
  flex-direction: column;
  touch-action: manipulation;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 1.1rem;
  font-weight: 600;
  margin-bottom: 8px;
}

.status-dot {
  color: #ff3b30;
  margin-right: 6px;
}

.battery {
  color: #34c759;
}

.tabs {
  display: flex;
  gap: 8px;
  margin: 12px 0;
}

.tab {
  background: #1c2333;
  border: none;
  color: #8a96a8;
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 500;
}

.tab.active {
  background: #0066ff;
  color: white;
}

.status-box {
  background: #1a2238;
  text-align: center;
  padding: 16px;
  border-radius: 12px;
  font-size: 1.4rem;
  font-weight: bold;
  margin: 12px 0;
  letter-spacing: 1px;
}

.status-box.idle {
  color: #8a96a8;
  background: #1a2238;
}

.status-box.active {
  color: #00ff9d;
  background: rgba(0, 255, 157, 0.12);
}

/* Joystick */
.joystick-container {
  position: relative;
  width: 100%;
  max-width: 280px;
  margin: 0 auto 20px;
  aspect-ratio: 1 / 1;
}

.joystick-base {
  width: 100%;
  height: 100%;
  background: radial-gradient(circle at 50% 50%, #1c2333, #0f1625);
  border-radius: 50%;
  border: 2px solid #2a354d;
  position: relative;
  overflow: hidden;
  touch-action: none;
}

.center {
  position: absolute;
  inset: 0;
  margin: auto;
  width: 60px;
  height: 60px;
  background: radial-gradient(circle, #2a354d, #1a2238);
  border-radius: 50%;
  border: 2px solid #3a475f;
  pointer-events: none;
}

.stick {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 48px;
  height: 48px;
  background: #0066ff;
  border-radius: 50%;
  box-shadow: 0 0 20px rgba(0, 102, 255, 0.6);
  transform: translate(-50%, -50%);
  transition: transform 0.05s ease-out;
  pointer-events: none;
}

/* PWM Section */
.pwm-section {
  margin: 16px 0;
  background: #141b2b;
  padding: 16px;
  border-radius: 12px;
}

.pwm-label {
  font-size: 0.9rem;
  color: #8a96a8;
  margin-bottom: 6px;
}

.pwm-value {
  font-size: 1.8rem;
  font-weight: bold;
  color: #00aaff;
  text-align: center;
  margin-bottom: 8px;
}

.pwm-slider {
  width: 100%;
  height: 12px;
  -webkit-appearance: none;
  background: #2a354d;
  border-radius: 6px;
  outline: none;
}

.pwm-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 24px;
  height: 24px;
  background: #00aaff;
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

/* Arrow Buttons */
.arrow-buttons {
  display: flex;
  gap: 12px;
  margin: 16px 0;
}

.arrow-btn {
  flex: 1;
  background: #1c2333;
  border: 1px solid #3a475f;
  color: #a0b0c8;
  padding: 14px;
  border-radius: 10px;
  font-weight: 600;
  font-size: 1rem;
}

.emergency {
  width: 100%;
  background: #ff2d55;
  color: white;
  border: none;
  padding: 18px;
  font-size: 1.2rem;
  font-weight: bold;
  border-radius: 12px;
  margin: 16px 0;
  box-shadow: 0 4px 12px rgba(255, 45, 85, 0.35);
}

.robot-ip {
  background: #1a2238;
  padding: 12px 16px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 0.95rem;
}

.ip-value {
  color: #00ff9d;
  font-weight: 500;
}

.footer-bar {
  margin-top: auto;
  padding: 12px;
  text-align: center;
  color: #5a6b8c;
  font-size: 0.85rem;
  border-top: 1px solid #2a354d;
}
</style>