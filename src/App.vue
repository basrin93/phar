<script setup>
import { ref, computed, onMounted } from 'vue'
import { medications } from './medications.js'

// Состояния приложения
const searchQuery = ref('')
const selectedCategory = ref('Все')
const selectedMedication = ref(null)
const showModal = ref(false)
const showFilters = ref(false)
const viewMode = ref('grid') // 'grid' или 'list'
const favorites = ref([])
const isMobile = ref(false)

// Проверка мобильного устройства
onMounted(() => {
  checkMobile()
  window.addEventListener('resize', checkMobile)
  
  // Загружаем избранное из localStorage
  const saved = localStorage.getItem('favorites')
  if (saved) {
    favorites.value = JSON.parse(saved)
  }
})

function checkMobile() {
  isMobile.value = window.innerWidth <= 768
  if (isMobile.value) {
    viewMode.value = 'list' // На мобильных по умолчанию список
  }
}

// Вычисляемые свойства
const categories = computed(() => {
  const cats = ['Все', ...new Set(medications.map(m => m.category))]
  return cats.sort()
})

const filteredMedications = computed(() => {
  let filtered = [...medications]

  // Фильтр по категории
  if (selectedCategory.value !== 'Все') {
    filtered = filtered.filter(m => m.category === selectedCategory.value)
  }

  // Фильтр по поиску
  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    filtered = filtered.filter(m => 
      m.name.toLowerCase().includes(query) ||
      m.latinName.toLowerCase().includes(query) ||
      (m.altName && m.altName.toLowerCase().includes(query)) ||
      (m.indication && m.indication.toLowerCase().includes(query))
    )
  }

  return filtered
})

// Функции для работы с избранным
function toggleFavorite(id) {
  const index = favorites.value.indexOf(id)
  if (index > -1) {
    favorites.value.splice(index, 1)
  } else {
    favorites.value.push(id)
  }
  localStorage.setItem('favorites', JSON.stringify(favorites.value))
}

function isFavorite(id) {
  return favorites.value.includes(id)
}

// Функции для модального окна
function openModal(medication) {
  selectedMedication.value = medication
  showModal.value = true
  document.body.style.overflow = 'hidden'
}

function closeModal() {
  showModal.value = false
  selectedMedication.value = null
  document.body.style.overflow = 'auto'
}

// Функция для копирования рецепта
function copyRecipe(medication) {
  const text = `Rp.: ${medication.recipe}\nS. ${medication.dosage}`
  navigator.clipboard.writeText(text).then(() => {
    alert('Рецепт скопирован в буфер обмена!')
  })
}

// Функция для сброса фильтров
function resetFilters() {
  searchQuery.value = ''
  selectedCategory.value = 'Все'
  showFilters.value = false
}

// Функция для печати рецепта
function printRecipe(medication) {
  const printContent = `
    <div style="padding: 20px; font-family: Arial, sans-serif;">
      <h2>${medication.name}</h2>
      <p style="font-style: italic; color: #666;">${medication.latinName}</p>
      ${medication.altName ? `<p>Торговое название: ${medication.altName}</p>` : ''}
      <hr>
      <h3>Рецепт:</h3>
      <pre style="background: #f5f5f5; padding: 15px; border-radius: 5px;">Rp.: ${medication.recipe}</pre>
      <p><strong>S.</strong> ${medication.dosage}</p>
      <hr>
      <p><strong>Форма выпуска:</strong> ${medication.form}</p>
      ${medication.indication ? `<p><strong>Показания:</strong> ${medication.indication}</p>` : ''}
      <p><strong>Категория:</strong> ${medication.category}</p>
    </div>
  `
  
  const printWindow = window.open('', '', 'width=800,height=600')
  printWindow.document.write(`
    <html>
      <head>
        <title>Рецепт: ${medication.name}</title>
      </head>
      <body>
        ${printContent}
      </body>
    </html>
  `)
  printWindow.document.close()
  printWindow.print()
}

// Функция для переключения вида
function toggleView() {
  viewMode.value = viewMode.value === 'grid' ? 'list' : 'grid'
}
</script>

