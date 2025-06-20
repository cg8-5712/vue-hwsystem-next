<template>
  <div class="bg-white dark:bg-gray-800 rounded-3xl shadow-lg border border-gray-200/50 dark:border-gray-700/50 overflow-hidden transition-all duration-200 hover:shadow-xl">
    <div class="p-4 sm:p-6 border-b border-gray-200 dark:border-gray-700">
      <div class="flex flex-col sm:flex-row sm:items-center justify-between mb-4 gap-3 sm:gap-0">
        <h2 class="text-lg sm:text-xl font-bold text-gray-900 dark:text-white">
          {{ title || $t('dashboard.assignments') }}
        </h2>
        <div class="flex items-center gap-2">
          <span class="text-sm text-gray-500 dark:text-gray-400">
            {{ filteredAndSortedAssignments.length }} {{ $t('dashboard.total') }}
          </span>
          <slot name="header-actions"></slot>
        </div>
      </div>
      
      <!-- 筛选器 - 移动端优化 -->
      <div class="flex items-center gap-2 flex-wrap">
        <button
          v-for="filter in filterOptions"
          :key="filter.value"
          @click="selectedFilter = filter.value"
          :class="[
            'px-3 py-1.5 rounded-lg text-xs font-medium transition-all duration-200 touch-manipulation',
            selectedFilter === filter.value
              ? 'bg-indigo-100 dark:bg-indigo-900/30 text-indigo-700 dark:text-indigo-300 border border-indigo-300 dark:border-indigo-600'
              : 'bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600'
          ]"
        >
          <span class="hidden sm:inline">{{ filter.label }}</span>
          <span class="sm:hidden">{{ filter.label.slice(0, 2) }}</span>
          <span v-if="filter.count !== undefined" class="ml-1 opacity-75">({{ filter.count }})</span>
        </button>
      </div>
    </div>

    <div class="max-h-[60vh] sm:max-h-[600px] overflow-y-auto">
      <div v-if="isListView" class="divide-y divide-gray-100 dark:divide-gray-700">
        <!-- 列表视图 - 移动端优化 -->
        <div
          v-for="assignment in filteredAndSortedAssignments"
          :key="assignment.id"
          :class="[
            'p-4 sm:p-6 hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-all duration-200 group cursor-pointer touch-manipulation',
            getAssignmentRowClass(assignment)
          ]"
            @click="$emit('assignment-click', assignment)"
        >
          <div class="flex items-start justify-between">
            <div class="flex-1 min-w-0">
              <div class="flex items-start sm:items-center gap-2 sm:gap-3 mb-3 flex-col sm:flex-row">
                <div class="flex items-center gap-2">
                  <div :class="getStatusIndicatorClass(assignment)"></div>
                  <h3 :class="[
                    'text-base sm:text-lg font-semibold group-hover:text-indigo-600 dark:group-hover:text-indigo-400 transition-colors duration-200 break-words',
                    getAssignmentTitleClass(assignment)
                  ]">
                    {{ assignment.title }}
                  </h3>
                </div>
                
                <!-- 移动端状态标签 -->
                <div class="flex items-center gap-2 flex-wrap">
                  <!-- 紧急/过期状态 -->
                  <span v-if="isOverdue(assignment)" class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-red-100 dark:bg-red-900/30 text-red-800 dark:text-red-200 animate-pulse">
                    {{ $t('common.overdue') || '已过期' }}
                  </span>
                  <span v-else-if="isUrgent(assignment)" class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-orange-100 dark:bg-orange-900/30 text-orange-800 dark:text-orange-200 animate-pulse">
                    {{ $t('common.urgent') || '紧急' }}
                  </span>
                  
                  <!-- 班级进度显示 -->
                  <div v-if="showClassProgress" class="flex items-center gap-1 px-2 py-1 rounded-lg bg-blue-100 dark:bg-blue-900/30 text-blue-800 dark:text-blue-200">
                    <UsersIcon class="h-3 w-3" />
                    <span class="text-xs font-medium">{{ assignment.classSubmitted || assignment.submitted }}/{{ assignment.classTotal || assignment.totalStudents }}</span>
                  </div>
                  
                  <!-- 普通状态标签 -->
                  <span v-if="showStatusBadge && !isOverdue(assignment) && !isUrgent(assignment)" :class="getStatusBadgeClass(assignment)">
                    {{ getStatusText(assignment) }}
                  </span>
                </div>
              </div>

              <!-- 移动端垂直布局，桌面端网格布局 -->
              <div class="flex flex-col sm:grid gap-2 sm:gap-4 text-sm sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
                <div class="flex items-center gap-2 text-gray-600 dark:text-gray-400">
                  <CalendarIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ $t('dashboard.assigned') }}: {{ assignment.assignDate }}</span>
                </div>
                <div class="flex items-center gap-2 text-gray-600 dark:text-gray-400">
                  <ClockIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ $t('dashboard.due') }}: {{ assignment.dueDate }}</span>
                </div>
                <div v-if="showAttempts" class="flex items-center gap-2 text-gray-600 dark:text-gray-400">
                  <ArrowPathIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ $t('dashboard.attempts') }}: {{ assignment.attempts }}</span>
                </div>
                <div v-if="showProgress" class="flex items-center gap-2 text-gray-600 dark:text-gray-400">
                  <ChartBarIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ progressLabel }}: {{ getProgressPercentage(assignment) }}%</span>
                </div>
              </div>
            </div>
            
            <!-- 右侧操作按钮 - 移动端优化 -->
            <div class="ml-2 sm:ml-4 flex flex-col items-end gap-2">
              <div class="flex items-center gap-1">
                <slot name="assignment-actions" :assignment="assignment">
                  <button class="inline-flex items-center gap-1 px-2 sm:px-3 py-1.5 text-xs font-medium text-indigo-600 dark:text-indigo-400 hover:bg-indigo-50 dark:hover:bg-indigo-900/20 rounded-lg transition-all duration-200 touch-manipulation">
                    <span class="hidden sm:inline">{{ $t('dashboard.viewDetails') }}</span>
                    <span class="sm:hidden">{{ $t('common.view') || '详情' }}</span>
                    <ChevronRightIcon class="h-3 w-3" />
                  </button>
                </slot>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      <div v-else class="p-4 sm:p-6">
        <!-- 网格视图 - 移动端单列，平板双列，桌面端多列 -->
        <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-2 xl:grid-cols-3 gap-4 sm:gap-6">
          <div
            v-for="assignment in filteredAndSortedAssignments"
            :key="assignment.id"
            :class="[
              'relative group bg-gradient-to-br from-white to-gray-50 dark:from-gray-700 dark:to-gray-800 rounded-2xl p-4 sm:p-6 border border-gray-200 dark:border-gray-600 hover:shadow-xl transform hover:-translate-y-1 transition-all duration-300 cursor-pointer touch-manipulation',
              getAssignmentCardClass(assignment),
              getCardHoverClass(assignment)
            ]"
              @click="$emit('assignment-click', assignment)"
          >
            <div class="absolute top-3 sm:top-4 right-3 sm:right-4">
              <div :class="getStatusIndicatorClass(assignment)"></div>
              <!-- 紧急状态在hover时显示动画点 -->
              <div v-if="isUrgent(assignment)" class="absolute top-0 left-0 w-3 h-3 bg-orange-500 rounded-full opacity-0 group-hover:opacity-100 group-hover:animate-ping transition-opacity duration-200"></div>
            </div>

            <div class="mb-4">
              <h3 :class="[
                'text-base sm:text-lg font-semibold mb-2 pr-6 sm:pr-8 break-words',
                getAssignmentTitleClass(assignment)
              ]">
                {{ assignment.title }}
              </h3>

              <div class="space-y-2 text-sm text-gray-600 dark:text-gray-400">
                <div class="flex items-center gap-2">
                  <CalendarIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ assignment.assignDate }} - {{ assignment.dueDate }}</span>
                </div>
                <div v-if="showAttempts" class="flex items-center gap-2">
                  <ArrowPathIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ assignment.attempts }} {{ $t('common.attempts') || '次提交' }}</span>
                </div>
                <div v-if="showClassProgress" class="flex items-center gap-2">
                  <UsersIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ classProgressLabel }}: {{ assignment.classSubmitted || assignment.submitted }}/{{ assignment.classTotal || assignment.totalStudents }} {{ $t('common.submitted') || '已提交' }}</span>
                </div>
                <div v-if="showProgress" class="flex items-center gap-2">
                  <ChartBarIcon class="h-4 w-4 flex-shrink-0" />
                  <span class="truncate">{{ getProgressPercentage(assignment) }}% {{ $t('common.completion') || '完成率' }}</span>
                </div>
              </div>
            </div>

            <div class="flex items-center justify-between">
              <div></div>
              <div class="opacity-0 group-hover:opacity-100 sm:opacity-100 flex items-center gap-1 transition-all duration-200">
                <slot name="grid-actions" :assignment="assignment">
                  <button class="inline-flex items-center gap-1 px-2 py-1.5 text-xs font-medium text-indigo-600 dark:text-indigo-400 hover:bg-indigo-50 dark:hover:bg-indigo-900/20 rounded-lg transition-all duration-200 touch-manipulation">
                    {{ $t('common.view') || '查看' }}
                    <ChevronRightIcon class="h-3 w-3" />
                  </button>
                </slot>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  CalendarIcon,
  ClockIcon,
  ArrowPathIcon,
  ChevronRightIcon,
  ChartBarIcon,
  UsersIcon
} from '@heroicons/vue/24/outline'

