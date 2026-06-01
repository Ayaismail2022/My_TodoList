<template>
  <div class="container" :class="{ dark: darkMode }">
    <h1>
      ✨ My Goals
      <button style="margin-right: 20px" class="add" @click="darkMode = !darkMode">
        {{ darkMode ? '☀️ Light' : '🌙 Dark' }}
      </button>
    </h1>

    <div class="stats">
      <span>الكل: {{ goals.length }}</span>
      <span>المكتملة: {{ completedGoals }}</span>
      <span>المتبقية: {{ pendingGoals }}</span>
    </div>

    <div class="filters">
      <button class="add" @click="filterType = 'all'">الكل</button>
      <button class="add" @click="filterType = 'completed'">المكتملة</button>
      <button class="add" @click="filterType = 'pending'">غير المكتملة</button>
    </div>

    <div class="input-group">
      <input
        ref="inputRef"
        v-model="newGoalText"
        type="text"
        placeholder="اكتب هدفك..."
        @keypress="handleKeyPress"
      />
      <button class="add" @click="addGoal">إضافة</button>
    </div>

    <div id="error">{{ errorMessage }}</div>

    <ul>
      <li v-for="(goal, index) in filteredGoals" :key="goal.id" class="goal-item">
        <div style="flex: 1">
          <!-- ✏️ Inline Edit -->
          <input
            v-if="editingIndex === index"
            :id="'edit-input-' + index"
            v-model="editedText"
            class="edit-input"
            @keyup.enter="saveEdit(index)"
            @keyup.esc="cancelEdit"
          />

          <!-- 📝 Normal view -->
          <span
            v-else
            class="task-text"
            :class="{ completed: goal.completed }"
            @click="toggleComplete(index)"
          >
            {{ goal.text }}
          </span>

          <small class="date">
            {{ new Date(goal.createdAt).toLocaleDateString() }}
          </small>
        </div>

        <div class="actions">
          <button class="edit" @click="handleEditButton(index, goal)">
            {{ editingIndex === index ? 'حفظ' : 'تعديل' }}
          </button>

          <button class="delete" @click="deleteGoal(index)">حذف</button>
        </div>
      </li>
    </ul>

    <div v-if="goals.length === 0" class="empty-state">
      🎯 لا توجد أهداف حالياً، أضف هدفك الأول!
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'

// State
const goals = ref([])
const newGoalText = ref('')
const errorMessage = ref('')
const inputRef = ref(null)

const filterType = ref('all')
const darkMode = ref(false)

// ✨ Inline editing state
const editingIndex = ref(null)
const editedText = ref('')

const STORAGE_KEY = 'goals'

// Load
const loadGoals = () => {
  const stored = localStorage.getItem(STORAGE_KEY)
  if (stored) {
    try {
      goals.value = JSON.parse(stored)
    } catch {
      goals.value = []
    }
  }
}

// Save
const saveGoals = () => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(goals.value))
}

// Validation
const validateGoal = (text) => {
  if (!text || !text.trim()) {
    errorMessage.value = '⚠️ الرجاء كتابة الهدف أولاً!'
    return false
  }

  const regex = /^[a-zA-Z0-9\u0600-\u06FF\s]+$/

  if (!regex.test(text)) {
    errorMessage.value = '⚠️ غير مسموح باستخدام الرموز الخاصة'
    return false
  }

  errorMessage.value = ''
  return true
}

const handleEditButton = (index, goal) => {
  if (editingIndex.value === index) {
    saveEdit(index)
  } else {
    startEdit(index, goal.text)
  }
}

// Add
const addGoal = () => {
  const text = newGoalText.value.trim()

  if (!validateGoal(text)) return

  goals.value.push({
    id: Date.now(),
    text,
    completed: false,
    createdAt: new Date().toISOString(),
  })

  newGoalText.value = ''

  nextTick(() => {
    inputRef.value?.focus()
  })
}

// Toggle complete
const toggleComplete = (index) => {
  if (editingIndex.value === index) return
  goals.value[index].completed = !goals.value[index].completed
}

