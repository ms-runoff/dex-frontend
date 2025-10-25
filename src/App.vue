<template>
  <div id="app">
    <div class="page-wrapper">
      <!-- Хедер -->
      <header class="header">
        <div class="logo">
          <span class="logo-icon">🔄</span>
          <span class="logo-text">DEX Swap</span>
        </div>
        <button class="btn-connect" @click="handleHeaderButton">
          {{ walletAddress ? `${walletAddress.slice(0, 6)}...${walletAddress.slice(-4)}` : 'Подключить' }}
        </button>
      </header>

      <!-- Основной контент -->
      <SwapForm
        :walletConnected="!!walletAddress"
        :walletAddress="walletAddress"
        :walletBalance="walletBalance"
        @wallet-connected="onWalletConnected"
        @wallet-disconnected="onWalletDisconnected"
      />
    </div>

    <!-- Оверлей -->
    <div v-if="showWalletInfo" class="overlay" @click="showWalletInfo = false"></div>

    <!-- Меню кошелька -->
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
</template>

<script setup>
import { ref, onMounted } from 'vue'
import SwapForm from './components/SwapForm.vue'
import WalletInfo from './components/WalletInfo.vue'

const walletAddress = ref('')
const walletBalance = ref('0.00')
const showWalletInfo = ref(false)

onMounted(() => {
  if (window.Telegram && window.Telegram.WebApp) {
    window.Telegram.WebApp.ready()
    window.Telegram.WebApp.expand()
  }
  window.addEventListener('open-wallet-menu', () => {})
  window.addEventListener('disconnect-wallet', handleDisconnect)
})

function onWalletConnected(data) {
  walletAddress.value = data.address
  walletBalance.value = data.balance
}

function onWalletDisconnected() {
  walletAddress.value = ''
  walletBalance.value = '0.00'
  showWalletInfo.value = false
}

function handleHeaderButton() {
  if (walletAddress.value) {
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

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  width: 100%;
  min-height: 100vh;
  overflow-x: hidden;
  background: #FFFFFF;
}

#app {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Helvetica', 'Arial', sans-serif;
  background: #FFFFFF;
  min-height: 100vh;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0 16px; /* ДОБАВЛЕНО! */
}

.page-wrapper {
  width: 100%;
  max-width: 480px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 0; /* УМЕНЬШЕНО с 20px */
  width: 100%;
  flex-shrink: 0;
}

.logo {
  display: flex;
  align-items: center;
  gap: 8px;
}

.logo-icon {
  font-size: 20px; /* УМЕНЬШЕНО с 24px */
}

.logo-text {
  font-size: 16px; /* УМЕНЬШЕНО с 18px */
  font-weight: 700;
  color: #4A90E2;
}

.btn-connect {
  background: #4A90E2;
  color: #fff;
  border: none;
  padding: 8px 16px; /* УМЕНЬШЕНО с 10px 20px */
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.btn-connect:hover {
  background: #357ABD;
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
</style>


