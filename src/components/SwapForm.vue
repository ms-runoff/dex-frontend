<template>
  <div class="swap-form-wide">
    <span class="swap-type">Своп</span>

    <!-- Продать -->
    <div class="swap-card-wide">
      <div class="swap-label">Продать</div>
      <div class="swap-input-row">
        <input
          class="swap-input"
          type="number"
          v-model="fromAmount"
          placeholder="0"
          @input="onFromInput"
          min="0"
        />
        <div class="swap-token-row-right">
          <img class="token-icon" src="../assets/eth.png" alt="ETH" />
          <span class="token-code">ETH</span>
          <span class="token-arrow">▼</span>
        </div>
      </div>
      <div class="swap-secondary">{{ formatUSD(fromAmount * ethRate) }}</div>
    </div>

    <div class="swap-arrow-wide">
      <span>↓</span>
    </div>

    <!-- Купить -->
    <div class="swap-card-wide">
      <div class="swap-label">Купить</div>
      <div class="swap-input-row">
        <input
          class="swap-input"
          type="number"
          v-model="toAmount"
          placeholder="0"
          @input="onToInput"
          min="0"
        />
        <div class="swap-token-row-right">
          <img class="token-icon" src="../assets/usdt.png" alt="USDT" />
          <span class="token-code">USDT</span>
          <span class="token-arrow">▼</span>
        </div>
      </div>
      <div class="swap-secondary">{{ formatUSD(toAmount) }}</div>
    </div>

    <!-- Главная кнопка действий -->
    <button class="btn-wallet-wide" @click="handleMainButton">
      {{ mainButtonText }}
    </button>
    <div class="swap-footer-rate-wide">
      1 USDT = {{ ethPerUsdt }} ETH (1,00 $)
    </div>
  </div>

  <!-- Меню выбора кошелька -->
  <div v-if="isWalletMenuOpen" class="overlay" @click="closeWalletMenu"></div>
  <transition name="slide">
    <div v-if="isWalletMenuOpen" class="wallet-menu">
      <div class="wallet-menu-header">
        <h3>Выберите кошелёк</h3>
        <button class="close-btn" @click="closeWalletMenu">✕</button>
      </div>
      <div class="wallet-list">
        <div class="wallet-item" @click="selectWallet">
          <div class="wallet-icon">🦄</div>
          <span class="wallet-name">Uniswap Wallet</span>
        </div>
      </div>
    </div>
  </transition>

  <!-- QR-модалка -->
  <div v-if="showQRModal" class="qr-overlay" @click="closeQRModal"></div>
  <transition name="fade">
    <div v-if="showQRModal" class="qr-modal">
      <div class="qr-modal-content">
        <div class="qr-header">
          <h3>Подключить Uniswap Wallet</h3>
          <button class="close-btn" @click="closeQRModal">✕</button>
        </div>
        <div class="qr-body">
          <div class="qr-code">
            <canvas ref="qrCanvas"></canvas>
          </div>
          <p class="qr-instruction">
            Отсканируйте QR-код или откройте приложение напрямую
          </p>
          <button 
            v-if="walletConnectUri" 
            @click="openInUniswap" 
            class="btn-open-wallet"
          >
            🦄 Открыть в Uniswap Wallet
          </button>
          <p v-if="connectionStatus" class="connection-status">
            {{ connectionStatus }}
          </p>
        </div>
      </div>
    </div>
  </transition>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch, defineProps, defineEmits } from 'vue'
import SignClient from '@walletconnect/sign-client'
import QRCode from 'qrcode'
import { ethers } from 'ethers'

const props = defineProps({
  walletConnected: Boolean,
  walletAddress: String,
  walletBalance: String
})

const emit = defineEmits(['wallet-connected', 'wallet-disconnected', 'open-wallet-info'])

const ethRate = ref(3894.537)
const fromAmount = ref('')
const toAmount = ref('')
const isWalletMenuOpen = ref(false)
const showQRModal = ref(false)
const qrCanvas = ref(null)
const connectionStatus = ref('')
const walletAddress = ref('')
const walletConnectUri = ref('')
const walletBalance = ref('0,00')

let signClient = null
let session = null