interface Assignment {
  id: string | number
  title: string
  assignDate: string
  dueDate: string
  status: string
  attempts?: number
  submitted?: number
  totalStudents?: number
  classSubmitted?: number
  classTotal?: number
}

interface Props {
  title?: string
  assignments: Assignment[]
  isListView: boolean
  showAttempts?: boolean
  showProgress?: boolean
  showClassProgress?: boolean
  showStatusBadge?: boolean
  progressLabel?: string
  classProgressLabel?: string
  statusColors?: Record<string, string>
  statusBadgeColors?: Record<string, string>
  statusTexts?: Record<string, string>
}

const props = withDefaults(defineProps<Props>(), {
  title: '',
  showAttempts: true,
  showProgress: false,
  showClassProgress: false,
  showStatusBadge: false,
  progressLabel: '进度',
  classProgressLabel: '班级',
  statusColors: () => ({
    submitted: 'bg-green-500',
    pending: 'bg-yellow-500',
    active: 'bg-green-500',
    draft: 'bg-gray-500',
    closed: 'bg-blue-500'
  }),
  statusBadgeColors: () => ({
    submitted: 'bg-green-100 dark:bg-green-900/30 text-green-800 dark:text-green-200',
    pending: 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-800 dark:text-yellow-200',
    active: 'bg-green-100 dark:bg-green-900/30 text-green-800 dark:text-green-200',
    draft: 'bg-gray-100 dark:bg-gray-900/30 text-gray-800 dark:text-gray-200',
    closed: 'bg-blue-100 dark:bg-blue-900/30 text-blue-800 dark:text-blue-200'
  }),
  statusTexts: () => ({
    submitted: '已提交',
    pending: '未提交',
    active: '进行中',
    draft: '草稿',
    closed: '已结束'
  })
})

