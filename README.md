# 🤖 Behavior Trees для ботов в CS:GO и CS2

> [!NOTE]
> Практическое руководство по KV3-деревьям поведения ботов в Counter-Strike: Global Offensive и Counter-Strike 2. Основано на реверс-инжиниринге официальных дампов KV3-файлов и документации Valve Developer Community / Source2 Wiki.

---

## 📋 Содержание

### Часть I. Основы
- [1. Что такое Behavior Tree](#1-что-такое-behavior-tree)
- [2. Быстрый старт](#2-быстрый-старт)
- [3. Формат KV3](#3-формат-kv3)
- [4. Конфиг бота](#4-конфиг-бота)

### Часть II. Узлы
- [5. Композиты](#5-композиты)
- [6. Декораторы](#6-декораторы)
- [7. Условия](#7-условия)
- [8. Действия](#8-действия)

### Часть III. Выполнение
- [9. Модель выполнения](#9-модель-выполнения)
- [10. Blackboard, домены, память](#10-blackboard-домены-память)
- [11. Зарезервированные переменные](#11-зарезервированные-переменные)

### Часть IV. Практика
- [12. Отладка](#12-отладка)
- [13. CS:GO vs CS2](#13-csgo-vs-cs2)
- [14. Готовые модули](#14-готовые-модули)
- [15. Паттерны](#15-паттерны)
- [16. Подводные камни](#16-подводные-камни)

### Часть V. Справочники
- [17. Скриптовые установки гранат](#17-скриптовые-установки-гранат)
- [18. Координация через глобальные счётчики](#18-координация-через-глобальные-счётчики)
- [19. Паттерны-предохранители](#19-паттерны-предохранители)
- [20. Парашют (CS2 Blacksire)](#20-парашют-cs2-blacksire)
- [21. Карты Guardian](#21-карты-guardian)
- [22. Practice: полная таблица раундов](#22-practice-полная-таблица-раундов)
- [23. Полный список доменов](#23-полный-список-доменов)
- [24. Быстрый справочник: aim_projectile vs throw_utility](#24-быстрый-справочник-aim_projectile-vs-throw_utility)

---

# Часть I. Основы

## 1. Что такое Behavior Tree

**1.1.** Behavior Tree (BT) — KV3-файл, описывающий поведение бота как дерево узлов. Игра обходит дерево каждый тик: проверяет условия, выполняет действия. Бот не «думает» — он строго следует дереву.

**1.2.** Где лежат файлы:

| Что | Где |
|---|---|
| Деревья и модули | `scripts/ai/...` внутри `game/csgo/pak01_dir.vpk` (извлекаются через Source2 Viewer) |
| Своё дерево | `scripts/ai/mymode/bt_default.kv3` |
| Конфиг | `scripts/ai/<mode>/bt_config.kv3` |

**1.3.** В предоставленных дампах файлы названы `.txt`, но в игре используются `.kv3`. Внутри — стандартная KV3-разметка.

**1.4.** Как включить:
```kv3
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
mp_restartgame 1
```

**1.5.** С `developer 1` боты при спавне пишут:
```text
[AI BT]: Loaded behavior tree '<file>'
```
> [!WARNING]
> Ошибки всегда имеют префикс `[AI BT]`.

**1.6.** Официальные режимы:

| Режим | Дерево | Строк в `attack` (CS2) | Строк в `attack` (CS:GO) |
|---|---|---|---|
| Deathmatch / DM Freeforall | `scripts/ai/deathmatch/bt_default.kv3` | **13** | **20** |
| Arms Race | `scripts/ai/armsrace/bt_default.kv3` | **13** | **20** |
| Rush | `scripts/ai/rush/bt_default.kv3` | **13** | **20** |
| New User Training | `scripts/ai/practice/newplayer_dust2_01.kv3` | **13** (отдельный конфиг) | **20** |
| Co-op (Guard/Roam/Rush/Sniper) | `scripts/ai/coop/bt_*.kv3` | **20** | **20** |
| Guardian | `scripts/ai/guardian/bt_op*.kv3` | **20** | **20** |

> [!IMPORTANT]
> - **CS2:** Deathmatch/Rush/Practice/Arms Race имеют ровно **13** строк в `attack`; Co-op/Guardian — **20**.
> - **CS:GO:** Во **всех** конфигах (coop, deathmatch, guardian) — **20** строк. Если перепутать, игра выдаст: `[AI BT]: Config node 'attack' must contain 13 entries`.
> - CS:GO-конфиги **не идентичны** между собой: coop и deathmatch различаются `look_around_*`, guardian отличается строкой RIFLE. В CS2 конфиги armsrace, deathmatch, practice и rush байт-идентичны.

---

## 2. Быстрый старт

**2.1.** Создайте `scripts/ai/mymode/bt_default.kv3`:

```kv3
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
{
    config = "scripts/ai/deathmatch/bt_config.kv3"
    root =
    {
        type = "decorator_repeat"
        child =
        {
            type = "sequencer"
            children =
            [
                {
                    type = "action_move_to"
                    destination = "bot_target"
                    movement_type = "BT_ACTION_MOVETO_RUN"
                    route_type = "BT_ACTION_MOVETO_FASTEST_ROUTE"
                },
                {
                    type = "action_look_at"
                    input_location = "bot_target"
                },
                {
                    type = "action_wait"
                    wait_time_min = 2
                    wait_time_max = 3
                }
            ]
        }
    }
}
```

**2.2.** Что делает дерево:

| Узел | Действие |
|---|---|
| `decorator_repeat` | Бесконечный цикл |
| `sequencer` | Выполняет детей по порядку |
| `action_move_to` | Бот бежит к `bot_target`. Если ключа нет в blackboard, игра ищет энтити с таким `targetname` и берёт её `origin` |
| `action_look_at` | Смотрит на `bot_target` |
| `action_wait` | Ждёт 2–3 секунды |

> [!TIP]
> В `practice/newplayer_dust2_01.kv3` Valve оставила готовую строку-комментарий: `// mp_bot_ai_bt "scripts/ai/practice/newplayer_dust2_01.kv3"`. Удобно копировать как шаблон запуска.

---

## 3. Формат KV3

**3.1.** Каждый файл начинается с обязательного заголовка:
```kv3
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
```

**3.2.** Далее — объект с двумя ключами:

| Ключ | Обязателен | Что означает |
|---|---|---|
| `config` | ✅ (для корневого дерева) | Путь к конфигу |
| `root` | ✅ | Корневой узел |

**3.3.** Узел — таблица с ключом `type`. Декораторы и условия имеют одного ребёнка `child`. Композиты — массив `children`. Subtree-файлы содержат один узел верхнего уровня без `config` и `root`.

**3.4.** Синтаксис:

| Элемент | Пример | Примечание |
|---|---|---|
| Комментарии | `// текст` | Однострочные |
| Ключи | `type`, `input` | Регистронезависимы: `Name = name` |
| Строки | `"scripts/ai/..."` | Часто `domain = "'PressAntennaButton'"` (двойная строка с одинарными кавычками внутри) |
| Числа | `3`, `0.5` | — |
| Массивы | `[ ... ]` | — |
| Объекты | `{ ... }` | — |

> [!WARNING]
> Не «исправляйте» `domain = "'PressAntennaButton'"` на `domain = "PressAntennaButton"`. В файлах Valve для `decorator_try_lock`, `decorator_token_service`, `decorator_run_once` используются именно строки с внутренними одинарными кавычками. Исключение: `decorator_hiding_spot_service` и `memory_to_expire` используют `domain = "AllBots"` (без внутренних кавычек).

---

## 4. Конфиг бота

**4.1.** Конфиг задаёт параметры действий: скорость прицеливания, реакцию, стрельбу. Он не управляет логикой — только «мышечной памятью».

### 4.2. Уровни скилла
| Версия | Уровни |
|---|---|
| CS:GO / CS2 | `default`, `low`, `fair`, `normal`, `tough`, `hard`, `very_hard`, `expert`, `elite` (9 уровней) |

> [!NOTE]
> Утверждение из старых версий вики «в CS:GO 8 уровней без elite» для этих дампов **неверно**: `elite` присутствует во всех трёх конфигах CS:GO.

### 4.3–4.5. Ключевые параметры (сокращенно)
| Группа | Ключи | Описание |
|---|---|---|
| **Прицеливание** | `aim_target_acquisition_lerp_time`, `aim_ready_angle_tolerance`, `aim_punch_angle_reaction_chance` | Время наводки, допуск угла, шанс реакции на отдачу |
| **Осмотр** | `look_around_awareness_yaw_range`, `look_around_focus_interval` | Макс. угол поворота, пауза между поворотами камеры |
| **Бой** | `reaction_time`, `combat_crouch_chance` | Задержка перед реакцией, шанс присесть в бою |

### 4.6. Таблица `attack`
**4.6.1.** Каждая строка — 5 чисел: `[burst duration, duration deviation, cadence, cooldown, cooldown deviation]`.

**4.6.2.** 13 строк (CS2 Deathmatch / Rush / Practice / Arms Race):
1. `KNIFE`, 2. `PISTOL`, 3. `SUBMACHINEGUN`, 4. `RIFLE`, 5. `SHOTGUN`, 6. `SNIPER_RIFLE`, 7. `MACHINEGUN`, 8. `C4`, 9. `TASER`, 10. `GRENADE`, 11. `EQUIPMENT`, 12. `STACKABLEITEM`, 13. `UNKNOWN`.

**4.6.3.** 20 строк (CS2 Co-op / Guardian, а также **все** CS:GO-конфиги):
Те же 13 + ещё 7: 14. `FISTS`, 15. `BREACHCHARGE`, 16. `BUMPMINE`, 17. `TABLET`, 18. `MELEE`, 19. `SHIELD`, 20. `WEAPONTYPE_ZONE_REPULSOR`.

> [!IMPORTANT]
> Массив содержит **ровно 20 элементов** в 20-строчных конфигах. Движок выдаст ошибку, если их количество не совпадает.

---

# Часть II. Узлы

## 5. Композиты

| Узел | Ключи | Что делает |
|---|---|---|
| `parallel` | `succeed_after_first` (bool), `children` | Запускает всех детей. Провал любого = провал всего. Успех = все дети успешны, либо первый успешный при `succeed_after_first = 1` |
| `selector` | `children` | Список приоритетов. Проверяет детей с первого. Первый прошедший запускается. Если более ранний ребёнок начал проходить — текущий останавливается |
| `sequencer` | `children` | Выполняет детей по порядку. Провал любого = стоп. Check-pass смотрит только на текущего ребёнка |
| `subtree` | `file`, `name`, `params` | Запускает другое дерево из файла. |

> [!WARNING]
> Ключ `params` в `subtree` **не встречается ни в одном** из 57 CS2-дампов и 544 CS:GO-дампов. Синтаксис взят из Source2 Wiki и не подтверждён. При попытке использовать `float`/`bool` движок выдаст `[AI BT]: unsupported type for subtree parameter`.

---

## 6. Декораторы

**6.1.** Почти все поддерживают `negated = 1` — инверсия check-результата.

### 6.2. Корневые сервисы
| Узел | Что делает | Статус |
|---|---|---|
| `decorator_bot_service` | Истекает память, управляет болтовнёй (`basic_chatter_enable = 1`) | ✅ |
| `decorator_buy_service` | Возвращает массив предметов к покупке в ключ `ShouldBuy` | ✅ |
| `decorator_hiding_spot_service` | Учитывает укрытия. Боты в одном домене не проверяют уже проверенные укрытия | ✅ |

### 6.3. Пикеры и Ранкеры
| Узел | Что делает | Статус |
|---|---|---|
| `decorator_picker_nearby` | Убирает сущности дальше `cutoff_distance` | ✅ |
| `decorator_picker_visible` | Убирает невидимые сущности. `check_fov = 0` игнорирует FOV | ✅ |
| `decorator_picker_max_score` | Берёт сущность с **максимальным score**. Если перед ним `decorator_ranker_dist` — берётся ближайшая | ✅ |
| `decorator_ranker_dist` | Сортирует сущности по дистанции, присваивает score (ближе = выше score) | ✅ |

### 6.4. Сенсоры и Память
| Узел | Что делает | Статус |
|---|---|---|
| `decorator_sensor` | Сканирует мир (`shape`: `sensor_shape_fov` или `sensor_shape_sphere`). `append_items = 1` добавляет к списку | ✅ (`append_items` работает и в CS:GO, и в CS2) |
| `decorator_memory` | Копирует сущности из `input` в `output`. `output_domain` — в общую память домена | ✅ |

### 6.5. Управление потоком и Теги
| Узел | Что делает | Статус |
|---|---|---|
| `decorator_run_once` | Без `domain` — каждый бот один раз. С `domain` — только один бот на сервере | ✅ |
| `decorator_try_lock` | Блокирует домен на время выполнения ребёнка. Только один бот одновременно | ✅ |
| `decorator_tag_entity` | Помечает сущность. `operation_type`: `BT_DECORATOR_TAG_ENTITY_SET` / `CLEAR` | ✅ |
| `decorator_token_service` | Раздаёт ботам «токены»-роли. Выход — `GroupId` и `GroupMemberId` | ✅ |

---

## 7. Условия

**7.1.** Все поддерживают `negated = 1`.

| Узел | Что делает | Статус |
|---|---|---|
| `condition_is_empty` | Проверяет, пуст ли ключ. `global = 1` — глобальный флаг | ✅ |
| `condition_is_equal` | Проверяет равенство. Поддерживает незадокументированный ключ `domain` | ✅ |
| `condition_distance_less` | Дистанция до `input` меньше случайного порога. **При `negated = 1` — «дистанция НЕ меньше порога», т.е. враг далеко** | ✅ |
| `condition_is_weapon_equipped` | Активное оружие — с этим classname | ✅ |
| `condition_inactive` | Нет активности в указанных ключах заданное время | ✅ |

---

## 8. Действия

**8.1.** Действия возвращают `running`, пока не завершатся, затем `success` или `failure`.

### 8.2. Движение и Позиционирование
| Узел | Что делает | Статус |
|---|---|---|
| `action_move_to` | Ведёт бота к `destination`. `movement_type`: WALK/RUN. | ✅ |
| `action_combat_positioning` | Уклонение/сближение. Читает `is_attacking` и не даёт боту стоять на месте | ✅ |
| `action_teleport` | Мгновенный перенос в `destination` | ✅ |
| `action_crouch` | Присесть (встречается в `bt_boost_molly`) | ✅ |

### 8.3. Прицеливание и Стрельба
| Узел | Что делает | Статус |
|---|---|---|
| `action_aim` | Поворачивает бота к точке. Ставит флаг `ready`. **Не разскопится со снайперки до смены оружия** | ✅ |
| `action_attack` | Стреляет по `input`, когда `ready` истинно. `output = true`, пока идёт атака | ✅ |
| `action_look_at` | Поворачивает бота. Принимает `input_angles` или `input_location` | ✅ |

### 8.4. Гранаты
| Узел | Что делает | Статус |
|---|---|---|
| `action_aim_projectile` | Рассчитывает углы. **Баг:** не учитывает траекторию, бросает неточно. Пишет **углы** в `output` | ✅ |
| `action_throw_utility` | **Прямой спавн гранаты** в мировой позиции `location` со скоростью `velocity`. Не зависит от взгляда бота | ✅ |

---

# Часть III. Выполнение

## 9. Модель выполнения

**9.1.** Каждый тик дерево обходится дважды:
1. **Check-pass:** Сверху вниз. Решает, какие ветки имеют право выполняться.
2. **Run-pass:** Выполняет выбранные ветки (движение, стрельба).

**9.2.** Поведение композитов:
| Композит | Check-pass | Run-pass | Прерывание |
|---|---|---|---|
| `sequencer` | Только текущий ребёнок | По порядку до провала | Нет |
| `selector` | С первого ребёнка каждый тик | Запускает первого прошедшего | Да (при появлении более приоритетного) |
| `parallel` | Все дети | Все дети каждый тик | Провал любого = стоп всех |

---

## 10. Blackboard, домены, память

**10.1.** Blackboard — хранилище ключей-значений бота.

### 10.2. Типы данных в ключах
| Тип | Пример | Примечание |
|---|---|---|
| Литерал строки | `"'CT'"` | Двойная строка с внутренними одинарными кавычками |
| ConVar | `@mp_plant_c4_anywhere` | Читается напрямую из консоли |
| Позиция | `"-1450 2230 2"` или `TargetArea` | Если это ключ blackboard, берётся первая сущность из списка |

### 10.3. Ключевые ключи Blackboard
| Ключ | Что хранит | Кто пишет | Кто читает |
|---|---|---|---|
| `Vision` | Враги в FOV | `decorator_sensor` | `decorator_picker_visible` |
| `ShortTermAttackMemory` | Враги, виденные недавно | `decorator_memory` | `bt_attack` |
| `HidingSpot` | Позиция укрытия | `decorator_hiding_spot_service` | `action_move_to` |
| `GroupId` / `GroupMemberId` | Роли бота | `decorator_token_service` | `condition_is_equal` |
| `ShouldBuy` | Предметы к покупке | `decorator_buy_service` | `condition_is_empty`, `action_buy` |

---

## 11. Зарезервированные переменные

Игра сама кладёт их в blackboard. Читать можно, создавать нельзя.

| Переменная | Тип | Использование в дампах |
|---|---|---|
| `NumberOfAliveTerrorists` | int | `bt_plant_bomb_if_owned` (проверка, последний ли ты) |
| `TeamName` | string | Practice: `condition_is_equal source="TeamName" destination="'T'"` |
| `RoundNumber` | int | Practice: загрузка оружия по раундам |
| `GuardianWaveNumber` | int | Guardian: стратегии по волнам |

---

# Часть IV. Практика

## 12. Отладка

| Команда | Назначение | Требования |
|---|---|---|
| `mp_bot_ai_bt "путь"` | Назначить дерево всем ботам | — |
| `mp_bot_ai_bt_clear_cache` | Сбросить кэш после правок | — |
| `cv_bot_ai_bt_debug_target <index>` | Нарисовать дерево бота | `sv_cheats 1` |
| `developer 1` | Писать `[AI BT]: Loaded...` при спавне | — |

---

## 13. CS:GO vs CS2

| Аспект | CS:GO | CS2 |
|---|---|---|
| Строк в `attack` | **20** во всех конфигах | 13 (DM/Rush/Practice/ArmsRace); 20 (Co-op/Guardian) |
| `append_items` в сенсорах | ✅ Работает (напр., для `hostage_entity`) | ✅ Работает |
| `TeamName` / `RoundNumber` | Не документированы | Активно используются в Practice |
| `action_wait` доп. параметры | 2 ключа | + `wait_modulo_tick_mod/add` |

---

## 14. Готовые модули

### 14.1. Deathmatch / Rush / Practice
| Модуль | Что делает |
|---|---|
| `bt_attack` | Берёт ближайшего врага → помечает в `EngagedEntities` → экипирует `BEST` → `aim` → `attack` |
| `bt_heal_if_needed` | Если HP < 60: экипирует `weapon_healthshot`, ждёт 1 с, `action_pull_trigger` |
| `bt_plant_bomb_if_owned` | Если есть C4 и нет флага `DontPlantBomb`: идёт на сайт, закладывает. С шансом 0.5 (если последний T) ставит флаг `DontPlantBomb` на 3–5 с |

### 14.2. Guardian
| Модуль | Что делает |
|---|---|
| `bt_item_strategy` | Покупка/выдача предметов по токенам стратегий (`action_assign_guardian_loadout`) |
| `action_throw_utility` | Прямые скриптовые броски дымов/молотовых (см. §17) |

---

## 15. Паттерны

### 15.1. Side-effect без остановки selector
```kv3
decorator_succeed
negated = 1
child = sequencer[ ... сделать что-то ... ]
```
Selector увидит `failure` и пойдёт дальше, но side-effect уже выполнен. Используется в `bt_clear_threats_within_fov` и `bt_select_nearby_threat`.

### 15.2. Переэкипировка без остановки атаки
```kv3
decorator_repeat
child = decorator_succeed
child = condition_out_of_ammo
child = action_equip_weapon weapon = "BEST"
```
«Если патроны кончились — переэкипировать `BEST`, и всегда возвращать success, чтобы `decorator_repeat` не останавливался».

---

## 16. Подводные камни

| ❌ Не делайте | ✅ Делайте вместо этого |
|---|---|
| Полагаться на `acquire_only` в `action_aim` | Используйте `action_look_at` (баг: aim не разскопится со снайперки) |
| Использовать 20 строк в `attack` для CS2 Deathmatch | Ровно 13, в фиксированном порядке |
| Передавать `float`/`bool` в `subtree params` | Только `int` или `string` (а лучше вообще не использовать `params`) |
| Писать `domain = PressAntennaButton` без кавычек | В файлах Valve: `domain = "'PressAntennaButton'"` |
| Ожидать, что `action_aim_projectile` точно бросит гранату | Баг: не учитывает траекторию. Используйте `action_throw_utility` |

> [!WARNING]
> **Мёртвый `ShouldBuy` в Arms Race:** В `armsrace/bt_default.kv3` есть `decorator_buy_service output = "ShouldBuy"`, но в `selector` **нет ветки покупки**. `ShouldBuy` вычисляется и никогда не читается. Копируя дерево как шаблон, проверяйте, что выход сервиса действительно используется.

---

# Часть V. Справочники

## 17. Скриптовые установки гранат

**17.1.** `action_throw_utility` — одно из самых мощных действий в CS2 Guardian. Оно позволяет **спавнить гранату в произвольной мировой позиции** с заданной скоростью, независимо от того, где стоит бот.

```kv3
action_throw_utility
name = "weapon_incgrenade"
location = "2333 -4365 -1465"
velocity = "0 0 -600"
```
| Поле | Смысл |
|---|---|
| `name` | classname гранаты (`weapon_incgrenade`, `weapon_smokegrenade` и т.д.) |
| `location` | **Мировая позиция**, куда спавнится граната (не позиция бота) |
| `velocity` | Начальная скорость. `"0 0 -600"` — падение вниз |

---

## 18. Координация через глобальные счётчики

Пример из `bt_coordinated_buy.kv3`:
1. Первый бот берёт `decorator_run_once domain = "'CoordinatedBuy'"`.
2. Устанавливает `WaitForCoordinatedBuy = 1` и `RushSite = 1`.
3. Ждёт 3 секунды, выполняет `action_coordinated_buy`.
4. Сбрасывает счётчики в `0`.
5. Остальные боты видят `WaitForCoordinatedBuy = 1` и ждут через `decorator_wait_success` → `action_compare_global_counter`.

---

## 19. Паттерны-предохранители

### 19.1. Самоуничтожение при бездействии
```kv3
condition_inactive
input = [ { key = "ShortTermAttackMemory" }, { key = "ShortTermAreaDamageMemory" } ]
round_start_threshold_seconds = 90
sensor_inactivity_threshold_seconds = 60
child = action_commit_suicide
```
«Если 90 секунд прошло с начала раунда и 60 секунд не было ни атаки, ни урона — бот убивает себя».

---

## 20. Парашют (CS2 Blacksire)

**20.1.** Уникальная механика CS2 Guardian. Боты телепортируются в воздух, получают парашют, и `action_parachute_positioning` рулит диагональным движением.

**20.2.** Токен-сервис парашюта:
В `bt_op11_blacksite_1.txt` токен **один**:
```kv3
tokens = [ "Parachute" ]
assignments = [ [ 1000 ] ]
```
Никаких 40 токенов нет. `Parachute1`…`Parachute40` — это значения **`GroupMemberId`**, которые движок генерирует автоматически. В `bt_parachute_1.kv3` используются 40 разных проверок `condition_is_equal` по `GroupMemberId = 'ParachuteN'`.

---

## 21. Карты Guardian

| Карта | Особенности |
|---|---|
| `op11_county_2` | **Скриптовые установки гранат** (firebomb, smoke_ring) + `action_entity_accept_input` для открытия дверей; override зрения 1500→4000 |
| `op11_blacksite_1` | **Парашют**; локальный override зрения: `sensor_shape_sphere` radius 25000 (видит всю карту) |
| `op11_blacksite_2/4` | Override зрения 1500→1850 |

---

## 22. Practice: полная таблица раундов

### 22.1. CT loadout (`bt_acquire_ct_weapons.kv3`)
| Раунд | Что покупается |
|---|---|
| 1 | `weapon_hkp2000` |
| 2 | `item_kevlar`, `weapon_p250` |
| 3 | `item_kevlar`, `weapon_mp9` |
| 4–6 | `item_assaultsuit`, `weapon_p90` |
| 7 | `weapon_hkp2000` (с `remove_all_items = 1` — сброс!) |
| 8+ | Цикл повторяется |

### 22.2. T стратегии по раундам
| Раунд | Группы T | Что делают |
|---|---|---|
| 1 | `t_1`–`t_5` | `bt_wait` → `bt_goto_a_bombsite_via_long_doors_from_t` |
| 2 | `t_1`–`t_5` | `bt_wait` → `bt_goto_a_via_long_doors_from_t` |
| 3 | `t_1`–`t_5` | `bt_wait` → `bt_goto_a_via_cat_from_t` |
| 4–5 | `t_1`–`t_5` | `bt_goto_b_site_from_t` |

---

## 23. Полный список доменов

| Домен | Где | Что блокирует |
|---|---|---|
| `"AllBots"` (без внутр. кавычек) | `memory_to_expire`, `decorator_run_once` | Общая память всех ботов |
| `"'AllBots'"` (**с** внутр. кавычками) | `op11_county_2`, волна 1 | **Единственное исключение** для `action_entity_accept_input` |
| `"'StratTokens'"` | `decorator_token_service` | Токены стратегий |
| `"'CoordinatedBuy'"` | `bt_coordinated_buy` | Координированная покупка (только 1 бот) |
| `"'ParachuteN'"` (N = 1..40) | `bt_parachute_1.kv3` | Индивидуальный «лок» на бота (через `condition_is_equal domain`) |

> [!WARNING]
> Не унифицируйте кавычки! `"AllBots"` без внутренних кавычек в `memory_to_expire`, но `"'StratTokens'"` и `"'CoordinatedBuy'"` — строго с ними.

---

## 24. Быстрый справочник: aim_projectile vs throw_utility

| Характеристика | `action_aim_projectile` | `action_throw_utility` |
|---|---|---|
| Использует позицию бота | Да | Нет (спавнит в `location`) |
| Учитывает траекторию | Нет (баг) | Задаётся вручную через `velocity` |
| Где используется | Co-op `bt_throw_utility_if_covered` | CS2 Guardian (`county_2`, `sirocco`) |
| Точность | Низкая | Высокая (скриптовая) |
| Когда использовать | Нужен «естественный» бросок от бота | Нужна точная установка дыма/огня/молотова |

---
*Документация сгенерирована на основе официальных дампов Valve. Актуально для CS:GO и CS2.*