const ethPerUsdt = computed(() => (1 / ethRate.value).toFixed(8))

const mainButtonText = computed(() => {
  if (!walletAddress.value) {
    return 'Подключить кошелек'
  }
  const balanceNum = parseFloat(walletBalance.value.replace(/\s/g, '').replace(',', '.'))
  if (balanceNum === 0 || isNaN(balanceNum)) {
    return 'Добавить средства для свопа'
  }
  return 'Своп'
})

onMounted(async () => {
  const projectId = import.meta.env.VITE_WALLETCONNECT_PROJECT_ID
  if (!projectId) {
    console.error('Project ID не найден в .env')
    return
  }

  try {
    signClient = await SignClient.init({
      projectId,
      metadata: {
        name: 'DEX MVP',
        description: 'Децентрализованная биржа',
        url: window.location.origin,
        icons: ['https://walletconnect.com/walletconnect-logo.png']
      }
    })
    console.log('WalletConnect initialized')

    // Восстановление сессии
    const sessions = signClient.session.getAll()
    if (sessions.length > 0) {
      session = sessions[0]
      const accounts = session.namespaces.eip155.accounts
      if (accounts && accounts.length > 0) {
        const addr = accounts[0].split(':')[2]
        walletAddress.value = addr
        await fetchBalance(addr)
        emitWalletConnected()
      }
    }

    // Обработчик отключения
    signClient.on('session_delete', () => {
      console.log('Session deleted')
      disconnectWallet()
    })
  } catch (error) {
    console.error('WalletConnect init error:', error)
  }

  // Слушаем события от App.vue
  window.addEventListener('open-wallet-menu', openWalletMenu)
  window.addEventListener('disconnect-wallet', disconnectWallet)
})

onUnmounted(() => {
  window.removeEventListener('open-wallet-menu', openWalletMenu)
  window.removeEventListener('disconnect-wallet', disconnectWallet)
})

watch(walletAddress, async (addr) => {
  if (addr) {
    await fetchBalance(addr)
  }
})

async function fetchBalance(address) {
  try {
    const provider = new ethers.JsonRpcProvider('https://ethereum.publicnode.com')
    const bal = await provider.getBalance(address)
    const ethVal = parseFloat(ethers.formatEther(bal))
    const usd = (ethVal * ethRate.value).toFixed(2)
    walletBalance.value = Number(usd).toLocaleString('ru-RU')
    console.log('Balance loaded:', ethVal, 'ETH =', usd, 'USD')
  } catch (error) {
    console.error('Balance fetch error:', error)
    walletBalance.value = '0,00'
  }
}

function formatUSD(val) {
  if (!val || isNaN(val)) return '0 $'
  return Number(val).toLocaleString('ru-RU', { maximumFractionDigits: 2 }) + ' $'
}

function onFromInput() {
  toAmount.value = fromAmount.value
    ? (parseFloat(fromAmount.value) * ethRate.value).toFixed(2)
    : ''
}

function onToInput() {
  fromAmount.value = toAmount.value
    ? (parseFloat(toAmount.value) / ethRate.value).toFixed(6)
    : ''
}

function handleMainButton() {
  if (!walletAddress.value) {
    // Кошелёк не подключен — открываем меню выбора
    openWalletMenu()
  } else {
    const balanceNum = parseFloat(walletBalance.value.replace(/\s/g, '').replace(',', '.'))
    if (balanceNum === 0 || isNaN(balanceNum)) {
      // Баланс 0 — открываем меню пополнения
      emit('open-wallet-info')
    } else {
      // Есть баланс — выполняем своп
      performSwap()
    }
  }
}

function openWalletMenu() {
  isWalletMenuOpen.value = true
}

function closeWalletMenu() {
  isWalletMenuOpen.value = false
}

async function selectWallet() {
  closeWalletMenu()
  showQRModal.value = true
  await connectWalletConnect()
}

