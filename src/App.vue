<template>
  <div id="app">
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
    <div v-if="showWalletInfo" class="overlay" @click="showWalletInfo = false"></div>
    
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
</template>

<script setup>
import { ref } from 'vue'
import SwapForm from './components/SwapForm.vue'
import WalletInfo from './components/WalletInfo.vue'

const walletConnected = ref(false)
const walletAddress = ref('')
const walletBalance = ref('0,00')
const showWalletInfo = ref(false)
const swapFormRef = ref(null)

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
    // Кошелёк подключен — открываем меню информации
    showWalletInfo.value = true
  } else {
    // Кошелёк не подключен — передаём событие в SwapForm для открытия меню выбора
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
</style>