const startEdit = (index, text) => {
  editingIndex.value = index
  editedText.value = text

  nextTick(() => {
    setTimeout(() => {
      const editInput = document.getElementById(`edit-input-${index}`)
      if (editInput) {
        editInput.focus()
        // خلي المؤشر في آخر النص بدل ما يحدد الكل
        editInput.setSelectionRange(editInput.value.length, editInput.value.length)
      }
    }, 50)
  })
}

// Save edit
const saveEdit = (index) => {
  const text = editedText.value.trim()

  if (!text) return

  goals.value[index].text = text
  editingIndex.value = null
  editedText.value = ''
}

// Cancel edit
const cancelEdit = () => {
  editingIndex.value = null
  editedText.value = ''
}

// Delete
const deleteGoal = (index) => {
  if (confirm('هل أنت متأكد من حذف هذا الهدف؟')) {
    goals.value.splice(index, 1)
  }
}

// Enter
const handleKeyPress = (event) => {
  if (event.key === 'Enter') {
    addGoal()
  }
}

// Computed
const completedGoals = computed(() => goals.value.filter((g) => g.completed).length)

const pendingGoals = computed(() => goals.value.filter((g) => !g.completed).length)

const filteredGoals = computed(() => {
  if (filterType.value === 'completed') {
    return goals.value.filter((g) => g.completed)
  }
  if (filterType.value === 'pending') {
    return goals.value.filter((g) => !g.completed)
  }
  return goals.value
})

// Watch save
watch(
  goals,
  () => {
    saveGoals()
  },
  { deep: true },
)

// Dark mode
watch(darkMode, (value) => {
  localStorage.setItem('darkMode', JSON.stringify(value))

  document.body.style.background = value
    ? 'linear-gradient(135deg,#1e1b4b,#4c1d95)'
    : 'linear-gradient(135deg,#667eea,#764ba2)'
})

// Mounted
onMounted(() => {
  loadGoals()

  // ✅ استرجاع وضع الدارك مود من localStorage
  const savedDarkMode = localStorage.getItem('darkMode')
  if (savedDarkMode !== null) {
    darkMode.value = JSON.parse(savedDarkMode)
  }

  nextTick(() => {
    inputRef.value?.focus()
  })
})
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Cairo', sans-serif;
}
body {
  min-height: 100vh;
  margin: 0;
  padding: 0;
}
.container {
  margin: 0 auto;
  width: 100%;
  max-width: 650px;
  background: #fff;
  padding: 30px;
  border-radius: 24px;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.12);
}

h1 {
  text-align: center;
  margin-bottom: 20px;
}

.stats {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
  padding: 15px;
  background: #f8f9ff;
  border-radius: 14px;
  font-weight: 700;
}

