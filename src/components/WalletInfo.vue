<template>
  <transition name="slide">
    <div v-if="isOpen" class="wallet-info-menu">
      <div class="wallet-header">
        <div class="wallet-avatar">
          <div class="avatar-icon">✨</div>
        </div>
        <div class="wallet-address-block">
          <div class="wallet-address">{{ formatAddress(address) }}</div>
          <div class="wallet-balance">{{ balance }}$</div>
        </div>
        <button class="close-btn" @click="close">✕</button>
      </div>

      <div class="wallet-welcome">
        <h3>Добро пожаловать!</h3>
        <p>Чтобы начать трейдинг, добавьте средства</p>
      </div>

      <div class="wallet-actions">
        <div class="action-card" @click="onBuyCrypto">
          <div class="action-icon bank">🏦</div>
          <div class="action-content">
            <div class="action-title">Покупка криптовалюты</div>
            <div class="action-description">
              Совершайте покупки с помощью дебетовой карты или банковского счета.
            </div>
          </div>
        </div>

        <div class="action-card" @click="onReceiveCrypto">
          <div class="action-icon receive">⬇</div>
          <div class="action-content">
            <div class="action-title">Получить криптовалюту</div>
            <div class="action-description">
              Позволяет перевести средства с другого кошелька.
            </div>
          </div>
        </div>
      </div>

      <div class="wallet-footer">
        <button class="btn-disconnect" @click="onDisconnect">
          Отключить кошелёк
        </button>
      </div>
    </div>
  </transition>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue'

const props = defineProps({
  isOpen: Boolean,
  address: String,
  balance: {
    type: String,
    default: '0,00'
  }
})

const emit = defineEmits(['close', 'disconnect', 'buy-crypto', 'receive-crypto'])

function formatAddress(addr) {
  if (!addr) return ''
  return `${addr.slice(0, 6)}...${addr.slice(-4)}`
}

function close() {
  emit('close')
}

function onDisconnect() {
  emit('disconnect')
  emit('close')
}

function onBuyCrypto() {
  emit('buy-crypto')
}

function onReceiveCrypto() {
  emit('receive-crypto')
}
</script>

<style scoped>
.wallet-info-menu {
  position: fixed;
  top: 0;
  right: 0;
  width: 450px;
  max-width: 95vw;
  height: 100vh;
  background: #ffffff;
  box-shadow: -4px 0 30px rgba(0, 0, 0, 0.15);
  z-index: 2000;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
}

.wallet-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 24px;
  border-bottom: 1px solid #e2e8f0;
  position: relative;
}

.wallet-avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: linear-gradient(135deg, #19b3ae 0%, #16a89f 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.avatar-icon {
  font-size: 24px;
}

.wallet-address-block {
  flex: 1;
}

.wallet-address {
  font-size: 16px;
  font-weight: 700;
  color: #1a202c;
  margin-bottom: 4px;
}

.wallet-balance {
  font-size: 24px;
  font-weight: 800;
  color: #1a202c;
}

.close-btn {
  background: transparent;
  border: none;
  font-size: 24px;
  color: #718096;
  cursor: pointer;
  transition: color 0.2s;
  position: absolute;
  top: 24px;
  right: 24px;
  line-height: 1;
}

.close-btn:hover {
  color: #1a202c;
}

.wallet-welcome {
  padding: 32px 24px;
  text-align: left;
}

.wallet-welcome h3 {
  margin: 0 0 8px 0;
  font-size: 20px;
  font-weight: 700;
  color: #1a202c;
}

.wallet-welcome p {
  margin: 0;
  font-size: 15px;
  color: #4a5568;
}

.wallet-actions {
  padding: 0 24px 24px 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.action-card {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  padding: 20px;
  background: #f9fafb;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.action-card:hover {
  background: #e8f5f4;
  border-color: #19b3ae;
  transform: translateX(-4px);
}

.action-icon {
  width: 48px;
  height: 48px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  flex-shrink: 0;
}

.action-icon.bank {
  background: linear-gradient(135deg, #ff007a 0%, #e7016a 100%);
}

.action-icon.receive {
  background: linear-gradient(135deg, #19b3ae 0%, #16a89f 100%);
  color: #ffffff;
  font-weight: 700;
}

.action-content {
  flex: 1;
}

.action-title {
  font-size: 16px;
  font-weight: 700;
  color: #1a202c;
  margin-bottom: 6px;
}

.action-description {
  font-size: 14px;
  color: #4a5568;
  line-height: 1.4;
}

.wallet-footer {
  padding: 24px;
  margin-top: auto;
  border-top: 1px solid #e2e8f0;
}

.btn-disconnect {
  width: 100%;
  padding: 14px;
  background: #f9fafb;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  font-size: 16px;
  font-weight: 600;
  color: #e53e3e;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-disconnect:hover {
  background: #fee;
  border-color: #e53e3e;
}

/* Анимация */
.slide-enter-active, .slide-leave-active {
  transition: transform 0.3s ease;
}

.slide-enter-from {
  transform: translateX(100%);
}

.slide-leave-to {
  transform: translateX(100%);
}
</style>
