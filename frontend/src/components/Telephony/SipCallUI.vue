<template>
  <div v-show="showCallPopup" v-bind="$attrs">
    <div
      ref="callPopup"
      class="fixed z-[60] flex w-60 cursor-move select-none flex-col rounded-lg bg-surface-gray-10 p-4 text-ink-gray-2 shadow-2xl"
      :style="style"
    >
      <div class="flex flex-row-reverse items-center gap-1">
        <MinimizeIcon
          class="h-4 w-4 cursor-pointer"
          @click="toggleCallWindow"
        />
      </div>
      <div class="flex flex-col items-center justify-center gap-3">
        <div class="flex flex-col items-center justify-center gap-1">
          <div
            class="text-2xl-medium text-center"
            :class="party.name ? 'cursor-pointer underline' : ''"
            @click="openParty"
          >
            {{ party.label || __('Unknown') }}
          </div>
          <div class="text-sm text-ink-gray-5">{{ phoneNumber }}</div>
        </div>
        <CountUpTimer ref="counterUp">
          <div v-if="onCall" class="my-1 text-base">
            {{ counterUp?.updatedTime }}
          </div>
        </CountUpTimer>
        <div v-if="!onCall" class="my-1 text-base">
          {{
            callStatus == 'ringing'
              ? __('Ringing...')
              : calling
                ? __('Calling...')
                : __('Incoming call...')
          }}
        </div>
        <div v-if="onCall" class="flex flex-col items-center gap-3">
          <div class="flex gap-2">
            <Button
              :icon="muted ? 'mic-off' : 'mic'"
              class="rounded-full"
              @click="toggleMute"
            />
            <Button
              class="rounded-full"
              :tooltip="__('Keypad')"
              @click="showKeypad = !showKeypad"
            >
              <template #icon>
                <DialpadIcon />
              </template>
            </Button>
            <Button
              class="rounded-full bg-surface-red-7 hover:bg-surface-red-8 rotate-[135deg] text-ink-base"
              :tooltip="__('Hang Up')"
              :icon="PhoneIcon"
              @click="hangUpCall"
            />
          </div>
          <div v-if="showKeypad" class="grid grid-cols-3 gap-2">
            <Button
              v-for="key in KEYPAD"
              :key="key"
              :label="key"
              class="!h-9 !w-12 text-lg"
              @click="sendTone(key)"
            />
          </div>
        </div>
        <div v-else-if="calling">
          <Button
            size="md"
            variant="solid"
            theme="red"
            :label="__('Cancel')"
            class="rounded-lg text-ink-base"
            @click="hangUpCall"
          >
            <template #prefix>
              <PhoneIcon class="rotate-[135deg]" />
            </template>
          </Button>
        </div>
        <div v-else class="flex gap-2">
          <Button
            size="md"
            variant="solid"
            theme="green"
            :label="__('Accept')"
            class="rounded-lg text-ink-base"
            :iconLeft="PhoneIcon"
            @click="acceptIncomingCall"
          />
          <Button
            size="md"
            variant="solid"
            theme="red"
            :label="__('Reject')"
            class="rounded-lg text-ink-base"
            @click="rejectIncomingCall"
          >
            <template #prefix>
              <PhoneIcon class="rotate-[135deg]" />
            </template>
          </Button>
        </div>
      </div>
    </div>
  </div>
  <div
    v-show="showSmallCallWindow"
    class="ml-2 flex cursor-pointer select-none items-center justify-between gap-3 rounded-lg bg-surface-gray-10 px-2 py-[7px] text-base text-ink-gray-2"
    v-bind="$attrs"
    @click="toggleCallWindow"
  >
    <div class="max-w-[120px] truncate">
      {{ party.label || phoneNumber }}
    </div>
    <div v-if="onCall" class="flex items-center gap-2">
      <div class="my-1 min-w-[40px] text-center">
        {{ counterUp?.updatedTime }}
      </div>
      <Button
        variant="solid"
        theme="red"
        class="!h-6 !w-6 rounded-full rotate-[135deg] text-ink-base"
        :icon="PhoneIcon"
        @click.stop="hangUpCall"
      />
    </div>
    <div v-else-if="calling" class="flex items-center gap-3">
      <div class="my-1">
        {{ callStatus == 'ringing' ? __('Ringing...') : __('Calling...') }}
      </div>
      <Button
        variant="solid"
        theme="red"
        class="!h-6 !w-6 rounded-full rotate-[135deg] text-ink-base"
        :icon="PhoneIcon"
        @click.stop="hangUpCall"
      />
    </div>
    <div v-else class="flex items-center gap-2">
      <Button
        variant="solid"
        theme="green"
        class="relative !h-6 !w-6 rounded-full animate-pulse text-ink-base"
        :tooltip="__('Accept Call')"
        :icon="PhoneIcon"
        @click.stop="acceptIncomingCall"
      />
      <Button
        variant="solid"
        theme="red"
        class="!h-6 !w-6 rounded-full rotate-[135deg] text-ink-base"
        :tooltip="__('Reject Call')"
        :icon="PhoneIcon"
        @click.stop="rejectIncomingCall"
      />
    </div>
  </div>
