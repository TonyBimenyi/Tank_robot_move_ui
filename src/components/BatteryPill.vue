<template>
  <div class="pill battery" :class="stateClass">
    <span class="battery-icon" aria-hidden="true">
      <span class="battery-fill" :style="{ width: `${percentSafe}%` }"></span>
      <span class="battery-cap"></span>
    </span>
    <span class="battery-text">{{ display }}</span>
  </div>
</template>

<script>
export default {
  name: "BatteryPill",
  props: {
    percent: { type: Number, default: null },
  },
  computed: {
    percentSafe() {
      if (!Number.isFinite(this.percent)) return 0
      return Math.max(0, Math.min(100, Math.round(this.percent)))
    },
    display() {
      return Number.isFinite(this.percent) ? `${this.percentSafe}%` : "—%"
    },
    stateClass() {
      if (!Number.isFinite(this.percent)) return "unknown"
      if (this.percentSafe <= 15) return "low"
      if (this.percentSafe <= 35) return "medium"
      return "high"
    },
  },
}
</script>

<style scoped>
.pill {
  padding: 7px 10px;
  border-radius: 999px;
  font-size: 0.82rem;
  color: rgba(208, 216, 224, 0.92);
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  backdrop-filter: blur(8px);
}

.pill.battery {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 7px 10px;
}

.battery-icon {
  position: relative;
  width: 26px;
  height: 14px;
  border-radius: 4px;
  border: 1px solid rgba(208, 216, 224, 0.7);
  box-sizing: border-box;
  overflow: hidden;
}

.battery-fill {
  position: absolute;
  inset: 2px;
  width: 0%;
  border-radius: 3px;
  background: linear-gradient(135deg, var(--mc-success), var(--mc-accent));
}

.battery-cap {
  position: absolute;
  right: -4px;
  top: 4px;
  width: 3px;
  height: 6px;
  border-radius: 2px;
  background: rgba(208, 216, 224, 0.7);
}

.battery-text {
  font-weight: 800;
  letter-spacing: 0.2px;
  font-size: 0.82rem;
}

.pill.battery.low .battery-fill {
  background: linear-gradient(135deg, var(--mc-danger), var(--mc-warn));
}

.pill.battery.medium .battery-fill {
  background: linear-gradient(135deg, var(--mc-warn), var(--mc-accent));
}

.pill.battery.unknown .battery-fill {
  background: rgba(208, 216, 224, 0.25);
}
</style>
