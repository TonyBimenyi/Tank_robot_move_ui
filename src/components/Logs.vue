<template>
  <div class="logs-view">
    <MissionHeader
      title="ROV-1"
      subtitle="Mission Control · Logs"
      robotIP="telemetry stream"
      :batteryPercent="null"
    />

    <MissionTabs active="logs" @switch="$emit('switch', $event)" />

    <div class="panel">
      <div class="controls">
        <label class="search">
          <span class="icon" aria-hidden="true">
            <svg viewBox="0 0 24 24" fill="none">
              <path
                d="M10.5 18a7.5 7.5 0 1 1 0-15 7.5 7.5 0 0 1 0 15Z"
                stroke="currentColor"
                stroke-width="1.8"
              />
              <path
                d="M16.2 16.2 21 21"
                stroke="currentColor"
                stroke-width="1.8"
                stroke-linecap="round"
              />
            </svg>
          </span>
          <input v-model.trim="query" class="input" placeholder="Search logs…" />
        </label>

        <select v-model="level" class="input select">
          <option value="all">All levels</option>
          <option value="info">Info</option>
          <option value="warn">Warn</option>
          <option value="error">Error</option>
        </select>

        <button class="btn" type="button" @click="clearLogs()">
          <span class="btn-icon" aria-hidden="true">
            <svg viewBox="0 0 24 24" fill="none">
              <path d="M6.5 7.5h11M10 7.5v-2h4v2" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
              <path d="M9 10v7M12 10v7M15 10v7" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>
              <path d="M7.5 7.5l.8 13a1.2 1.2 0 0 0 1.2 1.1h5a1.2 1.2 0 0 0 1.2-1.1l.8-13" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>
            </svg>
          </span>
          <span>Clear</span>
        </button>
      </div>

      <div class="list" ref="list">
        <div v-if="filtered.length === 0" class="empty">No logs.</div>

        <div v-for="item in filtered" :key="item.id" class="row" :class="item.level">
          <div class="meta">
            <span class="ts">{{ item.ts }}</span>
            <span class="tag">{{ item.source }}</span>
            <span class="tag lvl">{{ item.level }}</span>
          </div>
          <div class="msg">{{ item.message }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import MissionHeader from "./MissionHeader.vue"
import MissionTabs from "./MissionTabs.vue"

export default {
  name: "Logs",
  components: { MissionHeader, MissionTabs },
  props: {
    logs: { type: Array, required: true },
    clearLogs: { type: Function, required: true },
  },
  emits: ["switch"],
  data() {
    return {
      query: "",
      level: "all",
    }
  },
  computed: {
    filtered() {
      const q = this.query.toLowerCase()
      return this.logs.filter((l) => {
        if (this.level !== "all" && l.level !== this.level) return false
        if (!q) return true
        const text = `${l.ts} ${l.level} ${l.source} ${l.message}`.toLowerCase()
        return text.includes(q)
      })
    },
  },
  watch: {
    logs: {
      deep: true,
      handler() {
        this.$nextTick(() => {
          const el = this.$refs.list
          if (!el) return
          el.scrollTop = el.scrollHeight
        })
      },
    },
  },
}
</script>

<style scoped>
.logs-view {
  font-family: var(--mc-font);
  background: var(--mc-bg-gradient);
  color: var(--mc-text);
  min-height: 100vh;
  padding: 14px 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.panel {
  background: var(--mc-surface);
  border: 1px solid var(--mc-border);
  border-radius: 16px;
  padding: 12px;
  box-shadow: var(--mc-shadow);
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.controls {
  display: grid;
  grid-template-columns: 1fr 160px 120px;
  gap: 10px;
  align-items: center;
}

.search {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.95);
  background: rgba(20, 27, 43, 0.45);
}

.icon {
  width: 18px;
  height: 18px;
  display: inline-flex;
  color: var(--mc-muted);
}

.icon svg {
  width: 18px;
  height: 18px;
  display: block;
}

.input {
  width: 100%;
  border: 0;
  outline: none;
  background: transparent;
  color: var(--mc-text);
}

.select {
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.95);
  background: rgba(20, 27, 43, 0.45);
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.95);
  background: rgba(20, 27, 43, 0.45);
  font-weight: 900;
}

.btn:active {
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

.list {
  max-height: calc(100vh - 260px);
  overflow: auto;
  border-radius: 12px;
  border: 1px solid rgba(42, 53, 77, 0.75);
  background: rgba(10, 14, 23, 0.55);
}

.row {
  padding: 10px 12px;
  border-bottom: 1px solid rgba(42, 53, 77, 0.55);
}

.row:last-child {
  border-bottom: 0;
}

.meta {
  display: flex;
  gap: 8px;
  align-items: center;
  color: var(--mc-muted);
  font-size: 0.82rem;
  font-weight: 800;
}

.ts {
  color: rgba(208, 216, 224, 0.75);
}

.tag {
  padding: 4px 8px;
  border-radius: 999px;
  border: 1px solid rgba(42, 53, 77, 0.8);
  background: rgba(20, 27, 43, 0.5);
}

.tag.lvl {
  text-transform: uppercase;
}

.msg {
  margin-top: 6px;
  font-weight: 700;
  letter-spacing: 0.2px;
}

.row.info .tag.lvl {
  border-color: rgba(0, 170, 255, 0.5);
}

.row.warn .tag.lvl {
  border-color: rgba(255, 204, 0, 0.6);
}

.row.error .tag.lvl {
  border-color: rgba(255, 59, 48, 0.6);
}

.empty {
  padding: 18px;
  color: var(--mc-muted);
  text-align: center;
  font-weight: 800;
}

@media (max-width: 680px) {
  .controls {
    grid-template-columns: 1fr;
  }
  .list {
    max-height: calc(100vh - 340px);
  }
}
</style>
