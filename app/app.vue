<script setup>
// Imports
import cronstrue from 'cronstrue'
import config from '@/assets/config.json'

// Import Defined Data Sources (adds custom option for dropdown)
const definedDataSources = ref([...config.dataSources, {id: 99999, label: 'Custom', e5: false, free: false}])

// Cookie Settings
const helpPopupShown = useCookie('helpPopupShown')

// Setup Local Storage
const dataSources = useLocalStorage('data-sources', [])
const commitTier = useLocalStorage('commit-tier', 0)

// Component Hookups
const addModelOpen = ref(false)
const helpModalOpen = ref(false)
const clearModalOpen = ref(false)

// Fixed Component Setup
const analyticsCommitTiers = ref([
  {
    label: 'Pay as you go',
    cost: 4.3,
    cap: 0,
    onSelect: () => commitTier.value = 0
  },
  {
    label: '100 GB per Day',
    cost: 2.96,
    daily: 296,
    cap: 100,
    onSelect: () => commitTier.value = 1
  },
  {
    label: '200 GB per Day',
    cost: 2.74,
    daily: 548,
    cap: 200,
    onSelect: () => commitTier.value = 2
  },
  {
    label: '300 GB per Day',
    cost: 2.666,
    daily: 800,
    cap: 300,
    onSelect: () => commitTier.value = 3
  },
  {
    label: '400 GB per Day',
    cost: 2.593,
    daily: 1037.33,
    cap: 400,
    onSelect: () => commitTier.value = 4
  },
  {
    label: '500 GB per Day',
    cost: 2.53,
    daily: 1265,
    cap: 500,
    onSelect: () => commitTier.value = 5
  },
  {
    label: '1000 GB per Day',
    cost: 2.48,
    daily: 2480,
    cap: 1000,
    onSelect: () => commitTier.value = 6
  },
  {
    label: '2000 GB per Day',
    cost: 2.4,
    daily: 4800,
    cap: 2000,
    onSelect: () => commitTier.value = 7
  },
  {
    label: '5000 GB per Day',
    cost: 2.31,
    daily: 11550,
    cap: 5000,
    onSelect: () => commitTier.value = 8
  },
  {
    label: '10000 GB per Day',
    cost: 2.224,
    daily: 22240,
    cap: 10000,
    onSelect: () => commitTier.value = 9
  },
  {
    label: '25000 GB per Day',
    cost: 2.138,
    daily: 53450,
    cap: 25000,
    onSelect: () => commitTier.value = 10
  },
  {
    label: '50000 GB per Day',
    cost: 2.052,
    daily: 102600,
    cap: 50000,
    onSelect: () => commitTier.value = 11
  }
])
const dataLakeStorageCost = 0.026
const dataLakeIngestionCost = 0.05
const dataLakeProcessingCost = 0.1
const dataLakeQueryCost = 0.005

// Computed Data
const totalMb = computed(() => {
  let totalMb = 0
  for (let i = 0; i < dataSources.value.length; i++) {
    totalMb += dataSources.value[i].ingestPerDayMb
  }
  return totalMb
})

const analyticMb = computed(() => {
  let totalMb = 0
  const analyticSources = dataSources.value.filter(d => !d.dataLake)
  for (let i = 0; i < analyticSources.length; i++) {
    totalMb += analyticSources[i].ingestPerDayMb
  }
  return totalMb
})

const chargedAnalyticMb = computed(() => {
  let totalMb = 0
  const analyticSources = dataSources.value.filter(d => !d.dataLake)
  for (let i = 0; i < analyticSources.length; i++) {
    if (analyticSources[i].free) {
      continue
    }
    totalMb += analyticSources[i].ingestPerDayMb
  }
  return totalMb
})

// Need to update this to compute additional storage without affecting other metrics
const dataLakeMb = computed(() => {
  let totalMb = 0
  const dataLakeSources = dataSources.value.filter(d => d.dataLake)
  for (let i = 0; i < dataLakeSources.length; i++) {
    totalMb += dataLakeSources[i].ingestPerDayMb
  }
  return totalMb
})

const dataLakeStorageCostComputed = computed(() => {
  const currentMonth = dataLakeMb.value / 1024 * dataLakeStorageCost
  let retainedCost = 0
  const dataLakeSources = dataSources.value.filter(d => d.dataLake)
  for (let i = 0; i < dataLakeSources.length; i++) {
    retainedCost += (dataLakeSources[i].ingestPerDayMb / 1024) * dataLakeSources[i].retainedMonths * dataLakeStorageCost
  }
  return currentMonth + retainedCost
})

