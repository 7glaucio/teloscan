<script>
import TransactionTable from 'components/TransactionTable';
export default {
    name: 'TransactionsPage',
    components: {
        TransactionTable,
    },
};
</script>

<template lang='pug'>
q-page.column.justify-center.items-center
    .row.q-pa-lg
      .text-h4 Explore Transactions
    .row
      .col-12.q-px-sm
        TransactionTable( title="Recent transactions")
</template>
