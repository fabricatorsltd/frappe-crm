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
          <div
            v-if="digits"
            class="max-w-full truncate text-xl tabular-nums tracking-wider text-ink-gray-1"
          >
            {{ digits.slice(-16) }}
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
import { useDraggable, useEventListener, useWindowSize } from '@vueuse/core'
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
// the two frequencies of each key, as a phone plays them back to the caller
const DTMF = {
  1: [697, 1209],
  2: [697, 1336],
  3: [697, 1477],
  4: [770, 1209],
  5: [770, 1336],
  6: [770, 1477],
  7: [852, 1209],
  8: [852, 1336],
  9: [852, 1477],
  '*': [941, 1209],
  0: [941, 1336],
  '#': [941, 1477],
}

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
const digits = ref('')
let toneContext = null

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
  ringtone = new Audio(credentials.ringtone)
  ringtone.loop = true
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

// the ringtone the user picked on their profile, looped
let ringtone = null

function startRinging() {
  if (!ringtone) return
  ringtone.currentTime = 0
  ringtone.play().catch((error) => console.log('Ringtone blocked:', error))
}

function stopRinging() {
  ringtone?.pause()
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

// RFC 4733 events, what Asterisk expects from a WebRTC endpoint; they carry
// no sound, so the key is played here and shown, as on a phone
function sendTone(key) {
  if (!session) return
  session.sendDTMF(key, { transportType: 'RFC2833' })
  digits.value += key
  playTone(key)
}

function playTone(key) {
  try {
    toneContext ||= new AudioContext()
  } catch {
    return
  }
  const gain = toneContext.createGain()
  gain.gain.value = 0.08
  gain.connect(toneContext.destination)
  const end = toneContext.currentTime + 0.15
  for (const frequency of DTMF[key]) {
    const oscillator = toneContext.createOscillator()
    oscillator.frequency.value = frequency
    oscillator.connect(gain)
    oscillator.start()
    oscillator.stop(end)
  }
}

// the computer keyboard dials too, while a call is up
useEventListener(document, 'keydown', (e) => {
  if (!onCall.value || !(e.key in DTMF)) return
  if (e.target.closest?.('input, textarea, [contenteditable]')) return
  e.preventDefault()
  sendTone(e.key)
})

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
  digits.value = ''
  callStatus.value = ''
}

function toggleCallWindow() {
  showCallPopup.value = !showCallPopup.value
  showSmallCallWindow.value = !showSmallCallWindow.value
}

defineExpose({ makeOutgoingCall, setup })
</script>
