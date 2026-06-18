<template>
  <ScreenAdapter>
    <section class="screen">
      <header class="page-header"><div><p>Personal Carbon Profile</p><h1>个人碳档案</h1></div><RouterLink to="/">返回主屏</RouterLink></header>
      <div class="metric-row">
        <DataCard label="个人月均碳排" :value="monthlyAverage" :digits="1" suffix=" kgCO2e" />
        <DataCard label="可用绿色积分" :value="totalPoints" suffix=" pts" />
        <DataCard label="减排进度" :value="targetProgress" suffix="%" />
      </div>
      <div class="dashboard-grid">
        <section class="panel wide"><h2>月度碳足迹</h2><TrendLine :labels="monthLabels" :values="monthValues" /></section>
        <section class="panel"><h2>碳足迹分类</h2><CarbonGauge :value="realtimeAmount" :target="targetProgress" /></section>
        <section class="panel"><h2>减排建议</h2><ul class="suggestions"><li>优先选择轨道交通与步行通勤。</li><li>将高峰用电设备迁移到低碳时段。</li><li>每周设置一次低碳饮食日。</li></ul></section>
        <section class="panel wide">
          <h2>成就徽章墙</h2>
          <div class="badges">
            <div
              v-for="badge in badgeList"
              :key="badge.id"
              class="badge-item"
              :class="{ unlocked: badge.unlocked, locked: !badge.unlocked }"
            >
              <span class="badge-icon">{{ badge.icon }}</span>
              <span class="badge-name">{{ badge.name }}</span>
              <div class="badge-tooltip">
                <p class="tooltip-title">{{ badge.name }}</p>
                <p class="tooltip-condition">{{ badge.conditionText }}</p>
                <p class="tooltip-progress" v-if="!badge.unlocked">
                  当前进度：{{ badge.currentText }} / {{ badge.targetText }}
                </p>
                <p class="tooltip-status" :class="badge.unlocked ? 'unlocked' : 'locked'">
                  {{ badge.unlocked ? '✓ 已解锁' : '🔒 未解锁' }}
                </p>
              </div>
            </div>
          </div>
        </section>
      </div>
    </section>
  </ScreenAdapter>
</template>

<script setup lang="ts">
import { computed } from 'vue';
import { RouterLink } from 'vue-router';
import CarbonGauge from '@/components/charts/CarbonGauge.vue';
import TrendLine from '@/components/charts/TrendLine.vue';
import DataCard from '@/components/common/DataCard.vue';
import ScreenAdapter from '@/components/common/ScreenAdapter.vue';
import { useCarbonData } from '@/hooks/useCarbonData';
import { useRecyclingData } from '@/hooks/useRecyclingData';
import { useCharityData } from '@/hooks/useCharityData';
import { CarbonCategory } from '@/constants/enums';

const { monthlySummary, realtimeAmount, targetProgress, categorySummary } = useCarbonData();
const { totalPoints } = useRecyclingData();
const { totalParticipants } = useCharityData();
const monthLabels = computed(() => monthlySummary.value.map((item: { month: string; amount: number }) => item.month));
const monthValues = computed(() => monthlySummary.value.map((item: { month: string; amount: number }) => item.amount));
const monthlyAverage = computed(() => monthValues.value.reduce((sum: number, item: number) => sum + item, 0) / monthValues.value.length);

interface BadgeConfig {
  id: string;
  name: string;
  icon: string;
  target: number;
  targetText: string;
  getCurrent: () => number;
  getCurrentText: () => string;
  conditionText: string;
}

const badgeConfigs: BadgeConfig[] = [
  {
    id: 'low-carbon-commute',
    name: '低碳通勤',
    icon: '🚴',
    target: 60,
    targetText: '减排进度 ≥ 60%',
    getCurrent: () => targetProgress.value,
    getCurrentText: () => `${targetProgress.value.toFixed(1)}%`,
    conditionText: '个人减排进度达到60%以上，践行低碳通勤方式'
  },
  {
    id: 'recycling-expert',
    name: '回收达人',
    icon: '♻️',
    target: 300,
    targetText: '回收积分 ≥ 300 pts',
    getCurrent: () => totalPoints.value,
    getCurrentText: () => `${totalPoints.value} pts`,
    conditionText: '累计回收积分达到300分以上'
  },
  {
    id: 'water-saving',
    name: '节水先锋',
    icon: '💧',
    target: 20,
    targetText: '用水碳排 ≤ 20 kgCO2e',
    getCurrent: () => {
      const water = categorySummary.value.find((c) => c.category === CarbonCategory.WATER);
      return water ? water.amount : 0;
    },
    getCurrentText: () => {
      const water = categorySummary.value.find((c) => c.category === CarbonCategory.WATER);
      return `${water ? water.amount : 0} kgCO2e`;
    },
    conditionText: '用水类碳排放控制在20kgCO2e以下'
  },
  {
    id: 'charity-partner',
    name: '公益伙伴',
    icon: '❤️',
    target: 1,
    targetText: '参与公益 ≥ 1 次',
    getCurrent: () => (totalParticipants.value > 0 ? 1 : 0),
    getCurrentText: () => (totalParticipants.value > 0 ? '已参与' : '未参与'),
    conditionText: '参与任意一次公益捐赠项目'
  },
  {
    id: 'green-power',
    name: '绿电家庭',
    icon: '⚡',
    target: 80,
    targetText: '用电碳排 ≤ 80 kgCO2e',
    getCurrent: () => {
      const elec = categorySummary.value.find((c) => c.category === CarbonCategory.ELECTRICITY);
      return elec ? elec.amount : 0;
    },
    getCurrentText: () => {
      const elec = categorySummary.value.find((c) => c.category === CarbonCategory.ELECTRICITY);
      return `${elec ? elec.amount : 0} kgCO2e`;
    },
    conditionText: '用电类碳排放控制在80kgCO2e以下'
  }
];

const badgeList = computed(() => {
  return badgeConfigs.map((config) => {
    const current = config.getCurrent();
    let unlocked = false;
    if (config.id === 'low-carbon-commute') {
      unlocked = current >= config.target;
    } else if (config.id === 'recycling-expert') {
      unlocked = current >= config.target;
    } else if (config.id === 'water-saving') {
      unlocked = current <= config.target;
    } else if (config.id === 'charity-partner') {
      unlocked = totalParticipants.value > 0;
    } else if (config.id === 'green-power') {
      unlocked = current <= config.target;
    }
    return {
      ...config,
      unlocked,
      currentText: config.getCurrentText()
    };
  });
});
</script>