const { t } = useI18n()

defineEmits<{
  'assignment-click': [assignment: Assignment]
}>()

const gridCols = computed(() => {
  let cols = 'grid-cols-1 sm:grid-cols-2'
  if (props.showAttempts && props.showProgress) {
    cols = 'grid-cols-1 sm:grid-cols-4'
  } else if (props.showAttempts || props.showProgress) {
    cols = 'grid-cols-1 sm:grid-cols-3'
  }
  return cols
})

// 新增方法
const isUrgent = (assignment: Assignment) => {
  // 检查是否为紧急状态
  if (assignment.status === 'submitted') return false

  const now = new Date()
  now.setHours(0, 0, 0, 0) // 重置到当天00:00:00

  const dueDate = new Date(assignment.dueDate + 'T00:00:00') // 确保使用本地时间
  const diffDays = Math.ceil((dueDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24))

  return diffDays <= 3 && diffDays >= 0
}

const isOverdue = (assignment: Assignment) => {
  // 检查是否已过期
  if (assignment.status === 'submitted' || assignment.status === 'closed') return false

  const now = new Date()
  now.setHours(23, 59, 59, 999) // 设置到当天的最后一刻

  const dueDate = new Date(assignment.dueDate + 'T23:59:59') // 设置到截止日的最后一刻
  return dueDate < now
}

