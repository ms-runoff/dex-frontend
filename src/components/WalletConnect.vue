<template>
  <button @click="connect" class="connect-btn">
    {{ connected ? 'Отключить' : 'Подключить' }}
  </button>
</template>

<script setup>
import { ref } from 'vue'
import { ethers } from 'ethers'

const connected = ref(false)
const walletAddress = ref('')

const connect = async () => {
  if (!connected.value) {
    if (window.ethereum) {
      const provider = new ethers.BrowserProvider(window.ethereum)
      const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' })
      walletAddress.value = accounts[0]
      connected.value = true
    } else {
      alert('MetaMask не установлен')
    }
  } else {
    connected.value = false
    walletAddress.value = ''
  }
}
</script>

<style scoped>
.connect-btn {
  background: linear-gradient(135deg, #19b3ae 0%, #16a89f 100%);
  color: #fff;
  border: none;
  border-radius: 18px;
  padding: 0.7em 2em;
  font-size: 16px;
  cursor: pointer;
  font-weight: 700;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(25, 179, 174, 0.25);
}
.connect-btn:hover {
  background: linear-gradient(135deg, #16a89f 0%, #138785 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(25, 179, 174, 0.35);
}
</style>
