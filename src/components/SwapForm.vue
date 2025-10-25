<template>
  <div class="swap-wrapper">
    <div class="swap-card">
      <h2 class="swap-title">Обмен токенов</h2>

      <!-- Вы отдаёте -->
      <div class="input-section">
        <div class="input-header">
          <span class="label">Вы отдаёте</span>
          <span class="balance-text">Баланс: {{ fromBalance }}</span>
        </div>
        <div class="input-container">
                    <input
            ref="fromInput"
            type="text"
            inputmode="decimal"
            v-model="fromAmount"
            @input="onFromInput"
            @focus="onInputFocus"
            @blur="onInputBlur"
            placeholder="0.0"
            class="token-input"
          />
          <button class="token-selector" @click="showTokenModal = true">
            <div class="token-avatar" :style="{ background: fromToken.color }">
              {{ fromToken.icon }}
            </div>
            <span class="token-name">{{ fromToken.symbol }}</span>
            <span class="dropdown-arrow">▼</span>
          </button>
        </div>
      </div>

      <!-- Кнопка переворота -->
      <div class="swap-divider">
        <button class="swap-flip-btn">⇅</button>
      </div>

      <!-- Вы получите -->
      <div class="input-section">
        <div class="input-header">
          <span class="label">Вы получите</span>
          <span class="balance-text">Баланс: {{ toBalance }}</span>
        </div>
        <div class="input-container">
                    <input
            ref="toInput"
            type="text"
            inputmode="decimal"
            v-model="toAmount"
            @input="onToInput"
            @focus="onInputFocus"
            @blur="onInputBlur"
            placeholder="0.0"
            class="token-input"
          />
          <button class="token-selector" @click="showTokenModal = true">
            <div class="token-avatar" :style="{ background: toToken.color }">
              {{ toToken.icon }}
            </div>
            <span class="token-name">{{ toToken.symbol }}</span>
            <span class="dropdown-arrow">▼</span>
          </button>
        </div>
      </div>

      <!-- Детали свопа -->
      <div class="swap-info">
        <div class="info-row">
          <span class="info-label">Курс</span>
          <span class="info-value">1 {{ fromToken.symbol }} = {{ exchangeRate }} {{ toToken.symbol }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">Проскальзывание</span>
          <span class="info-value">{{ slippage }}%</span>
        </div>
        <div class="info-row">
          <span class="info-label">Комиссия сети</span>
          <span class="info-value">~{{ networkFee }} {{ fromToken.symbol }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">Мин. получение</span>
          <span class="info-value">{{ minReceived }} {{ toToken.symbol }}</span>
        </div>
      </div>

            <!-- Кнопка закрытия клавиатуры -->
      <button 
        v-if="isKeyboardOpen" 
        class="done-button" 
        @click="hideKeyboard"
      >
        ✓ Готово
      </button>

      <!-- Кнопка -->
      <button class="action-button" @click="handleAction">
        {{ buttonText }}
      </button>
    </div>

    <!-- История транзакций -->
    <div class="transactions-section" v-if="walletConnected">
      <h3 class="section-title">Последние транзакции</h3>
      <div class="tx-list">
        <div class="tx-item" v-for="tx in recentTxs" :key="tx.id">
          <div class="tx-pair">
            <span class="tx-from">{{ tx.fromAmt }} {{ tx.fromTkn }}</span>
            <span class="tx-arrow">→</span>
            <span class="tx-to">{{ tx.toAmt }} {{ tx.toTkn }}</span>
          </div>
          <span class="tx-time">{{ tx.time }}</span>
        </div>
      </div>
    </div>

    <!-- Модалка кошелька -->
    <div v-if="walletMenuOpen" class="modal-backdrop" @click="walletMenuOpen = false"></div>
    <transition name="slide-up">
      <div v-if="walletMenuOpen" class="wallet-modal">
        <div class="modal-title-bar">
          <h3>Подключить кошелёк</h3>
          <button class="close-modal-btn" @click="walletMenuOpen = false">✕</button>
        </div>
        <div class="wallet-options">
          <div class="wallet-option" @click="selectWallet">
            <div class="wallet-logo">🦄</div>
            <div class="wallet-details">
              <div class="wallet-title">Uniswap Wallet</div>
              <div class="wallet-subtitle">Официальный кошелёк</div>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- QR модалка -->
    <div v-if="qrModalOpen" class="modal-backdrop" @click="closeQR"></div>
    <transition name="fade">
      <div v-if="qrModalOpen" class="qr-modal-wrapper">
        <div class="qr-modal-card">
          <div class="modal-title-bar">
            <h3>Подключить Uniswap Wallet</h3>
            <button class="close-modal-btn" @click="closeQR">✕</button>
          </div>
          <div class="qr-content">
            <canvas ref="qrCanvas"></canvas>
            <p class="qr-hint">Отсканируйте код приложением кошелька</p>
            <button 
              v-if="wcUri" 
              @click="openUniswap" 
              class="open-app-btn"
            >
              🦄 Открыть в Uniswap Wallet
            </button>
            <p v-if="statusMsg" class="status-message">{{ statusMsg }}</p>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from 'vue'
import SignClient from '@walletconnect/sign-client'
import QRCode from 'qrcode'
import { ethers } from 'ethers'

const props = defineProps({
  walletConnected: Boolean,
  walletAddress: String,
  walletBalance: String
})

const emit = defineEmits(['wallet-connected', 'wallet-disconnected'])

const fromToken = ref({ symbol: 'TON', icon: 'T', color: '#0088CC', name: 'Toncoin' })
const toToken = ref({ symbol: 'USDT', icon: 'U', color: '#26A17B', name: 'Tether' })
const fromAmount = ref('')
const toAmount = ref('')
const fromBalance = ref('0.00')
const toBalance = ref('0.00')
const exchangeRate = ref('2.35')
const slippage = ref('0.5')
const networkFee = ref('0.05')
const walletMenuOpen = ref(false)
const qrModalOpen = ref(false)
const showTokenModal = ref(false)
const qrCanvas = ref(null)
const statusMsg = ref('')
const walletAddress = ref('')
const wcUri = ref('')
const walletBalance = ref('0,00')
const fromInput = ref(null)
const toInput = ref(null)
const isKeyboardOpen = ref(false)

const recentTxs = ref([
  { id: 1, fromAmt: '1.5', fromTkn: 'TON', toAmt: '3.52', toTkn: 'USDT', time: '2 мин назад' },
  { id: 2, fromAmt: '10', fromTkn: 'USDT', toAmt: '4.25', toTkn: 'TON', time: '15 мин назад' },
  { id: 3, fromAmt: '0.5', fromTkn: 'TON', toAmt: '1.17', toTkn: 'USDT', time: '1 час назад' }
])

let signClient = null
let session = null

const buttonText = computed(() => {
  if (!walletAddress.value) return 'Подключите кошелёк'
  const bal = parseFloat(walletBalance.value.replace(/\s/g, '').replace(',', '.'))
  if (bal === 0 || isNaN(bal)) return 'Недостаточно средств'
  return 'Обменять'
})

const minReceived = computed(() => {
  if (!toAmount.value) return '0.00'
  return (parseFloat(toAmount.value) * 0.995).toFixed(2)
})

onMounted(async () => {
  const pid = import.meta.env.VITE_WALLETCONNECT_PROJECT_ID
  if (!pid) return

  try {
    signClient = await SignClient.init({
      projectId: pid,
      metadata: {
        name: 'DEX Swap',
        description: 'DEX',
        url: window.location.origin,
        icons: []
      }
    })

    const sessions = signClient.session.getAll()
    if (sessions.length > 0) {
      session = sessions[0]
      const acc = session.namespaces.eip155.accounts[0].split(':')[2]
      walletAddress.value = acc
      await loadBalance(acc)
      notifyConnected()
    }

    signClient.on('session_delete', () => {
      disconnect()
    })
  } catch (err) {
    console.error(err)
  }

  window.addEventListener('open-wallet-menu', openWallet)
  window.addEventListener('disconnect-wallet', disconnect)
  
  // Настройка Telegram WebApp для работы с клавиатурой
  if (window.Telegram && window.Telegram.WebApp) {
    // Отключаем вертикальные свайпы чтобы клавиатура не мешала
    window.Telegram.WebApp.disableVerticalSwipes()
  }
})

onUnmounted(() => {
  window.removeEventListener('open-wallet-menu', openWallet)
  window.removeEventListener('disconnect-wallet', disconnect)
})

watch(walletAddress, async (addr) => {
  if (addr) await loadBalance(addr)
})

async function loadBalance(addr) {
  try {
    const provider = new ethers.JsonRpcProvider('https://ethereum.publicnode.com')
    const bal = await provider.getBalance(addr)
    const ethVal = parseFloat(ethers.formatEther(bal))
    const usd = (ethVal * parseFloat(exchangeRate.value) * 1000).toFixed(2)
    walletBalance.value = Number(usd).toLocaleString('ru-RU')
    fromBalance.value = ethVal.toFixed(4)
  } catch {
    walletBalance.value = '0,00'
    fromBalance.value = '0.00'
  }
}

function onFromInput() {
  if (fromAmount.value) {
    toAmount.value = (parseFloat(fromAmount.value) * parseFloat(exchangeRate.value)).toFixed(2)
  } else {
    toAmount.value = ''
  }
}

function onToInput() {
  if (toAmount.value) {
    fromAmount.value = (parseFloat(toAmount.value) / parseFloat(exchangeRate.value)).toFixed(2)
  } else {
    fromAmount.value = ''
  }
}

function handleAction() {
  if (!walletAddress.value) {
    openWallet()
  } else {
    alert('Обмен будет реализован позже')
  }
}

function openWallet() {
  walletMenuOpen.value = true
}

async function selectWallet() {
  walletMenuOpen.value = false
  qrModalOpen.value = true
  await connectWC()
}

async function connectWC() {
  if (!signClient) return

  try {
    statusMsg.value = 'Генерация QR...'

    const { uri, approval } = await signClient.connect({
      requiredNamespaces: {
        eip155: {
          methods: ['eth_sendTransaction', 'personal_sign'],
          chains: ['eip155:1'],
          events: ['accountsChanged']
        }
      }
    })

    if (uri) {
      wcUri.value = uri
      await nextTick()
      if (qrCanvas.value) {
        await QRCode.toCanvas(qrCanvas.value, uri, {
          width: 280,
          margin: 2,
          color: { dark: '#4A90E2', light: '#fff' }
        })
        statusMsg.value = 'Ожидание подтверждения...'
      }
    }

    session = await approval()
    const acc = session.namespaces.eip155.accounts[0].split(':')[2]
    walletAddress.value = acc
    statusMsg.value = '✅ Подключено'

    await loadBalance(acc)
    notifyConnected()

    setTimeout(closeQR, 2000)
  } catch {
    statusMsg.value = '❌ Ошибка'
  }
}

function notifyConnected() {
  emit('wallet-connected', {
    address: walletAddress.value,
    balance: walletBalance.value
  })
}

function disconnect() {
  walletAddress.value = ''
  walletBalance.value = '0,00'
  fromBalance.value = '0.00'
  session = null
  emit('wallet-disconnected')
}

function openUniswap() {
  if (!wcUri.value) return
  
  // Универсальная ссылка WalletConnect для всех кошельков
  const wcUniversalLink = `https://walletconnect.com/wc?uri=${encodeURIComponent(wcUri.value)}`
  
  // В Telegram это сработает
  if (window.Telegram && window.Telegram.WebApp) {
    window.Telegram.WebApp.openTelegramLink(wcUniversalLink)
  } else {
    window.open(wcUniversalLink, '_blank')
  }
}


function closeQR() {
  qrModalOpen.value = false
  statusMsg.value = ''
  wcUri.value = ''
}

function onInputFocus() {
  isKeyboardOpen.value = true
  // Telegram WebApp: разрешаем вертикальный скролл когда клавиатура открыта
  if (window.Telegram && window.Telegram.WebApp) {
    window.Telegram.WebApp.expand()
  }
}

function onInputBlur() {
  // Небольшая задержка для плавности
  setTimeout(() => {
    isKeyboardOpen.value = false
  }, 100)
}

function hideKeyboard() {
  // Убираем фокус со всех input полей
  if (fromInput.value) fromInput.value.blur()
  if (toInput.value) toInput.value.blur()
  
  // Дополнительно скроллим к началу для Telegram
  window.scrollTo({ top: 0, behavior: 'smooth' })
}
</script>

<style scoped>
.swap-wrapper {
  width: 100%;
}

.swap-card, .transactions-section {
  background: #fff;
  border-radius: 20px;
  padding: 24px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  margin-bottom: 16px;
  width: 100%;
}

.swap-title {
  font-size: 20px;
  font-weight: 700;
  color: #1A1A1A;
  margin: 0 0 20px 0;
}

.input-section {
  margin-bottom: 16px;
}

.input-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.label {
  font-size: 14px;
  color: #7D7D7D;
}

.balance-text {
  font-size: 13px;
  color: #B3B3B3;
}

.input-container {
  display: flex;
  align-items: center;
  background: #F5F5F5;
  border-radius: 12px;
  padding: 12px 16px;
  gap: 12px;
}

.token-input {
  flex: 1;
  min-width: 0; 
  font-size: 24px;
  font-weight: 600;
  border: none;
  background: transparent;
  outline: none;
  color: #1A1A1A;
  -webkit-user-select: text;
  user-select: text;
}

.token-input::placeholder {
  color: #D1D1D1;
}

/* Для iOS - убираем увеличение при фокусе */
@supports (-webkit-touch-callout: none) {
  .token-input {
    font-size: max(16px, 24px);
  }
}

.token-selector {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #fff;
  border: none;
  border-radius: 10px;
  padding: 8px 12px;
  cursor: pointer;
  transition: background 0.2s;
  flex-shrink: 0;
  white-space: nowrap; 
}


.token-selector:hover {
  background: #FAFAFA;
}

.token-avatar {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  font-weight: 700;
  font-size: 14px;
}

.token-name {
  font-weight: 600;
  font-size: 16px;
  color: #1A1A1A;
}

.dropdown-arrow {
  font-size: 10px;
  color: #B3B3B3;
}

.swap-divider {
  display: flex;
  justify-content: center;
  margin: 12px 0;
}

.swap-flip-btn {
  width: 36px;
  height: 36px;
  background: #fff;
  border: 2px solid #E5E5E5;
  border-radius: 50%;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.2s;
}

.swap-flip-btn:hover {
  border-color: #4A90E2;
  transform: rotate(180deg);
}

.swap-info {
  background: #FAFAFA;
  border-radius: 12px;
  padding: 16px;
  margin: 16px 0;
}

.info-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.info-row:last-child {
  margin-bottom: 0;
}

.info-label {
  font-size: 14px;
  color: #7D7D7D;
}

.info-value {
  font-size: 14px;
  font-weight: 600;
  color: #1A1A1A;
}

.done-button {
  width: 100%;
  background: #28a745;
  color: #fff;
  border: none;
  border-radius: 12px;
  padding: 14px;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s;
  margin-bottom: 12px;
  box-shadow: 0 2px 8px rgba(40, 167, 69, 0.3);
}

.done-button:active {
  transform: scale(0.98);
  background: #218838;
}

.action-button {
  width: 100%;
  background: #4A90E2;
  color: #fff;
  border: none;
  border-radius: 12px;
  padding: 16px;
  font-size: 16px;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s;
}

.action-button:hover {
  background: #357ABD;
}

.transactions-section {
  background: #fff;
  border-radius: 20px;
  padding: 24px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
}

.section-title {
  font-size: 18px;
  font-weight: 700;
  color: #1A1A1A;
  margin: 0 0 16px 0;
}

.tx-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.tx-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  background: #FAFAFA;
  border-radius: 10px;
}

.tx-pair {
  display: flex;
  align-items: center;
  gap: 8px;
}

.tx-from, .tx-to {
  font-weight: 600;
  font-size: 14px;
  color: #1A1A1A;
}

.tx-arrow {
  color: #B3B3B3;
}

.tx-time {
  font-size: 12px;
  color: #B3B3B3;
}

.modal-backdrop {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.5);
  z-index: 2000;
}

