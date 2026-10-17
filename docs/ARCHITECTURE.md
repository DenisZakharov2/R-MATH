# Архитектура приложения R MATH

Документ описывает внутреннее устройство одностраничного приложения `index.html` для разработчиков и контрибьюторов.

## 1. Обзор архитектуры

Приложение построено по принципу **Single-File Zero-Dependency SPA**:
* **Разметка (HTML5):** 5 основных экранов (`#menu-screen`, `#profile-screen`, `#shop-screen`, `#archive-screen`, `#game-screen`) и модальное окно итогов раунда (`#result-modal`), переключаемых через управление `style.display`.
* **Стили (CSS3):** модульная система стилей на базе CSS Custom Properties (`--neon-blue`, `--neon-green`, `--cell-size` и др.) с адаптивными медиазапросами для мобильных экранов, планшетов и десктопов.
* **Логика (Vanilla JS):** 7 логических модулей внутри единого тега `<script>`.

## 2. Модульная структура скрипта

| Модуль | Основные функции и константы | Назначение |
| :--- | :--- | :--- |
| **1. Справочники** | `SKIN_TONES`, `BASE_HAIR_STYLES`, `BASE_HAIR_COLORS`, `EYE_STYLES`, `SHOP_ITEMS`, `BLOCK_TITLES` | Хранение каталога из 38 предметов магазина, базовых настроек внешности и названий учебных блоков. |
| **2. Генератор SVG** | `buildHeroSVG(uid)`, `renderAllAvatars()` | Векторная сборка персонажа в координатной сетке `120×130` из 7 слоёв: фон/крылья (`backSVG`), задний слой волос (`hairBackSVG`), руки и торс с одеждой (`topDecor`), голова, передний слой причёски (`hairFrontSVG`), глаза (`eyesSVG`), очки (`glassesSVG`), головной убор (`hatSVG`) и питомец (`petSVG`). |
| **3. Звуковой движок** | `initAudio()`, `playSound(type)`, `toggleSound()` | Процедурный синтез звуков через `AudioContext` (`OscillatorNode` + `GainNode`) для событий `click`, `correct`, `wrong`, `fail`, `win`. |
| **4. Состояние и хранилище** | `state`, `loadProgress()`, `saveProgress()`, `ensureDailyDate()` | Централизованное хранение состояния игры и синхронизация с `localStorage` (`roblox_math_trainer_save_v8`). |
| **5. Экраны и оверлеи** | `hideAllScreens()`, `returnFromOverlay()`, `openProfileEditor()`, `openShop()`, `openArchive()`, `renderMissions()` | Навигация между экранами с сохранением паузы активного раунда (`state.canResumeGame` и `state.pendingNextQuestion`). |
| **6. Игровое ядро** | `startBlock()`, `loadQuestion()`, `highlightCurrentStep()`, `handleDigitInput()`, `buyRoundHint()`, `applyMistakePenalty()`, `completeQuestionSuccess()` | Пошаговая проверка ввода цифр, обработка платной подсказки, начисление `R$` и учёт сердечек. |
| **7. Генераторы заданий** | `buildBlock1Question()`, `buildBlock2Question()`, `buildBlock3Question()`, `buildMultiplicationColumn()`, `buildDivisionCorner()`, `buildBlock4Blitz()`, `buildBlock5WordProblem()`, `buildWordProblemColumnTemplate()` | Построение DOM-сеток математических заданий и формирование массива пошаговой проверки `state.steps`. |

## 3. Механизм пошаговой проверки (`state.steps`)

Каждое задание при генерации заполняет массив `state.steps`. Каждый элемент массива представляет собой один атомарный шаг ввода одной цифры (0–9):

```javascript
{
  el: HTMLElement,           // Ссылка на DOM-элемент клетки (.cell-input)
  expected: Number,          // Ожидаемая цифра (0..9)
  shortLabel: String,        // Краткое описание шага для записи в Архив при ошибке
  stepMsg: String,           // Текст подсказки для баннера (#status-banner)
  cardEl: HTMLElement,       // (Опционально) Карточка подшага, которую нужно разблокировать (.unlocked)
  revealLabelEl: HTMLElement,// (Опционально) Элемент заголовка подшага в Блоке 2
  revealLabelHTML: String,   // (Опционально) HTML, открывающийся при достижении этого шага
  altExpectedDigit: Number,  // (Опционально) Альтернативная первая цифра при перестановке слагаемых в Блоке 5
  allowSwapAddends: Function // (Опционально) Колбэк обмена слагаемых местами в Блоке 5
}
```

### Предварительные фазы (`state.phase`)
* `'digits'` — стандартная фаза ввода цифр с экранной или физической клавиатуры по массиву `state.steps`.
* `'order'` (Блок 2) — перед вводом цифр ученик должен нажать на кружочки `.order-circle` над знаками операций в правильном порядке (`1 → 2 → 3`). После выбора последнего действия фаза переключается на `'digits'`.
* `'wp_op'` (Блок 5) — перед записью решения задачи в столбик ученик выбирает арифметическое действие (`+`, `−`, `×`, `÷`). После верного выбора открывается блок решения и фаза переключается на `'digits'`.
