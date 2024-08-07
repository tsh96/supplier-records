<script setup lang="ts" generic="T extends any, O extends any">
import { startOfMonth, startOfQuarter, startOfYear, format as formatDate } from 'date-fns';
import { chain, sumBy } from 'lodash';
import { type FormItemRule, type FormInst } from 'naive-ui'
import { v4 as uuidv4 } from 'uuid';

defineOptions({
  name: 'IndexPage',
})

type SupplierAccount = {
  supplierName: string,
  items: SupplierAccountItem[],
}

type SupplierAccountItem = {
  id: string
  invoiceDate: number | null,
  invoiceNo: string,
  invoiceAmount: number,
  chequeDate: number | null,
  chequeNo: string,
  chequeAmount: number,
  remark: string,
}

const version = useLocalStorage('version', '1.0.0')
const supplierAccounts = useLocalStorage('supplierAccounts', [] as SupplierAccount[])
const selectedSupplierName = useLocalStorage<string | null>('selectedSupplierName', null)

const suppliersOption = computed(() =>
  supplierAccounts.value.map((supplierAccount) => ({
    label: supplierAccount.supplierName,
    value: supplierAccount.supplierName,
  })).sort((a, b) => a.label.localeCompare(b.label))
)

const supplierAccount = computed(() => {
  const supplierName = selectedSupplierName.value
  if (!supplierName)
    return null
  const supplierAccount = supplierAccounts.value.find((supplierAccount) => supplierAccount.supplierName === supplierName)
  if (!supplierAccount)
    return null
  return supplierAccount
})

onMounted(() => {
  if (version.value === '1.0.0') {
    supplierAccounts.value.forEach((supplierAccount) => {
      supplierAccount.items.forEach((item) => {
        item.id = uuidv4()
      })
    })
    version.value = '1.1.0'
  }
})

const yearFilter = ref(new Date().getFullYear())

watch(() => supplierAccount.value?.items, (items) => {
  if (!items) return
  items.sort((a, b) => (a.invoiceDate || Infinity) - (b.invoiceDate || Infinity))
}, { deep: true })


function addItem() {
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  if (!supplierAccount.value) return
  supplierAccount.value.items.push({
    id: uuidv4(),
    invoiceDate: null,
    invoiceNo: '',
    invoiceAmount: 0,
    chequeDate: null,
    chequeNo: '',
    chequeAmount: 0,
    remark: '',
  })
}

const addSupplierForm = ref({
  supplierName: '',
  show: false
})

function showAddSupplierForm() {
  addSupplierForm.value.supplierName = ''
  addSupplierForm.value.show = true
}

const addSupplierRule = {
  supplierName: [
    {
      required: true,
      trigger: ['input', 'blur'],
      validator(rule: FormItemRule, value: string) {
        if (!value) {
          return new Error('Supplier Name is required')
        } else if (supplierAccounts.value.find((supplierAccount) => supplierAccount.supplierName === value)) {
          return new Error('Supplier Name already exists')
        }
        return true
      }
    },
  ],
}

const addSupplierFormRef = ref<FormInst>()

async function addSupplier() {
  addSupplierFormRef.value?.validate((errors) => {
    if (errors) return
    addSupplierForm.value.show = false
    const supplierName = addSupplierForm.value.supplierName
    if (!supplierName) return
    supplierAccounts.value.push({
      supplierName,
      items: [],
    })
    selectedSupplierName.value = supplierName
  })
}


const updateSupplierForm = ref({
  supplierName: '',
  show: false
})

function showUpdateSupplierForm() {
  updateSupplierForm.value.supplierName = selectedSupplierName.value || ''
  updateSupplierForm.value.show = true
}

const updateSupplierRule = {
  supplierName: [
    {
      required: true,
      trigger: ['input', 'blur'],
      validator(rule: FormItemRule, value: string) {
        if (!value) {
          return new Error('Supplier Name is required')
        } else if (supplierAccounts.value.find((supplierAccount) => supplierAccount.supplierName === value)) {
          if (value === selectedSupplierName.value)
            return true
          else
            return new Error('Supplier Name already exists')
        }
        return true
      }
    },
  ],
}

const updateSupplierFormRef = ref<FormInst>()

async function updateSupplier() {
  updateSupplierFormRef.value?.validate((errors) => {
    if (errors) return
    updateSupplierForm.value.show = false
    const supplierName = updateSupplierForm.value.supplierName
    if (!supplierName) return
    if (!supplierAccount.value) return
    supplierAccount.value.supplierName = supplierName
    selectedSupplierName.value = supplierName
  })
}

