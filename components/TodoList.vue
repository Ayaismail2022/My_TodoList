<template>
  <div class="stars-container">
    <div class="stars layer-1"></div>
    <div class="stars layer-2"></div>
    <div class="stars layer-3"></div>
  </div>

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
          <input
            v-if="editingIndex === index"
            :id="'edit-input-' + index"
            v-model="editedText"
            class="edit-input"
            @keyup.enter="saveEdit(index)"
            @keyup.esc="cancelEdit"
          />

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

// الحالة (State)
const goals = ref([])
const newGoalText = ref('')
const errorMessage = ref('')
const inputRef = ref(null)

const filterType = ref('all')
const darkMode = ref(false)

// حالة التعديل الداخلي
const editingIndex = ref(null)
const editedText = ref('')

const STORAGE_KEY = 'goals'

// تحميل الأهداف
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

// حفظ الأهداف
const saveGoals = () => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(goals.value))
}

// التحقق من المدخلات
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

// إضافة هدف جديد
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

// تبديل حالة الاكتمال
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
        editInput.setSelectionRange(editInput.value.length, editInput.value.length)
      }
    }, 50)
  })
}

// حفظ التعديل
const saveEdit = (index) => {
  const text = editedText.value.trim()

  if (!text) return

  goals.value[index].text = text
  editingIndex.value = null
  editedText.value = ''
}

// إلغاء التعديل
const cancelEdit = () => {
  editingIndex.value = null
  editedText.value = ''
}

// حذف الهدف
const deleteGoal = (index) => {
  if (confirm('هل أنت متأكد من حذف هذا الهدف؟')) {
    goals.value.splice(index, 1)
  }
}

// معالجة الضغط على المفاتيح
const handleKeyPress = (event) => {
  if (event.key === 'Enter') {
    addGoal()
  }
}

// Computed Properties
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

// مراقبة الحفظ التلقائي
watch(goals, () => {
  saveGoals()
}, { deep: true })

// مراقبة وضع الدارك مود
watch(darkMode, (value) => {
  localStorage.setItem('darkMode', JSON.stringify(value))
  updateBodyBackground(value)
})

const updateBodyBackground = (isDarkMode) => {
  document.body.style.background = isDarkMode
    ? 'linear-gradient(135deg, #0f172a, #1e1b4b, #2e103f)'
    : 'linear-gradient(135deg, #667eea, #764ba2, #a18cd1)'
}

