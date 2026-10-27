# 🤖 Behavior Trees для ботов в CS:GO и CS2

> [!NOTE]
> Это практическое руководство по KV3-деревьям поведения ботов в Counter-Strike: Global Offensive и Counter-Strike 2. Основано на официальных дампах KV3-файлов и страницах Valve Developer Community / Source2 Wiki.

---

## 📋 Содержание

1. [Что такое Behavior Tree](#-что-такое-behavior-tree)
2. [Быстрый старт](#-быстрый-старт)
3. [Формат KV3](#-формат-kv3)
4. [Конфиг бота](#-конфиг-бота)
5. [Композиты](#-композиты)
6. [Декораторы](#-декораторы)
7. [Условия](#-условия)
8. [Действия](#-действия)
9. [Модель выполнения](#-модель-выполнения)
10. [Blackboard, домены, память](#-blackboard-домены-память)
11. [Зарезервированные переменные](#-зарезервированные-переменные)
12. [Отладка](#-отладка)
13. [CS:GO vs CS2](#-csgo-vs-cs2)
14. [Готовые модули](#-готовые-модули)
15. [Паттерны](#-паттерны)
16. [Подводные камни](#-подводные-камни)

---

## 🧠 Что такое Behavior Tree

**Behavior Tree (BT)** — KV3-файл, описывающий поведение бота как дерево узлов. Игра обходит дерево каждый тик: проверяет условия, выполняет действия. Бот не «думает» — он следует дереву.

### 📁 Где лежат файлы

| Что | Где |
|---|---|
| Деревья и модули | `scripts/ai/...` внутри `game/csgo/pak01_dir.vpk`, извлекаются Source2 Viewer |
| Своё дерево | `scripts/ai/mymode/bt_default.kv3` |
| Конфиг | `scripts/ai/<mode>/bt_config.kv3` |

В предоставленных дампах файлы названы `.txt`, но в игре используются `.kv3`. Внутри — KV3-разметка.

### ⚙️ Как включить

```cfg
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
mp_restartgame 1
```

С `developer 1` боты при спавне пишут:

```text
[AI BT]: Loaded behavior tree '<file>'
```

Ошибки — всегда с префиксом `[AI BT]`.

### 🎮 Официальные режимы

| Режим | Дерево | Конфиг |
|---|---|---|
| Deathmatch / DM Freeforall | `scripts/ai/deathmatch/bt_default.kv3` | 13 строк |
| Arms Race | `scripts/ai/armsrace/bt_default.kv3` | 13 строк |
| Rush | `scripts/ai/rush/bt_default.kv3` | 13 строк |
| New User Training | `scripts/ai/practice/newplayer_dust2_01.kv3` | 13 строк |
| Co-op (Guard/Roam/Rush/Sniper) | `scripts/ai/coop/bt_*.kv3` | 20 строк |
| Guardian | `scripts/ai/guardian/bt_op*.kv3` | 20 строк |

> [!IMPORTANT]
> Для CS2 Deathmatch/Rush/Practice используйте ровно **13 строк** в `attack`. Для Co-op/Guardian в предоставленных файлах — **20 строк**. Если перепутать, будет:
> ```text
> [AI BT]: Config node 'attack' must contain 13 entries
> ```

---

## 🚀 Быстрый старт

Создайте `scripts/ai/mymode/bt_default.kv3`:

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
                }
                {
                    type = "action_look_at"
                    input_location = "bot_target"
                }
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

### 🧠 Что делает это дерево

- `decorator_repeat` — бесконечный цикл.
- `sequencer` — выполняет детей по порядку.
- `action_move_to` — бот бежит к `bot_target`. Если `bot_target` нет в blackboard, игра попробует найти энтити с таким targetname и взять её origin.
- `action_look_at` — смотрит на `bot_target`.
- `action_wait` — ждёт 2–3 секунды.
- Цикл повторяется.

### 🧪 Как проверить

```cfg
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
mp_restartgame 1
```

Поставьте на карту `info_target` с targetname `bot_target`. Бот будет бежать к ней.

---

## 📄 Формат KV3

Каждый файл начинается с обязательного заголовка:

```kv3
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
```

Далее — объект с двумя ключами:

| Ключ | Обязателен | Что означает |
|---|---|---|
| `config` | ✅ для корневого дерева | Путь к конфигу |
| `root` | ✅ | Корневой узел |

Узел — таблица с ключом `type`.  
Декораторы и условия имеют одного ребёнка `child`.  
Композиты — массив `children`.  
Subtree-файлы содержат один узел верхнего уровня без `config` и `root`.

### ✍️ Синтаксис

| Элемент | Пример | Примечание |
|---|---|---|
| Комментарии | `// текст` | Однострочные |
| Ключи | `type`, `input`, `destination` | Регистронезависимы: `Name = name` встречается |
| Строки | `"scripts/ai/..."` | В файлах Valve часто `domain = "'PressAntennaButton'"` — двойная строка с одинарными кавычками внутри |
| Числа | `3`, `0.5` | — |
| Массивы | `[ ... ]` | — |
| Объекты | `{ ... }` | — |

> [!WARNING]
> Не «исправляйте» `domain = "'PressAntennaButton'"` на `domain = "PressAntennaButton"` без проверки. В предоставленных файлах для `decorator_try_lock`, `decorator_token_service`, `decorator_run_once`, `action_set_global_counter` и т.п. используются именно строки с внутренними одинарными кавычками. При этом `decorator_hiding_spot_service` `domain = "AllBots"` и `memory_to_expire { domain = "AllBots" }` — без внутренних кавычек.

### 📄 Пример subtree-файла

```kv3
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
{
    type = "decorator_picker_reaction_time"
    input = "ShortTermAttackMemory"
    output = "Enemy"
    child = { ... }
}
```

---

## ⚙️ Конфиг бота

Конфиг задаёт параметры действий: скорость прицеливания, реакцию, стрельбу. Он не управляет логикой — только «мышечной памятью».

### 🎚️ Уровни скилла

| Версия | Уровни |
|---|---|
| CS:GO | `default`, `low`, `fair`, `normal`, `tough`, `hard`, `very_hard`, `expert` (8) |
| CS2 | + `elite` (9) |

Каждый уровень наследует недостающие ключи из `default`. В предоставленных файлах есть все уровни.

### 🎯 4.1 Параметры прицеливания

| Ключ | Тип | Описание |
|---|---|---|
| `aim_target_acquisition_lerp_time` | `float` | Время «доезда» прицела до новой цели. Больше = медленнее наводится. |
| `aim_target_acquisition_lerp_time_deviation` | `float` | Случайный разброс времени. |
| `aim_target_acquisition_angle_penalty` | `float` | Штраф за большой угол до цели. |
| `aim_target_acquisition_angle_penalty_deviation` | `float` | Разброс штрафа. |
| `aim_target_acquisition_angle_penalty_reduction_ratio` | `float` | Насколько штраф уменьшается со временем. |
| `aim_target_acquisition_angle_tolerance` | `float` | Допуск угла при захвате. |
| `aim_target_acquisition_angle_lerp_bias` | `float` | Сглаживание угла захвата. |
| `aim_target_tracking_lerp_time` | `float` | Скорость сопровождения цели. |
| `aim_target_tracking_lerp_time_deviation` | `float` | Разброс сопровождения. |
| `aim_target_tracking_focus_interval` | `float` | Интервал «фокуса» при трекинге. |
| `aim_target_tracking_focus_interval_deviation` | `float` | Разброс интервала. |
| `aim_target_tracking_angle_lerp_bias` | `float` | Сглаживание трекинга. |
| `aim_ready_angle_tolerance` | `float` | Угол, при котором `action_attack` считает прицел готовым. |
| `aim_new_target_angle_tolerance` | `float` | Угол для переключения на новую цель. |
| `aim_max_duration` | `float` | Максимальное время прицеливания. |
| `aim_max_duration_deviation` | `float` | Разброс макс. времени. |
| `aim_punch_angle_reaction_chance` | `float` | Шанс реакции на отдачу. |

### 👀 4.2 Параметры осмотра

| Ключ | Описание |
|---|---|
| `look_around_awareness_yaw_range` | Макс. угол поворота по горизонтали. |
| `look_around_awareness_pitch_range` | Макс. угол по вертикали. |
| `look_around_focus_interval` | Пауза между поворотами камеры. |
| `look_around_focus_interval_deviation` | Разброс паузы. |
| `look_around_lerp_time` | Время поворота камеры. |
| `look_around_lerp_time_deviation` | Разброс времени. |
| `look_around_lerp_bias` | Сглаживание движения камеры. |

### ⚔️ 4.3 Реакция и бой

| Ключ | Описание |
|---|---|
| `reaction_time` | Задержка перед реакцией на событие. |
| `combat_crouch_chance` | Шанс присесть в бою. |
| `combat_dodge_command_duration` | Длительность команды уклонения. |
| `combat_dodge_command_duration_deviation` | Разброс длительности. |

### 🔫 4.4 Таблица attack

Каждая строка — 5 чисел:

```text
burst duration, duration deviation, cadence, cooldown, cooldown deviation
```

| Поле | Что означает |
|---|---|
| `burst duration` | Длительность одной очереди. |
| `duration deviation` | Разброс длительности. |
| `cadence` | Пауза между выстрелами внутри очереди; 0 = без паузы. |
| `cooldown` | Пауза между очередями. |
| `cooldown deviation` | Разброс паузы. |

#### 📋 13 строк — Deathmatch / Rush / Practice / Arms Race

Строгий порядок:

1. `KNIFE`
2. `PISTOL`
3. `SUBMACHINEGUN`
4. `RIFLE`
5. `SHOTGUN`
6. `SNIPER_RIFLE`
7. `MACHINEGUN`
8. `C4`
9. `TASER`
10. `GRENADE`
11. `EQUIPMENT`
12. `STACKABLEITEM`
13. `UNKNOWN`

#### 📋 20 строк — Co-op / Guardian

Те же 13 + ещё 7: `FISTS`, `BREACHCHARGE`, `BUMPMINE`, `TABLET`, `MELEE`, `SHIELD`, `WEAPONTYPE_ZONE_REPULSOR`, `UNKNOWN`.

В файлах порядок после `STACKABLEITEM`: `FISTS`, `BREACHCHARGE`, `BUMPMINE`, `TABLET`, `MELEE`, `SHIELD`, `WEAPONTYPE_ZONE_REPULSOR`, `UNKNOWN`. Итого 20 строк.

### 📊 Пример RIFLE

| Конфиг | Значение | Интерпретация |
|---|---|---|
| Deathmatch | `[ 0.22, 0.07, 0.0, 0.45, 0.15 ]` | Очередь ~0.22 с, без паузы внутри, новая очередь через 0.45 с. |
| Rush/Guardian | `[ 0.25, 0.55, 0.0, 0.95, 0.25 ]` | Очередь ~0.25 с, очень большой разброс, cooldown 0.95 с. |

---

## 🧩 Композиты

| Узел | Ключи | Что делает |
|---|---|---|
| `parallel` | `succeed_after_first` (`bool`), `children` | Запускает всех детей каждый тик. Провал любого = провал всего. Успех = все дети успешны, либо первый успешный при `succeed_after_first = 1`. |
| `selector` | `children` | Список приоритетов. Проверяет детей с первого каждый тик. Первый прошедший запускается. Если более ранний ребёнок начал проходить — текущий останавливается. |
| `sequencer` | `children` | Выполняет детей по порядку. Провал любого = стоп. Успех = все прошли. Check-pass смотрит только на текущего ребёнка; пройденные условия не перепроверяются. |
| `subtree` | `file`, `name`, `params` | Запускает другое дерево из файла. `params` пишут значения в blackboard до старта. Только `int` или `string`; `float`/`bool` отвергаются. Имена ключей регистронезависимы. |

### 📄 Пример subtree с параметрами

```kv3
{
    type = "subtree"
    file = "scripts/ai/mymode/bt_goto.kv3"
    name = "GotoA"
    params =
    [
        { key = "Target" value = "site_a_target" }
        { key = "WaitTime" value = 3 }
    ]
}
```

---

## 🎭 Декораторы

Почти все поддерживают `negated = 1` — инверсия check-результата.

### 🏗️ 6.1 Корневые сервисы

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_bot_service` | `memory_to_expire` (`{key, time, distance, domain}[]`), `tagged_entities_to_expire` (`string[]`), `input_chatter_enemies` (`string`), `basic_chatter_enable` (`bool`), `chatter_outnumbered_threshold` (`int`), `child` | Корневой сервис. Истекает память, управляет болтовнёй. | ✅ кроме `chatter_outnumbered_threshold` ⚠️ |
| `decorator_buy_service` | `output` (`string`), `child` | Возвращает массив предметов к покупке. Выход — ключ blackboard, обычно `ShouldBuy`. | ✅ |
| `decorator_hiding_spot_service` | `domain` (`string`), `output_hiding_spot` (`string`), `expiration_time` (`float`), `distance_threshold` (`float`), `child` | Учитывает укрытия. Боты в одном домене не проверяют уже проверенные укрытия. | ✅ |

### 🎣 6.2 Пикеры

| Узел | Доп. ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_picker_nearby` | `cutoff_distance` (`float`) | Убирает сущности дальше `cutoff_distance`. | ✅ |
| `decorator_picker_visible` | `check_fov` (`bool`, по умолчанию 1) | Убирает невидимые сущности. `check_fov = 0` — игнорирует FOV. | ✅ |
| `decorator_picker_reaction_time` | — | Задерживает выход на `reaction_time` бота. | ✅ |
| `decorator_picker_dedup` | `against` (`string`), `distance_threshold` (`float`) | Убирает дубликаты — сущности ближе `distance_threshold` к сущностям из `against`. | ✅ |
| `decorator_picker_random_by_distance` | `distance_min` (`float`), `distance_max` (`float`) | Вероятностный фильтр по дистанции. | ✅ |
| `decorator_picker_weight_as_distance` | — | Убирает сущности, чья дистанция больше их «веса». | ✅ |
| `decorator_picker_grenade_type` | `types[]` (`FLASH`, `FIRE`, `EXPLOSIVE`, `DECOY`, `SMOKE`) | Фильтр гранат по типу. | ✅ |
| `decorator_picker_blocked_by_smoke` | `distance_threshold` (`float`) | Оставляет только сущности за дымом. | ✅, но `distance_threshold` в файлах есть, по вики игнорируется |
| `decorator_picker_max_score` | — | Берёт первую сущность из списка. Если перед ним `decorator_ranker_dist` — берётся ближайшая. | ✅ |

### 📊 6.3 Ранкеры

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_ranker_dist` | `input` (`string`) | Сортирует сущности по дистанции. | ✅ |

### 📡 6.4 Сенсоры

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_sensor` | `output` (`string`), `class_name` (`string`), `priority` (`int`), `orphan_only` (`bool`), `append_items` (`bool`), `entity_type_filter` (`enum`), `team_filter` (`enum`), `attacker_filter` (`enum`), `shape` (`{type, radius}`), `child` | Сканирует мир и записывает найденные сущности в `output`. `shape`: `sensor_shape_fov` или `sensor_shape_sphere` с `radius`. `orphan_only = 1` — только «бесхозные». `append_items = 1` — добавить к списку. | ✅ кроме `attacker_filter` ⚠️ |

### 💾 6.5 Память

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_memory` | `input` (`string`), `output` (`string`), `output_domain` (`string`), `child` | Копирует сущности из `input` в `output`. Если указан `output_domain` — в общую память домена. | ✅ |
| `decorator_set_reaction_time` | `input` (`string`), `child` | Задерживает прохождение `input` на `reaction_time` бота. | ✅ |

### 🔀 6.6 Управление потоком

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_repeat` | `amount` (`int`), `child` | Перезапускает ребёнка каждый раз, когда он завершается. Без `amount` — вечно. С `amount = N` — N раз. | ✅ `amount` ⚠️ |
| `decorator_maybe` | `chance` (`float`, 0–1), `child` | Выполняет ребёнка с вероятностью `chance`. | ✅ |
| `decorator_succeed` | `negated` (`bool`), `child` | Без `negated`: всегда проходит check-pass, после ребёнка возвращает success. С `negated = 1`: всегда проходит check-pass, после ребёнка возвращает failure. | ✅ |
| `decorator_invert` | `child` | Инвертирует check-результат. В CS2 вместо него используют `negated = 1`. | ⚠️ не встречен |
| `decorator_run_once` | `domain` (`string`), `max_attempts` (`int`), `child` | Ограничивает запуск. Без `domain` — каждый бот один раз. С `domain` — только один бот на сервере. | ✅ |
| `decorator_try_lock` | `domain` (`string`), `child` | Блокирует домен на время выполнения ребёнка. Только один бот одновременно. | ✅ |
| `decorator_wait_success` | `timeout` (`float`), `child` | Ждёт success от ребёнка или `timeout`. | ✅ |
| `decorator_need_healing` | `health_threshold` (`int`), `child` | Выполняет ребёнка, если HP ниже порога. | ✅ |
| `decorator_random_int` | `min` (`int`), `max` (`int`), `output` (`string`), `child` | Пишет случайное целое в `output`. | ✅ |
| `decorator_random_approach_point` | `output` (`string`), `child` | Пишет случайную видимую точку на навмеше — «вход» в область. | ✅ |

### 🏷️ 6.7 Теги и счётчики

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_tag_entity` | `input` (`string`), `output` (`string`), `expiration_time` (`float`), `operation_type` (`enum`), `child` | Помечает первую сущность из `input` в `output`. `operation_type`: `BT_DECORATOR_TAG_ENTITY_SET` / `BT_DECORATOR_TAG_ENTITY_CLEAR`. | ✅ |
| `decorator_tag_threshold` | `entity_input` (`string`), `tagged_entities_input` (`string`), `amount` (`int`), `check_type` (`enum`), `child` | Сравнивает количество помеченных. `AT_LEAST` / `AT_MOST`. | ✅ |
| `decorator_token_service` | `config` (`{tokens[], assignments[][], team_restriction[]}`), `output_token_name` (`string`), `output_token_domain` (`string`), `domain` (`string`), `child` | Раздаёт ботам «токены»-роли. Выход — `GroupId` и `GroupMemberId`. | ✅ |
| `decorator_remove` | `input_domain` (`string`), `input` (`string`), `remove` (`string`), `child` | Удаляет сущности из `input`, которые есть в `remove`. | ✅ |
| `decorator_remove_key` | `input` (`string`), `child` | Удаляет ключ blackboard. | ✅ |
| `decorator_dec_global_counter` | `input_name` (`string`), `child` | Уменьшает глобальный счётчик на 1. | ✅ |

### 🧩 6.8 Прочие декораторы

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `decorator_find_utility_strat` | `distance_threshold` (`float`), `input_location` (`position`), `output_location` (`string`), `output_angles` (`string`), `output_weapon` (`string`), `utilities` (`{x, y, z, ang_pitch, ang_yaw, ang_roll, weapon}[]`), `child` | Предположительно ищет подходящую позицию для использования гранаты/утилиты. Не используется в официальных деревьях. | ⚠️ не встречен |
| `decorator_route_service` | `config` (`{routes: waypoint[][], strategies: int[][]}`), `output_waypoint` (`string`), `output_waypoint_name` (`string`), `output_domain` (`string`), `child` | Предположительно управляет маршрутами. Не используется. | ⚠️ не встречен |
| `decorator_set_barrier` | ключи не документированы | Не используется. | ⚠️ не встречен |

---

## ✅ Условия

Все поддерживают `negated = 1`.

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `condition_is_empty` | `input` (`string`), `global` (`bool`), `negated`, `child` | Проверяет, пуст ли ключ или его нет. `global = 1` — глобальный флаг. | ✅ |
| `condition_is_equal` | `source` (`string`), `destination` (`int` или `string`), `negated`, `child` | Проверяет равенство. `destination` может быть числом или строкой, включая `"'ct_1'"`. | ✅ |
| `condition_is_greater_equal` | `source`, `destination`, `negated`, `child` | `source >= destination`. | ✅ |
| `condition_is_less_equal` | `source`, `destination`, `negated`, `child` | `source <= destination`. | ✅ |
| `condition_is_greater` | `source`, `destination`, `negated`, `child` | `source > destination`. | ⚠️ не встречен |
| `condition_is_less` | `source`, `destination`, `negated`, `child` | `source < destination`. | ⚠️ не встречен |
| `condition_is_inv_slot_empty` | `slot` (`enum`: `RIFLE`, `PISTOL`, `KNIFE`, `GRENADES`, `C4`), `negated`, `child` | Проверяет пустоту слота. | ✅ |
| `condition_is_weapon_equipped` | `weapon` (`string`), `negated`, `child` | Активное оружие — с этим classname. | ✅ |
| `condition_out_of_ammo` | `negated`, `child` | В активном оружии нет патронов. | ✅ |
| `condition_owns_item` | `item` (`string`), `items_one_of` (`string[]`), `negated`, `child` | Есть ли предмет. | ✅ |
| `condition_is_at_bomb_site` | `negated`, `child` | Бот на бомбсайте. | ✅ |
| `condition_distance_less` | `input` (`string`), `distance_threshold_min` (`float`), `distance_threshold_max` (`float`), `negated`, `child` | Дистанция до `input` меньше случайного порога между min и max. | ✅ |
| `condition_has_parachute` | `negated`, `child` | Есть ли парашют. | ✅ |
| `condition_inactive` | `input` (`{key}[]`), `round_start_threshold_seconds` (`float`), `sensor_inactivity_threshold_seconds` (`float`), `negated`, `child` | Нет активности в указанных ключах заданное время и с начала раунда прошло не меньше порога. | ✅ |
| `condition_is_airborne` | `negated`, `child` | Бот в воздухе. | ✅ |
| `condition_is_reloading` | `negated`, `child` | Оружие перезаряжается. | ⚠️ не встречен |
| `condition_is_weapon_suitable` | `input`, `weapon`, `negated`, `child` | Подходит ли оружие для дистанции до `input`. | ⚠️ не встречен |
| `condition_barrier` | `input_domain`, `input_name`, `radius`, `negated`, `child` | — | ⚠️ не встречен |

---

## 🎬 Действия

Действия возвращают `running`, пока не завершатся, затем `success` или `failure`.

| Узел | Ключи | Что делает | Статус |
|---|---|---|---|
| `action_aim` | `input` (`position`), `ready` (`string`) | Поворачивает бота к точке `input`. Когда прицел готов — ставит флаг `ready`. `acquire_only` в файлах передаётся, но по вики игнорируется. Не разскопится со снайперки до смены оружия. | ✅ |
| `action_attack` | `input` (`string`), `ready` (`string`), `output` (`string`) | Стреляет по `input`, когда `ready` истинно. Если `ready` не указан — стреляет сразу. `output = true`, пока идёт атака. | ✅ |
| `action_move_to` | `destination` (`position`), `hiding_spot` (`number`), `threat` (`string`), `auto_look_adjust` (`bool`), `damaging_areas_penalty_cost` (`float`), `arrival_epsilon` (`float`), `additional_arrival_epsilon_2d` (`float`), `hiding_spot_check_distance_threshold` (`float`), `nearest_area_distance_threshold` (`float`), `movement_type` (`enum`), `route_type` (`enum`) | Ведёт бота к `destination`. `movement_type`: `WALK`/`RUN`. `route_type`: `FASTEST_ROUTE`/`SAFEST_ROUTE`. `destination` — позиция, ключ blackboard или targetname. | ✅ |
| `action_pull_trigger` | — (в файлах есть `ratio`, но игнорируется) | Нажимает ЛКМ. | ✅ |
| `action_use` | — | Нажимает `+use`: использовать, разминировать, подобрать. | ✅ |
| `action_look_at` | `input_angles` (`string`) или `input_location` (`string`) | Поворачивает бота. Указывать одно из двух. | ✅ |
| `action_wait` | `wait_time_min` (`number`), `wait_time_max` (`number`), `wait_modulo_tick_mod` (`number`), `wait_modulo_tick_add` (`number`) | Замораживает ИИ на случайное время. | ✅ кроме modulo ⚠️ |
| `action_equip_weapon` | `weapon` (`string`, classname или `BEST`) | Выбирает оружие. | ✅ |
| `action_equip_item` | `item` (`string`), `items_one_of` (`string[]`) | Выбирает предмет по classname. | ✅ |
| `action_buy` | — | Случайная покупка. | ✅ |
| `action_choose_bomb_site_area` | `input` (`number`), `output` (`string`) | Позиция бомбсайта по индексу. `input` может быть `0/1`, ключом blackboard или ConVar `@mp_guardian_target_site`. | ✅ |
| `action_choose_random_waypoint` | `input` (`string`), `output` (`string`) | Случайный тайл навмеша из `input`. | ✅ |
| `action_choose_team_spawn_area` | `output` (`string`), `team` (`enum`) | Нав-зона со спавнами команды. | ✅ `team` ⚠️ |
| `action_choose_random_waypoint_within_radius` | `origin` (`position`), `radius` (`number`), `output` (`string`) | Случайная точка в радиусе. `radius` может быть числом, строкой-литералом или ключом blackboard. | ✅ |
| `action_combat_positioning` | `input` (`string`), `is_attacking` (`string`) | Уклонение/сближение. | ✅ |
| `action_flee_area_damage` | `input` (`string`), `output` (`string`), `max_search_range` (`float`), `threat_min_keep_distance` (`float`) | Ищет безопасную точку от урона по площади. | ✅ |
| `action_set_global_flag` | `name` (`string`), `expiration_time_min` (`float`), `expiration_time_max` (`float`) | Ставит глобальный флаг на случайное время. | ✅ |
| `action_set_global_counter` | `input_name` (`string`), `input_value` (`int`) | Ставит глобальный счётчик. | ✅ |
| `action_compare_global_counter` | `input_name` (`string`), `input_value` (`int`) | Сравнивает глобальный счётчик с числом. | ✅ |
| `action_set_value_float` | `key` (`string`), `value` (`float`) | Пишет `float` в blackboard. | ✅ |
| `action_set_value_vector` | `key` (`string`), `value` (`position`) | Пишет позицию в blackboard. | ✅ |
| `action_aim_projectile` | `input` (`position`), `output` (`string`) | Рассчитывает углы для броска гранаты. Известный баг: не учитывает траекторию, бот бросает неточно. | ✅ |
| `action_choose_guardian_bomb_plant_location` | `output` (`string`) | Выбирает позицию закладки бомбы Guardian. | ✅ |
| `action_commit_suicide` | — | Убивает бота. | ✅ |
| `action_jump` | — | Прыжок. | ✅ |
| `action_parachute_positioning` | — | Движение по диагонали для парашюта. | ✅ |
| `action_say` | `phrase` (`string`), `high_priority` (`bool`) | Радио-команда. | ✅ |
| `action_select_areas_within_radius` | `input` (`position`), `radius` (`number`), `output` (`string`) | Выбирает нав-зоны в радиусе. `radius` — число, строка или ключ. | ✅ |
| `action_entity_accept_input` | `name` (`string`), `input` (`string`) | Отправляет `input` энтити по targetname. | ✅ |
| `action_coordinated_buy` | `id_no_purchase` (`string`), `purchases` (`{items[], id}[]`), `team_filter` (`enum`), `save_threshold` (`int`) | Скоординированная покупка. Известный баг: один бот может заставить покупать всех. | ✅ |
| `action_assign_guardian_loadout` | `input_token_name` (`string`), `assignments` (`{token, loadout[]}[]`) | Раздаёт лутауты Guardian по токенам и волнам. В `loadout`: `items[]`, `wave_numbers[]` или `wave_number_min`, `weight`. | ✅ |
| `action_throw_utility` | `name` (`string`), `location` (`position`), `velocity` (`position`) | Прямой бросок гранаты с заданной позицией и скоростью. | ✅ |
| `action_acquire_items` | `items` (`string[]`), `remove_all_items` (`bool`) | Выдаёт предметы. `remove_all_items = 1` — предварительно очистить инвентарь. | ✅ |
| `action_teleport` | `destination` (`position`) | Телепорт. | ✅ |
| `action_crouch` | — | Присесть. | ✅ |
| `action_custom_buy` | `item_aliases` (`string[]`) | Покупка по алиасам. | ⚠️ не встречен |
| `action_hide` | `output` (`string`), `max_range` (`float`) | Случайное укрытие. | ⚠️ не встречен |
| `action_standup` | — | Встать. | ⚠️ не встречен |
| `action_reload` | — | Перезарядка. | ⚠️ не встречен |
| `action_drop_active_weapon` | — | Выбросить активное оружие. | ⚠️ не встречен |
| `action_inspect_current_weapon` | — | Анимация осмотра оружия. | ⚠️ не встречен |
| `action_secondary_attack` | — | Вторичная атака (ПКМ). | ⚠️ не встречен |

---

## 🔄 Модель выполнения

Каждый тик дерево обходится дважды:

1. **Check-pass** — сверху вниз. Решает, какие ветки имеют право выполняться. Если корень не проходит — тик «пустой».
2. **Run-pass** — выполняет выбранные ветки. Здесь бот двигается, целится, стреляет.

Поскольку check-pass повторяется каждый тик, условия перепроверяются постоянно. `selector` может прервать текущую ветку, если более приоритетная начала проходить проверку.

### 🧩 Поведение композитов

| Композит | Check-pass | Run-pass | Прерывание |
|---|---|---|---|
| `sequencer` | Только текущий ребёнок | По порядку до провала | Нет; условия позади не перепроверяются |
| `selector` | С первого ребёнка каждый тик | Запускает первого прошедшего | Да — останавливает текущего при появлении более приоритетного |
| `parallel` | Все дети | Все дети каждый тик | Провал любого = стоп всех |

### 🎬 Пример разворачивания

`sequencer[ move_to(bot_target), look_at(bot_target), wait(2..3) ]` в `repeat`:

- Тик 1: `move_to` двигает на 1 тик → `running`.
- Тики 2..N: игра идёт прямо в `move_to` — одно непрерывное движение.
- Прибытие: `move_to` success → `look_at` success.
- `wait` выбирает время окончания и возвращает `running`.
- Все три готовы → `sequencer` success → `repeat` заново.

> [!TIP]
> Узлы не двигают бота напрямую. `action_move_to` управляет pathfinding; `action_use`/`action_reload` «нажимают кнопку» на тик; `action_aim`/`action_attack` рулят встроенным aiming/shooting, пока не завершатся.

---

## 🧠 Blackboard, домены, память

**Blackboard** — хранилище ключей-значений бота. Сенсор пишет врагов в `output = "Enemy"`, действие атаки читает их через `input = "Enemy"`.

### 📝 Строки

| Тип | Пример | Примечание |
|---|---|---|
| Литерал | `"'CT'"` | В файлах Valve часто двойная строка с внутренними одинарными кавычками |
| Ключ blackboard | `TeamName` | Должен содержать строку |

### 🔢 Числа

| Тип | Пример | Примечание |
|---|---|---|
| Литерал | `3`, `0.5` | — |
| ConVar | `@mp_plant_c4_anywhere` | — |
| Ключ blackboard | `RoundNumber` | Если содержит строку — разрешается ещё раз, но только на 1 уровень |

### 📍 Позиции и имена сущностей

Ключи, ожидающие позицию (`destination`, `input_location`), принимают:

| Тип | Пример | Примечание |
|---|---|---|
| Литерал | `"-1450 2230 2"` | — |
| Ключ blackboard | `TargetArea` | Позиция, строка или список sensed-сущностей; берётся первая |
| Targetname энтити | `ant.button` | Если ни один ключ так не называется, берётся origin энтити |

Это позволяет управлять ботами с карты: Rush-боты идут к `ant.button` по имени; скрипт может двигать target-энтити, чтобы перенаправлять ботов.

### 🌐 Домены

Домены — способ делить состояние между ботами. Форма записи зависит от узла.

| Домен | Где встречается | Назначение |
|---|---|---|
| `AllBots` | `memory_to_expire`, `input_domain`, `output_domain`, `decorator_run_once` | Общая память всех ботов; блокировка для `run_once` |
| `'StratTokens'` | `decorator_token_service` | Токены стратегий |
| `'PressAntennaButton'` | `decorator_try_lock` | Блокировка нажатия кнопки |
| `'DefuseBombIfNearby'` | `decorator_try_lock` | Блокировка разминирования |
| `'PickupBombIfNearby'` | `decorator_try_lock` | Блокировка подбора бомбы |
| `'SetupGlobalCounters'` | `decorator_run_once` | Установка счётчиков |

### 💾 Память

Схема:

```text
decorator_sensor → output = "Vision"
decorator_picker_* → сужают "Vision"
decorator_memory → копирует в "ShortTermAttackMemory"
decorator_bot_service в корне → истекает память: {key, time, distance}
```

#### Стандартные имена памяти

| Ключ | Время | Дистанция | Что хранит |
|---|---|---|---|
| `ShortTermAttackMemory` | 0.7 | 0 | Врагов, виденных недавно |
| `LongTermMemory` | 10 | 500 | Врагов, виденных давно |
| `ShortTermInvestigateMemory` | 3 | 200 (2000 в card13) | События для исследования |
| `ShortTermDamageMemory` | 0.7 | 0 | Источники урона |
| `ShortTermAreaDamageMemory` | 3 | 0 | Урон по площади |
| `DamageThroughSmokeMemory` | 3 | 0 | Источники урона за дымом |
| `Threats` (`AllBots`) | 10 | 0 | Глобальные угрозы |

### 📦 Другие ключи blackboard

| Ключ | Что хранит | Кто пишет | Кто читает |
|---|---|---|---|
| `Vision` | Список врагов, видимых сенсору FOV | `decorator_sensor` в `bt_memorize_enemies_vision` | `decorator_picker_visible`, `decorator_memory` |
| `Hearing` | Список услышанных шумов | `decorator_sensor` (`NOISE`) в `bt_memorize_noises` | `decorator_picker_weight_as_distance`, `decorator_picker_random_by_distance`, `decorator_memory` |
| `Damage` | Сущности, нанёсшие урон | `decorator_sensor` (`DAMAGE`) в `bt_memorize_damage` | `bt_face_damage_inflictor`, `bt_memorize_threats` |
| `Grenades` | Список гранат в поле зрения | `decorator_sensor` (`GRENADE`) в `bt_memorize_area_damage_grenades` | `decorator_picker_grenade_type` |
| `Infernos` | Список `inferno`-энтити | `decorator_sensor` (`CLASSNAME` `inferno`) в `bt_memorize_area_damage_infernos` | `decorator_picker_nearby`, `decorator_memory` |
| `AreaDamage` | Список зон урона | `decorator_sensor` (`AREA_DAMAGE`) в `bt_memorize_area_damage_current` | `decorator_memory` → `ShortTermAreaDamageMemory` |
| `FleeSpot` | Позиция безопасной точки | `action_flee_area_damage` | `action_move_to` в `bt_flee_area_damage_threats` |
| `TargetArea` | Целевая позиция (нав-тайл) | `action_choose_random_waypoint`, `action_choose_guardian_bomb_plant_location`, `action_select_areas_within_radius` | `action_move_to` |
| `TargetHuntArea` | Позиция для охоты | `action_choose_random_waypoint` в ветке «hunt» | `action_move_to` |
| `HuntAreas` | Нав-зоны для охоты | `action_choose_team_spawn_area` / `action_choose_bomb_site_area` | `action_choose_random_waypoint` |
| `BombSiteArea` | Нав-зона бомбсайта | `action_choose_bomb_site_area` | `action_choose_random_waypoint` |
| `BombSiteIndex` | Индекс бомбсайта (`0`/`1`) | `decorator_random_int` | `action_choose_bomb_site_area` |
| `RandomRoamPoint` | Случайная точка для `bt_roam` | `action_choose_random_waypoint_within_radius` | `action_move_to` |
| `RushPoint` | Случайная точка для `bt_rush` | `action_choose_random_waypoint_within_radius` | `action_move_to` |
| `RushCenterLocation` | Центр раша | `action_set_value_vector` в `bt_determine_rush_center` | `action_choose_random_waypoint_within_radius` в `bt_rush` |
| `RushCenterRadius` | Радиус раша | `action_set_value_float` в `bt_determine_rush_center` | `action_choose_random_waypoint_within_radius` в `bt_rush` |
| `UtilityTarget` | Углы для броска гранаты | `action_aim_projectile` | `action_look_at` в `bt_throw_utility_if_covered` |
| `ApproachPoint` | Случайная точка на навмеше для осмотра | `decorator_random_approach_point` в `bt_look_around` | `action_aim` |
| `HidingSpot` | Позиция выбранного укрытия | `decorator_hiding_spot_service` | `action_move_to` (`hiding_spot = "HidingSpot"`) |
| `ShouldBuy` | Массив предметов к покупке | `decorator_buy_service` | `condition_is_empty`, `action_buy` |
| `NearestThreat` | Ближайшая видимая угроза | `decorator_picker_reaction_time` в `bt_select_nearby_threat` | `decorator_memory` |
| `NearestThreatMemory` | Копия ближайшей угрозы | `decorator_memory` в `bt_select_nearby_threat` | `action_move_to` (`threat = "NearestThreatMemory"`) |
| `Event` | Событие для исследования | `decorator_picker_reaction_time` в `bt_investigate_closest_memorized_event` | `action_look_at`, `action_move_to` |
| `VisibleShortTermInvestigateMemory` | Видимое событие | `decorator_picker_visible` в `bt_investigate_closest_memorized_event` | `action_look_at` |
| `ShortTermInvestigateMemoryWithinReactionTime` | Событие после reaction time | `decorator_picker_reaction_time` в `bt_guard` | `condition_distance_less`, `decorator_ranker_dist` |
| `Bombs` | Список `weapon_c4` рядом | `decorator_sensor` (`CLASSNAME` `weapon_c4`) в `bt_pickup_bomb_if_nearby` | `action_move_to` |
| `PlantedBomb` | `planted_c4` рядом | `decorator_sensor` (`CLASSNAME` `planted_c4`) в `bt_defuse_bomb_if_nearby` | `action_move_to`, `action_use` |
| `Hostage` | `hostage_entity` рядом | `decorator_sensor` (`CLASSNAME` `hostage_entity`) в `bt_pickup_hostage` | `action_move_to` |
| `HostageSurroundingAreas` | Нав-зоны вокруг хостажа | `action_select_areas_within_radius` | `action_choose_random_waypoint` |
| `GroupId` | Идентификатор группы токенов | `decorator_token_service` | `condition_is_equal` |
| `GroupMemberId` | Идентификатор участника группы | `decorator_token_service` | `condition_is_equal` |
| `EngagedEntities` | Помеченные враги | `decorator_tag_entity` | `decorator_tag_threshold` |
| `ThreatsWithinFOV` | Угрозы в FOV | `decorator_picker_reaction_time` в `bt_clear_threats_within_fov` | `decorator_picker_visible`, `decorator_remove` |
| `RushSite` | Глобальный счётчик «сайт раша» | `action_set_global_counter` в `bt_coordinated_buy` / `bt_strategy_smoke` | `action_compare_global_counter` |
| `DontPlantBomb` | Глобальный флаг | `action_set_global_flag` в `bt_plant_bomb_if_owned` | `condition_is_empty` (`global = 1`) |
| `PreparingToThrowSmoke` | Глобальный счётчик | `action_set_global_counter` в `bt_coordinated_buy` (`card7_2`) | `decorator_dec_global_counter` |
| `ThrowPalaceFlash` | Глобальный счётчик | `action_set_global_counter` | `decorator_dec_global_counter` |
| `PositioningForTopSmoke` | Глобальный счётчик | `action_set_global_counter` | `decorator_dec_global_counter` |
| `WaitForCoordinatedBuy` | Глобальный счётчик | `action_set_global_counter` в `bt_coordinated_buy` | `action_compare_global_counter` |
| `DidNotPurchase` | ID бота, который не купил | `action_coordinated_buy` | `condition_is_empty` |

### 🚩 Глобальные флаги и счётчики

| Узел | Что делает |
|---|---|
| `action_set_global_flag` | Ставит именованный флаг на случайное время между `expiration_time_min` и `expiration_time_max`. Проверяется через `condition_is_empty` с `global = 1`. |
| `action_set_global_counter` | Ставит именованный счётчик. |
| `decorator_dec_global_counter` | Уменьшает счётчик. |
| `action_compare_global_counter` | Сравнивает счётчик с `input_value`. |

### 📦 Параметры subtree

`subtree` может записывать значения в blackboard до старта. Только `int` или `string`. `float`/`bool` дают ошибку:

```text
[AI BT]: unsupported type for subtree parameter
```

---

## 🧾 Зарезервированные переменные

Игра сама кладёт их в blackboard. Читать можно, создавать нельзя.

| Переменная | Тип | Откуда | Где используется |
|---|---|---|---|
| `NumberOfTerrorists` | `int` | Движок | — |
| `NumberOfAliveTerrorists` | `int` | Движок | `bt_plant_bomb_if_owned` |
| `NumberOfCounterTerrorists` | `int` | Движок | — |
| `NumberOfAliveCounterTerrorists` | `int` | Движок | — |
| `AccountBalance` | `int` | Движок | — |
| `AmmoCount/<weapon>` | `int` | Движок | — |
| `AmmoCount/current` | `int` | Движок | — |
| `TeamName` | `string` | Движок | Practice: `condition_is_equal source="TeamName" destination="'T'"` |
| `RoundNumber` | `int` | Движок | Practice |
| `BlindnessPercentage` | `float` | Движок | — |
| `GuardianWaveNumber` | `int` | Движок (Guardian) | Guardian `county_2` |
| `LastCoopSpawnPointName` | `string` | Движок (Co-op) | Co-op `bt_determine_rush_center` |
| `LastCoopSpawnPointLocation` | `Vector` | Движок (Co-op) | Co-op `guard`/`roam`/`rush` |

> [!NOTE]
> `TeamName` и `RoundNumber` отсутствуют в списке Source2 Wiki, но активно используются в Practice. Список на вики неполный.

---

## 🛠️ Отладка

| Команда | Назначение | Требования |
|---|---|---|
| `mp_bot_ai_bt "путь"` | Назначить дерево всем ботам | — |
| `mp_bot_ai_bt_clear_cache` | Сбросить кэш после правок | — |
| `cv_bot_ai_bt_debug_target <index>` | Нарисовать дерево бота | `sv_cheats 1`, локальный/приватный сервер |
| `cv_bot_ai_bt_hiding_spot_show` | Показать укрытия | `sv_cheats 1` |
| `cv_bot_ai_bt_moveto_show_next_hiding_spot` | Показать следующее проверяемое укрытие | `sv_cheats 1` |
| `developer 1` | Писать `[AI BT]: Loaded behavior tree '<file>'` при спавне | — |

### 💬 Сообщения

| Сообщение | Причина |
|---|---|
| `[AI BT]: Loaded behavior tree '<file>'` | При `developer 1` |
| `[AI BT] warnings` | Всегда |
| `Blackboard entry 'X' does not exist nor it is a valid entity name` | Ключ не найден |
| `[AI BT]: Config node 'attack' must contain 13 entries` | Неверное число строк |
| `[AI BT]: unsupported type for subtree parameter` | `float`/`bool` в `params` |

---

## ⚖️ CS:GO vs CS2

| Аспект | CS:GO | CS2 |
|---|---|---|
| Уровни скилла | 8 (без `elite`) | 9 (с `elite`) |
| Строк в `attack` | В старых 16; в предоставленном файле 20 | 13 для Deathmatch/Rush/Practice, 20 для Co-op/Guardian |
| `aim_ready_angle_tolerance` | Не документирован | Есть в конфиге |
| `TeamName` / `RoundNumber` | Не документированы | Используются в Practice |
| Документация узлов | Неточная, много TODO | Точный список читаемых ключей |
| Узлы вне официальных деревьев | Не отмечены | 36 отмечены, но часть используется в Guardian/Co-op |
| Игнорируемые параметры | Не отмечены | `acquire_only`, `ratio`, `distance_threshold` |
| Модель выполнения | Не описана | Два прохода |
| Subtree params | Не описаны | Только `int`/`string` |
| `decorator_picker_*` | Без доменов | С `input_domain`/`output_domain` |
| `decorator_sensor` | Без `append_items` | С `append_items` |
| `action_wait` | 2 ключа | + `wait_modulo_tick_mod`/`add` |

---

## 🧱 Готовые модули

### 💀 Deathmatch / Rush / Practice

| Модуль | Что делает |
|---|---|
| `bt_attack` | Берёт `ShortTermAttackMemory` → reaction time → сортирует по дистанции → берёт ближайшего → если есть враг: помечает в `EngagedEntities` (SET, 1 с) → параллельно: экипирует `BEST`, повторяет `succeed[out_of_ammo → equip BEST]`, `selector[flee_area_damage`, если нож — бежит к врагу, иначе `combat_positioning`], повторяет `aim Enemy ready="AimReady"`, повторяет `attack Enemy ready="AimReady" output="Attacking"`. |
| `bt_memorize_enemies_vision` | Сенсор FOV, `PLAYERS`, `ENEMY` → nearby 1500 → visible → reaction time → `ShortTermAttackMemory` + `LongTermMemory`. |
| `bt_memorize_noises` | Сенсор сфера 3000, `NOISE` → `weight_as_distance` → `random_by_distance` 800–3000 → `dedup` 100 к `LongTermMemory` → reaction time → `LongTermMemory`. |
| `bt_memorize_damage` | Сенсор `DAMAGE` → reaction time → `ShortTermDamageMemory`; `blocked_by_smoke` 3000 → `DamageThroughSmokeMemory`. |
| `bt_memorize_area_damage_grenades` | Сенсор `GRENADE` → `grenade_type` (`FIRE`, `EXPLOSIVE`) → nearby 2500 → visible → reaction time → `ShortTermAreaDamageMemory`. |
| `bt_memorize_area_damage_infernos` | Сенсор `CLASSNAME` `inferno` → nearby 1000 → reaction time → `ShortTermAreaDamageMemory`. |
| `bt_memorize_area_damage_current` | Сенсор `AREA_DAMAGE` → `ShortTermAreaDamageMemory`. |
| `bt_memorize_nearest_investigation` | `succeed` → `condition_is_empty ShortTermInvestigateMemory` → если пусто: `memory LongTermMemory` → `ShortTermInvestigateMemory` → `ranker_dist` → `max_score`. Всегда успех, но если память была пуста — заполняет ближайшим событием. |
| `bt_investigate_closest_memorized_event` | Берёт `ShortTermInvestigateMemory` → reaction time → если есть событие: экипирует `BEST`, если видимо — смотрит и ждёт 0.2–0.5 с, затем идёт к событию (`move_to threat="NearestThreatMemory"`). |
| `bt_heal_if_needed` | Если HP < 60: экипирует `weapon_healthshot`, ждёт 1 с, `action_pull_trigger`. |
| `bt_face_damage_inflictor` | Берёт `ShortTermDamageMemory` → reaction time → если есть: целится (`acquire_only`), параллельно `succeed` → `blocked_by_smoke` 3000 → если есть: повторяет `action_attack output="Attacking"`. |
| `bt_attack_damage_inflictor_smoke` | Берёт `DamageThroughSmokeMemory` → reaction time → сортирует → берёт ближайшего → целится, повторяет `action_attack input="Damage" output="Attacking"`. |
| `bt_flee_area_damage_threats` | Если `ShortTermAreaDamageMemory` не пусто: `action_flee_area_damage` 500 → бежит к `FleeSpot` (`auto_look_adjust=0`). |
| `bt_look_around` | `random_approach_point` → `action_aim` (`acquire_only`) → ждёт 0.2–0.5 с. |
| `bt_plant_bomb_if_covered` | Если есть `weapon_c4` и в `ShortTermAttackMemory` ≥2 врага: выбирает бомбсайт, идёт, проверяет `@mp_plant_c4_anywhere` или `is_at_bomb_site`, экипирует C4, ждёт 0.5 с, повторяет `action_pull_trigger`. |
| `bt_plant_bomb_if_owned` | Если есть `weapon_c4` и нет флага `DontPlantBomb`: экипирует `BEST`, выбирает бомбсайт, идёт, проверяет site, экипирует C4, ждёт 1 с, параллельно: с шансом 0.5 и если `NumberOfAliveTerrorists=1` ставит флаг `DontPlantBomb` (3–5 с), и повторяет `action_pull_trigger`. Если флаг `DontPlantBomb` — смотрит по сторонам. |
| `bt_pickup_bomb_if_nearby` | Сенсор сфера 3000, `CLASSNAME` `weapon_c4`, `orphan_only` → сортирует → берёт ближайшее → `try_lock 'PickupBombIfNearby'` → `action_move_to` (`nearest_area_distance_threshold=20`). |
| `bt_defuse_bomb_if_nearby` | Сенсор сфера 3000, `CLASSNAME` `planted_c4`, `orphan_only` → сортирует → `try_lock 'DefuseBombIfNearby'` → идёт, смотрит, ждёт 0.25 с, повторяет `action_use`. |
| `bt_pickup_hostage` | Сенсор `CLASSNAME` `hostage_entity`, `orphan_only` → идёт (`arrival_epsilon=35`), смотрит, ждёт 1 с, повторяет `action_use`. |
| `bt_pickup_hostage_if_covered` | Берёт `ShortTermAttackMemory` → reaction time → сортирует → если ≥2 врага в `EngagedEntities` (`CLEAR`, `amount=2`, `AT_LEAST`) → `bt_pickup_hostage`. |
| `bt_clear_threats_within_fov` | `succeed negated=1` → если `ShortTermAttackMemory` пусто: берёт `Threats` (`AllBots`) → reaction time → visible → удаляет из `Threats`. Side-effect, всегда failure. |
| `bt_select_nearby_threat` | `selector[succeed negated=1` → `remove_key NearestThreatMemory`; `succeed negated=1` → `Threats` (`AllBots`) → reaction time → visible (`check_fov=0`) → `ranker_dist` → `max_score` → если есть: `memory` → `NearestThreatMemory`]. Всегда failure, но обновляет `NearestThreatMemory`. |
| `bt_memorize_threats` | Сенсор `DAMAGE` → reaction time → `Threats` (`AllBots`). |

### 🤝 Co-op

| Модуль | Что делает |
|---|---|
| `bt_basic_player_engagement` | `selector[throw_utility_if_covered, attack, face_damage_inflictor, parallel[flee_area_damage_threats, repeat look_around], attack_damage_inflictor_smoke, clear_threats_within_fov, select_nearby_threat, heal_if_needed]`. |
| `bt_basic_player_memorization` | `parallel[memorize_enemies_vision, memorize_noises, memorize_nearest_investigation, memorize_area_damage_grenades, memorize_area_damage_infernos, memorize_damage, memorize_threats, memorize_area_damage_current]`. |
| `bt_determine_rush_center` | `succeed negated=1` → `selector[condition_is_equal LastCoopSpawnPointName='enemy.waveXX'` → `parallel[set_value_vector RushCenterLocation, set_value_float RushCenterRadius]]`. Всегда failure, но ставит центр раша. |
| `bt_throw_utility_if_covered` | `sequencer[preconditions (reaction time, ≥2 врага в EngagedEntities, есть hegrenade/incgrenade), maybe 0.15 → sequencer[aim_projectile, equip_item, wait 0.5, look_at, pull_trigger, wait 0.5]]`. |
| `bt_guard` | `parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement, Investigate (look at), Investigate (distance_less 450–550), go back to spawn + look around + maybe say "Clear" (0.7) + wait 4–8]]`. |
| `bt_roam` | `parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement, Investigate, roam (choose_random_waypoint_within_radius LastCoopSpawnPointLocation 700, move_to, maybe say "Clear" (0.7), wait 1–3)]]`. |
| `bt_rush` | `parallel[repeat BasicPlayerMemorization, repeat selector[DetermineRushCenter, condition_is_empty RushCenterLocation → action_jump, BasicPlayerEngagement, Investigate, Rush (choose_random_waypoint_within_radius RushCenterLocation RushCenterRadius, move_to)]]`. |
| `bt_sniper` | `parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement]]`. |

### 🛡️ Guardian

| Модуль | Что делает |
|---|---|
| `bt_item_strategy` | Покупка/выдача предметов по токенам стратегий. |
| `bt_takeover_strategy` | Захват позиции по токенам стратегий. |
| `bt_attack_with_shield` | Атака со щитом. |
| `bt_investigate_with_shield` | Исследование со щитом. |
| `bt_coordinated_buy` | Скоординированная покупка (с известным багом). |
| `bt_strategy_smoke`, `bt_strategy_grenades` | Стратегии дымов/гранат. |
| `action_assign_guardian_loadout` | Раздача лутаутов по волнам в `item_strategy`. |
| `action_throw_utility` | Прямые броски дымов/молотовых в `county_2`. |

---

## 🧬 Паттерны

### 🔄 Восприятие + решение через parallel

```text
parallel
├── repeat → parallel (sensors, memory)
└── repeat → selector (priority list)
```

### 🎯 Selector как список приоритетов

```text
selector
├── condition_is_empty ShouldBuy (negated) → sequencer[wait, buy, remove_key]
├── sensor DAMAGE → aim
├── subtree Attack
├── subtree HealIfNeeded
└── subtree Hunt
```

### 🔒 Блокировка действия одним ботом

```kv3
decorator_try_lock
domain = "'PressAntennaButton'"
child = sequencer[ move_to ant.button, look_at, wait, use ]
```

### 🎭 Side-effect без остановки selector

```kv3
decorator_succeed
negated = 1
child = sequencer[ ... сделать что-то ... ]
```

Selector увидит failure и пойдёт дальше, но side-effect уже выполнен.

### 🧹 Истечение памяти в корне

```kv3
decorator_bot_service
memory_to_expire =
[
    { key = "ShortTermAttackMemory" time = 0.7 distance = 0 }
    { key = "LongTermMemory" time = 10 distance = 500 }
    { key = "ShortTermInvestigateMemory" time = 3 distance = 200 }
]
```

---

## ⚠️ Подводные камни

| ❌ Не делайте | ✅ Делайте вместо этого |
|---|---|
| Полагаться на `acquire_only`, `ratio`, `distance_threshold` | Эти параметры в ряде узлов игнорируются |
| Использовать `action_coordinated_buy` без `decorator_run_once` | `decorator_run_once` + `action_buy` |
| Забывать `mp_bot_ai_bt_clear_cache` | Всегда чистить кэш после правок |
| Передавать `float`/`bool` в `subtree params` | Только `int` или `string` |
| Использовать 20 строк в `attack` для CS2 Deathmatch | Ровно 13, в фиксированном порядке |
| Ожидать, что `action_aim` разскопится | Смените оружие или экипируйте то же заново |
| Полагаться на `condition_is_greater`/`is_less` | Используйте `greater_equal`/`less_equal` |
| Использовать `decorator_invert` в CS2 | Используйте `negated = 1` |
| Забывать, что `action_attack` без `ready` стреляет сразу | Указывайте `ready`, если нужна синхронизация с `action_aim` |
| Ожидать, что `action_aim_projectile` точно бросит гранату | Баг: не учитывает траекторию |
| Использовать `decorator_picker_blocked_by_smoke` без понимания | Он оставляет только сущности за дымом |
| Думать, что `action_acquire_items` не работает | ✅ Работает; встречается в Practice/Guardian |
| Думать, что `action_teleport` не работает | ✅ Работает; встречается в Guardian/Co-op |
| Думать, что `action_crouch` не работает | ✅ Работает; встречается в `op09_grunts_card2` |
| Игнорировать `decorator_dec_global_counter` | ✅ Встречается в `op09_grunts_card7_2` |
| Писать `domain = PressAntennaButton` без кавычек | В файлах Valve: `domain = "'PressAntennaButton'"` |
| Унифицировать все `domain` под `"AllBots"` | Смотрите конкретный узел: где-то `"AllBots"`, где-то `"'StratTokens'"` |
| Использовать `decorator_find_utility_strat`, `decorator_route_service` или `decorator_set_barrier` в своих деревьях | Эти узлы не встречаются в официальных деревьях, их поведение не подтверждено |
