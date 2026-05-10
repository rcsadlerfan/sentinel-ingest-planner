<script setup>
const props = defineProps({
    log_sources: Array
})

const cols = [
    {
        accessorKey: 'name',
        header: 'Log Source'
    },
    {
        accessorKey: 'ingestPerDayMb',
        header: 'Ingest (MB/Day)'
    },
    {
        accessorKey: 'actions',
        header: ''
    }
]

console.log(props.log_sources)
</script>

<template>
    <UTable :data="log_sources" :columns="cols">
        <template #ingestPerDayMb-cell="{ row }">
            <UInputNumber v-model="row.original.ingestPerDayMb" class="flex grow"></UInputNumber>
        </template>
        <template #actions-cell="{ row }">
            <div class="float-right">
                <UButton color="error" label="Remove"></UButton>
            </div>
        </template>
    </UTable>
</template>