.wallet-modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: #fff;
  border-radius: 20px;
  box-shadow: 0 6px 32px rgba(0,0,0,0.18);
  max-width: 380px;
  width: 90%;
  z-index: 2001;
}

.qr-modal-wrapper {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border-radius: 20px;
  max-width: 420px;
  width: 90%;
  z-index: 2001;
  background: #fff;
  box-shadow: 0 6px 32px rgba(0,0,0,0.18);
}

.qr-modal-card {
  background: #fff;
  border-radius: 20px;
}

.modal-title-bar {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 24px 60px 20px 24px; /* ← отступ справа под крестик */
  border-bottom: 1px solid #F0F0F0;
  position: relative;
}

.modal-title-bar h3 {
  font-size: 18px;
  font-weight: 700;
  margin: 0;
  color: #1A1A1A;
  text-align: center;
}

.close-modal-btn {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 32px;
  height: 32px;
  background: #F5F5F5;
  border: none;
  border-radius: 50%;
  font-size: 18px;
  cursor: pointer;
  color: #7D7D7D;
  display: flex;
  align-items: center;
  justify-content: center;
}

.wallet-options {
  padding: 16px 24px 24px 24px;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.wallet-option {
  display: flex;
  align-items: center;
  gap: 16px;
  background: #F8F8F8;
  border-radius: 14px;
  padding: 16px 20px;
  cursor: pointer;
  transition: background 0.2s;
  width: 100%;
}
.wallet-option:hover {
  background: #EBF4FF;
}

.wallet-logo {
  width: 48px;
  height: 48px;
  background: linear-gradient(135deg, #4A90E2, #357ABD);
  border-radius: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 28px;
  color: #fff;
  flex-shrink: 0;
  box-shadow: 0 2px 8px rgba(74,144,226,0.15);
}

.wallet-details {
  display: flex;
  flex-direction: column;
}

.wallet-title {
  font-weight: 700;
  font-size: 17px;
  color: #222;
}

.wallet-subtitle {
  font-size: 13px;
  color: #7D7D7D;
  margin-top: 2px;
}

.qr-content {
  padding: 32px 24px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.qr-content canvas {
  border-radius: 12px;
  margin: 0 auto 16px auto;
  display: block;
}

.qr-hint {
  font-size: 14px;
  color: #7D7D7D;
  margin-bottom: 16px;
}

.open-app-btn {
  width: 100%;
  background: #4A90E2;
  color: #fff;
  border: none;
  border-radius: 12px;
  padding: 14px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  display: none;
}

@media (max-width: 768px) {
  .open-app-btn {
    display: block;
  }
}

.status-message {
  margin-top: 12px;
  font-size: 13px;
  color: #4A90E2;
}

.slide-up-enter-active, .slide-up-leave-active {
  transition: transform 0.3s ease;
}

.slide-up-enter-from, .slide-up-leave-to {
  transform: translateY(100%);
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
