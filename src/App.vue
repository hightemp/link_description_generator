<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

interface Model {
  id: string
  name: string
  description?: string
  context_length?: number
  pricing?: {
    prompt: string
    completion: string
  }
}

const inputText = ref('')
const outputText = ref('')
const isLoading = ref(false)
const apiKey = ref(localStorage.getItem('openrouter_api_key') || '')
const selectedModel = ref(localStorage.getItem('selected_model') || 'openai/gpt-4o-mini')
const modelFilter = ref('')
const showModelDropdown = ref(false)
const availableModels = ref<Model[]>([])
const isLoadingModels = ref(false)
const showSettings = ref(false)

// Промпт по умолчанию
const defaultPrompt = `Проанализируй следующий текст и создай описание в стиле wiki в одном предложении. Каждый элемент должен содержать название, краткое описание и ссылку (если есть).
Формат ответа:
- Название — краткое описание. ссылка

Пример:
- Ace — голосовой помощник на базе собственной модели, способный управлять компьютером с высокой скоростью и точностью выполнения команд. https://generalagents.com/ace/

Текст для анализа: {input}`

const systemPrompt = ref(localStorage.getItem('system_prompt') || defaultPrompt)

// Функции для работы с localStorage
const saveApiKey = (key: string) => {
  if (key.trim()) {
    localStorage.setItem('openrouter_api_key', key)
  } else {
    localStorage.removeItem('openrouter_api_key')
  }
}

const saveSelectedModel = (modelId: string) => {
  localStorage.setItem('selected_model', modelId)
}

const saveSystemPrompt = (prompt: string) => {
  localStorage.setItem('system_prompt', prompt)
}

const resetPromptToDefault = () => {
  systemPrompt.value = defaultPrompt
  saveSystemPrompt(defaultPrompt)
}

const openSettings = () => {
  showSettings.value = true
}

const closeSettings = () => {
  showSettings.value = false
}

// Загрузка моделей из OpenRouter API
const loadModels = async () => {
  isLoadingModels.value = true
  try {
    const response = await fetch('https://openrouter.ai/api/v1/models', {
      headers: {
        'Content-Type': 'application/json'
      }
    })
    
    if (response.ok) {
      const data = await response.json()
      availableModels.value = data.data.map((model: any) => ({
        id: model.id,
        name: model.name || model.id,
        description: model.description,
        context_length: model.context_length,
        pricing: model.pricing
      }))
    }
  } catch (error) {
    console.error('Ошибка загрузки моделей:', error)
    // Fallback к базовым моделям
    availableModels.value = [
      { id: 'openai/gpt-4o-mini', name: 'GPT-4o Mini' },
      { id: 'anthropic/claude-sonnet-4', name: 'Claude Sonnet 4' }
    ]
  } finally {
    isLoadingModels.value = false
  }
}

// Фильтрованный список моделей
const filteredModels = computed(() => {
  if (!modelFilter.value) return availableModels.value
  
  const filter = modelFilter.value.toLowerCase()
  return availableModels.value.filter(model =>
    model.name.toLowerCase().includes(filter) ||
    model.id.toLowerCase().includes(filter) ||
    (model.description && model.description.toLowerCase().includes(filter))
  )
})

const selectModel = (modelId: string) => {
  selectedModel.value = modelId
  showModelDropdown.value = false
  modelFilter.value = ''
  saveSelectedModel(modelId)
}

// Watchers для автоматического сохранения
watch(apiKey, (newKey) => {
  saveApiKey(newKey)
})

watch(selectedModel, (newModel) => {
  saveSelectedModel(newModel)
})

watch(systemPrompt, (newPrompt) => {
  saveSystemPrompt(newPrompt)
})

const getSelectedModelName = () => {
  const model = availableModels.value.find(m => m.id === selectedModel.value)
  return model ? model.name : selectedModel.value
}

// Закрытие выпадающего списка при клике вне его
const closeDropdown = (event: Event) => {
  const target = event.target as HTMLElement
  if (!target.closest('.model-dropdown')) {
    showModelDropdown.value = false
    modelFilter.value = ''
  }
}

// Загружаем модели при монтировании компонента
onMounted(() => {
  loadModels()
  document.addEventListener('click', closeDropdown)
})

