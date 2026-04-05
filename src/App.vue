<template>
  <div>
    <RobotControl
      v-if="currentView === 'control'"
      @switch="currentView = $event"
    />

    <Telemetry
      v-if="currentView === 'telemetry'"
      @switch="currentView = $event"
    />
  </div>
</template>

<script>
import RobotControl from "./components/RobotControl.vue";
import Telemetry from "./components/Telemetry.vue";

export default {
  name: "App",
  components: {
    RobotControl,
    Telemetry,
  },
  data() {
    return {
      currentView: "control",
    };
  },
};
</script>