const dataLakeProcessingCostComputed = computed(() => {
  return dataLakeMb.value / 1024 * dataLakeProcessingCost
})

const dataLakeIngestionCostComputed = computed(() => {
  return dataLakeMb.value / 1024 * dataLakeIngestionCost
})

const dataLakeQueryCostComputed = computed(() => {
  return dataLakeMb.value / 1024 * dataLakeQueryCost
})

const dataLakeTotalCostComputed = computed(() => {
  return dataLakeStorageCostComputed.value + dataLakeProcessingCostComputed.value + dataLakeIngestionCostComputed.value
})

const overageComputed = computed(() => {
  if (commitTier.value < 0 || chargedAnalyticMb.value / 1024 < analyticsCommitTiers.value[commitTier.value].cap) return 0

  const overage = chargedAnalyticMb.value / 1024 - analyticsCommitTiers.value[commitTier.value].cap
  return overage * analyticsCommitTiers.value[commitTier.value].cost
})

const analyticStorageCostComputed = computed(() => {
  if (commitTier.value == 0) {
    return chargedAnalyticMb.value / 1024 * analyticsCommitTiers.value[0].cost
  } else {
    if (chargedAnalyticMb.value / 1024 < analyticsCommitTiers.value[commitTier.value].cap) {
      return analyticsCommitTiers.value[commitTier.value].daily
    } else {
      return analyticsCommitTiers.value[commitTier.value].daily + overageComputed.value
    }
  }
})

const totalCostComputed = computed(() => {
  return dataLakeTotalCostComputed.value + analyticStorageCostComputed.value
})

// UI Overrides
const analyticChipUiOverride = {
  base: 'bg-green-400'
}

const dataLakeChipUiOverride = {
  base: 'bg-blue-400'
}

// Form Inputs
const in_newDataSourceName = ref('')
const in_newDataSourceIngestMb = ref(0)
const in_newDataSourceDataLake = ref(false)
const in_e5Benefit = ref(false)
const in_dataSource = ref('')

// Functions
function addDataSource() {
  const dataSourceConfig = definedDataSources.value.filter(d => d.label == in_dataSource.value)[0]
  dataSources.value.push({
    id: dataSources.value.length + 1,
    name: in_dataSource.value === 'Custom' ? in_newDataSourceName.value : in_dataSource.value,
    ingestPerDayMb: in_newDataSourceIngestMb.value,
    dataLake: in_newDataSourceDataLake.value,
    retainedMonths: 0,
    e5: dataSourceConfig ? dataSourceConfig.e5 : false,
    free: dataSourceConfig ? dataSourceConfig.free : false
  })

  addModelOpen.value = false
  in_dataSource.value = ''
  in_newDataSourceName.value = ''
  in_newDataSourceIngestMb.value = 0
  in_newDataSourceDataLake.value = false
}

function removeDataSource(id) {
  const idx = dataSources.value.findIndex(d => d.id === id)
  dataSources.value = dataSources.value.toSpliced(idx, 1)
}

function getReadableCron(exp) {
  try {
    return cronstrue.toString(exp)
  } catch (e) {
    return e
  }
}

function clearStorage() {
  dataSources.value = []
  commitTier.value = 0
  clearModalOpen.value = false
}

// Page load steps
onMounted(() => {
  if (!(helpPopupShown.value)) {
    helpPopupShown.value = true
    helpModalOpen.value = true
  }
})
</script>

