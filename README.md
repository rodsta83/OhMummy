# Oh Mummy — Документация проекта

## Обзор

«Oh Mummy» — игра в стиле классической аркады, где игрок исследует лабиринт-пирамиду 5x4.
Игрок обходит ячейки по периметру, оставляя следы. Когда все 4 стороны ячейки пройдены —
ячейка открывается, revealing её содержимое: сокровища, ключ, свиток или стража-мумию.
Цель — собрать ключ, найти дверь и перейти на следующий уровень, избегая мумий.

---

## Структура проекта

```
ReplicatedStorage/
  OhMummy/
    Config (ModuleScript)       — конфигурация игры
    Events/ (Folder)           — RemoteEvent-ы для клиент-серверной связи
      UpdateUI                  — сервер → клиент: обновление UI
      LevelStart                — сервер → клиент: начало уровня
      LevelComplete             — сервер → клиент: завершение уровня
      GameOver                  — сервер → клиент: конец игры
      FootprintUpdate           — сервер → клиент: обновление следов
      CellOpened                — сервер → клиент: ячейка открыта
      GameStart                 — клиент → сервер: запрос на рестарт
    README (ModuleScript)       — этот файл документации

ServerScriptService/
  OhMummy/
    MazeGenerator (ModuleScript) — генерация лабиринта
    FootprintSystem (ModuleScript) — система следов и открытия ячеек
    MummyAI (ModuleScript)       — ИИ мумий
    GameManager (Script)        — главный серверный контроллер

StarterPlayer/
  StarterPlayerScripts/
    ClientController (LocalScript) — клиентский контроллер UI и ввода

StarterGui/
  OhMummyUI (ScreenGui)         — интерфейс игры
    TopBar (Frame)               — верхняя панель: уровень, счёт, сокровища, ключ, свиток
      LevelLabel (TextLabel)
      ScoreLabel (TextLabel)
      TreasureLabel (TextLabel)
      KeyIcon (TextLabel)
      ScrollIcon (Frame)
        ScrollText (TextLabel)
        TimerLabel (TextLabel)
    MessageLabel (TextLabel)     — сообщения (начало уровня, находки, game over)
    RestartButton (TextButton)  — кнопка перезапуска игры
    InstructionsLabel (TextLabel) — инструкции для игрока
```

---

## Скрипты и их функции

---

### 1. Config (ReplicatedStorage.OhMummy.Config)

**Назначение:** Центральная конфигурация всех игровых систем. Содержит все настраиваемые параметры.

**Константы:**

| Параметр | Тип | Описание |
|---|---|---|
| `GRID_COLUMNS` | number | Количество колонок сетки (5) |
| `GRID_ROWS` | number | Количество рядов сетки (4) |
| `CELL_SIZE` | number | Размер ячейки в студсах (24) |
| `CELL_HEIGHT` | number | Высота стен ячейки (20) |
| `PATH_WIDTH` | number | Ширина дорожки между ячейками (6) |
| `LEVELS_PER_PYRAMID` | number | Количество уровней в пирамиде (5) |
| `CURRENT_LEVEL` | number | Текущий уровень (старт: 1) |
| `CELL_DISTRIBUTION` | table | Распределение содержимого ячеек: Treasure=10, Empty=6, Key=1, Scroll=1, Guardian=2 |
| `TREASURE_VALUE` | number | Очки за одно сокровище (100) |
| `TREASURE_PER_LEVEL` | number | Количество сокровищ на уровень (10) |
| `MUMMY_SPEED` | number | Скорость мумии (8 studs/sec) |
| `MUMMY_COUNT_PER_LEVEL` | number | Количество мумий на уровень (2) |
| `MUMMY_DAMAGE` | number | Урон мумии (100 — мгновенная смерть) |
| `MUMMY_UPDATE_INTERVAL` | number | Интервал обновления ИИ (0.1 сек) |
| `MUMMY_GRACE_PERIOD` | number | Период неподвижности мумий на старте (3 сек) |
| `PLAYER_SPEED` | number | Скорость игрока (16) |
| `PLAYER_START_HEALTH` | number | Начальное здоровье игрока (100) |
| `SCROLL_PROTECTION_DURATION` | number | Длительность защиты свитком (10 сек) |
| `FOOTPRINT_SIZE` | number | Размер следа (3 studs) |
| `FOOTPRINT_FADE_TIME` | number | Время исчезновения следа (30 сек) |
| `CELLS_NEEDED_TO_OPEN` | number | Количество сторон для открытия ячейки (4) |
| `COLORS` | table | Цвета для всех элементов игры |
| `UI_COLORS` | table | Цвета интерфейса |

---

### 2. MazeGenerator (ServerScriptService.OhMummy.MazeGenerator)

**Назначение:** Генерирует лабиринт для каждого уровня. Создаёт ячейки, стены, дорожки, внешнюю границу и дверь.