</template>

<script setup>
import MinimizeIcon from '@/components/Icons/MinimizeIcon.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import DialpadIcon from '@/components/Icons/DialpadIcon.vue'
import CountUpTimer from '@/components/CountUpTimer.vue'
import { useDraggable, useWindowSize } from '@vueuse/core'
import { call, toast } from 'frappe-ui'
import JsSIP from 'jssip'
import { ref } from 'vue'
import { useRouter } from 'vue-router'

// An in-browser softphone registered on the office PBX as the user's WebRTC
// extension. The PBX rings it together with the user's other phones, so a
// call answered elsewhere is cancelled here.

const router = useRouter()

const MEDIA = { audio: true, video: false }
const PC_CONFIG = { iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] }
const KEYPAD = ['1', '2', '3', '4', '5', '6', '7', '8', '9', '*', '0', '#']

let ua = null
let session = null
let domain = ''
const remoteAudio = new Audio()
remoteAudio.autoplay = true

const showCallPopup = ref(false)
const showSmallCallWindow = ref(false)
const onCall = ref(false)
const calling = ref(false)
const muted = ref(false)
const showKeypad = ref(false)
const callPopup = ref(null)
const counterUp = ref(null)
const callStatus = ref('')
const phoneNumber = ref('')
const party = ref({})

const { width, height } = useWindowSize()

let { style } = useDraggable(callPopup, {
  initialValue: { x: width.value - 280, y: height.value - 480 },
  preventDefault: true,
})

async function setup() {
  if (ua) return
  let credentials
  try {
    credentials = await call('fab_crm.telephony.softphone_credentials')
  } catch (error) {
    console.log('Softphone not available:', error.message)
    return
  }
  domain = credentials.domain
  ua = new JsSIP.UA({
    sockets: [new JsSIP.WebSocketInterface(credentials.ws_url)],
    uri: credentials.uri,
    authorization_user: credentials.username,
    password: credentials.password,
    register: true,
    session_timers: false,
  })
  ua.on('registered', () => console.log('Softphone registered'))
  ua.on('registrationFailed', (e) =>
    console.log('Softphone registration failed:', e.cause),
  )
  ua.on('newRTCSession', ({ session: newSession, originator }) => {
    if (originator === 'remote') handleIncomingCall(newSession)
  })
  ua.start()
}

function playRemoteAudio(rtcSession) {
  const attach = (pc) =>
    pc.addEventListener('track', (event) => {
      remoteAudio.srcObject = event.streams[0]
    })
  if (rtcSession.connection) attach(rtcSession.connection)
  else rtcSession.on('peerconnection', (e) => attach(e.peerconnection))
}

// ring cadence of an Italian line: 425 Hz, one second on, four off
let ringContext = null
let ringTimer = null

