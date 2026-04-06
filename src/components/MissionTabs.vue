<template>
  <div class="mission-tabs" role="tablist" aria-label="Mission tabs">
    <button
      type="button"
      class="tab"
      :class="{ active: active === 'control' }"
      :aria-selected="active === 'control'"
      @click="$emit('switch', 'control')"
    >
      <span class="tab-icon" aria-hidden="true">
        <svg viewBox="0 0 24 24" fill="none">
          <path
            d="M7.5 9.5c-2.35 0-4.42 1.6-5 3.89l-.56 2.25A2.25 2.25 0 0 0 4.13 18h1.06c.9 0 1.7-.54 2.03-1.38l.24-.62h8.08l.24.62A2.18 2.18 0 0 0 17.85 18h1.06a2.25 2.25 0 0 0 2.19-2.36l-.56-2.25A5.15 5.15 0 0 0 15.5 9.5h-8Z"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
          <path
            d="M8 12.25h2.5M9.25 11v2.5"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
          />
          <path
            d="M15.75 11.75h.01M17.25 13.25h.01"
            stroke="currentColor"
            stroke-width="2.6"
            stroke-linecap="round"
          />
        </svg>
      </span>
      <span>CONTROL</span>
    </button>

    <button
      type="button"
      class="tab"
      :class="{ active: active === 'telemetry' }"
      :aria-selected="active === 'telemetry'"
      @click="$emit('switch', 'telemetry')"
    >
      <span class="tab-icon" aria-hidden="true">
        <svg viewBox="0 0 24 24" fill="none">
          <path
            d="M4.5 18V6.5A2 2 0 0 1 6.5 4.5H19.5"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
          <path
            d="M6.75 15.25 10.25 11.75 13 14.5 18 9.5"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
          <path
            d="M18 9.5h-2.5"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
          />
        </svg>
      </span>
      <span>TELEMETRY</span>
    </button>

    <button
      type="button"
      class="tab"
      :class="{ active: active === 'logs' }"
      :aria-selected="active === 'logs'"
      @click="$emit('switch', 'logs')"
    >
      <span class="tab-icon" aria-hidden="true">
        <svg viewBox="0 0 24 24" fill="none">
          <path
            d="M8 7.5h10M8 12h10M8 16.5h10"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
          />
          <path
            d="M5.5 7.5h.01M5.5 12h.01M5.5 16.5h.01"
            stroke="currentColor"
            stroke-width="2.8"
            stroke-linecap="round"
          />
        </svg>
      </span>
      <span>LOGS</span>
    </button>
  </div>
</template>

<script>
export default {
  name: "MissionTabs",
  props: {
    active: { type: String, required: true },
  },
  emits: ["switch"],
}
</script>

<style scoped>
.mission-tabs {
  display: flex;
  gap: 6px;
  padding: 6px;
  border-radius: 16px;
  background: var(--mc-surface-2);
  border: 1px solid var(--mc-border);
  backdrop-filter: blur(10px);
}

.tab {
  flex: 1;
  background: transparent;
  border: 0;
  color: var(--mc-muted);
  padding: 10px 12px;
  border-radius: 12px;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: transform 0.12s ease, background 0.12s ease;
}

.tab:active {
  transform: translateY(1px);
}

.tab.active {
  color: var(--mc-text);
  background: var(--mc-accent-gradient);
}

.tab-icon {
  width: 18px;
  height: 18px;
  display: inline-flex;
  color: currentColor;
  opacity: 0.95;
}

.tab-icon svg {
  width: 18px;
  height: 18px;
  display: block;
}
</style>