**Локальные функции:**

- `generateCellContents()` — Создаёт перемешанный список типов ячеек на основе `CELL_DISTRIBUTION`. Возвращает массив строк ("Treasure", "Empty", "Key", "Scroll", "Guardian").

- `createCell(col, row, cellType)` — Создаёт одну ячейку с полом, четырьмя стенами и контентом. Возвращает `(cellModel, worldX, worldZ)`.
  - Создаёт Model с именем `Cell_{col}_{row}`
  - Добавляет Part «Floor» (пол ячейки)
  - Добавляет Folder «CellWalls» с 4 стенами (WallN, WallS, WallE, WallW)
  - Добавляет Part «Content» (сокровище/ключ/свиток/страж), скрытый до открытия
  - Устанавливает атрибуты: `CellType`, `Col`, `Row`, `IsOpened`, `WorldX`, `WorldZ`

- `createPath(col, row)` — Создаёт горизонтальные и вертикальные дорожки между ячейками. Возвращает массив Part-ов.

- `createOuterPaths()` — Создаёт внешние дорожки по периметру сетки (север, юг, восток, запад + углы). Возвращает массив Part-ов.

- `createBoundary()` — Создает внешние стены-границы. Южная стена разделена на две части с зазором для двери. Возвращает массив Part-ов.

- `createDoor()` — Создаёт дверь перехода на следующий уровень в южной стене. Возвращает Part с атрибутами `IsDoor=true`, `IsLocked=true`.

**Публичные методы:**

- `MazeGenerator.generateLevel(levelNumber)` — Главная функция генерации. Создаёт Model уровня со всеми ячейками, дорожками, стенами и дверью. Возвращает `(mazeModel, cellData, door)`.
  - `cellData` — массив таблиц: `{ model, col, row, cellType, worldX, worldZ, isOpened }`

- `MazeGenerator.getSpawnPosition()` — Возвращает позицию спавна игрока (южный вход в лабиринт). `Vector3`

---

### 3. FootprintSystem (ServerScriptService.OhMummy.FootprintSystem)

**Назначение:** Отслеживает перемещение игрока вокруг ячеек. Каждая ячейка имеет 4 стороны (N, S, E, W). Когда все 4 стороны пройдены — ячейка открывается.

**Методы:**

- `FootprintSystem.new(cellData)` — Конструктор. Создаёт объект системы следов. Принимает `cellData` от MazeGenerator. Инициализирует состояние каждой ячейки: `{ North=false, South=false, East=false, West=false, isOpened=false }`.

- `FootprintSystem:checkPlayerPosition(playerPos)` — Проверяет позицию игрока относительно всех ячеек. Определяет, мимо какой стороны какой ячейки проходит игрок. Помечает пройденные стороны. Возвращает массив открытых ячеек (если ячейка открылась) или `nil`.
  - Проверяет 4 зоны вокруг каждой ячейки (в пределах ширины дорожки)
  - Вызывает `markSide()` для каждой подходящей стороны

- `FootprintSystem:markSide(cellModel, side, playerPos)` — Помечает сторону ячейки как пройденную. Создаёт визуальный след. Если все 4 стороны пройдены — помечает ячейку как открытую и возвращает `cellModel`, иначе `nil`.

- `FootprintSystem:createFootprint(cellModel, side, playerPos)` — Создаёт визуальный Part-след на дорожке. Плоский, неколлизионный, цвет `Footprint`.

- `FootprintSystem:openCell(cellModel, onCellOpened)` — Открывает ячейку: делает контент видимым, меняет цвет пола, удаляет стены ячейки. Вызывает колбэк `onCellOpened(cellModel, cellType)`.

- `FootprintSystem:getOpenedCount()` — Возвращает количество открытых ячеек.

- `FootprintSystem:getOpenedCountByType(cellType)` — Возвращает количество открытых ячеек указанного типа.

---

### 4. MummyAI (ServerScriptService.OhMummy.MummyAI)

**Назначение:** Управляет спавном, перемещением и враждебным поведением мумий. Мумии преследуют игрока через PathfindingService.

**Локальные функции:**

- `createMummy(position, isGuardian)` — Создаёт модель мумии. Состоит из:
  - `HumanoidRootPart` (3x6x3) — основное тело
  - `Head` (2.5x2.5x2.5) — голова, приваренная к HRP
  - `Humanoid` — с WalkSpeed = MUMMY_SPEED
  - Атрибуты: `IsMummy=true`, `IsGuardian=bool`
  - Возвращает Model

**Методы:**

- `MummyAI.new(maze, cellData, levelNumber)` — Конструктор. Создаёт контроллер ИИ мумий. Инициализирует: `mummies={}`, `mummyPaths={}`, `scrollProtectedMummies={}`.

