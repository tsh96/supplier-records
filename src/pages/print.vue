<script setup lang="ts">
import { useTransactions, format } from '~/composables/transaction';

const {
  transactions,
  accumulatedAmounts,
  dataClasses,
} = useTransactions()

function formatDate(date: number) {
  const d = new Date(date)
  return `${d.getFullYear()}-${(d.getMonth() + 1).toString().padStart(2, '0')}-${d.getDate().toString().padStart(2, "0")}`
}

onMounted(() => {
  window.print()
  window.close()
})
</script>

<template lang="pug">
.layout
  h1(text="2xl") Transactions
  n-table(size="small" :single-line="false")
    thead.sticky.top-0.z-3
      tr
        th.w-10
        th.w-40 Date
        th Description
        th.w-40 Credit
        th.w-40 Debit
        th.w-40 Accumulated
    tbody
      tr(v-for="transaction, i in transactions")
        td {{ i+1 }}
        td {{ formatDate(transaction.date) }}
        td {{ transaction.description }}
        td.text-right {{ format(transaction.credit) }}
        td.text-right {{ format(transaction.debit) }}
        td.text-right(:class="dataClasses[i].accumulated") {{ format(accumulatedAmounts[i]) }}
</template>

<style lang="scss" scoped>
@import "~/styles/common.scss";

:deep() {
  th {
    background-color: #bdbdbd;
  }
  tr {
    break-inside: avoid;
  }
}

@media print {
  .layout {
    zoom: 0.6;
  }
}

</style>