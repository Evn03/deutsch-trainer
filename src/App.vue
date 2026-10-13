<script setup>
import { computed, ref } from 'vue'

const fileInput = ref(null)
const words = ref([])
const errors = ref([])
const fileName = ref('')
const showUploader = ref(true)
const selectedId = ref(null)
const showTranslation = ref(false)
const checked = ref(false)
const placements = ref({})

const genders = [
  { id: 'm', label: 'Мужской род', short: 'М. р.', article: 'der', color: 'blue' },
  { id: 'f', label: 'Женский род', short: 'Ж. р.', article: 'die', color: 'rose' },
  { id: 'n', label: 'Средний род', short: 'Ср. р.', article: 'das', color: 'green' },
]

const demoText = `[der Mann, der Tag, der Baum]
Правильный ответ:
М.Р: Mann, Tag, Baum
Ж.Р:
Ср.Р:
Перевод слов:
Mann - Мужчина
Tag - День
Baum - Дерево

[die Frau, die Nacht, die Blume]
Правильный ответ:
М.Р:
Ж.Р: Frau, Nacht, Blume
Ср.Р:
Перевод слов:
Frau - Женщина
Nacht - Ночь
Blume - Цветок

[das Kind, das Haus, das Buch]
Правильный ответ:
М.Р:
Ж.Р:
Ср.Р: Kind, Haus, Buch
Перевод слов:
Kind - Ребёнок
Haus - Дом
Buch - Книга`

const articles = [
  'der', 'die', 'das',
  'ein', 'eine',
  'den', 'dem', 'des',
  'einen', 'einem', 'einer', 'eines',
  'kein', 'keine', 'keinen', 'keinem', 'keiner', 'keines',
  'mein', 'meine', 'meinen', 'meinem', 'meiner', 'meines',
  'dein', 'deine', 'deinen', 'deinem', 'deiner', 'deines',
  'sein', 'seine', 'seinen', 'seinem', 'seiner', 'seines',
  'ihr', 'ihre', 'ihren', 'ihrem', 'ihrer', 'ihres',
  'unser', 'unsere', 'unseren', 'unserem', 'unserer', 'unseres',
  'euer', 'eure', 'euren', 'eurem', 'eurer', 'eures',
  'dieser', 'diese', 'dieses', 'diesen', 'diesem',
  'jeder', 'jede', 'jedes', 'jeden', 'jedem',
  'welcher', 'welche', 'welches', 'welchen', 'welchem',
  'mancher', 'manche', 'manches', 'manchen', 'manchem',
  'solcher', 'solche', 'solches', 'solchen', 'solchem'
].sort((a, b) => b.length - a.length)