- `MummyAI:spawnMummies()` — Спавнит мумий для уровня:
  - `MUMMY_COUNT_PER_LEVEL` обычных мумий в случайных позициях на дорожках
  - Мумий-стражей из ячеек типа "Guardian"

- `MummyAI:getRandomPathPosition()` — Возвращает случайную позицию на дорожке лабиринта. `Vector3`

- `MummyAI:start(targetPlayer)` — Запускает ИИ мумий:
  - Замораживает мумий на `MUMMY_GRACE_PERIOD` секунд
  - Запускает цикл пересчёта путей (каждые 1 сек) через `computePath()`
  - Запускает цикл движения и проверки коллизий (каждые `MUMMY_UPDATE_INTERVAL` сек) через `updateMummy()`

- `MummyAI:computePath(mummy, player)` — Вычисляет путь к игроку через `PathfindingService:CreatePath()`:
  - Параметры: `AgentRadius=1.5`, `AgentHeight=6`, `AgentCanJump=false`
  - Если игрок дальше 120 studs — мумия блуждает (`getRandomPathPosition`)
  - Сохраняет waypoints в `self.mummyPaths[mummy]`

- `MummyAI:updateMummy(mummy, player)` — Обновляет движение одной мумии:
  - Проверяет оглушение (если есть свиток)
  - Двигается по waypoints через `humanoid:MoveTo()`
  - При сближении < 6 studs с игроком:
    - Если у игрока есть свиток — мумия оглушается (`stunMummy`)
    - Иначе — `playerHumanoid.Health = 0` (мгновенная смерть)

- `MummyAI:stunMummy(mummy)` — Оглушает мумию на `SCROLL_PROTECTION_DURATION` секунд:
  - Устанавливает `WalkSpeed = 0`
  - Делает мумию полупрозрачной (Transparency = 0.5)
  - Через `task.delay` восстанавливает скорость и непрозрачность

- `MummyAI:setScrollProtection(enabled)` — Включает/выключает защиту свитком. Устанавливает `self.playerHasScroll`.

- `MummyAI:stop()` — Останавливает все мумии, удаляет их из Workspace, очищает данные.

---

### 5. GameManager (ServerScriptService.OhMummy.GameManager)

**Назначение:** Главный серверный оркестратор. Управляет потоком уровней, состоянием игрока, открытием ячеек, сбором предметов, переходом между уровнями и рестартом.

**Глобальное состояние (`gameState`):**

| Поле | Тип | Описание |
|---|---|---|
| `currentLevel` | number | Текущий уровень |
| `score` | number | Текущий счёт |
| `treasuresCollected` | number | Собрано сокровищ на уровне |
| `hasKey` | bool | Найден ли ключ |
| `hasScroll` | bool | Активен ли свиток |
| `scrollEndTime` | number | Время окончания защиты свитком (os.clock) |
| `isPlaying` | bool | Идёт ли игра |
| `gameOverPending` | bool | Ожидание рестарта после смерти |
| `maze` | Model | Текущий лабиринт |
| `cellData` | table | Данные ячеек |
| `footprintSystem` | FootprintSystem | Текущая система следов |
| `mummyAI` | MummyAI | Текущий ИИ мумий |
| `door` | Part | Дверь текущего уровня |

**Локальные функции:**

- `cleanupLevel()` — Останавливает мумий (`mummyAI:stop()`), удаляет лабиринт из Workspace, очищает все ссылки.

- `updateUI(player)` — Отправляет клиенту данные для обновления UI через `Events.UpdateUI:FireClient()`. Передаёт: level, score, treasures, totalTreasures, hasKey, hasScroll, scrollTimeLeft.

- `startLevel(player, levelNumber)` — Запуск нового уровня:
  - Очищает предыдущий уровень (`cleanupLevel`)
  - Генерирует лабиринт (`MazeGenerator.generateLevel`)
  - Инициализирует `FootprintSystem` и `MummyAI`
  - Спавнит мумий и запускает ИИ
  - Телепортирует игрока на спавн
  - Настраивает обработку касания двери (`setupDoorTouch`)
  - Отправляет клиенту `LevelStart` и обновление UI
  - Запускает цикл отслеживания следов (каждые 0.2 сек):
    - Проверяет позицию игрока через `footprintSystem:checkPlayerPosition`
    - Открывает ячейки через `footprintSystem:openCell`
    - Обрабатывает содержимое: Treasure (+очки), Key (ключ), Scroll (защита), Guardian (мумия), Empty
  - Запускает цикл обновления таймера свитка (каждые 0.5 сек)

- `onPlayerDied(player)` — Обработка смерти игрока:
  - Отправляет `GameOver` клиенту
  - Устанавливает `isPlaying=false`, `gameOverPending=true`
  - Очищает уровень

