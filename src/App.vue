<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue'

const STORAGE_KEY = 'asset-snap-store'

const accountTypes = [
  { value: 'ASSET', label: '资产' },
  { value: 'LIABILITY', label: '负债' },
]

const subTypes = {
  ASSET: [
    { value: 'CASH', label: '现金/存款' },
    { value: 'INVEST', label: '投资' },
  ],
  LIABILITY: [
    { value: 'CREDIT', label: '信用/消费贷' },
    { value: 'LOAN', label: '长期贷款' },
  ],
}

const state = reactive({
  accounts: [],
  snapshots: [],
})

const activeTab = ref('snapshot')
const notice = ref('')
const editingAccountId = ref(null)
const fileInput = ref(null)

const accountForm = reactive({
  name: '',
  type: 'ASSET',
  subType: 'CASH',
})

const snapshotForm = reactive({
  date: todayDate(),
  note: '',
  balances: {},
  investmentPnl: {},
})

const activeChartMetric = ref('netWorth')
const selectedSnapshotId = ref('')

onMounted(() => {
  loadStore()
  hydrateSnapshotForm()
})

watch(
  () => [state.accounts, state.snapshots],
  () => saveStore(),
  { deep: true },
)

watch(
  () => accountForm.type,
  (type) => {
    accountForm.subType = subTypes[type][0].value
  },
)

watch(activeTab, (tab) => {
  if (tab === 'snapshot') hydrateSnapshotForm()
})

const activeAccounts = computed(() => state.accounts.filter((account) => !account.deletedAt))
const accountMap = computed(() => new Map(state.accounts.map((account) => [account.id, account])))
const assetAccounts = computed(() => activeAccounts.value.filter((account) => account.type === 'ASSET'))
const liabilityAccounts = computed(() =>
  activeAccounts.value.filter((account) => account.type === 'LIABILITY'),
)
const investmentAccounts = computed(() =>
  activeAccounts.value.filter((account) => account.type === 'ASSET' && account.subType === 'INVEST'),
)

const sortedSnapshots = computed(() =>
  [...state.snapshots].sort((a, b) => parseDate(a.date).getTime() - parseDate(b.date).getTime()),
)

const latestSnapshot = computed(() => sortedSnapshots.value.at(-1) || null)
const latestMetrics = computed(() => calculateMetrics(latestSnapshot.value))
const previousMetrics = computed(() => {
  const previous = sortedSnapshots.value.at(-2)
  return calculateMetrics(previous)
})

watch(latestSnapshot, (snapshot) => {
  if (snapshot && !selectedSnapshotId.value) selectedSnapshotId.value = snapshot.id
})

const metricCards = computed(() => [
  { label: '当前净资产', value: latestMetrics.value.netWorth, strong: true },
  { label: '总资产', value: latestMetrics.value.totalAssets },
  { label: '总负债', value: latestMetrics.value.totalLiabilities, negative: true },
  { label: '较上次变化', value: latestMetrics.value.netWorth - previousMetrics.value.netWorth, delta: true },
  { label: '投资账面', value: latestMetrics.value.investmentBook },
  { label: '浮盈/浮亏', value: latestMetrics.value.investmentPnl, delta: true },
  {
    label: '较上次净投入估算',
    value: latestMetrics.value.investmentCost - previousMetrics.value.investmentCost,
    delta: true,
  },
  { label: '投资成本估算', value: latestMetrics.value.investmentCost },
  { label: '现金及存款', value: latestMetrics.value.cash },
])

const periodCards = computed(() => {
  if (!latestSnapshot.value) return []

  return [
    buildPeriodCard('本周变化', startOfWeek(latestSnapshot.value.date)),
    buildPeriodCard('本月变化', startOfMonth(latestSnapshot.value.date)),
    buildPeriodCard('本年变化', startOfYear(latestSnapshot.value.date)),
  ].filter(Boolean)
})

const historyRows = computed(() =>
  sortedSnapshots.value
    .map((snapshot, index, snapshots) => {
      const metrics = calculateMetrics(snapshot)
      const previous = index > 0 ? calculateMetrics(snapshots[index - 1]) : null
      return {
        ...snapshot,
        metrics,
        delta: previous ? metrics.netWorth - previous.netWorth : 0,
        pnlDelta: previous ? metrics.investmentPnl - previous.investmentPnl : 0,
        netInvestmentDelta: previous ? metrics.investmentCost - previous.investmentCost : 0,
      }
    })
    .reverse(),
)