// Очистка обработчика при размонтировании
onUnmounted(() => {
  document.removeEventListener('click', closeDropdown)
})

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
        model: selectedModel.value,
        messages: [
          {
            role: 'user',
            content: systemPrompt.value.replace('{input}', inputText.value)
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
    } catch (err) {
      console.error('Ошибка копирования:', err)
    }
  }
}
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="header-content">
        <h1>Генератор описания ссылок</h1>
        <button @click="openSettings" class="settings-btn" title="Настройки">
          <svg viewBox="0 0 24 24" fill="currentColor">
            <path d="M12 15.5A3.5 3.5 0 0 1 8.5 12A3.5 3.5 0 0 1 12 8.5a3.5 3.5 0 0 1 3.5 3.5a3.5 3.5 0 0 1-3.5 3.5m7.43-2.53c.04-.32.07-.64.07-.97c0-.33-.03-.66-.07-1l2.11-1.63c.19-.15.24-.42.12-.64l-2-3.46c-.12-.22-.39-.31-.61-.22l-2.49 1c-.52-.39-1.06-.73-1.69-.98l-.37-2.65A.506.506 0 0 0 14 2h-4c-.25 0-.46.18-.5.42l-.37 2.65c-.63.25-1.17.59-1.69.98l-2.49-1c-.22-.09-.49 0-.61.22l-2 3.46c-.13.22-.07.49.12.64L4.57 11c-.04.34-.07.67-.07 1c0 .33.03.65.07.97l-2.11 1.66c-.19.15-.25.42-.12.64l2 3.46c.12.22.39.3.61.22l2.49-1.01c.52.4 1.06.74 1.69.99l.37 2.65c.04.24.25.42.5.42h4c.25 0 .46-.18.5-.42l.37-2.65c.63-.26 1.17-.59 1.69-.99l2.49 1.01c.22.08.49 0 .61-.22l2-3.46c.12-.22.07-.49-.12-.64l-2.11-1.66Z"/>
          </svg>
        </button>
      </div>
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

        <div class="model-section">
          <label for="model-select">Модель:</label>
          <div class="model-dropdown" :class="{ 'open': showModelDropdown }">
            <button
              type="button"
              class="model-select-btn"
              @click="showModelDropdown = !showModelDropdown"
              :disabled="isLoadingModels"
            >
              <span class="model-name">
                {{ isLoadingModels ? 'Загрузка моделей...' : getSelectedModelName() }}
              </span>
              <svg class="dropdown-icon" :class="{ 'rotated': showModelDropdown }" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
              </svg>
            </button>
            
            <div v-if="showModelDropdown" class="model-dropdown-content">
              <div class="model-search">
                <input
                  v-model="modelFilter"
                  placeholder="Поиск моделей..."
                  class="model-search-input"
                  @click.stop
                />
              </div>
              
              <div class="model-list">
                <button
                  v-for="model in filteredModels"
                  :key="model.id"
                  type="button"
                  class="model-option"
                  :class="{ 'selected': model.id === selectedModel }"
                  @click="selectModel(model.id)"
                >
                  <div class="model-info">
                    <div class="model-title">{{ model.name }}</div>
                    <div class="model-id">{{ model.id }}</div>
                    <div v-if="model.description" class="model-description">{{ model.description }}</div>
                  </div>
                </button>
              </div>
            </div>
          </div>
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

    <!-- Модальное окно настроек -->
    <div v-if="showSettings" class="settings-overlay" @click="closeSettings">
      <div class="settings-modal" @click.stop>
        <div class="settings-header">
          <h2>Настройки</h2>
          <button @click="closeSettings" class="close-btn">
            <svg viewBox="0 0 24 24" fill="currentColor">
              <path d="M19 6.41L17.59 5L12 10.59L6.41 5L5 6.41L10.59 12L5 17.59L6.41 19L12 13.41L17.59 19L19 17.59L13.41 12L19 6.41Z"/>
            </svg>
          </button>
        </div>
        
        <div class="settings-content">
          <div class="setting-group">
            <label for="system-prompt">Системный промпт:</label>
            <textarea
              id="system-prompt"
              v-model="systemPrompt"
              placeholder="Введите системный промпт..."
              rows="12"
              class="prompt-textarea"
            ></textarea>
            <div class="prompt-help">
              <p>Используйте <code>{input}</code> для вставки пользовательского текста</p>
              <button @click="resetPromptToDefault" class="reset-btn">
                Сбросить к значению по умолчанию
              </button>
            </div>
          </div>
        </div>
        
        <div class="settings-footer">
          <button @click="closeSettings" class="save-btn">
            Сохранить и закрыть
          </button>
        </div>
      </div>
    </div>
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

.header-content {
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  margin-bottom: 0.5rem;
}

.header h1 {
  font-size: clamp(2rem, 4vw, 2.5rem);
  font-weight: 700;
  margin: 0;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
  letter-spacing: -0.02em;
}

.settings-btn {
  position: absolute;
  right: 0;
  background: rgba(255, 255, 255, 0.2);
  border: none;
  border-radius: 8px;
  padding: 0.5rem;
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
  backdrop-filter: blur(10px);
}

.settings-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-1px);
}