<template>
  <div id="app">
    <!-- Шапка приложения -->
    <header class="app-header">
      <div class="header-content">
        <h1>
          <span class="icon">💊</span>
          <span class="title-text">Фармрецепты</span>
        </h1>
        <div class="header-stats">
          <span class="stat">Всего: {{ medications.length }}</span>
          <span class="stat">Найдено: {{ filteredMedications.length }}</span>
        </div>
      </div>
    </header>

    <!-- Панель поиска и фильтров -->
    <div class="search-panel">
      <div class="search-wrapper">
        <input 
          v-model="searchQuery"
          type="text" 
          placeholder="Поиск препарата..."
          class="search-input"
        >
        <button 
          v-if="searchQuery"
          @click="searchQuery = ''"
          class="clear-btn"
        >
          ✕
        </button>
      </div>
      
      <div class="action-buttons">
        <button 
          @click="showFilters = !showFilters" 
          class="filter-btn"
          :class="{ active: showFilters || selectedCategory !== 'Все' }"
        >
          <span class="icon">🔽</span>
          Фильтры
        </button>
        
        <button 
          @click="toggleView" 
          class="view-btn"
          v-if="!isMobile"
        >
          <span v-if="viewMode === 'grid'">📋</span>
          <span v-else>⚏</span>
        </button>
      </div>
    </div>

    <!-- Выпадающая панель фильтров -->
    <div v-if="showFilters" class="filters-panel">
      <div class="filter-group">
        <label>Категория препарата:</label>
        <select v-model="selectedCategory" class="category-select">
          <option v-for="cat in categories" :key="cat" :value="cat">
            {{ cat }} ({{ medications.filter(m => cat === 'Все' || m.category === cat).length }})
          </option>
        </select>
      </div>
      
      <div class="filter-actions">
        <button @click="resetFilters" class="reset-btn">
          Сбросить фильтры
        </button>
      </div>
    </div>

    <!-- Список/Сетка препаратов -->
    <div class="medications-container" :class="`view-${viewMode}`">
      <TransitionGroup name="fade">
        <div 
          v-for="med in filteredMedications" 
          :key="med.id"
          class="medication-card"
          @click="openModal(med)"
        >
          <!-- Кнопка избранного -->
          <button 
            @click.stop="toggleFavorite(med.id)"
            class="favorite-btn"
            :class="{ active: isFavorite(med.id) }"
          >
            {{ isFavorite(med.id) ? '★' : '☆' }}
          </button>
          
          <div class="card-content">
            <div class="card-header">
              <h3 class="med-name">{{ med.name }}</h3>
              <span class="med-latin">{{ med.latinName }}</span>
              <span v-if="med.altName" class="med-alt">{{ med.altName }}</span>
            </div>
            
            <div class="card-info">
              <div class="info-item">
                <span class="label">Форма:</span>
                <span class="value">{{ med.form }}</span>
              </div>
              
              <div v-if="med.indication" class="info-item indication">
                <span class="label">Показание:</span>
                <span class="value">{{ med.indication }}</span>
              </div>
            </div>
            
            <div class="card-footer">
              <span class="category-tag">{{ med.category }}</span>
              <span class="card-id">#{{ med.id }}</span>
            </div>
          </div>
        </div>
      </TransitionGroup>
    </div>

    <!-- Пустой результат поиска -->
    <div v-if="filteredMedications.length === 0" class="empty-state">
      <span class="empty-icon">🔍</span>
      <h3>Препараты не найдены</h3>
      <p>Попробуйте изменить параметры поиска или фильтры</p>
      <button @click="resetFilters" class="reset-btn">
        Сбросить фильтры
      </button>
    </div>

    <!-- Модальное окно с детальной информацией -->
    <Teleport to="body">
      <Transition name="modal">
        <div v-if="showModal" class="modal-overlay" @click.self="closeModal">
          <div class="modal" :class="{ mobile: isMobile }">
            <!-- Шапка модального окна -->
            <div class="modal-header">
              <div class="modal-title-group">
                <h2>{{ selectedMedication.name }}</h2>
                <p class="modal-latin">{{ selectedMedication.latinName }}</p>
                <p v-if="selectedMedication.altName" class="modal-alt">
                  {{ selectedMedication.altName }}
                </p>
              </div>
              <button class="modal-close" @click="closeModal">✕</button>
            </div>
            
            <!-- Тело модального окна -->
            <div class="modal-body">
              <!-- Основная информация -->
              <section class="modal-section">
                <h4>Основная информация</h4>
                <div class="detail-grid">
                  <div class="detail-item">
                    <span class="detail-label">Форма выпуска:</span>
                    <span class="detail-value">{{ selectedMedication.form }}</span>
                  </div>
                  
                  <div v-if="selectedMedication.indication" class="detail-item">
                    <span class="detail-label">Показания:</span>
                    <span class="detail-value">{{ selectedMedication.indication }}</span>
                  </div>
                  
                  <div class="detail-item">
                    <span class="detail-label">Категория:</span>
                    <span class="detail-value category-badge">{{ selectedMedication.category }}</span>
                  </div>
                </div>
              </section>
              
              <!-- Рецепт -->
              <section class="modal-section recipe-section">
                <h4>Рецепт</h4>
                <div class="recipe-box">
                  <pre class="recipe-text">Rp.: {{ selectedMedication.recipe }}</pre>
                  <div class="signature">
                    <strong>S.</strong> {{ selectedMedication.dosage }}
                  </div>
                </div>
              </section>
              
              <!-- Действия -->
              <section class="modal-actions">
                <button 
                  @click="toggleFavorite(selectedMedication.id)"
                  class="action-btn favorite"
                  :class="{ active: isFavorite(selectedMedication.id) }"
                >
                  <span>{{ isFavorite(selectedMedication.id) ? '★' : '☆' }}</span>
                  {{ isFavorite(selectedMedication.id) ? 'В избранном' : 'В избранное' }}
                </button>
                
                <button 
                  @click="copyRecipe(selectedMedication)"
                  class="action-btn copy"
                >
                  <span>📋</span>
                  Копировать
                </button>
                
                <button 
                  @click="printRecipe(selectedMedication)"
                  class="action-btn print"
                  v-if="!isMobile"
                >
                  <span>🖨️</span>
                  Печать
                </button>
              </section>
            </div>
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
}

