<template>
  <div id="app">
    <div class="layout">
      <header class="top-bar">
        <span class="top-title">Торговля</span>
        <button class="btn-connect" @click="handleHeaderButton">
          {{ walletConnected ? '🦄 Uniswap Wallet' : 'Подключить' }}
        </button>
      </header>

      <main class="centered-main">
        <SwapForm
          :walletConnected="walletConnected"
          :walletAddress="walletAddress"
          :walletBalance="walletBalance"
          @wallet-connected="onWalletConnected"
          @wallet-disconnected="onWalletDisconnected"
          @open-wallet-info="showWalletInfo = true"
        />
      </main>

      <!-- Оверлей для меню кошелька -->
      <div 
        v-if="showWalletInfo" 
        class="overlay" 
        @click="showWalletInfo = false">
      </div>

      <!-- Меню информации о кошельке -->
      <WalletInfo
        :isOpen="showWalletInfo"
        :address="walletAddress"
        :balance="walletBalance"
        @close="showWalletInfo = false"
        @disconnect="handleDisconnect"
        @buy-crypto="handleBuyCrypto"
        @receive-crypto="handleReceiveCrypto"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import SwapForm from './components/SwapForm.vue'
import WalletInfo from './components/WalletInfo.vue'

// --- Telegram и кошельки ---
const walletConnected = ref(false)
const walletAddress = ref('')
const walletBalance = ref('0,00')
const showWalletInfo = ref(false)
const isTelegramWebApp = ref(false)
const backgroundColor = ref('#ffffff')

// --- Telegram Web App API ---
onMounted(() => {
  if (window.Telegram && window.Telegram.WebApp) {
    const tg = window.Telegram.WebApp
    tg.ready()
    tg.expand()
    isTelegramWebApp.value = true

    // Используем оформление Telegram темы
    backgroundColor.value = tg.themeParams?.bg_color || '#ffffff'

    // Можно отправлять события обратно в Telegram
    console.log('Запустилось как Telegram WebApp')
    console.log('User:', tg.initDataUnsafe?.user)
    console.log('Theme:', tg.themeParams)

    // Пример — при изменении темы Telegram (меняется моментально)
    tg.onEvent('themeChanged', () => {
      backgroundColor.value = tg.themeParams?.bg_color || '#ffffff'
    })

    // Пример отправки данных пользователю в чат Telegram через sendData()
    // (бот получает событие, если настроен WebApp)
    window.sendSwapDataToTelegram = (data) => {
      // Отправляем JSON в Telegram Bot
      tg.sendData(JSON.stringify(data))
    }
  }
})

// --- Функции кошелька ---
function onWalletConnected(data) {
  walletConnected.value = true
  walletAddress.value = data.address
  walletBalance.value = data.balance
}

function onWalletDisconnected() {
  walletConnected.value = false
  walletAddress.value = ''
  walletBalance.value = '0,00'
  showWalletInfo.value = false
}

function handleHeaderButton() {
  if (walletConnected.value) {
    showWalletInfo.value = true
  } else {
    window.dispatchEvent(new CustomEvent('open-wallet-menu'))
  }
}

function handleDisconnect() {
  onWalletDisconnected()
  window.dispatchEvent(new CustomEvent('disconnect-wallet'))
}

function handleBuyCrypto() {
  alert('Покупка криптовалюты будет реализована позже')
}

function handleReceiveCrypto() {
  alert('Получение криптовалюты будет реализована позже')
}
</script>

<style scoped>
body {
  background: #fff;
}
#app {
  font-family: Inter, Arial, sans-serif;
  min-height: 100vh;
  width: 100vw;
  transition: background-color 0.3s;
}
.top-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 32px 48px 0 48px;
}
.top-title {
  font-size: 28px;
  color: #202020;
  font-weight: 600;
}
.btn-connect {
  background: linear-gradient(135deg, #19b3ae 0%, #16a89f 100%);
  color: #fff;
  padding: 0.7em 2em;
  border-radius: 18px;
  font-weight: 700;
  font-size: 16px;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(25, 179, 174, 0.25);
}
.btn-connect:hover {
  background: linear-gradient(135deg, #16a89f 0%, #138785 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(25, 179, 174, 0.35);
}
.centered-main {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  min-height: 70vh;
  margin-top: 32px;
}
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 1999;
}
.layout {
  display: flex;
  flex-direction: column;
  margin: 0 auto;
  padding: 0 16px;
  max-width: 1200px;
  width: 100%;
  box-sizing: border-box;
}
.btn-connect {
  background: linear-gradient(135deg, #19b3ae 0%, #16a89f 100%);
  color: #fff;
  padding: 0.7em 2em;
  border-radius: 18px;
  font-weight: 700;
  font-size: 16px;
  white-space: nowrap;
}

@media (max-width: 768px) {
  .top-bar {
    flex-direction: column;
    gap: 12px;
  }
  .btn-connect {
    font-size: 15px;
    padding: 0.6em 1.8em;
  }
  .top-title {
    font-size: 20px;
  }
}

</style>