.filters {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.input-group {
  display: flex;
  gap: 10px;
  margin-bottom: 12px;
}

input {
  flex: 1;
  padding: 14px;
  border-radius: 14px;
  border: 2px solid #eee;
  outline: none;
}

button {
  cursor: pointer;
  border: none;
}

button.add {
  padding: 12px;
  background: #6a11cb;
  color: white;
  border-radius: 12px;
}

ul {
  list-style: none;
  margin-top: 15px;
}

.goal-item {
  background: #fff;
  border: 1px solid #eee;
  padding: 16px;
  margin-bottom: 12px;
  border-radius: 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.task-text {
  cursor: pointer;
  font-weight: 600;
}

.completed {
  text-decoration: line-through;
  color: gray;
}

.actions {
  display: flex;
  gap: 8px;
}

.edit {
  background: #ffb400;
  color: white;
  padding: 6px 10px;
  border-radius: 8px;
}

.delete {
  background: #ff4d4d;
  color: white;
  padding: 6px 10px;
  border-radius: 8px;
}

/* ✨ Inline edit input */
.edit-input {
  padding: 8px;
  border-radius: 10px;
  border: 2px solid #6a11cb;
  width: 100%;
}

.container {
  margin: 20px auto;
  width: 100%;
  max-width: 650px;
  background: #ffffff;
  padding: 30px;
  border-radius: 24px;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.12);
  transition: all 0.3s ease;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}

.stats {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 20px;
  background: #f8f9ff;
  padding: 15px;
  border-radius: 14px;
  font-weight: 700;
  color: #444;
}

.stats span {
  flex: 1;
  text-align: center;
}

.dark-btn {
  width: 100%;
  margin-bottom: 15px;
  padding: 12px;
  border-radius: 12px;
  border: none;
  background: #222;
  color: white;
  font-weight: bold;
  cursor: pointer;
  transition: 0.3s;
}

.dark-btn:hover {
  background: #111;
}

.filters {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.filters button {
  flex: 1;
}

.input-group {
  display: flex;
  gap: 10px;
  margin-bottom: 12px;
}

input {
  flex: 1;
  padding: 14px;
  border-radius: 14px;
  border: 2px solid #eee;
  outline: none;
  transition: all 0.3s;
  font-size: 15px;
}

input:focus {
  border-color: #6a11cb;
  box-shadow: 0 0 10px rgba(106, 17, 203, 0.15);
}

button {
  border: none;
  cursor: pointer;
  transition: all 0.3s;
}

button:hover {
  transform: translateY(-2px);
}

button.add {
  padding: 12px 15px;
  background: #6a11cb;
  color: white;
  border-radius: 12px;
  font-weight: 600;
}

button.add:hover {
  background: #5011a8;
}

#error {
  color: #ff4d4d;
  font-size: 13px;
  margin-bottom: 8px;
  min-height: 18px;
  font-weight: 600;
}

.success {
  color: #28a745;
  font-size: 14px;
  margin-bottom: 10px;
  font-weight: 600;
}

ul {
  list-style: none;
  margin-top: 15px;
  max-height: 400px;
  overflow-y: auto;
  padding-right: 5px;
}

ul::-webkit-scrollbar {
  width: 6px;
}

ul::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 10px;
}

ul::-webkit-scrollbar-thumb {
  background: #6a11cb;
  border-radius: 10px;
}

.goal-item {
  background: white;
  border: 1px solid #eee;
  padding: 16px;
  margin-bottom: 12px;
  border-radius: 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  transition: all 0.3s ease;
  animation: fadeIn 0.3s ease;
}

.goal-item:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.08);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.task-text {
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  transition: 0.2s;
  word-break: break-word;
}

.task-text:hover {
  color: #6a11cb;
}

.completed {
  text-decoration: line-through;
  color: gray;
  opacity: 0.7;
}

.date {
  display: block;
  margin-top: 6px;
  color: #888;
  font-size: 12px;
}

.actions {
  display: flex;
  gap: 8px;
}

.actions button {
  padding: 7px 12px;
  border-radius: 10px;
  font-size: 12px;
  font-weight: 600;
}

.edit {
  background: #ffb400;
  color: white;
}

.edit:hover {
  background: #e5a200;
}

.delete {
  background: #ff4d4d;
  color: white;
}

.delete:hover {
  background: #e63e3e;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: #999;
  font-size: 15px;
  background: #f9f9f9;
  border-radius: 14px;
  margin-top: 15px;
}

/* Dark Mode */

.dark {
  background: #1f2937;
  color: white;
}

.dark h1 {
  color: white;
}

.dark .goal-item {
  background: #374151;
  border-color: #4b5563;
}

.dark .stats {
  background: #374151;
  color: white;
}

.dark input {
  background: #374151;
  color: white;
  border-color: #4b5563;
}

.dark .date {
  color: #d1d5db;
}

.dark .empty-state {
  background: #374151;
  color: white;
}

/* Mobile */

@media (max-width: 768px) {
  .container {
    width: 95%;
    padding: 20px;
  }

  .input-group {
    flex-direction: column;
  }

  .filters {
    flex-direction: column;
  }

  .stats {
    flex-direction: column;
  }

  .goal-item {
    flex-direction: column;
    align-items: flex-start;
  }

  .actions {
    width: 100%;
  }

  .actions button {
    flex: 1;
  }
}
</style>
