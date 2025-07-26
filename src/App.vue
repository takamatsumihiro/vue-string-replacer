<script setup>
import { ref, computed, watch, onMounted } from 'vue'
 
// ローカルストレージから復元 or 初期カテゴリ
const saved = localStorage.getItem('spending-categories')
const categories = ref(saved ? JSON.parse(saved) : [
  { name: '食費', amount: 0, color: '#f56565' },
  { name: '交通費', amount: 0, color: '#48bb78' },
  { name: '趣味', amount: 0, color: '#4299e1' },
  { name: '住居', amount: 0, color: '#ed8936' },
])
 
// 新規カテゴリ入力状態
const newCategoryName = ref('')
const newCategoryColor = ref('#a0aec0')
const errorMessage = ref('')
 
// Undo用スタック（最後に削除したカテゴリ）
const undoStack = ref([])
 
// 画面幅に応じてSVGサイズ設定（モバイル対応）
const svgSize = ref(200)
const updateSvgSize = () => {
  const w = window.innerWidth * 0.9
  svgSize.value = w > 200 ? 200 : w
}
onMounted(() => {
  updateSvgSize()
  window.addEventListener('resize', updateSvgSize)
})
 
// ローカルストレージにカテゴリを保存
watch(categories, (val) => {
  localStorage.setItem('spending-categories', JSON.stringify(val))
}, { deep: true })
 
// バリデーションとカテゴリ追加
const addCategory = () => {
  const name = newCategoryName.value.trim()
  if (!name) {
    errorMessage.value = 'カテゴリ名を入力してください。'
    return
  }
  if (categories.value.some(c => c.name === name)) {
    errorMessage.value = '同じ名前のカテゴリはすでに存在します。'
    return
  }
  categories.value.push({ name, amount: 0, color: newCategoryColor.value })
  newCategoryName.value = ''
  newCategoryColor.value = '#a0aec0'
  errorMessage.value = ''
}
 
// Enterキーでカテゴリ追加対応
const onNewCategoryKeyup = (e) => {
  if (e.key === 'Enter') addCategory()
}
 
// 削除確認＆Undo用にスタック保存
const confirmRemove = (index) => {
  const cat = categories.value[index]
  if (confirm(`カテゴリ「${cat.name}」を削除しますか？`)) {
    undoStack.value.push(cat)
    categories.value.splice(index, 1)
  }
}
 
// Undo機能（最後に削除したカテゴリを戻す）
const undoRemove = () => {
  if (undoStack.value.length === 0) return
  const cat = undoStack.value.pop()
  if (!categories.value.some(c => c.name === cat.name)) {
    categories.value.push(cat)
  }
}
 
// 合計支出
const total = computed(() =>
  categories.value.reduce((sum, c) => sum + Number(c.amount), 0)
)
 
// 最大支出（棒グラフの基準）
const maxAmount = computed(() => {
  if (categories.value.length === 0) return 0
  return Math.max(...categories.value.map(c => Number(c.amount)))
})
 
// 各カテゴリの割合計算（円グラフ用）
const categoryPercentages = computed(() =>
  categories.value
    .filter(c => c.amount > 0)
    .map(cat => ({
      ...cat,
      percent: total.value > 0 ? (cat.amount / total.value) * 100 : 0,
    }))
)
 
// SVG円グラフの描画計算関数
function polarToCartesian(cx, cy, radius, angleDegrees) {
  const angleRadians = (angleDegrees - 90) * Math.PI / 180.0
  return {
    x: cx + radius * Math.cos(angleRadians),
    y: cy + radius * Math.sin(angleRadians),
  }
}
function describeArc(cx, cy, radius, startAngle, endAngle) {
  const start = polarToCartesian(cx, cy, radius, endAngle)
  const end = polarToCartesian(cx, cy, radius, startAngle)
  const largeArcFlag = endAngle - startAngle <= 180 ? "0" : "1"
  return [
    `M ${cx} ${cy}`,
    `L ${start.x} ${start.y}`,
    `A ${radius} ${radius} 0 ${largeArcFlag} 0 ${end.x} ${end.y}`,
    "Z"
  ].join(" ")
}
const getArcPath = (index) => {
  const startAngle = categoryPercentages.value.slice(0, index).reduce((acc, c) => acc + c.percent * 3.6, 0)
  const endAngle = startAngle + categoryPercentages.value[index].percent * 3.6
  return describeArc(16, 16, 16, startAngle, endAngle)
}
</script>
 
