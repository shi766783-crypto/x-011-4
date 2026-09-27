<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import type { Domain, StudyLog } from '@/types'
import { DOMAINS } from '@/constants'
import StatCard from '@/components/StatCard.vue'
import { useLogsStore } from '@/stores/logs'
import { usePlansStore } from '@/stores/plans'

const logsStore = useLogsStore()
const plansStore = usePlansStore()
const router = useRouter()

type Filter = 'all' | 'pending' | 'resolved'
const filter = ref<Filter>('all')

/** 清单条目：直接引用日志本体，不复制数据，删除日志后自动从清单消失 */
interface MistakeEntry {
  log: StudyLog
  domain: Domain
  planName: string
  resolved: boolean
}

interface MistakeGroup {
  domain: Domain
  items: MistakeEntry[]
}

const planMap = computed(() => {
  const map = new Map<string, (typeof plansStore.plans)[number]>()
  plansStore.plans.forEach((p) => map.set(p.id, p))
  return map
})

/** 所有带问题内容的日志，按关联计划归属领域（未关联归入“其他”） */
const entries = computed<MistakeEntry[]>(() =>
  logsStore.logs
    .filter((l) => l.problem.trim().length > 0)
    .map((log) => {
      const plan = log.planId ? planMap.value.get(log.planId) : undefined
      return {
        log,
        domain: plan?.domain ?? '其他',
        planName: plan?.name ?? '未关联',
        resolved: Boolean(log.resolvedAt),
      }
    }),
)

const filtered = computed<MistakeEntry[]>(() =>
  entries.value.filter((e) => {
    if (filter.value === 'pending') return !e.resolved
    if (filter.value === 'resolved') return e.resolved
    return true
  }),
)

/** 按领域分组，领域顺序与 DOMAINS 常量保持一致 */
const groups = computed<MistakeGroup[]>(() => {
  const byDomain = new Map<Domain, MistakeEntry[]>()
  filtered.value.forEach((e) => {
    const list = byDomain.get(e.domain) ?? []
    list.push(e)
    byDomain.set(e.domain, list)
  })
  return DOMAINS.filter((d) => byDomain.has(d)).map((d) => ({
    domain: d,
    items: byDomain.get(d) ?? [],
  }))
})

const resolvedCount = computed(() => entries.value.filter((e) => e.resolved).length)
const pendingCount = computed(() => entries.value.length - resolvedCount.value)
const resolveRate = computed(() =>
  entries.value.length === 0 ? 0 : Math.round((resolvedCount.value / entries.value.length) * 100),
)

function toggleResolved(entry: MistakeEntry): void {
  logsStore.updateLog(entry.log.id, {
    resolvedAt: entry.resolved ? undefined : new Date().toISOString(),
  })
  ElMessage.success(entry.resolved ? '已重新标记为待解决' : '已标记为已解决')
}
</script>

<template>
  <div>
    <div class="page-header">
      <h2 class="page-title">错题回顾</h2>
      <el-radio-group v-model="filter">
        <el-radio-button value="all">全部</el-radio-button>
        <el-radio-button value="pending">待解决</el-radio-button>
        <el-radio-button value="resolved">已解决</el-radio-button>
      </el-radio-group>
    </div>

    <div class="card-grid summary">
      <StatCard label="待解决问题" :value="pendingCount" icon="❗" color="#f56c6c" />
      <StatCard label="已解决问题" :value="resolvedCount" icon="✅" color="#67c23a" />
      <StatCard label="解决率(%)" :value="resolveRate" icon="📈" color="#409eff" />
    </div>

    <el-empty v-if="entries.length === 0" description="还没有记录过问题，写日志时记下遇到的困难吧">
      <el-button type="primary" @click="router.push('/logs')">去写日志</el-button>
    </el-empty>

    <el-empty v-else-if="groups.length === 0" description="当前筛选下没有问题记录" />

    <template v-else>
      <section v-for="group in groups" :key="group.domain" class="domain-section">
        <div class="domain-header">
          <el-tag effect="dark">{{ group.domain }}</el-tag>
          <span class="domain-count">{{ group.items.length }} 条</span>
        </div>

        <el-card
          v-for="entry in group.items"
          :key="entry.log.id"
          class="mistake-card"
          :class="{ resolved: entry.resolved }"
          shadow="hover"
        >
          <div class="mistake-head">
            <span class="mistake-date">{{ entry.log.date }}</span>
            <el-tag size="small" type="info" effect="plain">{{ entry.planName }}</el-tag>
            <el-tag v-if="entry.resolved" size="small" type="success">已解决</el-tag>
            <el-tag v-else size="small" type="danger">待解决</el-tag>
          </div>

          <div class="mistake-section">
            <div class="mistake-label">问题</div>
            <div class="mistake-problem">{{ entry.log.problem }}</div>
          </div>

          <div v-if="entry.log.solution" class="mistake-section">
            <div class="mistake-label">解决方式</div>
            <div class="mistake-solution">{{ entry.log.solution }}</div>
          </div>

          <div class="mistake-foot">
            <span class="mistake-content">学习内容：{{ entry.log.content }}</span>
            <el-button
              size="small"
              :type="entry.resolved ? 'info' : 'success'"
              :plain="entry.resolved"
              @click="toggleResolved(entry)"
            >
              {{ entry.resolved ? '重新打开' : '标记已解决' }}
            </el-button>
          </div>
        </el-card>
      </section>
    </template>
  </div>
</template>

<style scoped>
.summary {
  margin-bottom: 20px;
}

.domain-section {
  margin-bottom: 24px;
}

.domain-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 12px;
}

.domain-count {
  font-size: 13px;
  color: #909399;
}

.mistake-card {
  margin-bottom: 12px;
}

.mistake-card.resolved {
  opacity: 0.72;
}

.mistake-head {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 12px;
}

.mistake-date {
  font-size: 13px;
  color: #606266;
  font-weight: 600;
}

.mistake-section {
  margin-bottom: 12px;
}

.mistake-label {
  font-size: 13px;
  color: #909399;
  margin-bottom: 6px;
}

.mistake-problem {
  font-size: 15px;
  font-weight: 600;
  color: #1f2d3d;
  line-height: 1.6;
  white-space: pre-wrap;
}

.mistake-solution {
  font-size: 14px;
  color: #303133;
  line-height: 1.7;
  white-space: pre-wrap;
  background: #f5f7fa;
  padding: 12px;
  border-radius: 8px;
}

.mistake-foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.mistake-content {
  font-size: 13px;
  color: #909399;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