- `setupDoorTouch(player)` — Настраивает обработку касания двери:
  - При касании двери игроком с ключом:
    - Если последний уровень — `LevelComplete` с `isFinal=true`, конец игры
    - Иначе — переход на следующий уровень (`startLevel`)
  - Использует debounce для защиты от повторных срабатываний

**Обработчики событий:**

- `Players.PlayerAdded` → `onPlayerAdded(player)` — При входе игрока:
  - Отслеживает `CharacterAdded`
  - При спавне персонажа: телепортирует в безопасную точку, ждёт 1 сек
  - Если `gameOverPending=true` — не запускает игру (ждёт ручного рестарта)
  - Иначе — запускает уровень 1
  - Отслеживает `humanoid.Died` → `onPlayerDied`

- `Events.GameStart.OnServerEvent` — При запросе рестарта от клиента:
  - Сбрасывает `gameOverPending=false`
  - Устанавливает `isPlaying=true`, `currentLevel=1`, `score=0`
  - Запускает уровень 1

- Цикл по `Players:GetPlayers()` — Обрабатывает игроков, уже находящихся в игре при старте скрипта.

---

### 6. ClientController (StarterPlayer.StarterPlayerScripts.ClientController)

**Назначение:** Клиентский контроллер. Обрабатывает события сервера, обновляет UI, управляет визуальными эффектами и кнопкой рестарта.

**Обработчики событий (RemoteEvent → OnClientEvent):**

- `Events.UpdateUI` — Обновление интерфейса:
  - LevelLabel → "Level X / 5"
  - ScoreLabel → "Score: X"
  - TreasureLabel → "Treasures: X / 10"
  - KeyIcon → видимость (data.hasKey)
  - ScrollIcon → видимость + TimerLabel → время до окончания защиты

- `Events.LevelStart` — Начало уровня:
  - Скрывает RestartButton
  - Показывает MessageLabel с "Level X"
  - Плавное исчезновение через TweenService (2 сек)

- `Events.LevelComplete` — Завершение уровня:
  - Если `isFinal=true`: "PYRAMID COMPLETE! Final Score: X" + показ RestartButton
  - Иначе: "Level X Complete! Advancing..." + плавное исчезновение

- `Events.GameOver` — Конец игры:
  - Показывает "GAME OVER\nYou were caught by a mummy!\nScore: X\nLevel: Y"
  - Показывает RestartButton

- `Events.CellOpened` — Открытие ячейки:
  - Treasure → "Treasure Found!"
  - Key → "Key Found!"
  - Scroll → "Scroll Found! Protection activated!"
  - Guardian → "A Guardian Mummy awakens!"
  - Empty → "Empty Tomb..."
  - Плавное исчезновение через TweenService (1.5 сек)

**Локальные функции:**

- `setupRestartButton()` — Настраивает кнопку рестарта:
  - При нажатии: скрывает кнопку и MessageLabel, отправляет `Events.GameStart:FireServer()`

**Дополнительно:**

- Инструкции (`InstructionsLabel`) скрываются через 8 секунд после старта
- Кнопка рестарта пересоздаётся при каждом респавне персонажа

---

## Игровой процесс

1. **Старт:** Игрок спавнится у южного входа в лабиринт
2. **Исследование:** Игрок ходит по дорожкам между ячейками, оставляя следы
3. **Открытие ячеек:** Обойдя все 4 стороны ячейки, игрок открывает её:
   - **Treasure** — +100 очков
   - **Key** — ключ для двери (необходим для перехода)
   - **Scroll** — 10 секунд защиты от мумий (мумии оглушаются при касании)
   - **Guardian** — спавнит дополнительную мумию-стража
   - **Empty** — пусто
4. **Угроза:** Мумии преследуют игрока через PathfindingService. Касание = мгновенная смерть
5. **Переход:** Найдя ключ, игрок касается двери на юге → следующий уровень
6. **Победа:** Пройдя 5 уровней — «PYRAMID COMPLETE!»
7. **Поражение:** При смерти — «GAME OVER» с кнопкой рестарта

---

## Remote Events

| Событие | Направление | Данные | Описание |
|---|---|---|---|
| `UpdateUI` | Server → Client | `{level, score, treasures, totalTreasures, hasKey, hasScroll, scrollTimeLeft}` | Обновление UI |
| `LevelStart` | Server → Client | `{level}` | Начало уровня |
| `LevelComplete` | Server → Client | `{score, level, isFinal}` | Завершение уровня/игры |
| `GameOver` | Server → Client | `{score, level}` | Смерть игрока |
| `CellOpened` | Server → Client | `{cellType, position}` | Ячейка открыта |
| `GameStart` | Client → Server | — | Запрос на рестарт |
| `FootprintUpdate` | Server → Client | — | Обновление следов (зарезервировано) |

---

*Документация сгенерирована для проекта «Oh Mummy»*
