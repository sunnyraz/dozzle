<template>
  <div class="flex gap-4">
    <StatMonitor
      :data="memoryData"
      label="mem"
      :stat-value="formatBytes(totalStat.memoryUsage)"
      :limit="formatBytes(limits.memory, { short: true, decimals: 1 })"
    />
    <StatMonitor
      :data="cpuData"
      label="load"
      :stat-value="Math.max(0, totalStat.cpu).toFixed(2) + '%'"
      :limit="roundCPU(limits.cpu) + ' CPU'"
    />
  </div>
</template>

<script lang="ts" setup>
import { Stat } from "@/models/Container";
import { Container } from "@/models/Container";

const { containers } = defineProps<{
  containers: Container[];
}>();

const totalStat = ref<Stat>({ cpu: 0, memory: 0, memoryUsage: 0 });
const { history, reset } = useSimpleRefHistory(totalStat, { capacity: 300 });
const { hosts } = useHosts();

const roundCPU = (num: number) => (Number.isInteger(num) ? num.toFixed(0) : num.toFixed(1));

watch(
  () => containers,
  () => {
    const initial: Stat[] = [];
    for (let i = 1; i <= 300; i++) {
      const stat = containers.reduce(
        (acc, { statsHistory }) => {
          const item = statsHistory.at(-i);
          if (!item) {
            return acc;
          }
          return {
            cpu: acc.cpu + item.cpu,
            memory: acc.memory + item.memory,
            memoryUsage: acc.memoryUsage + item.memoryUsage,
          };
        },
        { cpu: 0, memory: 0, memoryUsage: 0 },
      );
      initial.push(stat);
    }
    reset({ initial: initial.reverse() });
  },
  { immediate: true },
);

const limits = computed(() =>
  containers.reduce(
    (acc, c) => ({
      cpu: acc.cpu + (c.cpuLimit > 0 ? c.cpuLimit : hosts.value[c.host].nCPU),
      memory: acc.memory + (c.memoryLimit > 0 ? c.memoryLimit : hosts.value[c.host].memTotal),
    }),
    { cpu: 0, memory: 0 },
  ),
);

useIntervalFn(() => {
  totalStat.value = containers.reduce(
    (acc, { stat }) => {
      return {
        cpu: acc.cpu + stat.cpu,
        memory: acc.memory + stat.memory,
        memoryUsage: acc.memoryUsage + stat.memoryUsage,
      };
    },
    { cpu: 0, memory: 0, memoryUsage: 0 },
  );
}, 1000);

const cpuData = computed(() =>
  history.value.map((stat, i) => ({
    x: i,
    y: Math.max(0, stat.cpu),
    value: Math.max(0, stat.cpu).toFixed(2) + "%",
  })),
);

const memoryData = computed(() =>
  history.value.map((stat, i) => ({
    x: i,
    y: stat.memoryUsage,
    value: formatBytes(stat.memoryUsage),
  })),
);
</script>
