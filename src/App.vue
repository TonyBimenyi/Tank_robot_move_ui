<template>
  <div>
    <RobotControl
      v-if="currentView === 'control'"
      :updatePwmTelemetry="updatePwmTelemetry"
      :addLog="addLog"
      @switch="currentView = $event"
    />

    <Telemetry
      v-if="currentView === 'telemetry'"
      :pwm="telemetry.pwm"
      :pwmTarget="telemetry.pwmTarget"
      :addLog="addLog"
      @switch="currentView = $event"
    />

    <Logs
      v-if="currentView === 'logs'"
      :logs="logs"
      :clearLogs="clearLogs"
      @switch="currentView = $event"
    />
  </div>
</template>

<script>
import RobotControl from "./components/RobotControl.vue";
import Telemetry from "./components/Telemetry.vue";
import Logs from "./components/Logs.vue";

export default {
  name: "App",
  components: {
    RobotControl,
    Telemetry,
    Logs,
  },
  data() {
    return {
      currentView: "control",
      telemetry: {
        pwm: 0,
        pwmTarget: 0,
      },
      logs: [],
    };
  },
  methods: {
    updatePwmTelemetry(pwm, pwmTarget) {
      this.telemetry = { pwm, pwmTarget };
    },
    addLog(entry) {
      const now = new Date();
      const ts = now.toLocaleTimeString(undefined, { hour12: false });
      const safeEntry = typeof entry === "string" ? { level: "info", message: entry } : (entry ?? {});
      const logItem = {
        id: `${Date.now()}-${Math.random().toString(16).slice(2)}`,
        ts,
        level: safeEntry.level ?? "info",
        source: safeEntry.source ?? "app",
        message: safeEntry.message ?? "",
        data: safeEntry.data ?? null,
      };

      const next = [...this.logs, logItem];
      this.logs = next.length > 500 ? next.slice(next.length - 500) : next;
    },
    clearLogs() {
      this.logs = [];
    },
  },
};
</script>