// 筛选状态
const selectedFilter = ref('all')

// 筛选选项
const filterOptions = computed(() => [
  { 
    label: t('common.all') || '全部', 
    value: 'all', 
    count: props.assignments.length 
  },
  { 
    label: t('common.urgent') || '紧急', 
    value: 'urgent', 
    count: props.assignments.filter(a => isUrgent(a)).length 
  },
  { 
    label: t('common.overdue') || '过期', 
    value: 'overdue', 
    count: props.assignments.filter(a => isOverdue(a)).length 
  },
  { 
    label: t('common.pending') || '待完成', 
    value: 'pending', 
    count: props.assignments.filter(a => a.status === 'pending' || a.status === 'active' || a.status === 'draft').length 
  },
  { 
    label: t('common.completed') || '已完成', 
    value: 'completed', 
    count: props.assignments.filter(a => a.status === 'submitted' || a.status === 'closed').length 
  }
])

// 筛选逻辑
const filteredAssignments = computed(() => {
  switch (selectedFilter.value) {
    case 'urgent':
      return props.assignments.filter(a => isUrgent(a))
    case 'overdue':
      return props.assignments.filter(a => isOverdue(a))
    case 'pending':
      return props.assignments.filter(a => a.status === 'pending' || a.status === 'active' || a.status === 'draft')
    case 'completed':
      return props.assignments.filter(a => a.status === 'submitted' || a.status === 'closed')
    default:
      return props.assignments
  }
})

// 排序逻辑：过期 -> 紧急 -> 普通未完成 -> 已完成
const filteredAndSortedAssignments = computed(() => {
  return [...filteredAssignments.value].sort((a, b) => {
    const aOverdue = isOverdue(a)
    const bOverdue = isOverdue(b)
    const aUrgent = isUrgent(a)
    const bUrgent = isUrgent(b)
    const aCompleted = a.status === 'submitted' || a.status === 'closed'
    const bCompleted = b.status === 'submitted' || b.status === 'closed'

    // 过期的排在最前面
    if (aOverdue && !bOverdue) return -1
    if (!aOverdue && bOverdue) return 1

    // 紧急的排在第二位
    if (aUrgent && !bUrgent) return -1
    if (!aUrgent && bUrgent) return 1

    // 已完成的排在最后面
    if (aCompleted && !bCompleted) return 1
    if (!aCompleted && bCompleted) return -1

    // 同类型内按截止日期排序
    return new Date(a.dueDate).getTime() - new Date(b.dueDate).getTime()
  })
})

const getAssignmentRowClass = (assignment: Assignment) => {
  if (isOverdue(assignment)) {
    return 'bg-gray-100/50 dark:bg-gray-700/30'
  }
  if (isUrgent(assignment)) {
    return 'bg-orange-50/50 dark:bg-orange-900/10'
  }
  return ''
}

const getAssignmentCardClass = (assignment: Assignment) => {
  if (isOverdue(assignment)) {
    return 'border-gray-400 dark:border-gray-600 bg-gradient-to-br from-gray-100 to-gray-50 dark:from-gray-800/50 dark:to-gray-700/30 shadow-gray-300 dark:shadow-gray-900/30'
  }
  if (isUrgent(assignment)) {
    return 'border-orange-300 dark:border-orange-700 bg-gradient-to-br from-orange-50 to-yellow-50 dark:from-orange-900/20 dark:to-yellow-900/20 shadow-orange-200 dark:shadow-orange-900/20'
  }
  return ''
}