/* Шапка приложения */
.app-header {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  box-shadow: 0 2px 20px rgba(0, 0, 0, 0.1);
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-content {
  max-width: 1400px;
  margin: 0 auto;
  padding: 15px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-header h1 {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #333;
  font-size: 1.5rem;
}

.app-header .icon {
  font-size: 1.8rem;
}

.header-stats {
  display: flex;
  gap: 15px;
  font-size: 0.9rem;
  color: #666;
}

.stat {
  padding: 5px 10px;
  background: #f0f0f0;
  border-radius: 15px;
}

/* Панель поиска */
.search-panel {
  max-width: 1400px;
  margin: 20px auto;
  padding: 0 20px;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.search-wrapper {
  flex: 1;
  min-width: 250px;
  position: relative;
}

.search-input {
  width: 100%;
  padding: 12px 40px 12px 15px;
  border: none;
  border-radius: 25px;
  background: white;
  font-size: 1rem;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.1);
  transition: all 0.3s;
}

.search-input:focus {
  outline: none;
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.15);
}

.clear-btn {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  background: #e0e0e0;
  border: none;
  width: 25px;
  height: 25px;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.3s;
}

.clear-btn:hover {
  background: #d0d0d0;
}

.action-buttons {
  display: flex;
  gap: 10px;
}

.filter-btn,
.view-btn {
  padding: 12px 20px;
  background: white;
  border: none;
  border-radius: 25px;
  font-size: 0.95rem;
  cursor: pointer;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.1);
  transition: all 0.3s;
  display: flex;
  align-items: center;
  gap: 5px;
}

.filter-btn:hover,
.view-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.15);
}

.filter-btn.active {
  background: #667eea;
  color: white;
}