const selectedSnapshot = computed(() => {
  if (!sortedSnapshots.value.length) return null
  return (
    sortedSnapshots.value.find((snapshot) => snapshot.id === selectedSnapshotId.value) ||
    latestSnapshot.value
  )
})

const selectedMetrics = computed(() => calculateMetrics(selectedSnapshot.value))

const selectedPreviousSnapshot = computed(() => {
  const index = sortedSnapshots.value.findIndex((snapshot) => snapshot.id === selectedSnapshot.value?.id)
  return index > 0 ? sortedSnapshots.value[index - 1] : null
})

const accountBreakdown = computed(() => {
  const snapshot = selectedSnapshot.value
  if (!snapshot) return []

  return state.accounts
    .filter((account) => snapshot.balances && Object.hasOwn(snapshot.balances, account.id))
    .map((account) => {
      const balance = Math.abs(Number(snapshot.balances[account.id]) || 0)
      const pnl = account.subType === 'INVEST' ? getSnapshotPnl(snapshot, account.id) : null
      const previousBalance = Math.abs(Number(selectedPreviousSnapshot.value?.balances?.[account.id]) || 0)
      const previousPnl =
        account.subType === 'INVEST' ? getSnapshotPnl(selectedPreviousSnapshot.value, account.id) : null
      const cost = pnl === null ? null : balance - pnl
      const previousCost = previousPnl === null ? null : previousBalance - previousPnl
      return {
        ...account,
        balance,
        pnl,
        cost,
        netInvestmentDelta: cost === null || previousCost === null ? null : cost - previousCost,
        weight: selectedMetrics.value.totalAssets
          ? (balance / selectedMetrics.value.totalAssets) * 100
          : 0,
      }
    })
    .sort((a, b) => b.balance - a.balance)
})

const investmentBreakdown = computed(() =>
  accountBreakdown.value.filter((account) => account.type === 'ASSET' && account.subType === 'INVEST'),
)

const chartOptions = [
  { value: 'netWorth', label: '净资产' },
  { value: 'assetsLiabilities', label: '资产/负债' },
  { value: 'investmentPnl', label: '浮盈/浮亏' },
]

const chartPoints = computed(() => {
  if (!sortedSnapshots.value.length) return { series: [], labels: [] }

  const rows = sortedSnapshots.value.map((snapshot, index, snapshots) => {
    const metrics = calculateMetrics(snapshot)
    const previous = index > 0 ? calculateMetrics(snapshots[index - 1]) : null
    return {
      date: snapshot.date,
      netWorth: metrics.netWorth,
      delta: previous ? metrics.netWorth - previous.netWorth : 0,
      totalAssets: metrics.totalAssets,
      totalLiabilities: metrics.totalLiabilities,
      investmentPnl: metrics.investmentPnl,
      timestamp: parseDate(snapshot.date).getTime(),
    }
  })

  const config = {
    netWorth: [
      { key: 'netWorth', label: '净资产', className: 'worth-line' },
      { key: 'delta', label: '较上次变化', className: 'delta-line' },
    ],
    assetsLiabilities: [
      { key: 'totalAssets', label: '总资产', className: 'worth-line' },
      { key: 'totalLiabilities', label: '总负债', className: 'liability-line' },
    ],
    investmentPnl: [{ key: 'investmentPnl', label: '浮盈/浮亏', className: 'worth-line' }],
  }[activeChartMetric.value]

  const values = rows.flatMap((row) => config.map((serie) => row[serie.key]))
  const min = Math.min(...values)
  const max = Math.max(...values)
  const range = max - min || 1
  const width = 760
  const height = 240
  const pad = 28
  const minTime = Math.min(...rows.map((row) => row.timestamp))
  const maxTime = Math.max(...rows.map((row) => row.timestamp))
  const timeRange = maxTime - minTime || 1

  const toPoint = (row, key) => {
    const x =
      rows.length === 1
        ? width / 2
        : pad + ((row.timestamp - minTime) / timeRange) * (width - pad * 2)
    const y = height - pad - ((row[key] - min) / range) * (height - pad * 2)
    return `${x.toFixed(1)},${y.toFixed(1)}`
  }

  return {
    series: config.map((serie) => ({
      ...serie,
      points: rows.map((row) => toPoint(row, serie.key)).join(' '),
    })),
    labels: rows.filter((_, index) => index === 0 || index === rows.length - 1),
  }
})

