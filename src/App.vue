<script setup lang="ts">
import { ref } from 'vue'

const inputText = ref('')
const outputText = ref('')
const isLoading = ref(false)
const apiKey = ref('')

const generateDescription = async () => {
  if (!inputText.value.trim() || !apiKey.value.trim()) {
    alert('Пожалуйста, введите текст и API ключ')
    return
  }

  isLoading.value = true
  outputText.value = ''

  try {
    const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiKey.value}`,
        'Content-Type': 'application/json',
        'HTTP-Referer': window.location.origin,
        'X-Title': 'Link Description Generator'
      },
      body: JSON.stringify({
        model: 'openai/gpt-3.5-turbo',
        messages: [
          {
            role: 'user',
            content: `Проанализируй следующий текст и создай описание в формате markdown-списка. Каждый элемент должен содержать название, краткое описание и ссылку (если есть). Формат ответа:

- Название — краткое описание. ссылка

Текст для анализа: ${inputText.value}`
          }
        ]
      })
    })

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    const data = await response.json()
    outputText.value = data.choices[0].message.content
  } catch (error) {
    console.error('Error:', error)
    outputText.value = 'Ошибка при генерации описания. Проверьте API ключ и попробуйте снова.'
  } finally {
    isLoading.value = false
  }
}

const copyToClipboard = async () => {
  if (outputText.value) {
    try {
      await navigator.clipboard.writeText(outputText.value)
      alert('Скопировано в буфер обмена!')
    } catch (err) {
      console.error('Ошибка копирования:', err)
    }
  }
}
</script>

<template>
  <div class="app">
    <header class="header">
      <h1>Генератор описания ссылок</h1>
      <p>Введите текст для анализа и получите структурированное описание</p>
    </header>

    <main class="main">
      <div class="input-section">
        <div class="api-key-section">
          <label for="api-key">OpenRouter API Key:</label>
          <input
            id="api-key"
            type="password"
            v-model="apiKey"
            placeholder="Введите ваш API ключ"
            class="api-input"
          />
        </div>

        <div class="text-input-section">
          <label for="input-text">Текст для анализа:</label>
          <textarea
            id="input-text"
            v-model="inputText"
            placeholder="Вставьте сюда текст, который нужно проанализировать..."
            rows="10"
            class="text-input"
          ></textarea>
        </div>

        <button 
          @click="generateDescription" 
          :disabled="isLoading || !inputText.trim() || !apiKey.trim()"
          class="generate-btn"
        >
          {{ isLoading ? 'Генерация...' : 'Сгенерировать описание' }}
        </button>
      </div>

      <div class="output-section" v-if="outputText || isLoading">
        <div class="output-header">
          <h3>Результат:</h3>
          <button 
            v-if="outputText && !isLoading" 
            @click="copyToClipboard"
            class="copy-btn"
          >
            Копировать
          </button>
        </div>

        <div v-if="isLoading" class="loader">
          <div class="spinner"></div>
          <p>Генерация описания...</p>
        </div>

        <div v-else class="output-content">
          <pre class="output-text">{{ outputText }}</pre>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  padding: 2rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.header {
  text-align: center;
  color: white;
  margin-bottom: 3rem;
}

.header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
}

.header p {
  font-size: 1.2rem;
  opacity: 0.9;
}

.main {
  max-width: 1200px;
  margin: 0 auto;
  display: grid;
  gap: 2rem;
  grid-template-columns: 1fr 1fr;
}

.input-section, .output-section {
  background: white;
  border-radius: 12px;
  padding: 2rem;
  box-shadow: 0 8px 32px rgba(0,0,0,0.1);
}

.api-key-section, .text-input-section {
  margin-bottom: 1.5rem;
}

.api-key-section label, .text-input-section label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
  color: #333;
}

.api-input, .text-input {
  width: 100%;
  padding: 1rem;
  border: 2px solid #e1e5e9;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.3s ease;
}

.api-input:focus, .text-input:focus {
  outline: none;
  border-color: #667eea;
}

.text-input {
  resize: vertical;
  min-height: 200px;
  font-family: inherit;
}

.generate-btn {
  width: 100%;
  padding: 1rem 2rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.generate-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

.generate-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.output-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.copy-btn {
  padding: 0.5rem 1rem;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
}

.copy-btn:hover {
  background: #218838;
}

.loader {
  text-align: center;
  padding: 2rem;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #667eea;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.output-text {
  background: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 6px;
  padding: 1.5rem;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  line-height: 1.6;
  max-height: 400px;
  overflow-y: auto;
}

@media (max-width: 768px) {
  .main {
    grid-template-columns: 1fr;
  }
  
  .app {
    padding: 1rem;
  }
  
  .header h1 {
    font-size: 2rem;
  }
}
</style>