/* Панель фильтров */
.filters-panel {
  max-width: 1400px;
  margin: 0 auto 20px;
  padding: 20px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 15px;
  margin-left: 20px;
  margin-right: 20px;
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.filter-group {
  margin-bottom: 15px;
}

.filter-group label {
  display: block;
  margin-bottom: 8px;
  color: #666;
  font-weight: 500;
}

.category-select {
  width: 100%;
  padding: 10px 15px;
  border: 1px solid #ddd;
  border-radius: 10px;
  font-size: 0.95rem;
  background: white;
  cursor: pointer;
}

.filter-actions {
  text-align: right;
}

.reset-btn {
  padding: 10px 20px;
  background: #ff6b6b;
  color: white;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s;
}

.reset-btn:hover {
  background: #ff5252;
  transform: translateY(-2px);
}

/* Контейнер препаратов */
.medications-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 0 20px 40px;
}

.medications-container.view-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
}

.medications-container.view-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

/* Карточка препарата */
.medication-card {
  background: white;
  border-radius: 15px;
  padding: 20px;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.1);
  position: relative;
}

.medication-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.15);
}

.favorite-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  background: white;
  border: 2px solid #e0e0e0;
  width: 35px;
  height: 35px;
  border-radius: 50%;
  font-size: 1.2rem;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.favorite-btn:hover {
  transform: scale(1.1);
}

.favorite-btn.active {
  color: #ffd700;
  border-color: #ffd700;
  background: #fffbf0;
}

.card-content {
  padding-right: 40px;
}

.card-header {
  margin-bottom: 15px;
  padding-bottom: 15px;
  border-bottom: 1px solid #f0f0f0;
}

.med-name {
  color: #333;
  font-size: 1.2rem;
  margin-bottom: 5px;
}

.med-latin {
  color: #666;
  font-style: italic;
  font-size: 0.9rem;
  display: block;
}

.med-alt {
  color: #999;
  font-size: 0.85rem;
  display: block;
  margin-top: 3px;
}

.card-info {
  margin-bottom: 15px;
}

.info-item {
  margin-bottom: 8px;
  display: flex;
  align-items: flex-start;
}

.info-item .label {
  font-weight: 600;
  color: #666;
  margin-right: 8px;
  min-width: 70px;
  font-size: 0.9rem;
}

.info-item .value {
  color: #333;
  font-size: 0.9rem;
}

.info-item.indication .value {
  color: #667eea;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.category-tag {
  display: inline-block;
  padding: 5px 12px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  border-radius: 15px;
  font-size: 0.8rem;
  font-weight: 500;
}

.card-id {
  color: #999;
  font-size: 0.8rem;
}

/* Режим списка */
.view-list .medication-card {
  display: flex;
  align-items: center;
}

.view-list .card-content {
  flex: 1;
  display: flex;
  gap: 20px;
  align-items: center;
  padding-right: 50px;
}

.view-list .card-header {
  border: none;
  padding: 0;
  margin: 0;
  flex: 1;
}

.view-list .card-info {
  flex: 2;
  margin: 0;
  display: flex;
  gap: 20px;
}

.view-list .card-footer {
  flex-shrink: 0;
}

/* Пустое состояние */
.empty-state {
  text-align: center;
  padding: 60px 20px;
  color: white;
}

.empty-icon {
  font-size: 4rem;
  display: block;
  margin-bottom: 20px;
}

.empty-state h3 {
  font-size: 1.5rem;
  margin-bottom: 10px;
}

.empty-state p {
  margin-bottom: 20px;
  opacity: 0.9;
}

/* Модальное окно */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  z-index: 1000;
}

.modal {
  background: white;
  border-radius: 20px;
  max-width: 700px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
}

.modal.mobile {
  position: fixed;
  top: 10px;
  left: 10px;
  right: 10px;
  bottom: 10px;
  max-width: none;
  max-height: none;
  height: calc(100vh - 20px);
  border-radius: 15px;
}

