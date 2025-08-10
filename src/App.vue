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
        model: 'openai/gpt-4o-mini',
        messages: [
          {
            role: 'user',
            content: `Проанализируй следующий текст и создай описание в стиле wiki в одном предложении. Каждый элемент должен содержать название, краткое описание и ссылку (если есть). 
Формат ответа:
- Название — краткое описание. ссылка

Пример:
- Ace — голосовой помощник на базе собственной модели, способный управлять компьютером с высокой скоростью и точностью выполнения команд. https://generalagents.com/ace/

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
  padding: 2rem 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  width: 100%;
}

.header {
  text-align: center;
  color: white;
  margin-bottom: 3rem;
  padding: 0 1rem;
}

.header h1 {
  font-size: clamp(2rem, 4vw, 2.5rem);
  font-weight: 700;
  margin-bottom: 0.5rem;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
  letter-spacing: -0.02em;
}

.header p {
  font-size: clamp(1rem, 2.5vw, 1.2rem);
  opacity: 0.9;
  font-weight: 400;
  max-width: 600px;
  margin: 0 auto;
}

.main {
  max-width: 1400px;
  margin: 0 auto;
  display: grid;
  gap: 2rem;
  grid-template-columns: 1fr 1fr;
  width: 100%;
  padding: 0 1rem;
}

.input-section, .output-section {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  box-shadow: 0 10px 40px rgba(0,0,0,0.08);
  border: 1px solid rgba(255,255,255,0.2);
  backdrop-filter: blur(10px);
}

.api-key-section, .text-input-section {
  margin-bottom: 1.5rem;
}

.api-key-section label, .text-input-section label {
  display: block;
  margin-bottom: 0.75rem;
  font-weight: 500;
  color: #374151;
  font-size: 0.95rem;
  letter-spacing: -0.01em;
}

.api-input, .text-input {
  width: 100%;
  padding: 0.875rem 1rem;
  border: 1.5px solid #d1d5db;
  border-radius: 10px;
  font-size: 0.95rem;
  font-weight: 400;
  background-color: #fafafa;
  transition: all 0.2s ease;
  line-height: 1.5;
}

.api-input:focus, .text-input:focus {
  outline: none;
  border-color: #667eea;
  background-color: #ffffff;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
  transform: translateY(-1px);
}

.api-input:hover, .text-input:hover {
  border-color: #9ca3af;
  background-color: #ffffff;
}

.text-input {
  resize: vertical;
  min-height: 200px;
  font-family: inherit;
  line-height: 1.6;
}

.api-input::placeholder, .text-input::placeholder {
  color: #9ca3af;
  font-weight: 400;
}

.generate-btn {
  width: 100%;
  padding: 1rem 2rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  letter-spacing: -0.01em;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
}

.generate-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
  background: linear-gradient(135deg, #5a6fd8 0%, #6a4190 100%);
}

.generate-btn:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
}

.generate-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.2);
}

.output-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.copy-btn {
  padding: 0.625rem 1.25rem;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.875rem;
  font-weight: 500;
  transition: all 0.2s ease;
  letter-spacing: -0.01em;
  box-shadow: 0 2px 8px rgba(16, 185, 129, 0.3);
}

.copy-btn:hover {
  background: #059669;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.4);
}

.copy-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 8px rgba(16, 185, 129, 0.3);
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
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1.5rem;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', 'Consolas', monospace;
  font-size: 0.9rem;
  line-height: 1.7;
  max-height: 400px;
  overflow-y: auto;
  color: #374151;
  font-weight: 400;
}

.output-header h3 {
  margin: 0;
  font-size: 1.125rem;
  font-weight: 600;
  color: #1f2937;
  letter-spacing: -0.025em;
}

@media (max-width: 1024px) {
  .main {
    max-width: 800px;
    grid-template-columns: 1fr;
    gap: 1.5rem;
  }
}

@media (max-width: 768px) {
  .main {
    padding: 0 1rem;
  }
  
  .app {
    padding: 1.5rem 0;
  }
  
  .input-section, .output-section {
    padding: 1.5rem;
  }
  
  .api-input, .text-input {
    padding: 0.75rem;
    font-size: 16px; /* Предотвращает зум на iOS */
  }
  
  .generate-btn {
    padding: 0.875rem 1.5rem;
  }
}

@media (max-width: 480px) {
  .header {
    margin-bottom: 2rem;
  }
  
  .input-section, .output-section {
    padding: 1.25rem;
  }
  
  .text-input {
    min-height: 150px;
  }
}
</style>