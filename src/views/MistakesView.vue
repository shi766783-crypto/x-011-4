<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import type { Domain, StudyLog } from '@/types'
import { DOMAINS } from '@/constants'
import { useLogsStore } from '@/stores/logs'
import { usePlansStore } from '@/stores/plans'

const logsStore = useLogsStore()
const plansStore = usePlansStore()
const router = useRouter()

type StatusFilter = 'all' | 'unresolved' | 'resolved'

interface MistakeGroup {
  domain: Domain | '未关联计划'
  items: StudyLog[]
}

const statusFilter = ref<StatusFilter>('all')

const planDomainMap = computed(() => {
  const map = new Map<string, Domain>()
  plansStore.plans.forEach((p) => map.set(p.id, p.domain))
  return map
})

/** 错题清单直接由日志实时计算：只收录填了「遇到的问题」的日志，删除日志后自动消失 */
const problemLogs = computed(() => logsStore.logs.filter((l) => l.problem.trim() !== ''))

const unresolvedCount = computed(() => problemLogs.value.filter((l) => !l.resolvedAt).length)
const resolvedCount = computed(() => problemLogs.value.length - unresolvedCount.value)

const groups = computed<MistakeGroup[]>(() => {
  const map = new Map<string, StudyLog[]>()
  for (const log of problemLogs.value) {
    if (statusFilter.value === 'unresolved' && log.resolvedAt) continue
    if (statusFilter.value === 'resolved' && !log.resolvedAt) continue
    const domain = (log.planId && planDomainMap.value.get(log.planId)) || '未关联计划'
    const list = map.get(domain)
    if (list) {
      list.push(log)
    } else {
      map.set(domain, [log])
    }
  }
  // 组按领域固定顺序排列，未关联计划的排在最后
  return [...DOMAINS, '未关联计划' as const]
    .filter((d) => map.has(d))
    .map((d) => ({ domain: d, items: map.get(d)! }))
})

function toggleResolved(log: StudyLog): void {
  logsStore.toggleResolved(log.id)
  ElMessage.success(log.resolvedAt ? '已标记为已解决' : '已重新标记为待解决')
}
</script>

<template>
  <div>
    <div class="page-header">
      <h2 class="page-title">错题回顾</h2>
      <el-radio-group v-model="statusFilter">
        <el-radio-button value="all">全部 {{ problemLogs.length }}</el-radio-button>
        <el-radio-button value="unresolved">待解决 {{ unresolvedCount }}</el-radio-button>
        <el-radio-button value="resolved">已解决 {{ resolvedCount }}</el-radio-button>
      </el-radio-group>
    </div>

    <el-empty
      v-if="problemLogs.length === 0"
      description="还没有收录任何问题，写日志时填写「遇到的问题」即可加入错题清单"
    >
      <el-button type="primary" @click="router.push('/logs')">去写日志</el-button>
    </el-empty>

    <el-empty
      v-else-if="groups.length === 0"
      :description="statusFilter === 'resolved' ? '还没有已解决的问题' : '没有待解决的问题，全部搞定了 🎉'"
    />

    <div v-else class="group-list">
      <el-card v-for="group in groups" :key="group.domain" shadow="never" class="group-card">
        <template #header>
          <div class="group-header">
            <el-tag effect="plain">{{ group.domain }}</el-tag>
            <span class="group-count">{{ group.items.length }} 条</span>
          </div>
        </template>

        <div
          v-for="log in group.items"
          :key="log.id"
          class="mistake-item"
          :class="{ resolved: !!log.resolvedAt }"
        >
          <div class="mistake-main">
            <div class="mistake-meta">
              <span class="mistake-date">{{ log.date }}</span>
              <el-tag v-if="log.resolvedAt" type="success" size="small">已解决</el-tag>
              <el-tag v-else type="danger" size="small">待解决</el-tag>
            </div>
            <div class="mistake-problem">{{ log.problem }}</div>
            <div v-if="log.solution" class="mistake-solution">解决方式：{{ log.solution }}</div>
          </div>
          <el-button
            size="small"
            :type="log.resolvedAt ? 'default' : 'success'"
            class="mistake-action"
            @click="toggleResolved(log)"
          >
            {{ log.resolvedAt ? '标记待解决' : '标记已解决' }}
          </el-button>
        </div>
      </el-card>
    </div>
  </div>
</template>

<style scoped>
.group-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.group-header {
  display: flex;
  align-items: center;
  gap: 10px;
}

.group-count {
  font-size: 13px;
  color: #909399;
}

.mistake-item {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  padding: 14px 0;
  border-bottom: 1px solid #ebeef5;
}

.mistake-item:last-child {
  border-bottom: none;
  padding-bottom: 0;
}

.mistake-item:first-child {
  padding-top: 0;
}

.mistake-main {
  flex: 1;
  min-width: 0;
}

.mistake-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;
}

.mistake-date {
  font-size: 13px;
  color: #909399;
}

.mistake-problem {
  font-size: 15px;
  color: #1f2d3d;
  line-height: 1.6;
  white-space: pre-wrap;
}

.mistake-item.resolved .mistake-problem {
  color: #909399;
  text-decoration: line-through;
}

.mistake-solution {
  margin-top: 6px;
  font-size: 13px;
  color: #606266;
  line-height: 1.6;
  white-space: pre-wrap;
  background: #f5f7fa;
  padding: 8px 12px;
  border-radius: 6px;
}

.mistake-action {
  flex-shrink: 0;
}
</style>