async function connectWalletConnect() {
  if (!signClient) {
    console.error('SignClient not initialized')
    connectionStatus.value = 'WalletConnect не инициализирован'
    return
  }

  try {
    connectionStatus.value = 'Генерация QR-кода...'

    const { uri, approval } = await signClient.connect({
      requiredNamespaces: {
        eip155: {
          methods: ['eth_sendTransaction', 'personal_sign', 'eth_signTypedData'],
          chains: ['eip155:1'],
          events: ['chainChanged', 'accountsChanged']
        }
      }
    })

    if (uri) {
      walletConnectUri.value = uri
      await nextTick()
      if (qrCanvas.value) {
        await QRCode.toCanvas(qrCanvas.value, uri, {
          width: 280,
          margin: 2,
          color: { dark: '#1a202c', light: '#ffffff' }
        })
        connectionStatus.value = 'Ожидание подтверждения...'
      }
    }

    session = await approval()
    const accounts = session.namespaces.eip155.accounts
    if (accounts && accounts.length > 0) {
      const acct = accounts[0].split(':')[2]
      walletAddress.value = acct
      console.log('✅ Wallet address set:', acct)
      connectionStatus.value = `✅ Подключено: ${acct.slice(0, 6)}...${acct.slice(-4)}`

      await fetchBalance(acct)
      emitWalletConnected()

      setTimeout(closeQRModal, 2000)
    }
  } catch (error) {
    console.error('WalletConnect error:', error)
    connectionStatus.value = '❌ Ошибка подключения'
  }
}

function emitWalletConnected() {
  emit('wallet-connected', {
    address: walletAddress.value,
    balance: walletBalance.value
  })
}

function disconnectWallet() {
  walletAddress.value = ''
  walletBalance.value = '0,00'
  session = null
  emit('wallet-disconnected')
}

function openInUniswap() {
  const link = `uniswap://wc?uri=${encodeURIComponent(walletConnectUri.value)}`
  window.location.href = link
  setTimeout(() => {
    connectionStatus.value = 'Если Uniswap Wallet не открылся, установите его'
  }, 1000)
}

function closeQRModal() {
  showQRModal.value = false
  connectionStatus.value = ''
  walletConnectUri.value = ''
}

function performSwap() {
  alert('Функция свопа будет реализована позже')
}
</script>