function deleteSupplier() {
  const supplierName = selectedSupplierName.value
  if (!supplierName) return
  const index = supplierAccounts.value.findIndex((supplierAccount) => supplierAccount.supplierName === supplierName)
  if (index === -1) return
  supplierAccounts.value.splice(index, 1)
  selectedSupplierName.value = null
}


function removeItem(id: string) {
  if (!supplierAccount.value) return
  const index = supplierAccount.value.items.findIndex((item) => item.id === id)
  if (index === -1) return
  supplierAccount.value.items.splice(index, 1)
}

const draggableRow = ref<number>()
useEventListener('mouseup', () => {
  draggableRow.value = -1
})
const draggingIndex = ref(-1)

function dropItem(index: number) {
  const items = supplierAccount.value?.items
  if (!items) return
  const draggingTransaction = items[draggingIndex.value]
  items.splice(draggingIndex.value, 1)
  items.splice(index, 0, draggingTransaction)
}

const invoiceDateFilter = ref<[number, number]>()
const chequeDateFilter = ref<[number, number]>()

const filteredSupplierItems = computed(() => {
  if (!supplierAccount.value) return []
  const items = supplierAccount.value.items.filter(item => !item.invoiceDate || startOfYear(item.invoiceDate).getFullYear() === yearFilter.value)
  if (!invoiceDateFilter.value && !chequeDateFilter.value) return items
  return items.filter(item => {
    if (invoiceDateFilter.value) {
      if (!item.invoiceDate) return false
      if (item.invoiceDate < invoiceDateFilter.value[0] || item.invoiceDate > invoiceDateFilter.value[1]) return false
    }
    if (chequeDateFilter.value) {
      if (!item.chequeDate) return false
      if (item.chequeDate < chequeDateFilter.value[0] || item.chequeDate > chequeDateFilter.value[1]) return false
    }
    return true
  })
})

const showStatisticModal = ref(false)
const statisticAmountType = ref<'invoice' | 'cheque'>('invoice')

const statisticPeriodType = ref<'all' | 'monthly' | 'quarterly' | 'yearly'>('all')
const statisticMonthlyPeriod = ref<number | null>()
const statisticQuarterlyPeriod = ref<number | null>()
const statisticYearlyPeriod = ref<number | null>()

const statisticPeriodOptions = computed(() => {
  const monthly = chain(supplierAccounts.value)
    .flatMap(supplierAccount => supplierAccount.items)
    .map(item => {
      if (statisticAmountType.value === 'invoice')
        return item.invoiceDate && startOfMonth(item.invoiceDate).getTime()
      return item.chequeDate && startOfMonth(item.chequeDate).getTime()
    })
    .compact()
    .uniq()
    .sortBy()
    .value()

  const quarterly = chain(monthly)
    .map(date => startOfQuarter(date).getTime())
    .uniq()
    .value()

  const yearly = chain(monthly)
    .map(date => startOfYear(date).getTime())
    .uniq()
    .value()

  return {
    monthly: monthly.map(date => ({
      label: formatDate(new Date(date), 'yyyy-MM'),
      value: date,
    })),
    quarterly: quarterly.map(date => ({
      label: formatDate(new Date(date), 'yyyy-QQQ'),
      value: date,
    })),
    yearly: yearly.map(date => ({
      label: formatDate(new Date(date), 'yyyy'),
      value: date,
    })),
  }
})