function stripArticle(value) {
  let word = value.trim().replace(/^[\s"'“”„«»([{]+|[\s"'“”„«»)\]}.,;:!?]+$/g, '')
  let changed = true
  while (changed) {
    changed = false
    for (const article of articles) {
      const re = new RegExp('^' + article.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + '\\s+', 'i')
      if (re.test(word)) {
        word = word.replace(re, '').trim()
        changed = true
        break
      }
    }
  }
  return word
}

function parseText(text) {
  const lines = text.replace(/^\uFEFF/, '').split(/\r?\n/)
  const parsed = []
  const foundErrors = []
  let current = null
  let section = 'words'

  const finish = () => {
    if (!current) return
    const line = current.startLine
    if (!current.rawWords.length) foundErrors.push(`Строка ${line}: не найден список существительных в квадратных скобках.`)
    if (!current.genderSeen.m) foundErrors.push(`Блок со строки ${line}: отсутствует строка «М.Р:».`)
    if (!current.genderSeen.f) foundErrors.push(`Блок со строки ${line}: отсутствует строка «Ж.Р:».`)
    if (!current.genderSeen.n) foundErrors.push(`Блок со строки ${line}: отсутствует строка «Ср.Р:».`)
    if (!current.translationSeen) foundErrors.push(`Блок со строки ${line}: отсутствует раздел «Перевод слов:».`)

    const expectedByWord = new Map()
    for (const g of genders) {
      for (const w of current.answers[g.id]) {
        const key = w.toLocaleLowerCase('de-DE')
        if (expectedByWord.has(key)) {
          foundErrors.push(`Блок со строки ${line}: слово «${w}» указано в нескольких родах.`)
        } else {
          expectedByWord.set(key, g.id)
        }
      }
    }

    const cleanWords = current.rawWords.map(raw => {
      const noun = stripArticle(raw)
      const key = noun.toLocaleLowerCase('de-DE')
      const expected = expectedByWord.get(key)
      if (!noun) {
        foundErrors.push(`Строка ${current.wordsLine}: не удалось выделить существительное из «${raw}».`)
        return null
      }
      if (!expected) {
        foundErrors.push(`Блок со строки ${line}: для слова «${noun}» не указан правильный род в разделе «Правильный ответ».`)
      }
      const translation = current.translations.get(key)
      if (!translation) {
        foundErrors.push(`Блок со строки ${line}: нет перевода для слова «${noun}».`)
      }
      return { id: `${line}-${key}-${parsed.length}`, word: noun, expected: expected || null, translation: translation || '', placed: null }
    }).filter(Boolean)

    for (const [key, translation] of current.translations) {
      if (!cleanWords.some(w => w.word.toLocaleLowerCase('de-DE') === key)) {
        foundErrors.push(`Блок со строки ${line}: перевод «${key} - ${translation}» относится к слову, которого нет в списке.`)
      }
    }
    const unique = new Set()
    for (const w of cleanWords) {
      const key = w.word.toLocaleLowerCase('de-DE')
      if (unique.has(key)) foundErrors.push(`Блок со строки ${line}: слово «${w.word}» повторяется в списке.`)
      unique.add(key)
    }
    parsed.push(...cleanWords)
    current = null
  }

  const parseList = (raw, lineNo) => {
    const content = raw.trim().replace(/^\[/, '').replace(/\]$/, '')
    const items = content.split(',').map(s => s.trim()).filter(Boolean)
    if (!items.length) foundErrors.push(`Строка ${lineNo}: список пуст. Ожидается формат [der Mann, die Frau, das Kind].`)
    return items
  }

  lines.forEach((rawLine, index) => {
    const lineNo = index + 1
    const line = rawLine.trim()
    if (!line || line.startsWith('#')) return

    if (/^\[.*\]$/.test(line)) {
      finish()
      current = {
        startLine: lineNo, wordsLine: lineNo, rawWords: parseList(line, lineNo),
        answers: { m: [], f: [], n: [] }, genderSeen: { m: false, f: false, n: false },
        translations: new Map(), translationSeen: false
      }
      section = 'words'
      return
    }

    if (!current) {
      foundErrors.push(`Строка ${lineNo}: текст вне блока. Каждый набор начинайте со списка в квадратных скобках, например [der Mann, die Frau].`)
      return
    }

    if (/^Правильный\s+ответ\s*:/i.test(line)) {
      section = 'answers'
      return
    }
    if (/^Перевод\s+слов\s*:/i.test(line)) {
      section = 'translations'
      current.translationSeen = true
      return
    }

    const genderMatch = line.match(/^(М\.?\s*Р\.?|Мужской\s+род|Ж\.?\s*Р\.?|Женский\s+род|Ср\.?\s*Р\.?|Средний\s+род)\s*[:\-]\s*(.*)$/i)
    if (genderMatch) {
      const label = genderMatch[1].toLocaleLowerCase('ru-RU').replace(/\s/g, '')
      let id = null
      if (label.startsWith('м.') || label.startsWith('муж')) id = 'm'
      else if (label.startsWith('ж.') || label.startsWith('жен')) id = 'f'
      else id = 'n'
      current.genderSeen[id] = true
      current.answers[id].push(...genderMatch[2].split(/[,;]/).map(s => stripArticle(s)).filter(Boolean))
      section = 'answers'
      return
    }

    if (section === 'translations') {
      const match = line.match(/^(.+?)\s*[-–—]\s*(.+)$/)
      if (!match) {
        foundErrors.push(`Строка ${lineNo}: неверная запись перевода. Используйте «Mann - Мужчина».`)
        return
      }
      const word = stripArticle(match[1]).toLocaleLowerCase('de-DE')
      const translation = match[2].trim()
      if (current.translations.has(word)) foundErrors.push(`Строка ${lineNo}: перевод для «${word}» указан повторно.`)
      else current.translations.set(word, translation)
      return
    }

    if (section === 'words' && !current.rawWords.length) {
      foundErrors.push(`Строка ${lineNo}: ожидался список существительных в квадратных скобках.`)
    } else {
      foundErrors.push(`Строка ${lineNo}: не удалось распознать строку «${line}». Проверьте название раздела или формат записи.`)
    }
  })
  finish()

  if (!parsed.length && !foundErrors.length) foundErrors.push('Файл не содержит ни одного блока со списком существительных.')
  return { parsed, foundErrors }
}

function shuffleWords(items) {
  const shuffled = [...items]
  // Алгоритм Фишера — Йетса: перемешиваем копию, не меняя исходный массив.
  for (let i = shuffled.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]]
  }
  return shuffled
}