const getCardHoverClass = (assignment: Assignment) => {
  if (isOverdue(assignment)) {
    return 'hover:border-gray-500 dark:hover:border-gray-400'
  }
  if (isUrgent(assignment)) {
    return 'hover:border-orange-400 dark:hover:border-orange-300'
  }
  return 'hover:border-indigo-300 dark:hover:border-indigo-500'
}

const getAssignmentTitleClass = (assignment: Assignment) => {
  if (isOverdue(assignment)) {
    return 'text-gray-600 dark:text-gray-400'
  }
  if (isUrgent(assignment)) {
    return 'text-orange-900 dark:text-orange-100'
  }
  return 'text-gray-900 dark:text-white'
}

const getStatusIndicatorClass = (assignment: Assignment) => {
  const baseClasses = ['w-3 h-3 rounded-full']
  const statusColor = props.statusColors[assignment.status] || 'bg-gray-500'

  if (isOverdue(assignment)) {
    baseClasses.push('bg-gray-500 dark:bg-gray-400')
  } else if (isUrgent(assignment)) {
    baseClasses.push('bg-orange-500', 'group-hover:animate-pulse', 'transition-all', 'duration-200')
  } else {
    baseClasses.push(statusColor)
  }

  return baseClasses
}

const getStatusBadgeClass = (assignment: Assignment) => {
  const baseClasses = ['inline-flex items-center px-3 py-1.5 rounded-full text-xs font-medium transition-colors duration-200']

  if (isOverdue(assignment)) {
    baseClasses.push('bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-300')
  } else if (isUrgent(assignment)) {
    baseClasses.push('bg-orange-100 dark:bg-orange-900/30 text-orange-800 dark:text-orange-200 border border-orange-300 dark:border-orange-700')
  } else {
    const statusBadgeColor = props.statusBadgeColors[assignment.status] || 'bg-gray-100 dark:bg-gray-900/30 text-gray-800 dark:text-gray-200'
    baseClasses.push(statusBadgeColor)
  }

  return baseClasses
}

const getStatusText = (assignment: Assignment) => {
  // 不在这里返回紧急/过期状态，这些在模板中单独处理
  const statusKey = assignment.status
  const statusTranslations = {
    submitted: t('common.submitted') || '已提交',
    pending: t('common.pending') || '待提交',
    active: t('common.active') || '进行中',
    draft: t('common.draft') || '草稿',
    closed: t('common.closed') || '已结束'
  }
  return statusTranslations[statusKey] || assignment.status
}

const getProgressPercentage = (assignment: Assignment) => {
  const submitted = assignment.submitted || assignment.classSubmitted || 0
  const total = assignment.totalStudents || assignment.classTotal || 1
  return Math.round((submitted / total) * 100)
}
</script>

<style scoped>
/* 自定义滚动条 */
.max-h-\[60vh\]::-webkit-scrollbar,
.max-h-\[600px\]::-webkit-scrollbar {
  width: 6px;
}

.max-h-\[60vh\]::-webkit-scrollbar-track,
.max-h-\[600px\]::-webkit-scrollbar-track {
  background: transparent;
}

.max-h-\[60vh\]::-webkit-scrollbar-thumb,
.max-h-\[600px\]::-webkit-scrollbar-thumb {
  background: #d1d5db;
  border-radius: 3px;
}

.dark .max-h-\[60vh\]::-webkit-scrollbar-thumb,
.dark .max-h-\[600px\]::-webkit-scrollbar-thumb {
  background: #4b5563;
}

/* 移动端触摸优化 */
@media (max-width: 640px) {
  .touch-manipulation {
    touch-action: manipulation;
    -webkit-tap-highlight-color: transparent;
  }
}
</style>
