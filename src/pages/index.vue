<script setup lang="ts" generic="T extends any, O extends any">
import { type FormItemRule, type FormInst } from 'naive-ui'

defineOptions({
  name: 'IndexPage',
})

type SupplierAccount = {
  supplierName: string,
  items: SupplierAccountItem[],
}

type SupplierAccountItem = {
  invoiceDate: number,
  invoiceNo: string,
  invoiceAmount: number,
  chequeDate: number,
  chequeNo: string,
  chequeAmount: number,
  remark: string,
}

const supplierAccounts = useLocalStorage('supplierAccounts', [] as SupplierAccount[])

const suppliersOption = computed(() => supplierAccounts.value.map((supplierAccount) => {
  return {
    label: supplierAccount.supplierName,
    value: supplierAccount.supplierName,
  }
}))

const selectedSupplierName = useLocalStorage<string | null>('selectedSupplierName', null)

const supplierAccount = computed(() => {
  const supplierName = selectedSupplierName.value
  if (!supplierName)
    return null
  const supplierAccount = supplierAccounts.value.find((supplierAccount) => supplierAccount.supplierName === supplierName)
  if (!supplierAccount)
    return null
  return supplierAccount
})


function addItem() {
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  if (!supplierAccount.value) return
  supplierAccount.value.items.push({
    invoiceDate: today.getTime(),
    invoiceNo: '',
    invoiceAmount: 0,
    chequeDate: today.getTime(),
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


function removeItem(index: number) {
  supplierAccount.value?.items.splice(index, 1)
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
    n-popconfirm(@positive-click="upload()")
      div
        div Restore supplier records from a file?
        div All unsaved changes will be lost.
        div Please backup your supplier records before restoring.
      template(#trigger)
        n-button(title="Upload" type="info") Restore
    n-button(@click="download()" title="Download" type="success") Backup
  .flex.gap-x-2.py-2.items-center
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
          th.w-38.min-w-38 Date
          th.w-50.min-w-50 Invoice No.
          th.w-50.min-w-50 Amt #[n-tag(type="success") SUM: {{ format(supplierAccount.items.reduce((sum, item) => sum + item.invoiceAmount, 0)) }}]
          th.w-38.min-w-38 Date
          th.w-50.min-w-50 Cheque No.
          th.w-50.min-w-50 Amt #[n-tag(type="success") SUM: {{ format(supplierAccount.items.reduce((sum, item) => sum + item.chequeAmount, 0)) }}]
          th.min-w-30 Remark
          th.w-10
            .i-carbon-add-alt(hover="bg-green-7 cursor-pointer" @click="addItem")
      tbody(v-if="supplierAccount")
        tr(
          v-for="item, i in supplierAccount.items"
          :draggable="i === draggableRow"
          @dragstart="draggingIndex = i"
          @dragover.prevent
          @drop="dropItem(i)"
        )
          td.cursor-move(@mousedown="draggableRow = i") {{ i+1 }}
          td
            n-date-picker(v-model:value="item.invoiceDate" type="date" size="small")
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
            n-date-picker(v-model:value="item.chequeDate" type="date" size="small")
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
            .i-carbon-subtract-alt(hover="bg-red-7 cursor-pointer" @click="removeItem(i)")
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
</template>

<style lang="scss" scoped>
@import "~/styles/common.scss";
</style>