function resetRoundState() {
  checked.value = false
  selectedId.value = null
  showTranslation.value = false
  placements.value = {}
}

function loadText(text, name = 'Текстовый ввод') {
  const result = parseText(text)
  fileName.value = name
  errors.value = result.foundErrors
  resetRoundState()

  if (result.foundErrors.length || result.parsed.length === 0) {
    words.value = []
    showUploader.value = true
    const firstError = result.foundErrors[0] || 'Файл не содержит слов для упражнения.'
    window.alert(`Не удалось загрузить упражнение.
${firstError}

Исправь файл и попробуй снова.`)
    return
  }

  words.value = shuffleWords(result.parsed).map(w => ({ ...w, placed: null }))
  showUploader.value = false
}

function openFilePicker() {
  fileInput.value?.click()
}

function onFileCancel() {
  // Если пользователь отменил выбор, сохраняем уже загруженный урок.
  if (words.value.length && !errors.value.length) {
    showUploader.value = false
    return
  }
  window.alert('Файл не выбран. Выбери файл .txt, чтобы загрузить свой урок, или нажми «Пройти тестовый вариант».')
}

function onFileChange(event) {
  const file = event.target.files?.[0]
  event.target.value = ''
  if (!file) return

  if (!file.name.toLowerCase().endsWith('.txt')) {
    errors.value = ['Выбран неподдерживаемый формат. Загрузите файл с расширением .txt.']
    words.value = []
    fileName.value = file.name
    showUploader.value = true
    window.alert('Неверный формат файла. Поддерживаются только файлы .txt.')
    return
  }

  const reader = new FileReader()
  reader.onload = () => loadText(String(reader.result || ''), file.name)
  reader.onerror = () => {
    errors.value = ['Не удалось прочитать файл. Проверьте, что он доступен и сохранён в текстовом формате UTF-8.']
    words.value = []
    showUploader.value = true
    window.alert(errors.value[0])
  }
  reader.readAsText(file, 'UTF-8')
}

function placeWord(genderId) {
  if (!selectedId.value || checked.value) return
  placements.value = { ...placements.value, [selectedId.value]: genderId }
  selectedId.value = null
  showTranslation.value = false
}

function toggleSelect(word) {
  if (checked.value) return
  if (selectedId.value === word.id) {
    selectedId.value = null
    showTranslation.value = false
  } else {
    selectedId.value = word.id
    showTranslation.value = false
  }
}

function revealTranslation() {
  if (selectedId.value) showTranslation.value = !showTranslation.value
}

