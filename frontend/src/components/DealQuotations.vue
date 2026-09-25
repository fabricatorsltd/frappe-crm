<template>
  <div v-if="data?.enabled" class="border-b p-1 sm:p-3">
    <Section
      labelClass="px-2 font-semibold"
      headerClass="h-8"
      :label="__('Quotations')"
      :count="data.quotations.length || undefined"
      :opened="true"
    >
      <template #actions>
        <Button
          v-if="data.can_create"
          variant="ghost"
          class="mr-2 w-7"
          icon="lucide-plus"
          :tooltip="__('Create Quotation')"
          @click="createQuotation"
        />
      </template>
      <div class="mt-1 flex flex-col">
        <a
          v-for="q in data.quotations"
          :key="q.name"
          :href="q.url"
          target="_blank"
          rel="noopener"
          class="flex items-center justify-between gap-3 rounded px-3 py-1.5 text-base hover:bg-surface-gray-2"
        >
          <span class="flex min-w-0 flex-col">
            <span class="truncate text-ink-gray-8">{{ q.name }}</span>
            <span class="text-sm text-ink-gray-5">
              {{ formatDate(q.transaction_date, 'D MMM YYYY') }} ·
              {{ q.docstatus == 0 ? __('Draft') : __(q.status) }}
            </span>
          </span>
          <span class="shrink-0 tabular-nums text-ink-gray-7">
            {{ formatCurrency(q.grand_total, '', q.currency) }}
          </span>
        </a>
        <div
          v-if="!data.quotations.length"
          class="px-3 py-1.5 text-base text-ink-gray-5"
        >
          {{ __('No Quotations') }}
        </div>
      </div>
    </Section>
  </div>
</template>

<script setup>
import Section from '@/components/CollapsibleSection.vue'
import { formatDate } from '@/utils'
import { formatCurrency } from '@/utils/numberFormat'
import { useEventListener } from '@vueuse/core'
import { call, createResource } from 'frappe-ui'
import { computed } from 'vue'

// Quotations tied to the deal by the ERPNext integration. The list reloads
// when the window regains focus, so a quotation saved in the ERPNext tab
// shows up here.

const props = defineProps({
  deal: { type: String, required: true },
  organization: { type: String, default: '' },
})

const quotations = createResource({
  url: 'fab_crm.quotations.deal_quotations',
  params: { deal: props.deal },
  auto: true,
})
const data = computed(() => quotations.data)

async function createQuotation() {
  const url = await call(
    'crm.fcrm.doctype.erpnext_crm_settings.erpnext_crm_settings.get_quotation_url',
    { crm_deal: props.deal, organization: props.organization },
  )
  window.open(url, '_blank')
}

useEventListener(window, 'focus', () => quotations.reload())
</script>