.settings-btn svg {
  width: 20px;
  height: 20px;
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

.api-key-section, .text-input-section, .model-section {
  margin-bottom: 1.5rem;
}

.api-key-section label, .text-input-section label, .model-section label {
  display: block;
  margin-bottom: 0.75rem;
  font-weight: 500;
  color: #374151;
  font-size: 0.95rem;
  letter-spacing: -0.01em;
}

/* Стили для выпадающего списка моделей */
.model-dropdown {
  position: relative;
}

.model-select-btn {
  width: 100%;
  padding: 0.875rem 1rem;
  border: 1.5px solid #d1d5db;
  border-radius: 10px;
  font-size: 0.95rem;
  font-weight: 400;
  background-color: #fafafa;
  transition: all 0.2s ease;
  line-height: 1.5;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  text-align: left;
}

.model-select-btn:hover:not(:disabled) {
  border-color: #9ca3af;
  background-color: #ffffff;
}

.model-select-btn:focus {
  outline: none;
  border-color: #667eea;
  background-color: #ffffff;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.model-select-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.model-name {
  flex: 1;
  color: #374151;
}

.dropdown-icon {
  width: 20px;
  height: 20px;
  color: #6b7280;
  transition: transform 0.2s ease;
}

.dropdown-icon.rotated {
  transform: rotate(180deg);
}

.model-dropdown-content {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  z-index: 50;
  background: white;
  border: 1.5px solid #d1d5db;
  border-radius: 10px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.15);
  margin-top: 0.25rem;
  max-height: 300px;
  overflow: hidden;
}

.model-search {
  padding: 0.75rem;
  border-bottom: 1px solid #e5e7eb;
}

.model-search-input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.875rem;
  background-color: #f9fafb;
  transition: all 0.2s ease;
}

.model-search-input:focus {
  outline: none;
  border-color: #667eea;
  background-color: #ffffff;
  box-shadow: 0 0 0 2px rgba(102, 126, 234, 0.1);
}

.model-list {
  max-height: 200px;
  overflow-y: auto;
}

.model-option {
  width: 100%;
  padding: 0.75rem;
  border: none;
  background: none;
  text-align: left;
  cursor: pointer;
  transition: background-color 0.2s ease;
  border-bottom: 1px solid #f3f4f6;
}

.model-option:hover {
  background-color: #f8fafc;
}

.model-option.selected {
  background-color: #eff6ff;
  border-left: 3px solid #667eea;
}

.model-option:last-child {
  border-bottom: none;
}

.model-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.model-title {
  font-weight: 500;
  color: #1f2937;
  font-size: 0.9rem;
}

.model-id {
  font-size: 0.8rem;
  color: #6b7280;
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', 'Consolas', monospace;
}

.model-description {
  font-size: 0.8rem;
  color: #9ca3af;
  line-height: 1.4;
  margin-top: 0.25rem;
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
/* Стили для модального окна настроек */
.settings-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  backdrop-filter: blur(4px);
}

.settings-modal {
  background: white;
  border-radius: 16px;
  width: 90%;
  max-width: 600px;
  max-height: 80vh;
  overflow: hidden;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
}

.settings-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.5rem 2rem;
  border-bottom: 1px solid #e5e7eb;
  background: #f8fafc;
}

.settings-header h2 {
  margin: 0;
  font-size: 1.25rem;
  font-weight: 600;
  color: #1f2937;
}

.close-btn {
  background: none;
  border: none;
  padding: 0.5rem;
  cursor: pointer;
  border-radius: 6px;
  color: #6b7280;
  transition: all 0.2s ease;
}

.close-btn:hover {
  background: #e5e7eb;
  color: #374151;
}

.close-btn svg {
  width: 20px;
  height: 20px;
}

.settings-content {
  flex: 1;
  padding: 2rem;
  overflow-y: auto;
}

.setting-group {
  margin-bottom: 1.5rem;
}

.setting-group label {
  display: block;
  margin-bottom: 0.75rem;
  font-weight: 500;
  color: #374151;
  font-size: 0.95rem;
}

.prompt-textarea {
  width: 100%;
  padding: 1rem;
  border: 1.5px solid #d1d5db;
  border-radius: 10px;
  font-size: 0.9rem;
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', 'Consolas', monospace;
  line-height: 1.6;
  resize: vertical;
  min-height: 200px;
  background-color: #fafafa;
  transition: all 0.2s ease;
}

.prompt-textarea:focus {
  outline: none;
  border-color: #667eea;
  background-color: #ffffff;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.prompt-help {
  margin-top: 1rem;
  padding: 1rem;
  background: #f0f9ff;
  border: 1px solid #e0f2fe;
  border-radius: 8px;
}

.prompt-help p {
  margin: 0 0 0.75rem 0;
  font-size: 0.875rem;
  color: #0369a1;
}

.prompt-help code {
  background: #e0f2fe;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', 'Consolas', monospace;
  font-size: 0.8rem;
  color: #0c4a6e;
}

.reset-btn {
  background: #f59e0b;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.reset-btn:hover {
  background: #d97706;
  transform: translateY(-1px);
}

.settings-footer {
  padding: 1.5rem 2rem;
  border-top: 1px solid #e5e7eb;
  background: #f8fafc;
}

.save-btn {
  width: 100%;
  padding: 0.875rem 1.5rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.save-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

@media (max-width: 768px) {
  .settings-modal {
    width: 95%;
    max-height: 90vh;
  }
  
  .settings-header, .settings-content, .settings-footer {
    padding: 1rem 1.5rem;
  }
  
  .header-content {
    flex-direction: column;
    gap: 1rem;
  }
  
  .settings-btn {
    position: static;
  }
}