const placedCount = computed(() => Object.keys(placements.value).filter(id => placements.value[id]).length)
const allPlaced = computed(() => words.value.length > 0 && placedCount.value === words.value.length)
const correctCount = computed(() => words.value.filter(w => placements.value[w.id] === w.expected).length)
const unplaced = computed(() => words.value.filter(w => !placements.value[w.id]))
function placedWords(genderId) {
  return words.value.filter(w => placements.value[w.id] === genderId)
}
function wordStatus(word) {
  if (!checked.value) return ''
  return placements.value[word.id] === word.expected ? 'correct' : 'wrong'
}
function loadDemo() { loadText(demoText, 'Тестовый вариант') }
function startAnotherUpload() {
  showUploader.value = true
  errors.value = []
}
function resetExercise() {
  placements.value = {}
  checked.value = false
  selectedId.value = null
  showTranslation.value = false
  words.value = shuffleWords(words.value)
}
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <div class="brand-mark">N</div>
      <div>
        <div class="eyebrow">DEUTSCH · TRAINER</div>
        <h1>Род немецких существительных</h1>
      </div>
      <div class="topbar-spacer"></div>
      <span class="status-pill"><span class="status-dot"></span> Локальный режим</span>
    </header>

    <section class="intro">
      <div>
        <p class="eyebrow accent">ARTIKEL TRAINING</p>
        <h2>Убери артикль.<br /><span>Определи род.</span></h2>
        <p class="intro-copy">Загрузи свой список, распределяй существительные по родам и проверь результат. Артикли из списка удаляются автоматически.</p>
      </div>
      <div class="hero-symbol" aria-hidden="true">
        <div class="symbol-card der">der</div>
        <div class="symbol-card die">die</div>
        <div class="symbol-card das">das</div>
      </div>
    </section>

    <section v-if="showUploader || !words.length" class="upload-panel panel">
      <div class="section-heading">
        <div>
          <h3>1. Загрузить список</h3>
          <p>Выбери свой TXT-файл или попробуй тестовый вариант.</p>
        </div>
        <button class="button secondary" @click="loadDemo">Пройти тестовый вариант</button>
      </div>
      <input ref="fileInput" class="hidden-input" type="file" accept=".txt,text/plain" @change="onFileChange" @cancel="onFileCancel" />
      <button class="upload-zone upload-zone-button" type="button" @click="openFilePicker">
        <span class="upload-icon">↑</span>
        <span class="upload-title">Загрузить файл .txt</span>
        <span class="upload-subtitle">Файл обрабатывается в браузере и никуда не отправляется</span>
      </button>
      <div v-if="fileName && errors.length" class="file-info"><span class="file-icon">TXT</span><span>{{ fileName }}</span></div>
      <div v-if="errors.length" class="validation-box">
        <div class="validation-title"><span class="validation-icon">!</span> Проверка файла: {{ errors.length }} {{ errors.length === 1 ? 'ошибка' : 'ошибок' }}</div>
        <p class="validation-help">Исправь ошибки в файле и загрузи его снова.</p>
        <ol><li v-for="(error, i) in errors" :key="i">{{ error }}</li></ol>
      </div>
      <details class="format-help">
        <summary>Как должен выглядеть TXT-файл?</summary>
        <pre>[der Mann, die Frau, das Kind]