<style scoped>
.swap-form-wide {
  margin: 0 auto;
  background: #ffffff;
  border-radius: 28px;
  box-shadow: 0 8px 40px rgba(0, 0, 0, 0.08);
  width: 100%;
  max-width: 700px;
  padding: 28px 60px 28px 60px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.swap-type {
  background: #e8f5f4;
  color: #0d7a73;
  font-size: 18px;
  font-weight: 800;
  border-radius: 14px;
  padding: 6px 24px;
  align-self: flex-start;
  margin-bottom: 8px;
}
.swap-card-wide {
  background: #f9fafb;
  border-radius: 18px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.06);
  border: 1px solid #e8ebed;
  padding: 18px 20px 16px 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  align-items: flex-start;
}
.swap-label {
  font-size: 17px;
  color: #4a5568;
  margin-bottom: 2px;
  font-weight: 600;
}
.swap-input-row {
  width: 100%;
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 12px;
}
.swap-input {
  flex: 1 1 auto;
  font-size: 36px;
  font-weight: 700;
  color: #1a202c;
  border: none;
  background: transparent;
  outline: none;
  min-width: 60px;
}
.swap-input::placeholder {
  color: #cbd5e0;
  opacity: 1;
}
.swap-secondary {
  font-size: 15px;
  color: #718096;
  font-weight: 500;
}
.swap-token-row-right {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 6px;
  padding: 8px 14px;
  background: #ffffff;
  border-radius: 14px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  border: 1px solid #e2e8f0;
}
.token-icon {
  width: 28px;
  height: 28px;
}
.token-code {
  font-weight: 700;
  font-size: 17px;
  color: #2d3748;
}
.token-arrow {
  font-size: 18px;
  color: #a0aec0;
}
.swap-arrow-wide {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 20px;
  font-size: 22px;
  color: #718096;
  margin: 2px 0 2px 0;
}
.btn-wallet-wide {
  width: 100%;
  background: linear-gradient(135deg, #19b3ae 0%, #16a89f 100%);
  color: #ffffff;
  border: none;
  border-radius: 18px;
  padding: 16px 0px;
  font-size: 20px;
  font-weight: 800;
  margin: 14px 0 0;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 6px 20px rgba(25, 179, 174, 0.3);
}
.btn-wallet-wide:hover {
  background: linear-gradient(135deg, #16a89f 0%, #138785 100%);
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(25, 179, 174, 0.4);
}
.swap-footer-rate-wide {
  text-align: left;
  color: #4a5568;
  font-size: 14px;
  margin-top: 8px;
  margin-left: 2px;
  font-weight: 500;
}
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 999;
}
.wallet-menu {
  position: fixed;
  top: 0;
  right: 0;
  width: 400px;
  max-width: 90vw;
  height: 100vh;
  background: #ffffff;
  box-shadow: -4px 0 20px rgba(0, 0, 0, 0.15);
  z-index: 1000;
  display: flex;
  flex-direction: column;
}
.wallet-menu-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 28px;
  border-bottom: 1px solid #e2e8f0;
}
.wallet-menu-header h3 {
  margin: 0;
  font-size: 22px;
  color: #1a202c;
  font-weight: 700;
}
.close-btn {
  background: transparent;
  border: none;
  font-size: 28px;
  color: #718096;
  cursor: pointer;
  transition: color 0.2s;
  line-height: 1;
}
.close-btn:hover {
  color: #1a202c;
}
.wallet-list {
  padding: 20px;
  overflow-y: auto;
}
.wallet-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 18px 20px;
  background: #f9fafb;
  border-radius: 14px;
  border: 1px solid #e2e8f0;
  cursor: pointer;
  transition: all 0.2s ease;
  margin-bottom: 12px;
}
.wallet-item:hover {
  background: #e8f5f4;
  border-color: #19b3ae;
  transform: translateX(-4px);
}
.wallet-icon {
  font-size: 32px;
}
.wallet-name {
  font-size: 18px;
  font-weight: 600;
  color: #2d3748;
}
.qr-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  z-index: 1500;
}
.qr-modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1600;
  width: 420px;
  max-width: 90vw;
}
.qr-modal-content {
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  overflow: hidden;
}
.qr-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 28px;
  border-bottom: 1px solid #e2e8f0;
}
.qr-header h3 {
  margin: 0;
  font-size: 22px;
  color: #1a202c;
  font-weight: 700;
}
.qr-body {
  padding: 32px 28px;
  text-align: center;
}
.qr-code {
  margin-bottom: 20px;
  display: flex;
  justify-content: center;
}
.qr-code canvas {
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
}
.qr-instruction {
  color: #4a5568;
  font-size: 16px;
  line-height: 1.5;
  margin: 0 0 16px 0;
}
.btn-open-wallet {
  width: 100%;
  padding: 14px 24px;
  background: linear-gradient(135deg, #ff007a 0%, #e7016a 100%);
  color: #ffffff;
  border: none;
  border-radius: 14px;
  font-size: 16px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 16px rgba(255, 0, 122, 0.25);
}
.btn-open-wallet:hover {
  background: linear-gradient(135deg, #e7016a 0%, #cc0059 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(255, 0, 122, 0.35);
}
.connection-status {
  margin-top: 16px;
  padding: 12px;
  background: #e8f5f4;
  color: #0d7a73;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  line-height: 1.4;
}
.swap-form-wide {
  margin: 0 auto;
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.08);
  width: 100%;
  max-width: 700px;
  padding: 24px 32px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}
/* На мобильных — компактнее */
@media (max-width: 768px) {
  .swap-form-wide {
    max-width: 90vw;
    padding: 16px 18px;
    border-radius: 16px;
  }
  .swap-input {
    font-size: 24px;
  }
  .btn-wallet-wide {
    font-size: 18px;
    padding: 12px 0;
  }
  .swap-type {
    font-size: 15px;
    padding: 4px 16px;
  }
}
.slide-enter-active, .slide-leave-active {
  transition: transform 0.3s ease;
}
.slide-enter-from {
  transform: translateX(100%);
}
.slide-leave-to {
  transform: translateX(100%);
}
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
.btn-open-wallet {
  display: none;
}
@media (max-width: 768px) {
  .btn-open-wallet {
    display: block;
  }
}
</style>