<template>
  <UApp>
    <UBanner color="warning" icon="i-lucide-info" title="These are rough estimates. Actual cost will vary based on agreement with Microsoft."></UBanner>
    <UHeader title="Microsoft Sentinel Ingest Planner">
      <template #right>
        <div class="flex grow space-x-4 items-center">
          <div class="flex grow flex-col w-full space-y-1">
            <div class="flex items-center justify-between">
              <p>Tier Breakdown</p>
              <div class="flex space-x-6">
                <div class="flex items-center space-x-2">
                  <UChip :ui="analyticChipUiOverride"></UChip>
                  <p class="italic">Analytic Tier</p>
                </div>
                <div class="flex items-center space-x-2">
                  <UChip :ui="dataLakeChipUiOverride"></UChip>
                  <p class="italic">Data Lake Tier</p>
                </div>
              </div>
            </div>
            <div class="flex grow w-full h-2 bg-gray-500 rounded-xl overflow-hidden">
                <!-- Analytic Tier -->
                <div class="bg-green-400" :style="{ width: (analyticMb / totalMb) * 100 + '%'}"></div>
                <!-- Data Lake Tier -->
                <div class="bg-blue-400" :style="{ width: (dataLakeMb / totalMb) * 100 + '%'}"></div>
            </div>
          </div>
          <UModal v-model:open="helpModalOpen" title="How to use this calculator">
            <UTooltip text="Help">
              <UButton icon="lucide:circle-question-mark" variant="outline" color="neutral"></UButton>

            </UTooltip>
            <template #body>
              <p>This site is designed to help you calculate Microsoft Sentinel Ingest Costs to better plan your architecture. The calculator is broken up into Analytic and Data Lake tiers to reflect how Sentinel looks at data.</p>
              <br>
              <p>To add a log source, use the "Add Log Source" button in the top right. This is where you will define which tier you want to place the log source in as well as how much data is ingested per day. You can move these between the different tiers if needed.</p>
              <br>
              <p>For the Analytic Tier, you can select a commitment tier which helps cut down ingest cost if you have a known level of ingest per day. When one is selected, the cost breakdown will show the daily rate as well as the overage calculation.</p>
              <br>
              <p>Currently the Data Lake Tier calculations only look at ingest, processing, and storage. This will be updated in the future to allow factoring in querying.</p>
              <br>
              <p>The monthly cost breakdown will be on the side and dynamically updates as you enter information into the calculator. The sidebar also has links to Microsoft resources to help understand the pricing.</p>
              <br>
              <p>The meter at the top of the page shows the breakdown of data between the two tiers based on the data sources added.</p>
            </template>
          </UModal>
          <UModal v-model:open="clearModalOpen" title="Reset">
            <UTooltip text="Reset">
              <UButton icon="i-lucide-trash" color="error"></UButton>
            </UTooltip>
            <template #body>
              <p>Are you sure you want to clear all data?</p>
            </template>
            <template #footer>
              <div class="w-full flex">
                <UButton @click="clearStorage">Confirm</UButton>
              </div>
            </template>
          </UModal>
        </div>
      </template>
    </UHeader>
    <UMain>
      <UPage>
        <UPageBody>
          <UContainer class="space-y-8">
            <div class="flex justify-between items-center">
              <div class="flex items-center space-x-4">
                <h1 class="text-2xl">Analytics Tier</h1>
                <UDropdownMenu :items="analyticsCommitTiers">
                  <UButton variant="outline" color="neutral" icon="lucide:chevron-down"><b>Commitment Tier:</b> {{ analyticsCommitTiers[commitTier].label }}</UButton>
                </UDropdownMenu>
              </div>
              <UModal v-model:open="addModelOpen" title="Add Data Source">
                <UButton icon="lucide:plus">Add Data Source</UButton>

                <template #body>
                  <div class="space-y-4">
                    <UFormField label="Data Source">
                      <USelect v-model="in_dataSource" value-key="label" class="w-full" :items="definedDataSources" trailing-icon="i-lucide-chevron-down"></USelect>
                    </UFormField>
                    <p v-if="in_dataSource && definedDataSources.filter(d => d.label === in_dataSource)[0].e5" class="italic">This log source qualifies for the E5 benefit</p>
                    <p v-if="in_dataSource && definedDataSources.filter(d => d.label === in_dataSource)[0].free" class="italic">Ingest for this log source is not charged</p>
                    <UFormField v-if="in_dataSource == 'Custom'" label="Data Source Name" required>
                      <UInput v-model="in_newDataSourceName" class="w-full"></UInput>
                    </UFormField>
                    <UFormField label="Ingest (MB) / day" required>
                      <UInputNumber v-model="in_newDataSourceIngestMb" class="w-full"></UInputNumber>
                    </UFormField>
                    <UFormField>
                      <USwitch v-model="in_newDataSourceDataLake" label="Data Lake"></USwitch>
                    </UFormField>
                  </div>
                </template>

                <template #footer>
                  <UButton :disabled="(in_dataSource == '' || (in_dataSource == 'Custom' && in_newDataSourceName == '') || in_newDataSourceIngestMb < 0)" @click="addDataSource">Add</UButton>
                </template>
              </UModal>
            </div>
            <UPageGrid>
              <p v-if="dataSources.filter(d => !d.dataLake).length==0" class="italic">No data sources added.</p>
              <UCard v-else v-for="d in dataSources.filter(d => !d.dataLake)">
                <template #header>
                  <div class="space-y-2">
                    <h1 class="text-xl">{{ d.name }}</h1>
                    <UBadge v-if="d.free" color="success">Free</UBadge>
                    <UBadge v-if="d.e5" color="info">E5 Benefit</UBadge>
                  </div>
                </template>
                <div>
                  <UFormField label="Ingest (MB) / day">
                    <UInputNumber
                      v-model="d.ingestPerDayMb"
                      class="flex grow"
                      placeholder="Size in MB"
                    ></UInputNumber>
                  </UFormField>
                </div>
                <template #footer>
                  <div class="flex justify-between items-center">
                    <UButton @click="removeDataSource(d.id)" variant="outline" color="error">Remove</UButton>
                    <UButton @click="d.dataLake = true">Move to Data Lake</UButton>
                  </div>
                </template>
              </UCard>
            </UPageGrid>
            <h1 class="text-2xl">Data Lake Tier</h1>
            <UPageGrid>
              <p v-if="dataSources.filter(d => d.dataLake).length==0" class="italic">No data sources added.</p>
              <UCard v-else v-for="d in dataSources.filter(d => d.dataLake)">
                <template #header>
                  <h1 class="text-xl">{{ d.name }}</h1>
                </template>
                <div class="space-y-4">
                  <UFormField label="Ingest (MB) / day">
                    <UInputNumber
                      v-model="d.ingestPerDayMb"
                      class="flex grow"
                      placeholder="Size in MB"
                    ></UInputNumber>
                  </UFormField>
                  <UFormField label="Months of retention" help="Added to current month">
                    <UInputNumber
                      v-model="d.retainedMonths"
                      class="flex grow"
                      placeholder="12"
                    ></UInputNumber>
                  </UFormField>
                  <!-- Review adding at a later date -->
                  <!-- <UFormField label="How often will this data be queried" description="Enter as Cron Expression" :help=getReadableCron(d.cron)>
                    <UInput v-model="d.cron" class="w-full"></UInput>
                  </UFormField> -->
                </div>
                <template #footer>
                  <div class="flex justify-between items-center">
                    <UButton @click="removeDataSource(d.id)" variant="outline" color="error">Remove</UButton>
                    <UButton @click="d.dataLake = false">Move to Analytic</UButton>
                  </div>
                </template>
              </UCard>
            </UPageGrid>
          </UContainer>
        </UPageBody>
        <template #right>
          <UPageAside>
            <h1 class="text-2xl mb-5">Monthly Cost Breakdown</h1>
            <div class="space-y-4">
              <div class="space-y-2">
                <h2 class="text-xl">Analytics Tier - {{ (chargedAnalyticMb / 1024 * 31).toFixed(2) }}GB</h2>
                <div class="flex justify-between">
                  <p>{{ analyticsCommitTiers[commitTier].label }}</p>
                  <p v-if="commitTier === 0">${{ analyticsCommitTiers[commitTier].cost.toFixed(2) }}/GB</p>
                  <p v-else>${{ analyticsCommitTiers[commitTier].daily.toFixed(2)}}</p>
                </div>
                <div v-if="commitTier > 0" class="flex justify-between">
                  <p>Overage</p>
                  <p>${{ overageComputed.toFixed(2) }}</p>
                </div>
                <div class="flex justify-between">
                  <p class="bold">Total</p>
                  <p>${{ analyticStorageCostComputed.toFixed(2) }}</p>
                </div>
              </div>
              <div class="space-y-2">
                <h2 class="text-xl">Data Lake Tier - {{ (dataLakeMb / 1024 * 31).toFixed(2) }}GB</h2>
                <div class="flex justify-between">
                  <p>Ingestion</p>
                  <p>${{ dataLakeIngestionCostComputed.toFixed(2) }}</p>
                </div>
                <div class="flex justify-between">
                  <p>Processing</p>
                  <p>${{ dataLakeProcessingCostComputed.toFixed(2) }}</p>
                </div>
                <div class="flex justify-between">
                  <p>Storage</p>
                  <p>${{ dataLakeStorageCostComputed.toFixed(2) }}</p>
                </div>
                <div class="flex justify-between">
                  <p class="bold">Total</p>
                  <p>${{ dataLakeTotalCostComputed.toFixed(2) }}</p>
                </div>
              </div>
              <USeparator></USeparator>
              <div>
                <h2 class="text-xl">Total Cost - ${{ totalCostComputed.toFixed(2) }}</h2>
                <p>{{ (totalMb / 1024 * 31).toFixed(2) }}GB</p>
              </div>
            </div>
          </UPageAside>
        </template>
      </UPage>
    </UMain>
    <UFooter>
      <template #left>
        <p class="text-muted text-sm">Copyright © {{ new Date().getFullYear() }}</p>
      </template>
      <template #right>
        <UButton
          icon="i-simple-icons-github"
          color="neutral"
          variant="ghost"
          to="https://github.com/nuxt/nuxt"
          target="_blank"
          aria-label="GitHub"
        ></UButton>
      </template>
    </UFooter>
  </UApp>
</template>