function parseDate(value) {
  const [year, month, day] = value.split('-').map(Number)
  return new Date(year, month - 1, day)
}

function todayDate() {
  return toDateString(new Date())
}

function toDateString(date) {
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

function toBackupStamp(date) {
  const day = toDateString(date).replaceAll('-', '')
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  const seconds = String(date.getSeconds()).padStart(2, '0')
  return `${day}_${hours}${minutes}${seconds}`
}

function startOfWeek(value) {
  const date = parseDate(value)
  const day = date.getDay() || 7
  date.setDate(date.getDate() - day + 1)
  return toDateString(date)
}

function startOfMonth(value) {
  const date = parseDate(value)
  date.setDate(1)
  return toDateString(date)
}

function startOfYear(value) {
  const date = parseDate(value)
  date.setFullYear(date.getFullYear(), 0, 1)
  return toDateString(date)
}

function buildPeriodCard(label, boundaryDate) {
  const baseline = [...sortedSnapshots.value]
    .reverse()
    .find((snapshot) => snapshot.date < boundaryDate)
  if (!baseline || !latestSnapshot.value) return null

  const baselineMetrics = calculateMetrics(baseline)
  return {
    label,
    fromDate: baseline.date,
    boundaryDate,
    netWorthDelta: latestMetrics.value.netWorth - baselineMetrics.netWorth,
    assetDelta: latestMetrics.value.totalAssets - baselineMetrics.totalAssets,
    liabilityDelta: latestMetrics.value.totalLiabilities - baselineMetrics.totalLiabilities,
    pnlDelta: latestMetrics.value.investmentPnl - baselineMetrics.investmentPnl,
    netInvestmentDelta: latestMetrics.value.investmentCost - baselineMetrics.investmentCost,
  }
}

function makeId(prefix) {
  if (globalThis.crypto?.randomUUID) return `${prefix}_${crypto.randomUUID()}`
  return `${prefix}_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`
}

function loadStore() {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (!raw) return

  try {
    const parsed = JSON.parse(raw)
    if (!validateStoreData(parsed)) {
      showNotice('本地数据结构异常，请先导出备份后检查。')
      return
    }
    state.accounts = parsed.accounts
    state.snapshots = parsed.snapshots
  } catch {
    showNotice('本地数据读取失败，已保持当前空状态。')
  }
}

function saveStore() {
  try {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({
        accounts: state.accounts,
        snapshots: state.snapshots,
      }),
    )
  } catch {
    showNotice('本地存储空间不足，请先导出 JSON 备份。')
  }
}

function showNotice(message) {
  notice.value = message
  window.clearTimeout(showNotice.timer)
  showNotice.timer = window.setTimeout(() => {
    notice.value = ''
  }, 2600)
}

function formatMoney(value) {
  const amount = Number(value) || 0
  return new Intl.NumberFormat('zh-CN', {
    style: 'currency',
    currency: 'CNY',
    maximumFractionDigits: 2,
  }).format(amount)
}

function subtypeLabel(value) {
  return [...subTypes.ASSET, ...subTypes.LIABILITY].find((item) => item.value === value)?.label || value
}

function validateStoreData(data) {
  if (!data || !Array.isArray(data.accounts) || !Array.isArray(data.snapshots)) return false

  const accountIds = new Set()
  const accountsValid = data.accounts.every((account) => {
    if (!account || typeof account !== 'object') return false
    if (typeof account.id !== 'string' || typeof account.name !== 'string') return false
    if (accountIds.has(account.id)) return false
    if (!['ASSET', 'LIABILITY'].includes(account.type)) return false
    if (![...subTypes.ASSET, ...subTypes.LIABILITY].some((item) => item.value === account.subType)) {
      return false
    }
    accountIds.add(account.id)
    return true
  })

  if (!accountsValid) return false

  return data.snapshots.every((snapshot) => {
    if (!snapshot || typeof snapshot !== 'object') return false
    if (typeof snapshot.id !== 'string' || !isValidDateString(snapshot.date)) return false
    if (!snapshot.balances || typeof snapshot.balances !== 'object' || Array.isArray(snapshot.balances)) {
      return false
    }
    const balancesValid = Object.entries(snapshot.balances).every(
      ([accountId, value]) => accountIds.has(accountId) && Number.isFinite(Number(value)),
    )
    if (!balancesValid) return false
    if (snapshot.investmentPnl === undefined) return true
    if (!snapshot.investmentPnl || typeof snapshot.investmentPnl !== 'object' || Array.isArray(snapshot.investmentPnl)) {
      return false
    }
    return Object.entries(snapshot.investmentPnl).every(
      ([accountId, value]) => accountIds.has(accountId) && Number.isFinite(Number(value)),
    )
  })
}