function startRinging() {
  stopRinging()
  try {
    ringContext = new AudioContext()
  } catch {
    return
  }
  const beep = () => {
    const oscillator = ringContext.createOscillator()
    const gain = ringContext.createGain()
    oscillator.frequency.value = 425
    gain.gain.value = 0.2
    oscillator.connect(gain).connect(ringContext.destination)
    oscillator.start()
    oscillator.stop(ringContext.currentTime + 1)
  }
  beep()
  ringTimer = setInterval(beep, 5000)
}

function stopRinging() {
  clearInterval(ringTimer)
  ringTimer = null
  ringContext?.close()
  ringContext = null
}

async function lookUpParty(number) {
  party.value = {}
  try {
    party.value = await call('fab_crm.telephony.softphone_party', { number })
  } catch {
    // an unknown caller is shown by number
  }
}

function openParty() {
  if (!party.value.name) return
  const route = party.value.doctype === 'CRM Deal' ? 'Deal' : 'Lead'
  const param = route === 'Deal' ? 'dealId' : 'leadId'
  router.push({ name: route, params: { [param]: party.value.name } })
}

function trackSession(rtcSession) {
  session = rtcSession
  playRemoteAudio(rtcSession)
  // an incoming session reports progress too, when it rings here
  rtcSession.on('progress', () => {
    if (calling.value) callStatus.value = 'ringing'
  })
  rtcSession.on('confirmed', () => {
    stopRinging()
    calling.value = false
    onCall.value = true
    counterUp.value?.start()
  })
  rtcSession.on('ended', resetCall)
  rtcSession.on('failed', (e) => {
    if (calling.value && e.originator !== 'local') {
      toast.error(__('Call failed: {0}', [e.cause]))
    }
    resetCall()
  })
}

function handleIncomingCall(rtcSession) {
  if (session) {
    rtcSession.terminate({ status_code: 486 })
    return
  }
  phoneNumber.value = rtcSession.remote_identity.uri.user
  party.value = { label: rtcSession.remote_identity.display_name }
  lookUpParty(phoneNumber.value)
  trackSession(rtcSession)
  showCallPopup.value = true
  startRinging()
}

function acceptIncomingCall() {
  stopRinging()
  session?.answer({ mediaConstraints: MEDIA, pcConfig: PC_CONFIG })
}

function rejectIncomingCall() {
  session?.terminate({ status_code: 486 })
}

function hangUpCall() {
  session?.terminate()
}

function toggleMute() {
  if (!session) return
  if (session.isMuted().audio) session.unmute({ audio: true })
  else session.mute({ audio: true })
  muted.value = session.isMuted().audio
}

// in-band tones, what Asterisk expects from a WebRTC endpoint
function sendTone(key) {
  session?.sendDTMF(key, { transportType: 'RFC2833' })
}

function makeOutgoingCall(number) {
  if (!ua?.isRegistered()) {
    toast.error(__('The softphone is not connected to the PBX'))
    return
  }
  if (session) return
  // * and # stay: PBX feature codes such as *99101 are dialled as typed
  phoneNumber.value = (number || '').replace(/(?!^\+)[^\d*#]/g, '')
  if (!phoneNumber.value) return
  lookUpParty(phoneNumber.value)
  calling.value = true
  showCallPopup.value = true
  trackSession(
    ua.call(`sip:${phoneNumber.value}@${domain}`, {
      mediaConstraints: MEDIA,
      pcConfig: PC_CONFIG,
    }),
  )
}

function resetCall() {
  stopRinging()
  counterUp.value?.stop()
  session = null
  remoteAudio.srcObject = null
  showCallPopup.value = false
  showSmallCallWindow.value = false
  onCall.value = false
  calling.value = false
  muted.value = false
  showKeypad.value = false
  callStatus.value = ''
}

function toggleCallWindow() {
  showCallPopup.value = !showCallPopup.value
  showSmallCallWindow.value = !showSmallCallWindow.value
}

defineExpose({ makeOutgoingCall, setup })
</script>
