<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick } from 'vue'

type QType = 'add' | 'sub' | 'mix'
type Mode = 'new' | 'wrong'

interface Question {
  a: number
  b: number
  op: '+' | '-'
  answer: number
}

const types: { value: QType; label: string }[] = [
  { value: 'add', label: '加法' },
  { value: 'sub', label: '减法' },
  { value: 'mix', label: '加减混合' },
]

const qtype = ref<QType>('mix')
const min = ref(1)
const max = ref(20)
const mode = ref<Mode>('new')

const question = ref<Question | null>(null)
const input = ref('')
const feedback = ref<'none' | 'right' | 'wrong'>('none')
const answered = ref(false)
const inputRef = ref<HTMLInputElement | null>(null)

const total = ref(0)
const correct = ref(0)
const wrongList = ref<Question[]>([])

const accuracy = computed(() =>
  total.value === 0 ? '—' : Math.round((correct.value / total.value) * 100) + '%'
)

const key = (q: Question) => `${q.a}${q.op}${q.b}`

function rand(lo: number, hi: number) {
  return Math.floor(Math.random() * (hi - lo + 1)) + lo
}

function genQuestion(): Question {
  const lo = Math.max(0, Math.min(Number(min.value) || 0, Number(max.value) || 0))
  const hi = Math.max(0, Math.max(Number(min.value) || 0, Number(max.value) || 0))
  let op: '+' | '-'
  if (qtype.value === 'add') op = '+'
  else if (qtype.value === 'sub') op = '-'
  else op = Math.random() < 0.5 ? '+' : '-'
  let a = rand(lo, hi)
  let b = rand(lo, hi)
  if (op === '-' && a < b) [a, b] = [b, a] // 减法不出现负数
  return { a, b, op, answer: op === '+' ? a + b : a - b }
}

function next() {
  if (mode.value === 'wrong') {
    if (wrongList.value.length === 0) {
      question.value = null
      return
    }
    question.value = { ...wrongList.value[rand(0, wrongList.value.length - 1)] }
  } else {
    question.value = genQuestion()
  }
  input.value = ''
  feedback.value = 'none'
  answered.value = false
  nextTick(() => inputRef.value?.focus())
}

function judge() {
  const q = question.value!
  answered.value = true
  total.value++
  if (Number(input.value) === q.answer) {
    correct.value++
    feedback.value = 'right'
    const i = wrongList.value.findIndex(w => key(w) === key(q))
    if (i !== -1) {
      wrongList.value.splice(i, 1)
      save()
    }
  } else {
    feedback.value = 'wrong'
    if (!wrongList.value.some(w => key(w) === key(q))) {
      wrongList.value.push({ ...q })
      save()
    }
  }
}

function onEnter() {
  if (!question.value) return
  if (answered.value) {
    next()
    return
  }
  if (input.value.trim() === '') return
  judge()
}

function save() {
  localStorage.setItem('wrong-list', JSON.stringify(wrongList.value))
}

watch([qtype, min, max, mode], next)

onMounted(() => {
  try {
    wrongList.value = JSON.parse(localStorage.getItem('wrong-list') || '[]')
  } catch {
    wrongList.value = []
  }
  next()
})
</script>

<template>
  <div class="min-h-screen bg-slate-100 flex items-center justify-center p-4">
    <div class="w-full max-w-md bg-white rounded-2xl shadow p-6 space-y-6">
      <h1 class="text-2xl font-bold text-center">口算练习</h1>

      <div class="space-y-3 text-sm">
        <div class="flex items-center gap-3">
          <span class="w-16 shrink-0 text-slate-500">题型</span>
          <label v-for="t in types" :key="t.value" class="flex items-center gap-1 cursor-pointer">
            <input type="radio" v-model="qtype" :value="t.value" />{{ t.label }}
          </label>
        </div>
        <div class="flex items-center gap-2">
          <span class="w-16 shrink-0 text-slate-500">范围</span>
          <input v-model.number="min" type="number" min="0" class="w-20 border rounded px-2 py-1" />
          <span>~</span>
          <input v-model.number="max" type="number" min="0" class="w-20 border rounded px-2 py-1" />
        </div>
        <div class="flex items-center gap-3">
          <span class="w-16 shrink-0 text-slate-500">题目来源</span>
          <label class="flex items-center gap-1 cursor-pointer">
            <input type="radio" v-model="mode" value="new" />新题
          </label>
          <label class="flex items-center gap-1 cursor-pointer">
            <input type="radio" v-model="mode" value="wrong" />错题合集（{{ wrongList.length }}）
          </label>
        </div>
      </div>

      <div v-if="question" class="text-center space-y-4">
        <div class="text-4xl font-mono font-bold tracking-wider">
          {{ question.a }} {{ question.op }} {{ question.b }} = ?
        </div>
        <input
          ref="inputRef"
          v-model="input"
          type="text"
          inputmode="numeric"
          placeholder="输入答案，回车提交"
          class="w-full border-2 rounded-lg px-4 py-3 text-2xl text-center focus:outline-none"
          :class="{
            'border-slate-300 focus:border-blue-500': feedback === 'none',
            'border-green-500': feedback === 'right',
            'border-red-500': feedback === 'wrong',
          }"
          @keyup.enter="onEnter"
        />
        <div class="h-6 text-sm font-bold">
          <span v-if="feedback === 'right'" class="text-green-600">✓ 回答正确，回车下一题</span>
          <span v-else-if="feedback === 'wrong'" class="text-red-600">
            ✗ 正确答案：{{ question.answer }}，已收入错题，回车继续
          </span>
        </div>
      </div>
      <div v-else class="py-8 text-center text-slate-400">
        错题合集为空，请切换到「新题」练习
      </div>

      <div class="flex justify-around border-t pt-4 text-center text-sm">
        <div>
          <div class="text-xl font-bold">{{ total }}</div>
          <div class="text-slate-500">已答</div>
        </div>
        <div>
          <div class="text-xl font-bold text-green-600">{{ correct }}</div>
          <div class="text-slate-500">正确</div>
        </div>
        <div>
          <div class="text-xl font-bold text-blue-600">{{ accuracy }}</div>
          <div class="text-slate-500">正确率</div>
        </div>
      </div>
    </div>
  </div>
</template>