function isValidDateString(value) {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) return false
  return toDateString(parseDate(value)) === value
}

function calculateMetrics(snapshot) {
  if (!snapshot) {
    return {
      totalAssets: 0,
      totalLiabilities: 0,
      netWorth: 0,
      cash: 0,
      investmentBook: 0,
      investmentPnl: 0,
      investmentCost: 0,
      investmentPnlRate: 0,
    }
  }

  const totals = Object.entries(snapshot.balances || {}).reduce(
    (totals, [accountId, rawBalance]) => {
      const account = accountMap.value.get(accountId)
      if (!account) return totals

      const value = Math.abs(Number(rawBalance) || 0)
      if (account.type === 'LIABILITY') {
        totals.totalLiabilities += value
      } else if (account.subType === 'INVEST') {
        totals.totalAssets += value
        totals.investmentBook += value
        totals.investmentPnl += getSnapshotPnl(snapshot, accountId)
      } else {
        totals.totalAssets += value
        totals.cash += value
      }
      totals.netWorth = totals.totalAssets - totals.totalLiabilities
      return totals
    },
    {
      totalAssets: 0,
      totalLiabilities: 0,
      netWorth: 0,
      cash: 0,
      investmentBook: 0,
      investmentPnl: 0,
      investmentCost: 0,
      investmentPnlRate: 0,
    },
  )

  totals.investmentCost = totals.investmentBook - totals.investmentPnl
  totals.investmentPnlRate = totals.investmentCost
    ? (totals.investmentPnl / Math.abs(totals.investmentCost)) * 100
    : 0
  return totals
}

function hydrateSnapshotForm() {
  const latest = latestSnapshot.value
  snapshotForm.date = todayDate()
  snapshotForm.note = ''
  snapshotForm.balances = {}
  snapshotForm.investmentPnl = {}

  activeAccounts.value.forEach((account) => {
    snapshotForm.balances[account.id] = latest?.balances?.[account.id] ?? ''
    if (account.type === 'ASSET' && account.subType === 'INVEST') {
      snapshotForm.investmentPnl[account.id] = latest?.investmentPnl?.[account.id] ?? ''
    }
  })
}

function normalizeAmount(value) {
  const number = Number(value)
  if (!Number.isFinite(number)) return 0
  return Math.round(Math.abs(number) * 100) / 100
}

function normalizeSignedAmount(value) {
  if (value === '' || value === null || value === undefined) return null
  const number = Number(value)
  if (!Number.isFinite(number)) return null
  return Math.round(number * 100) / 100
}

function getSnapshotPnl(snapshot, accountId) {
  const value = snapshot?.investmentPnl?.[accountId]
  return Number.isFinite(Number(value)) ? Number(value) : 0
}

function findPreviousPnl(accountId, beforeDate) {
  return [...sortedSnapshots.value]
    .reverse()
    .find((snapshot) => snapshot.date < beforeDate && snapshot.investmentPnl?.[accountId] !== undefined)
    ?.investmentPnl?.[accountId]
}

function formatPercent(value) {
  return `${(Number(value) || 0).toFixed(1)}%`
}