<template>
  <div class="p-4 max-w-xl mx-auto w-full text-sm">
    <h1 class="text-2xl font-bold mb-6 text-center">支出管理アプリ</h1>
 
    <!-- 新規カテゴリ追加 -->
    <section class="mb-6 flex flex-wrap gap-3 items-center justify-center">
      <input
        v-model="newCategoryName"
        placeholder="カテゴリ名"
        @keyup="onNewCategoryKeyup"
        class="border rounded px-3 py-2 text-lg flex-grow min-w-[140px]"
        aria-label="新規カテゴリ名"
      />
      <input
        type="color"
        v-model="newCategoryColor"
        title="カテゴリ色を選択"
        class="w-12 h-12 p-0 border rounded cursor-pointer"
        aria-label="新規カテゴリ色"
      />
      <button
        @click="addCategory"
        class="bg-green-600 text-white px-5 py-2 rounded text-lg hover:bg-green-700 transition"
        aria-label="カテゴリ追加"
      >
        追加
      </button>
    </section>
    <p v-if="errorMessage" class="text-red-600 text-center mb-4" role="alert">{{ errorMessage }}</p>
 
    <!-- 既存カテゴリ一覧 -->
    <section>
      <div v-for="(cat, i) in categories" :key="cat.name" class="mb-6">
        <div class="flex flex-wrap justify-between items-center gap-3 mb-2">
          <label class="font-semibold flex items-center gap-4 min-w-[130px]">
            {{ cat.name }}
            <input
              type="color"
              v-model="cat.color"
              class="w-10 h-10 p-0 border rounded cursor-pointer"
              title="色を変更"
              aria-label="カテゴリ色変更"
            />
          </label>
          <button
            @click="confirmRemove(i)"
            class="text-red-600 text-sm px-3 py-1 border border-red-600 rounded hover:bg-red-100 transition"
            aria-label="カテゴリ削除"
          >
            削除
          </button>
        </div>
        <input
          type="number"
          v-model.number="cat.amount"
          min="0"
          class="border rounded px-3 py-2 w-full text-lg"
          inputmode="numeric"
          aria-label="支出金額入力"
        />
        <div class="h-6 bg-gray-300 rounded mt-2">
          <div
            class="h-6 rounded transition-all duration-300"
            :style="{
              width: maxAmount > 0 ? (cat.amount / maxAmount) * 100 + '%' : '0%',
              backgroundColor: cat.color,
            }"
          ></div>
        </div>
      </div>
    </section>
 
    <!-- Undoボタン -->
    <div class="text-center mb-4">
      <button
        @click="undoRemove"
        :disabled="undoStack.length === 0"
        class="bg-yellow-500 text-white px-4 py-2 rounded disabled:opacity-50"
        aria-label="削除取り消し"
      >
        削除を取り消す
      </button>
    </div>
 
    <!-- 合計支出 -->
    <div class="mt-2 font-semibold text-center text-lg">
      合計支出: {{ total.toLocaleString() }} 円
    </div>
 
    <!-- 円グラフ -->
    <section class="mt-8">
      <h2 class="font-bold mb-3 text-center text-lg">カテゴリ別支出割合</h2>
      <svg
        :width="svgSize"
        :height="svgSize"
        viewBox="0 0 32 32"
        class="block mx-auto"
        role="img"
        aria-label="カテゴリ別支出割合の円グラフ"
      >
        <g v-if="categoryPercentages.length">
          <template v-for="(cat, i) in categoryPercentages" :key="cat.name">
            <path
              :d="getArcPath(i)"
              :fill="cat.color"
              stroke="#fff"
              stroke-width="0.5"
            />
          </template>
        </g>
        <circle v-else cx="16" cy="16" r="16" fill="#eee" />
      </svg>
    </section>
 
    <!-- 支出割合リスト -->
    <section class="mt-6 max-w-md mx-auto">
      <ul>
        <li
          v-for="(cat, i) in categoryPercentages"
          :key="cat.name"
          class="flex justify-between items-center border-b py-2"
        >
          <span class="flex items-center gap-3">
            <span
              class="inline-block w-6 h-6 rounded-full"
              :style="{ backgroundColor: cat.color }"
              aria-hidden="true"
            ></span>
            <span>{{ cat.name }}</span>
          </span>
          <span>
            {{ cat.percent.toFixed(1) }}%（{{ Number(cat.amount).toLocaleString() }}円）
          </span>
        </li>
      </ul>
    </section>
  </div>
</template>
 
<style scoped>
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
button:disabled {
  cursor: not-allowed;
}
@media (max-width: 480px) {
  .text-lg {
    font-size: 1rem;
  }
  input[type='color'] {
    width: 36px;
    height: 36px;
  }
}
</style>
