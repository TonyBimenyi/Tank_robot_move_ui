<template>
  <div class="header">
    <div class="brand">
      <span class="status-dot" aria-hidden="true">●</span>
      <div class="brand-text">
        <div class="title">{{ title }}</div>
        <div class="subtitle">{{ subtitle }}</div>
      </div>
    </div>

    <div class="header-right">
      <div class="pill live">LIVE</div>
      <BatteryPill :percent="batteryPercent" />
      <div v-if="robotIP" class="pill ip">{{ robotIP }}</div>
    </div>
  </div>
</template>

<script>
import BatteryPill from "./BatteryPill.vue"

export default {
  name: "MissionHeader",
  components: { BatteryPill },
  props: {
    title: { type: String, required: true },
    subtitle: { type: String, required: true },
    robotIP: { type: String, default: "" },
    batteryPercent: { type: Number, default: null },
  },
}
</script>

<style scoped>
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}

.brand {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.title {
  font-size: 1.08rem;
  font-weight: 700;
  letter-spacing: 0.4px;
}

.subtitle {
  font-size: 0.86rem;
  color: var(--mc-muted);
}

.status-dot {
  color: var(--mc-danger);
  font-size: 1.1rem;
}

.header-right {
  display: flex;
  gap: 8px;
  align-items: center;
  flex-wrap: wrap;
  justify-content: flex-end;
}

.pill {
  padding: 7px 10px;
  border-radius: 999px;
  font-size: 0.82rem;
  color: rgba(208, 216, 224, 0.92);
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  backdrop-filter: blur(8px);
}

.pill.live {
  color: #08111f;
  background: linear-gradient(135deg, var(--mc-success), var(--mc-accent));
  border: 0;
  font-weight: 700;
}

.pill.ip {
  max-width: min(46vw, 380px);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
