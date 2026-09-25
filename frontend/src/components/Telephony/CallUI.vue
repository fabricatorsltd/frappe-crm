<template>
  <Button
    v-if="isAnyEnabled"
    variant="ghost"
    :tooltip="__('Make a Call')"
    :icon="PhoneIcon"
    @click="openDialer"
  />
  <TwilioCallUI ref="twilio" />
  <ExotelCallUI ref="exotel" />
  <SipCallUI ref="sip" />
  <Dialog
    v-model:open="show"
    :title="__('Make Call')"
    :actions="[
      {
        label:
          enabledIntegrations.length > 1
            ? __('Call using {0}', [callMedium])
            : __('Call'),
        variant: 'solid',
        onClick: makeCallUsing,
      },
    ]"
  >
    <template #default>
      <div class="flex flex-col gap-4" @keydown.enter="makeCallUsing">
        <FormControl
          v-model="mobileNumber"
          type="tel"
          :label="__('Phone Number')"
          autofocus
        />
        <FormControl
          v-if="enabledIntegrations.length > 1"
          v-model="callMedium"
          type="select"
          :label="__('Calling Medium')"
          :options="enabledIntegrations.map((i) => i.label)"
        />
        <div v-if="enabledIntegrations.length > 1" class="flex flex-col gap-1">
          <FormControl
            v-model="isDefaultMedium"
            type="checkbox"
            :label="__('Make {0} as default calling medium', [callMedium])"
          />

          <div v-if="isDefaultMedium" class="text-sm text-ink-gray-4">
            {{
              __('You can change the default calling medium from the settings')
            }}
          </div>
        </div>
      </div>
    </template>
  </Dialog>
</template>
<script setup>
import TwilioCallUI from '@/components/Telephony/TwilioCallUI.vue'
import ExotelCallUI from '@/components/Telephony/ExotelCallUI.vue'
import SipCallUI from '@/components/Telephony/SipCallUI.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import { defaultCallingMedium, useTelephony } from '@/composables/telephony'
import { globalStore } from '@/stores/global'
import { FormControl, call, toast } from 'frappe-ui'
import { computed, nextTick, ref, watch } from 'vue'

const { setMakeCall } = globalStore()
const { isEnabled, isAnyEnabled } = useTelephony()

const twilio = ref(null)
const exotel = ref(null)
const sip = ref(null)

const callMedium = ref('Twilio')
const isDefaultMedium = ref(false)

const show = ref(false)
const mobileNumber = ref('')

const enabledIntegrations = computed(() =>
  [
    { key: 'twilio', label: 'Twilio', ref: twilio },
    { key: 'exotel', label: 'Exotel', ref: exotel },
    { key: 'sip', label: 'SIP', ref: sip },
  ].filter(({ key }) => isEnabled(key)),
)

function makeCall(number) {
  if (enabledIntegrations.value.length > 1 && !defaultCallingMedium.value) {
    mobileNumber.value = number
    show.value = true
    return
  }

  callMedium.value = enabledIntegrations.value[0]?.label ?? 'Twilio'
  if (defaultCallingMedium.value) {
    callMedium.value = defaultCallingMedium.value
  }

  mobileNumber.value = number
  makeCallUsing()
}

// a number typed by hand, for calls that have no record to start from
function openDialer() {
  mobileNumber.value = ''
  callMedium.value =
    defaultCallingMedium.value || enabledIntegrations.value[0]?.label
  show.value = true
}

function makeCallUsing() {
  if (!mobileNumber.value?.trim()) return
  if (isDefaultMedium.value && callMedium.value) {
    setDefaultCallingMedium()
  }

  if (callMedium.value === 'Twilio') {
    twilio.value.makeOutgoingCall(mobileNumber.value)
  }

  if (callMedium.value === 'Exotel') {
    exotel.value.makeOutgoingCall(mobileNumber.value)
  }

  if (callMedium.value === 'SIP') {
    sip.value.makeOutgoingCall(mobileNumber.value)
  }
  show.value = false
}

async function setDefaultCallingMedium() {
  await call('crm.integrations.api.set_default_calling_medium', {
    medium: callMedium.value,
  })

  defaultCallingMedium.value = callMedium.value
  toast.success(
    __('Default calling medium set successfully to {0}', [callMedium.value]),
  )
}

watch(
  isAnyEnabled,
  () =>
    nextTick(() => {
      for (const {
        key,
        label,
        ref: integrationRef,
      } of enabledIntegrations.value) {
        integrationRef.value.setup()
        callMedium.value = label
      }

      if (isAnyEnabled.value) {
        callMedium.value = enabledIntegrations.value[0]?.label ?? 'Twilio'
        setMakeCall(makeCall)
      }
    }),
  { immediate: true },
)
</script>