onMounted(() => {
  loadGoals()

  const savedDarkMode = localStorage.getItem('darkMode')
  if (savedDarkMode !== null) {
    darkMode.value = JSON.parse(savedDarkMode)
  }

  updateBodyBackground(darkMode.value)

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
:global(body) {
  min-height: 100vh;
  margin: 0;
  padding: 0;
  overflow-x: hidden;
  position: relative;
  transition: background 0.3s ease;
}

/* ✨ ✨ ✨ نظام النجوم عالي الأداء (Super Lightweight) ✨ ✨ ✨ */
.stars-container {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 0;
  overflow: hidden;
  will-change: transform; /* تحضير المتصفح لتفعيل تسريع الكارت */
}

.stars {
  position: absolute;
  width: 100%;
  height: 100%;
  background-repeat: repeat;
  /* تحويل الإضاءة لجعلها ناصعة وخفيفة بالـ opacity بدلاً من الفلاتر الثقيلة */
  opacity: 0.8;
}

/* الطبقة الأولى: نجوم صغيرة بيضاء صافية */
.layer-1 {
  background-image:
    radial-gradient(1.5px 1.5px at 20px 30px, #ffffff 100%, rgba(255,255,255,0)),
    radial-gradient(2px 2px at 75px 120px, #ffffff 100%, rgba(255,255,255,0)),
    radial-gradient(1.5px 1.5px at 140px 70px, #ffffff 100%, rgba(255,255,255,0));
  background-size: 200px 200px;
  /* استخدام translate3d يُجبر المتصفح على تشغيل كرت الشاشة للتحريك */
  animation: moveFast 40s linear infinite;
}

/* الطبقة الثانية: نجوم متوسطة بيضاء صافية بنمط توزيع مختلف */
.layer-2 {
  background-image:
    radial-gradient(2px 2px at 45px 85px, #ffffff 100%, rgba(255,255,255,0)),
    radial-gradient(2.5px 2.5px at 115px 40px, #ffffff 100%, rgba(255,255,255,0));
  background-size: 280px 280px;
  animation: moveMedium 65s linear infinite;
  opacity: 0.6;
}

/* الطبقة الثالثة: نجوم متناثرة تومض مكانها فقط لتقليل حركات الاتجاهات المتداخلة الثقيلة */
.layer-3 {
  background-image:
    radial-gradient(1.2px 1.2px at 15px 15px, #ffffff 100%, rgba(255,255,255,0)),
    radial-gradient(2.2px 2.2px at 90px 160px, #ffffff 100%, rgba(255,255,255,0));
  background-size: 350px 350px;
  animation: justTwinkle 4s ease-in-out infinite;
  opacity: 0.7;
}

/* 🔄 أنيميشن خفيف جداً يغير المحاور بالـ Transform */
@keyframes moveFast {
  from { transform: translate3d(0, 0, 0); }
  to { transform: translate3d(100px, -200px, 0); }
}

@keyframes moveMedium {
  from { transform: translate3d(0, 0, 0); }
  to { transform: translate3d(-150px, 150px, 0); }
}

@keyframes justTwinkle {
  0%, 100% { opacity: 0.2; }
  50% { opacity: 0.8; }
}

/* الحاوية وتأثير الزجاج الخفيف المحسّن */
.container {
  position: relative;
  z-index: 10;
  margin: 40px auto;
  width: 100%;
  max-width: 650px;
  background: rgba(255, 255, 255, 0.92); /* زيادة التعتيم قليلاً لتخفيف عبء الـ blur */
  padding: 30px;
  border-radius: 24px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, background 0.3s ease;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
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
  border: 2px solid #e2e8f0;
  outline: none;
  transition: border-color 0.2s;
  font-size: 15px;
  background: #fff;
}

input:focus {
  border-color: #6a11cb;
}

button {
  border: none;
  cursor: pointer;
  font-weight: 600;
  transition: background 0.2s, transform 0.2s;
}

button:hover {
  transform: translateY(-2px);
}

button.add {
  padding: 12px 20px;
  background: #6a11cb;
  color: white;
  border-radius: 12px;
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
}

ul::-webkit-scrollbar-thumb {
  background: #6a11cb;
  border-radius: 10px;
}

.goal-item {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  padding: 16px;
  margin-bottom: 12px;
  border-radius: 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  transition: transform 0.2s;
}

.goal-item:hover {
  transform: translateY(-2px);
}

.task-text {
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  word-break: break-word;
}

.completed {
  text-decoration: line-through;
  color: #94a3b8;
  opacity: 0.65;
}

.date {
  display: block;
  margin-top: 6px;
  color: #64748b;
  font-size: 12px;
}

.actions {
  display: flex;
  gap: 8px;
}

.actions button {
  padding: 7px 14px;
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

.edit-input {
  padding: 8px;
  border-radius: 10px;
  border: 2px solid #6a11cb;
  width: 100%;
  background: white;
  color: #333;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: #64748b;
  font-size: 15px;
  background: #f8fafc;
  border-radius: 14px;
  margin-top: 15px;
}

/* 🌙 Dark Mode */
.dark {
  background: rgba(30, 41, 59, 0.94);
  color: #f8fafc;
}

.dark h1 {
  color: #ffffff;
}

.dark .goal-item {
  background: #334155;
  border-color: #475569;
}

.dark .stats {
  background: #1e293b;
  color: #e2e8f0;
}

.dark input {
  background: #334155;
  color: #ffffff;
  border-color: #475569;
}

.dark .edit-input {
  background: #1e293b;
  color: #fff;
}

.dark .date {
  color: #94a3b8;
}

.dark .empty-state {
  background: #1e293b;
  color: #94a3b8;
}

/* 📱 Mobile */
@media (max-width: 768px) {
  .container {
    width: 92%;
    padding: 20px;
  }

  .input-group, .filters, .stats, .goal-item {
    flex-direction: column;
    align-items: stretch;
  }

  .stats span {
    padding: 4px 0;
  }

  .actions {
    width: 100%;
    margin-top: 10px;
  }

  .actions button {
    flex: 1;
  }
}
</style>