Правильный ответ:
М.Р: Mann
Ж.Р: Frau
Ср.Р: Kind
Перевод слов:
Mann - Мужчина
Frau - Женщина
Kind - Ребёнок</pre>
        <p>Каждый блок начинается с отдельной строки списка в квадратных скобках. Пустая строка между блоками необязательна. Поддерживаются формы артиклей: der/die/das, ein/eine и падежные формы, а также притяжательные и указательные формы (mein, dieser и др.). В списке указывай артикль вместе со словом; в ответах и переводах — само существительное.</p>
      </details>
    </section>

    <section v-else class="loaded-file-panel panel">
      <div class="loaded-file-icon">✓</div>
      <div class="loaded-file-copy">
        <strong>Урок загружен</strong>
        <span>{{ fileName }} · {{ words.length }} {{ words.length === 1 ? 'слово' : words.length < 5 ? 'слова' : 'слов' }}</span>
      </div>
      <button class="button secondary" @click="startAnotherUpload">Загрузить другой файл</button>
    </section>

    <section v-if="words.length && !showUploader" class="exercise">
      <div class="exercise-heading">
        <div>
          <div class="eyebrow accent">2. ПРАКТИКА</div>
          <h3>Распредели слова по колонкам</h3>
          <p>Нажми на слово, затем выбери колонку. Кнопка «Перевод» покажет перевод выбранного слова.</p>
        </div>
        <div class="progress-block">
          <div class="progress-label"><span>Распределено</span><strong>{{ placedCount }} / {{ words.length }}</strong></div>
          <div class="progress-track"><div class="progress-fill" :style="{ width: (placedCount / words.length * 100) + '%' }"></div></div>
        </div>
      </div>

      <div class="word-bank panel">
        <div class="bank-head"><div><h4>Слова для распределения</h4><p>Нажми на существительное, чтобы выбрать его</p></div><span class="count-badge">{{ unplaced.length }} осталось</span></div>
        <div v-if="unplaced.length" class="word-chips">
          <button v-for="word in unplaced" :key="word.id" class="word-chip" :class="{ selected: selectedId === word.id }" @click="toggleSelect(word)">{{ word.word }} <span class="chip-arrow">↗</span></button>
        </div>
        <div v-else class="all-placed">Все слова распределены. Можно проверять результат.</div>
        <div v-if="selectedId" class="selected-actions">
          <div class="selected-word">Выбрано: <strong>{{ words.find(w => w.id === selectedId)?.word }}</strong></div>
          <button class="button secondary" @click="revealTranslation">{{ showTranslation ? 'Скрыть перевод' : 'Перевод' }}</button>
          <span v-if="showTranslation" class="translation-reveal">{{ words.find(w => w.id === selectedId)?.translation || 'Перевод не указан' }}</span>
        </div>
      </div>

      <div class="columns">
        <section v-for="gender in genders" :key="gender.id" class="gender-column" :class="gender.color">
          <div class="column-head">
            <div class="article-token">{{ gender.article }}</div>
            <div class="column-title"><h4>{{ gender.label }}</h4><span>{{ placedWords(gender.id).length }} слов</span></div>
          </div>
          <button class="drop-target" :disabled="!selectedId || checked" @click="placeWord(gender.id)">
            <span class="drop-plus">+</span> {{ selectedId ? `Переместить сюда «${words.find(w => w.id === selectedId)?.word}»` : 'Выбери слово выше, затем нажми сюда' }}
          </button>
          <div class="placed-list">
            <div v-for="word in placedWords(gender.id)" :key="word.id" class="placed-word" :class="wordStatus(word)">
              <span class="placed-name">{{ word.word }}</span>
              <span v-if="checked" class="result-mark">{{ wordStatus(word) === 'correct' ? '✓' : '×' }}</span>
              <span
                v-if="checked && wordStatus(word) === 'wrong' && word.expected"
                class="correct-gender-hint"
                style="display: block; flex-basis: 100%; font-size: 0.82rem; margin-top: 3px; color: #287a43;"
              >Правильно: {{ genders.find(g => g.id === word.expected)?.article }} — {{ genders.find(g => g.id === word.expected)?.label }}</span>
              <button v-else-if="!checked" class="remove-word" title="Вернуть в список" @click="placements = { ...placements, [word.id]: null }; selectedId = null">×</button>
            </div>
            <div v-if="!placedWords(gender.id).length" class="empty-column">Здесь пока пусто</div>
          </div>
        </section>
      </div>

      <div class="check-panel panel">
        <div v-if="!checked">
          <div><h4>Готов проверить себя?</h4><p>{{ allPlaced ? 'Все слова распределены — можно проверить ответы.' : `Осталось распределить ${words.length - placedCount} ${words.length - placedCount === 1 ? 'слово' : 'слов'}. Проверить можно и раньше.` }}</p></div>
          <button class="button primary" :disabled="!allPlaced" @click="checked = true">Проверить ответы <span>→</span></button>
        </div>
        <div v-else class="results-summary">
          <div class="score-circle" :class="{ perfect: correctCount === words.length }">{{ correctCount }}<span>/ {{ words.length }}</span></div>
          <div class="result-copy"><h4>{{ correctCount === words.length ? 'Отлично! Все ответы верны.' : 'Проверка завершена' }}</h4><p>Зелёный — верно, красный — ошибка. Не распределённые слова считаются неверными.</p></div>
          <button class="button primary" @click="resetExercise">Попробовать снова</button>
        </div>
      </div>
    </section>
    <footer><span>DEUTSCH · NOMEN</span><span>Тренажёр родов существительных</span></footer>
  </main>
</template>