const statistic = computed(() => {
  switch (statisticPeriodType.value) {
    case 'all':
      break;
    case 'monthly':
      if (!statisticMonthlyPeriod.value) return null
      break;
    case 'quarterly':
      if (!statisticQuarterlyPeriod.value) return null
      break;
    case 'yearly':
      if (!statisticYearlyPeriod.value) return null
      break;
  }

  const suppliers = chain(supplierAccounts.value)
    .map((supplierAccount) => {
      return {
        supplierName: supplierAccount.supplierName,
        totalAmount: sumBy(supplierAccount.items, item => {

          if (statisticAmountType.value === 'invoice') {
            if (!item.invoiceDate) return 0
            switch (statisticPeriodType.value) {
              case 'all':
                break;
              case 'monthly':
                if (statisticMonthlyPeriod.value !== startOfMonth(item.invoiceDate).getTime()) return 0
                break;
              case 'quarterly':
                if (statisticQuarterlyPeriod.value !== startOfQuarter(item.invoiceDate).getTime()) return 0
                break;
              case 'yearly':
                if (statisticYearlyPeriod.value !== startOfYear(item.invoiceDate).getTime()) return 0
                break;
            }
            return item.invoiceAmount
          }
          else {
            if (!item.chequeDate) return 0
            switch (statisticPeriodType.value) {
              case 'all':
                break;
              case 'monthly':
                if (statisticMonthlyPeriod.value !== startOfMonth(item.chequeDate).getTime()) return 0
                break;
              case 'quarterly':
                if (statisticQuarterlyPeriod.value !== startOfQuarter(item.chequeDate).getTime()) return 0
                break;
              case 'yearly':
                if (statisticYearlyPeriod.value !== startOfYear(item.chequeDate).getTime()) return 0
                break;
            }
            return item.chequeAmount
          }
        })
      }
    })
    .sortBy(item => -item.totalAmount)
    .value()

  const totalAmount = sumBy(suppliers, item => item.totalAmount)

  return {
    suppliers,
    totalAmount,
  }
})

const showPendingInvoicesModal = ref(false)
const pendingInvoices = computed(() => {
  const suppliers = supplierAccounts.value
    .map(supplierAccount => ({
      supplierName: supplierAccount.supplierName,
      items: supplierAccount.items.filter(item => !item.chequeDate)
    }))
    .filter(supplierAccount => supplierAccount.items.length)

  const count = sumBy(suppliers, supplierAccount => supplierAccount.items.length)
  return {
    suppliers,
    count,
  }
})

function download() {
  const element = document.createElement('a')
  element.setAttribute('href', `data:text/plain;charset=utf-8,${encodeURIComponent(JSON.stringify(supplierAccounts.value))}`)
  const nowDate = new Date()
  const now = `${nowDate.getFullYear()}-${(nowDate.getMonth() + 1).toString().padStart(2, '0')}-${nowDate.getDate().toString().padStart(2, "0")} ${nowDate.getHours().toString().padStart(2, '0')}-${nowDate.getMinutes().toString().padStart(2, '0')}-${nowDate.getSeconds().toString().padStart(2, '0')}`
  element.setAttribute('download', `suppliers ${now}.json`)

  element.style.display = 'none'
  document.body.appendChild(element)

  element.click()

  document.body.removeChild(element)
}

function upload() {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = 'application/json'
  input.onchange = () => {
    if (input.files?.length) {
      const file = input.files[0]
      const reader = new FileReader()
      reader.onload = () => {
        const text = reader.result as string
        try {
          const json = JSON.parse(text)
          if (Array.isArray(json))
            supplierAccounts.value = json
          else
            throw new Error('Invalid JSON')
        }
        catch (e) {
          alert('Invalid JSON')
        }
      }
      reader.readAsText(file)
    }
  }
  input.click()
}

</script>