.modal-header {
  background: linear-gradient(135deg, #667eea, #764ba2);
  padding: 25px;
  border-radius: 20px 20px 0 0;
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: start;
}

.modal-title-group h2 {
  font-size: 1.6rem;
  margin-bottom: 5px;
}

.modal-latin {
  font-style: italic;
  opacity: 0.9;
}

.modal-alt {
  font-size: 0.9rem;
  opacity: 0.8;
  margin-top: 5px;
}

.modal-close {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  width: 35px;
  height: 35px;
  border-radius: 50%;
  font-size: 1.5rem;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-close:hover {
  background: rgba(255, 255, 255, 0.3);
}

.modal-body {
  padding: 25px;
}

.modal-section {
  margin-bottom: 25px;
}

.modal-section h4 {
  color: #333;
  font-size: 1.1rem;
  margin-bottom: 15px;
  padding-bottom: 8px;
  border-bottom: 2px solid #f0f0f0;
}

.detail-grid {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-item {
  display: flex;
  align-items: flex-start;
}

.detail-label {
  font-weight: 600;
  color: #666;
  margin-right: 10px;
  min-width: 120px;
}

.detail-value {
  color: #333;
  flex: 1;
}

.category-badge {
  display: inline-block;
  padding: 4px 12px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  border-radius: 12px;
  font-size: 0.9rem;
}

.recipe-section {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 12px;
}

.recipe-box {
  background: white;
  padding: 15px;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
}

.recipe-text {
  font-family: 'Courier New', monospace;
  color: #2c3e50;
  white-space: pre-wrap;
  margin-bottom: 15px;
  line-height: 1.6;
}

.signature {
  padding-top: 15px;
  border-top: 1px solid #e0e0e0;
  color: #555;
}

.modal-actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.action-btn {
  flex: 1;
  min-width: 120px;
  padding: 12px 20px;
  border: none;
  border-radius: 10px;
  font-size: 0.95rem;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-weight: 500;
}

.action-btn.favorite {
  background: #fffbf0;
  color: #f39c12;
  border: 2px solid #f39c12;
}

.action-btn.favorite.active {
  background: #f39c12;
  color: white;
}

.action-btn.copy {
  background: #e8f4fd;
  color: #2196f3;
  border: 2px solid #2196f3;
}

.action-btn.print {
  background: #f0f0f0;
  color: #666;
  border: 2px solid #999;
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
}

/* Анимации */
.fade-enter-active,
.fade-leave-active {
  transition: all 0.3s;
}

.fade-enter-from {
  opacity: 0;
  transform: translateY(10px);
}

.fade-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.modal-enter-active,
.modal-leave-active {
  transition: all 0.3s;
}

.modal-enter-from,
.modal-leave-from {
  opacity: 0;
}

.modal-enter-from .modal,
.modal-leave-to .modal {
  transform: scale(0.9);
}

/* Адаптивность для мобильных устройств */
@media (max-width: 768px) {
  .header-content {
    padding: 12px 15px;
  }
  
  .app-header h1 {
    font-size: 1.2rem;
  }
  
  .title-text {
    display: none;
  }
  
  .header-stats {
    font-size: 0.8rem;
  }
  
  .stat {
    padding: 3px 8px;
  }
  
  .search-panel {
    margin: 15px auto;
    padding: 0 15px;
  }
  
  .search-wrapper {
    min-width: 200px;
  }
  
  .medications-container {
    padding: 0 15px 30px;
  }
  
  .medications-container.view-grid {
    grid-template-columns: 1fr;
    gap: 15px;
  }
  
  .medication-card {
    padding: 15px;
  }
  
  .med-name {
    font-size: 1.1rem;
  }
  
  .modal-header {
    padding: 20px;
    border-radius: 15px 15px 0 0;
  }
  
  .modal-title-group h2 {
    font-size: 1.3rem;
  }
  
  .modal-body {
    padding: 20px;
  }
  
  .detail-label {
    min-width: 100px;
    font-size: 0.9rem;
  }
  
  .modal-actions {
    flex-direction: column;
  }
  
  .action-btn {
    min-width: auto;
    width: 100%;
  }
}

@media (max-width: 480px) {
  .medications-container.view-list .card-content {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }
  
  .view-list .card-info {
    flex-direction: column;
    width: 100%;
  }
  
  .view-list .card-footer {
    width: 100%;
    justify-content: space-between;
    margin-top: 10px;
  }
}

/* Скроллбар для модального окна */
.modal::-webkit-scrollbar {
  width: 8px;
}

.modal::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 10px;
}

.modal::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 10px;
}

.modal::-webkit-scrollbar-thumb:hover {
  background: #555;
}
</style>