function saveSnapshot() {
  if (!activeAccounts.value.length) {
    activeTab.value = 'accounts'
    showNotice('请先添加至少一个账户。')
    return
  }

  const existingIndex = state.snapshots.findIndex((snapshot) => snapshot.date === snapshotForm.date)
  if (
    existingIndex !== -1 &&
    !window.confirm(`已存在 ${snapshotForm.date} 的记录，要覆盖它吗？`)
  ) {
    return
  }

  const balances = {}
  const investmentPnl = {}
  activeAccounts.value.forEach((account) => {
    balances[account.id] = normalizeAmount(snapshotForm.balances[account.id])
    if (account.type === 'ASSET' && account.subType === 'INVEST') {
      const inputPnl = normalizeSignedAmount(snapshotForm.investmentPnl[account.id])
      investmentPnl[account.id] =
        inputPnl ?? findPreviousPnl(account.id, snapshotForm.date) ?? getSnapshotPnl({}, account.id)
    }
  })

  const snapshot = {
    id: existingIndex === -1 ? makeId('snap') : state.snapshots[existingIndex].id,
    date: snapshotForm.date,
    timestamp: parseDate(snapshotForm.date).getTime(),
    balances,
    investmentPnl,
    note: snapshotForm.note.trim(),
  }

  if (existingIndex === -1) state.snapshots.push(snapshot)
  else state.snapshots.splice(existingIndex, 1, snapshot)

  hydrateSnapshotForm()
  activeTab.value = 'dashboard'
  selectedSnapshotId.value = snapshot.id
  showNotice('快照已保存。')
}

function submitAccount() {
  const name = accountForm.name.trim()
  if (!name) {
    showNotice('账户名称不能为空。')
    return
  }

  if (editingAccountId.value) {
    const account = state.accounts.find((item) => item.id === editingAccountId.value)
    if (account) {
      account.name = name
      account.type = accountForm.type
      account.subType = accountForm.subType
    }
  } else {
    state.accounts.push({
      id: makeId('acc'),
      name,
      type: accountForm.type,
      subType: accountForm.subType,
      createdAt: Date.now(),
    })
  }

  resetAccountForm()
  hydrateSnapshotForm()
  showNotice('账户已保存。')
}

function editAccount(account) {
  editingAccountId.value = account.id
  accountForm.name = account.name
  accountForm.type = account.type
  accountForm.subType = account.subType
}

function resetAccountForm() {
  editingAccountId.value = null
  accountForm.name = ''
  accountForm.type = 'ASSET'
  accountForm.subType = 'CASH'
}

function deleteAccount(account) {
  if (!window.confirm(`删除「${account.name}」后，它不会再出现在新快照中，历史记录会保留。继续吗？`)) {
    return
  }
  account.deletedAt = Date.now()
  hydrateSnapshotForm()
  showNotice('账户已停用。')
}

function deleteSnapshot(snapshot) {
  if (!window.confirm(`确定删除 ${snapshot.date} 的快照吗？`)) return
  state.snapshots = state.snapshots.filter((item) => item.id !== snapshot.id)
  hydrateSnapshotForm()
  showNotice('快照已删除。')
}

function clearZero(event) {
  if (event.target.value === '0' || event.target.value === '0.00') event.target.value = ''
}

function exportData() {
  const payload = JSON.stringify({ accounts: state.accounts, snapshots: state.snapshots }, null, 2)
  const blob = new Blob([payload], { type: 'application/json' })
  const anchor = document.createElement('a')
  const stamp = toBackupStamp(new Date())

  anchor.href = URL.createObjectURL(blob)
  anchor.download = `assets_backup_${stamp}.json`
  anchor.click()
  URL.revokeObjectURL(anchor.href)
  showNotice('备份文件已导出。')
}

function openImportPicker() {
  fileInput.value?.click()
}