<template lang="pug">
.mx-8.flex.flex-col.h-screen.gap-y-2
  h1.gap-x-2.py-1(text="4xl" flex items-center) Supplier Records
    sub(text="sm") v1.0.1
    n-popconfirm(@positive-click="upload()")
      div
        div Restore supplier records from a file?
        div All unsaved changes will be lost.
        div Please backup your supplier records before restoring.
      template(#trigger)
        n-button(type="info") #[.i-carbon-upload] Restore
    n-button(@click="download()" type="success") #[.i-carbon-download] Backup
    n-button(@click="showStatisticModal=true" type="warning") #[.i-carbon-chart-bar] Statistics
    n-tooltip(:disabled="!!pendingInvoices.count") You have no pending invoices.
      template(#trigger)
        n-badge(:value="pendingInvoices.count" type="error")
          n-button(@click="showPendingInvoicesModal=true" type="error" :disabled="!pendingInvoices.count") #[.i-carbon-document] Pending Invoices
  .flex.gap-x-2.py-2.items-center
    .text-xl Year:
    n-input-number.font-mono(v-model:value="yearFilter" min="1900" max="3000" style="width: 180px; min-width: 180px;" placeholder="Year")
    n-divider(vertical)
    .text-xl Supplier:
    n-select(v-model:value="selectedSupplierName" :options="suppliersOption" size="large" filterable clearable)
    n-button(v-if="supplierAccount" @click="showUpdateSupplierForm()" type="info") Update Supplier
    n-popconfirm(v-if="supplierAccount" @positive-click="deleteSupplier()")
      span Are you sure to delete this supplier?
      template(#trigger)
        n-button(type="error") Delete Supplier
    n-button(type="primary" @click="showAddSupplierForm()") Add Supplier
  .overflow-auto.flex-grow.outline.outline-1.outline-grey.pb-100
    n-table(v-if="supplierAccount" :bordered="false" style="overflow: visible;" size="small")
      thead.sticky.top-0.z-3
        tr
          th.w-10.min-w-10
          th.w-38.min-w-38
            .flex.gap-x-2 Date
              n-popover(trigger="click")
                template(#trigger)
                  n-badge(:show="!!invoiceDateFilter" dot)
                    n-button(text) #[.i-carbon-filter]
                n-date-picker(v-model:value="invoiceDateFilter" type="daterange" clearable)
          th.w-50.min-w-50 Invoice No.
          th.w-50.min-w-50 Amt #[n-tag(type="success") SUM: {{ format(sumBy(filteredSupplierItems, item => item.invoiceAmount)) }}]
          th.w-38.min-w-38
            .flex.gap-x-2 Date
              n-popover(trigger="click")
                template(#trigger)
                  n-badge(:show="!!chequeDateFilter" dot)
                    n-button(text) #[.i-carbon-filter]
                n-date-picker(v-model:value="chequeDateFilter" type="daterange" clearable)
          th.w-50.min-w-50 Cheque No.
          th.w-50.min-w-50 Amt #[n-tag(type="success") SUM: {{ format(sumBy(filteredSupplierItems, item => item.chequeAmount)) }}]
          th.min-w-30 Remark
          th.w-10
            n-tooltip(:disabled="!invoiceDateFilter && !chequeDateFilter") Disabled when filter is applied.
              template(#trigger)
                n-button(text @click="addItem" :disabled="!!invoiceDateFilter || !!chequeDateFilter")
                  .i-carbon-add-alt(hover="bg-green-7")
      tbody(v-if="supplierAccount")
        transition-group(name="table-row")
          tr(
            v-for="item, i in filteredSupplierItems"
            :draggable="i === draggableRow"
            @dragstart="draggingIndex = i"
            @dragover.prevent
            @drop="dropItem(i)"
            :key="item.id"
          )
            td.cursor-move(@mousedown="draggableRow = i") {{ i+1 }}
            td
              n-date-picker(v-model:value="item.invoiceDate" type="date" size="small" clearable)
            td
              n-input(v-model:value="item.invoiceNo" size="small")
            td.text-right
              n-input-number(
                v-model:value="item.invoiceAmount"
                step="0.01"
                :format="format"
                :parse="parse"
                :show-button="false" placeholder=""
                size="small"
              )
            td
              n-date-picker(v-model:value="item.chequeDate" type="date" size="small" clearable)
            td
              n-input(v-model:value="item.chequeNo" size="small")
            td.text-right
              n-input-number(
                v-model:value="item.chequeAmount"
                step="0.01"
                :format="format"
                :parse="parse"
                :show-button="false" placeholder=""
                size="small"
              )
            td.text-right
              n-tooltip(:disabled="!item.remark")
                div {{ item.remark }}
                template(#trigger)
                  n-input(v-model:value="item.remark" size="small")
            td
              .i-carbon-subtract-alt(hover="bg-red-7 cursor-pointer" @click="removeItem(item.id)")
    .flex.flex-col.items-center.justify-center.h-full(v-else)
      n-empty(size="huge")
        div
          .text-center.text-3xl No Supplier Selected
          .text-center.text-3xl Please select a supplier to view its records.
          .text-center.text-3xl.flex.items-center You can add a new supplier by clicking
            n-button(type="primary" @click="showAddSupplierForm()" style="margin-left: 8px") Add Supplier
  n-modal(v-model:show="addSupplierForm.show" preset="dialog" title="Add Supplier")
    n-form(:model="addSupplierForm" :rules="addSupplierRule" ref="addSupplierFormRef")
      n-form-item(label="Supplier Name" path="supplierName")
        n-input(v-model:value="addSupplierForm.supplierName" placeholder="Supplier Name" @keydown.enter.prevent="addSupplier")
    template(#action)
      n-button(@click="addSupplier" type="primary") OK

  n-modal(v-model:show="updateSupplierForm.show" preset="dialog" title="Update Supplier")
    n-form(:model="updateSupplierForm" :rules="updateSupplierRule" ref="updateSupplierFormRef")
      n-form-item(label="Supplier Name" path="supplierName")
        n-input(v-model:value="updateSupplierForm.supplierName" placeholder="Supplier Name" @keydown.enter.prevent="updateSupplier")
    template(#action)
      n-button(@click="updateSupplier" type="primary") OK

  n-modal(v-model:show="showStatisticModal" preset="dialog" style="width: calc(100vw - 200px); height: calc(100vh - 200px);")
    template(#header)
      .flex.items-center.gap-x-2.w-full Statistic
        n-radio-group(v-model:value="statisticAmountType")
          n-radio-button(value="invoice") Invoice
          n-radio-button(value="cheque") Cheque
        n-radio-group(v-model:value="statisticPeriodType")
          n-radio-button(value="all") All
          n-radio-button(value="monthly") Monthly
          n-radio-button(value="quarterly") Quarterly
          n-radio-button(value="yearly") Yearly
        .w-50
          n-select(v-show="statisticPeriodType === 'monthly'" v-model:value="statisticMonthlyPeriod" :options="statisticPeriodOptions.monthly" filterable clearable)
          n-select(v-show="statisticPeriodType === 'quarterly'" v-model:value="statisticQuarterlyPeriod" :options="statisticPeriodOptions.quarterly" filterable clearable)
          n-select(v-show="statisticPeriodType === 'yearly'" v-model:value="statisticYearlyPeriod" :options="statisticPeriodOptions.yearly" filterable clearable)
    n-scrollbar.pr-2(style="height: calc(100vh - 280px);")
      n-table(v-if="statistic" style="overflow: visible;" size="small")
        thead.sticky.top-0.z-3
          tr
            th Supplier
            th
              span.cursor-pointer {{ statisticAmountType == 'invoice' ? 'Invoice' : 'Cheque'  }} Amount
              n-tag.float-right.font-bold(type="success") SUM: {{ format(statistic.totalAmount) }}
        tbody
          tr(v-for="supplierAccount in statistic.suppliers")
            td {{ supplierAccount.supplierName }}
            td.text-right.relative.font-bold {{ format(supplierAccount.totalAmount) }}
              .absolute.left-0.bg-gray.h-full.top-0.opacity-30(:style="`width: ${supplierAccount.totalAmount / statistic.totalAmount * 100}%`")
      .flex.items-center.justify-center.w-full(v-else size="huge" style="height: calc(100vh - 280px);")
        n-empty
          .text-4xl Please select a period to view statistic.
          n-select(v-show="statisticPeriodType === 'monthly'" v-model:value="statisticMonthlyPeriod" :options="statisticPeriodOptions.monthly" filterable clearable size="large")
          n-select(v-show="statisticPeriodType === 'quarterly'" v-model:value="statisticQuarterlyPeriod" :options="statisticPeriodOptions.quarterly" filterable clearable size="large")
          n-select(v-show="statisticPeriodType === 'yearly'" v-model:value="statisticYearlyPeriod" :options="statisticPeriodOptions.yearly" filterable clearable size="large")

  n-modal(v-model:show="showPendingInvoicesModal" preset="dialog" title="Pending Invoices" style="width: calc(100vw - 200px); height: calc(100vh - 200px);")
    n-scrollbar.pr-2(style="height: calc(100vh - 280px);")
      template(v-for="supplierAccount in pendingInvoices.suppliers")
        n-table(style="overflow: visible;" size="small" :single-line="false")
          thead.sticky.top-0.z-3
            tr
              th(colspan="4")
                .text-xl.font-bold.text-center {{ supplierAccount.supplierName }}
            tr
              th.w-35 Date
              th.w-50 Invoice No
              th.w-60 Amount
                n-tag.font-mono.float-right(type="error") SUM: {{ format(sumBy(supplierAccount.items, item => item.invoiceAmount)) }}
              th Remark
          tbody
            tr(v-for="item in supplierAccount.items")
              td.font-mono {{ item.invoiceDate && formatDate(new Date(item.invoiceDate), 'yyyy-MM-dd') }}
              td.font-mono {{ item.invoiceNo }}
              td.font-mono.text-right {{ format(item.invoiceAmount) }}
              td {{ item.remark }}

</template>

<style lang="scss" scoped>
@import "~/styles/common.scss";

.font-mono {
  font-family: Consolas, 'Courier New', Courier, monospace;
}

.table-row-move, /* apply transition to moving elements */
.table-row-enter-active,
.table-row-leave-active {
  transition: all 0.5s ease;
}

.table-row-enter-from,
.table-row-leave-to {
  opacity: 0;
}
</style>