function importData(event) {
  const file = event.target.files?.[0]
  event.target.value = ''
  if (!file) return
  if (!window.confirm('导入会替换当前所有本地数据，确定继续吗？')) return

  const reader = new FileReader()
  reader.onload = () => {
    try {
      const parsed = JSON.parse(String(reader.result))
      if (!validateStoreData(parsed)) {
        showNotice('导入失败：JSON 结构不符合账户/快照格式。')
        return
      }
      state.accounts = parsed.accounts
      state.snapshots = parsed.snapshots
      hydrateSnapshotForm()
      showNotice('数据已导入。')
    } catch {
      showNotice('导入失败：文件不是有效 JSON。')
    }
  }
  reader.readAsText(file)
}
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <div>
        <p class="eyebrow">本地资产快照</p>
        <h1>Asset Snap</h1>
      </div>
      <div class="data-actions">
        <button class="ghost-button" type="button" @click="exportData">导出 JSON</button>
        <button class="dark-button" type="button" @click="openImportPicker">导入</button>
        <input ref="fileInput" hidden type="file" accept="application/json" @change="importData" />
      </div>
    </header>

    <nav class="tabs" aria-label="主导航">
      <button :class="{ active: activeTab === 'snapshot' }" @click="activeTab = 'snapshot'">记录快照</button>
      <button :class="{ active: activeTab === 'dashboard' }" @click="activeTab = 'dashboard'">总览</button>
      <button :class="{ active: activeTab === 'accounts' }" @click="activeTab = 'accounts'">账户设置</button>
    </nav>

    <p v-if="notice" class="notice">{{ notice }}</p>

    <section v-if="activeTab === 'snapshot'" class="workspace">
      <div class="section-head">
        <div>
          <p class="eyebrow">Snapshot</p>
          <h2>记录一次完整财务状态</h2>
        </div>
        <label class="date-field">
          日期
          <input v-model="snapshotForm.date" type="date" />
        </label>
      </div>

      <div v-if="!activeAccounts.length" class="empty-state">
        <h3>先添加你的金融账户</h3>
        <p>比如微信余额、工资卡、基金账户、信用卡或房贷。添加后，这里会自动生成录入表单。</p>
        <button class="dark-button" type="button" @click="activeTab = 'accounts'">去添加账户</button>
      </div>

      <form v-else class="snapshot-form" @submit.prevent="saveSnapshot">
        <div class="balance-grid" :class="{ single: !liabilityAccounts.length }">
          <section class="balance-group">
            <div class="group-title">
              <h3>资产</h3>
              <span>{{ assetAccounts.length }} 项</span>
            </div>
            <label v-for="account in assetAccounts" :key="account.id" class="money-field">
              <span>{{ account.name }}</span>
              <input
                v-model="snapshotForm.balances[account.id]"
                inputmode="decimal"
                min="0"
                step="0.01"
                type="number"
                @focus="clearZero"
              />
            </label>
          </section>

          <section v-if="liabilityAccounts.length" class="balance-group">
            <div class="group-title">
              <h3>负债</h3>
              <span>{{ liabilityAccounts.length }} 项</span>
            </div>
            <label v-for="account in liabilityAccounts" :key="account.id" class="money-field liability">
              <span>{{ account.name }}</span>
              <input
                v-model="snapshotForm.balances[account.id]"
                inputmode="decimal"
                min="0"
                step="0.01"
                type="number"
                @focus="clearZero"
              />
            </label>
          </section>
        </div>

        <section v-if="investmentAccounts.length" class="balance-group investment-pnl-group">
          <div class="group-title">
            <h3>投资浮盈/浮亏</h3>
            <span>留空则沿用上一次</span>
          </div>
          <label v-for="account in investmentAccounts" :key="account.id" class="money-field pnl-field">
            <span>{{ account.name }}</span>
            <input
              v-model="snapshotForm.investmentPnl[account.id]"
              inputmode="decimal"
              step="0.01"
              type="number"
              placeholder="可填正数或负数"
              @focus="clearZero"
            />
          </label>
        </section>

        <label class="note-field">
          备注
          <textarea v-model="snapshotForm.note" rows="3" placeholder="例如：发薪、还款、市场波动..." />
        </label>

        <button class="save-button" type="submit">保存快照</button>
      </form>
    </section>

    <section v-if="activeTab === 'dashboard'" class="workspace">
      <div class="section-head">
        <div>
          <p class="eyebrow">Dashboard</p>
          <h2>宏观资产总览</h2>
        </div>
      </div>

      <div v-if="!latestSnapshot" class="empty-state">
        <h3>还没有任何快照</h3>
        <p>先添加账户，再记录第一次快照；之后这里会显示净资产、变化和趋势。</p>
        <button class="dark-button" type="button" @click="activeTab = 'accounts'">先添加账户</button>
      </div>

      <template v-else>
        <div class="metric-grid">
          <article
            v-for="card in metricCards"
            :key="card.label"
            class="metric-card"
            :class="{ strong: card.strong, negative: card.negative, positive: card.delta && card.value > 0 }"
          >
            <span>{{ card.label }}</span>
            <strong>{{ formatMoney(card.value) }}</strong>
          </article>
        </div>

        <section v-if="periodCards.length" class="period-panel">
          <div class="group-title">
            <h3>周期变化</h3>
            <span>周一为周初，1 号为月初</span>
          </div>
          <div class="period-grid">
            <article v-for="card in periodCards" :key="card.label" class="period-card">
              <div class="period-card-head">
                <span>{{ card.label }}</span>
                <small>基准 {{ card.fromDate }}</small>
              </div>
              <strong :class="{ gain: card.netWorthDelta > 0, loss: card.netWorthDelta < 0 }">
                {{ formatMoney(card.netWorthDelta) }}
              </strong>
              <dl>
                <div>
                  <dt>资产</dt>
                  <dd :class="{ gain: card.assetDelta > 0, loss: card.assetDelta < 0 }">
                    {{ formatMoney(card.assetDelta) }}
                  </dd>
                </div>
                <div>
                  <dt>负债</dt>
                  <dd :class="{ loss: card.liabilityDelta > 0, gain: card.liabilityDelta < 0 }">
                    {{ formatMoney(card.liabilityDelta) }}
                  </dd>
                </div>
                <div>
                  <dt>浮盈/浮亏</dt>
                  <dd :class="{ gain: card.pnlDelta > 0, loss: card.pnlDelta < 0 }">
                    {{ formatMoney(card.pnlDelta) }}
                  </dd>
                </div>
                <div>
                  <dt>估算净投入</dt>
                  <dd :class="{ gain: card.netInvestmentDelta > 0, loss: card.netInvestmentDelta < 0 }">
                    {{ formatMoney(card.netInvestmentDelta) }}
                  </dd>
                </div>
              </dl>
            </article>
          </div>
        </section>

        <section class="chart-panel">
          <div class="group-title">
            <h3>趋势</h3>
            <div class="segmented-control">
              <button
                v-for="option in chartOptions"
                :key="option.value"
                type="button"
                :class="{ active: activeChartMetric === option.value }"
                @click="activeChartMetric = option.value"
              >
                {{ option.label }}
              </button>
            </div>
          </div>
          <svg class="trend-chart" viewBox="0 0 760 240" role="img" aria-label="净资产趋势图">
            <line x1="28" y1="212" x2="732" y2="212" />
            <polyline
              v-for="serie in chartPoints.series"
              :key="serie.key"
              :points="serie.points"
              :class="serie.className"
            />
            <circle
              v-for="point in chartPoints.series[0]?.points.split(' ').filter(Boolean)"
              :key="point"
              :cx="point.split(',')[0]"
              :cy="point.split(',')[1]"
              r="4"
            />
          </svg>
          <div class="chart-legend">
            <span v-for="serie in chartPoints.series" :key="serie.key">
              <i :class="serie.className"></i>{{ serie.label }}
            </span>
          </div>
        </section>

        <section class="insight-grid">
          <article class="insight-panel">
            <div class="group-title">
              <h3>快照详情</h3>
              <select v-model="selectedSnapshotId" class="compact-select">
                <option v-for="snapshot in sortedSnapshots" :key="snapshot.id" :value="snapshot.id">
                  {{ snapshot.date }}
                </option>
              </select>
            </div>
            <div class="detail-stats">
              <div>
                <span>投资占资产</span>
                <strong>
                  {{
                    formatPercent(
                      selectedMetrics.totalAssets
                        ? (selectedMetrics.investmentBook / selectedMetrics.totalAssets) * 100
                        : 0,
                    )
                  }}
                </strong>
              </div>
              <div>
                <span>浮盈/浮亏率</span>
                <strong :class="{ gain: selectedMetrics.investmentPnl > 0, loss: selectedMetrics.investmentPnl < 0 }">
                  {{ formatPercent(selectedMetrics.investmentPnlRate) }}
                </strong>
              </div>
            </div>
            <div class="breakdown-list">
              <div v-for="account in accountBreakdown" :key="account.id" class="breakdown-row">
                <div>
                  <strong>{{ account.name }}</strong>
                  <span>{{ account.type === 'ASSET' ? '资产' : '负债' }} / {{ subtypeLabel(account.subType) }}</span>
                </div>
                <div>
                  <strong>{{ formatMoney(account.balance) }}</strong>
                  <span v-if="account.type === 'ASSET'">{{ formatPercent(account.weight) }}</span>
                </div>
              </div>
            </div>
          </article>

          <article class="insight-panel">
            <div class="group-title">
              <h3>投资明细</h3>
              <span>{{ investmentBreakdown.length }} 项</span>
            </div>
            <div v-if="investmentBreakdown.length" class="breakdown-list">
              <div v-for="account in investmentBreakdown" :key="account.id" class="breakdown-row">
                <div>
                  <strong>{{ account.name }}</strong>
                  <span>成本估算 {{ formatMoney(account.cost) }}</span>
                </div>
                <div>
                  <strong :class="{ gain: account.pnl > 0, loss: account.pnl < 0 }">
                    {{ formatMoney(account.pnl) }}
                  </strong>
                  <span>账面 {{ formatMoney(account.balance) }}</span>
                  <span
                    v-if="account.netInvestmentDelta !== null"
                    :class="{ gain: account.netInvestmentDelta > 0, loss: account.netInvestmentDelta < 0 }"
                  >
                    净投入 {{ formatMoney(account.netInvestmentDelta) }}
                  </span>
                </div>
              </div>
            </div>
            <p v-else class="muted">暂无投资账户。</p>
          </article>
        </section>

        <section class="history-panel">
          <div class="group-title">
            <h3>历史快照</h3>
            <span>{{ historyRows.length }} 条</span>
          </div>
          <div class="table-wrap">
            <table>
              <thead>
                <tr>
                  <th>日期</th>
                  <th>净资产</th>
                  <th>变化</th>
                  <th>浮盈/浮亏</th>
                  <th>估算净投入</th>
                  <th>备注</th>
                  <th></th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="row in historyRows" :key="row.id">
                  <td>{{ row.date }}</td>
                  <td>{{ formatMoney(row.metrics.netWorth) }}</td>
                  <td :class="{ gain: row.delta > 0, loss: row.delta < 0 }">{{ formatMoney(row.delta) }}</td>
                  <td :class="{ gain: row.metrics.investmentPnl > 0, loss: row.metrics.investmentPnl < 0 }">
                    {{ formatMoney(row.metrics.investmentPnl) }}
                  </td>
                  <td :class="{ gain: row.netInvestmentDelta > 0, loss: row.netInvestmentDelta < 0 }">
                    {{ formatMoney(row.netInvestmentDelta) }}
                  </td>
                  <td class="note-cell">{{ row.note || '—' }}</td>
                  <td><button class="text-button" type="button" @click="deleteSnapshot(row)">删除</button></td>
                </tr>
              </tbody>
            </table>
          </div>
        </section>
      </template>
    </section>

    <section v-if="activeTab === 'accounts'" class="workspace">
      <div class="section-head">
        <div>
          <p class="eyebrow">Accounts</p>
          <h2>账户设置</h2>
        </div>
      </div>

      <form class="account-form" @submit.prevent="submitAccount">
        <label>
          名称
          <input v-model="accountForm.name" type="text" placeholder="例如：招行工资卡" />
        </label>
        <label>
          类型
          <select v-model="accountForm.type">
            <option v-for="type in accountTypes" :key="type.value" :value="type.value">{{ type.label }}</option>
          </select>
        </label>
        <label>
          分类
          <select v-model="accountForm.subType">
            <option v-for="type in subTypes[accountForm.type]" :key="type.value" :value="type.value">
              {{ type.label }}
            </option>
          </select>
        </label>
        <div class="form-actions">
          <button class="dark-button" type="submit">{{ editingAccountId ? '保存修改' : '添加账户' }}</button>
          <button v-if="editingAccountId" class="ghost-button" type="button" @click="resetAccountForm">取消</button>
        </div>
      </form>

      <div class="account-list">
        <article v-for="account in activeAccounts" :key="account.id" class="account-row">
          <div>
            <strong>{{ account.name }}</strong>
            <span>{{ account.type === 'ASSET' ? '资产' : '负债' }} / {{ subtypeLabel(account.subType) }}</span>
          </div>
          <div class="row-actions">
            <button class="ghost-button" type="button" @click="editAccount(account)">编辑</button>
            <button class="text-button" type="button" @click="deleteAccount(account)">删除</button>
          </div>
        </article>
        <p v-if="!activeAccounts.length" class="muted">暂无账户。添加后即可开始记录快照。</p>
      </div>
    </section>

    <footer class="site-footer">
      <a href="https://github.com/senzi/panorama" target="_blank" rel="noreferrer">senzi/panorama</a>
      <span>MIT</span>
      <span>Vibe coding by Codex</span>
    </footer>
  </main>
</template>
