# Behavior Trees ботов CS2 и CS:GO — Полная вики

> ✅ Проверено по всем предоставленным файлам: конфиги (13 и 20 строк), деревья Deathmatch, Rush, Practice, Co-op (Guard/Roam/Rush/Sniper), Guardian (op09/op10/op11), все модули.
>
> 📌 Оригинальные файлы — истина. Узлы, которые не встречаются в предоставленных деревьях, помечены как «требует тестирования» — это не значит, что они не работают, просто их поведение не подтверждено на практике.

---

## 📋 Содержание

1. [Что такое Behavior Tree](#1-что-такое-behavior-tree)
2. [Быстрый старт](#2-быстрый-старт)
3. [Формат KV3](#3-формат-kv3)
4. [Конфиг бота](#4-конфиг-бота)
5. [Композиты](#5-композиты)
6. [Декораторы](#6-декораторы)
7. [Условия](#7-условия)
8. [Действия](#8-действия)
9. [Модель выполнения](#9-модель-выполнения)
10. [Blackboard, домены, память](#10-blackboard-домены-память)
11. [Зарезервированные переменные](#11-зарезервированные-переменные)
12. [Отладка](#12-отладка)
13. [CS:GO vs CS2](#13-csgo-vs-cs2)
14. [Готовые модули](#14-готовые-модули)
15. [Паттерны](#15-паттерны)
16. [Подводные камни](#16-подводные-камни)

---

## 1. Что такое Behavior Tree

**Behavior Tree (BT)** — KV3-файл, описывающий поведение бота как дерево узлов. Игра обходит дерево **каждый тик** (~64 раза/сек): проверяет условия, выполняет действия. Бот не «думает» — он следует дереву.

### 📁 Где лежат файлы

| Что | Где |
|-----|-----|
| Деревья и модули | `scripts/ai/...` внутри `game/csgo/pak01_dir.vpk` (извлекаются Source2 Viewer) |
| Своё дерево | `scripts/ai/mymode/bt_default.kv3` |
| Конфиг | `scripts/ai/<mode>/bt_config.kv3` |

### ⚙️ Как включить

```cfg
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
```

Затем перезапуск раунда (`mp_restartgame 1` или `endround`), чтобы боты переспавнились. С `developer 1` боты при спавне пишут `[AI BT]: Loaded behavior tree '<file>'`. Ошибки — всегда с префиксом `[AI BT]`.

### 🎮 Официальные режимы

| Режим | Дерево | Конфиг |
|-------|--------|--------|
| Deathmatch / DM Freeforall | `scripts/ai/deathmatch/bt_default.kv3` | 13 строк |
| Arms Race | `scripts/ai/armsrace/bt_default.kv3` | 13 строк |
| Rush | `scripts/ai/rush/bt_default.kv3` | 13 строк |
| New User Training | `scripts/ai/practice/newplayer_dust2_01.kv3` | 13 строк |
| Co-op (Guard/Roam/Rush/Sniper) | `scripts/ai/coop/bt_*.kv3` | 20 строк |
| Guardian | `scripts/ai/guardian/bt_op*.kv3` | 20 строк |

> ⚠️ **Про 13 и 20 строк:** в предоставленных файлах конфиги Deathmatch/Rush/Practice имеют **13 строк** в таблице `attack`, конфиги Co-op/Guardian — **20 строк**. Source2 Wiki утверждает, что CS2 требует ровно 13 строк, иначе ошибка `[AI BT]: Config node 'attack' must contain 13 entries`. Вероятно, Guardian/Co-op используют другой путь загрузки или это CS:GO-совместимые конфиги. **Для своих деревьев в Deathmatch-стиле используйте 13 строк.**

---

## 2. Быстрый старт

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
                { type = "action_move_to" destination = "bot_target" movement_type = "BT_ACTION_MOVETO_RUN" route_type = "BT_ACTION_MOVETO_FASTEST_ROUTE" }
                { type = "action_look_at" input_location = "bot_target" }
                { type = "action_wait" wait_time_min = 2 wait_time_max = 3 }
            ]
        }
    }
}
```

### 🧠 Что делает это дерево

1. `decorator_repeat` — бесконечный цикл.
2. `sequencer` — выполняет детей по порядку.
3. `action_move_to` — бот бежит к энтити `bot_target` (пока не прибудет).
4. `action_look_at` — смотрит на `bot_target`.
5. `action_wait` — ждёт 2–3 секунды.
6. Цикл повторяется.

### 🧪 Как проверить

```cfg
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
mp_restartgame 1
```

Поставьте на карту энтити с targetname `bot_target` (например, `info_target`). Бот будет бежать к ней.

---

## 3. Формат KV3

Каждый файл начинается с обязательного заголовка:

```kv3
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
```

Далее — объект с двумя ключами:

| Ключ | Обязателен | Что означает |
|------|-----------|--------------|
| `config` | ✅ для корневого дерева | Путь к конфигу |
| `root` | ✅ | Корневой узел |

**Узел** — таблица с ключом `type`. Декораторы и условия имеют одного ребёнка `child`; композиты — массив `children`. Subtree-файлы содержат **один узел верхнего уровня** без `config` и `root`.

### ✍️ Синтаксис

| Элемент | Пример | Примечание |
|---------|--------|------------|
| Комментарии | `// текст` | Однострочные |
| Ключи | `type`, `input`, `destination` | Регистронезависимы (`Name` = `name`) |
| Строки-литералы | `'CT'` | В одинарных кавычках; кавычки отбрасываются |
| Строки в двойных кавычках | `"scripts/ai/..."` | Одинарные внутри сохраняются как часть строки: `domain = "'PressAntennaButton'"` → значение `'PressAntennaButton'` |
| Числа | `3`, `0.5` | — |
| Массивы | `[ ... ]` | — |
| Объекты | `{ ... }` | — |

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

## 4. Конфиг бота

Конфиг задаёт параметры действий: скорость прицеливания, реакцию, стрельбу. Он не управляет логикой — только «мышечной памятью».

### 🎚️ Уровни скилла

| Версия | Уровни |
|--------|--------|
| CS:GO | `default`, `low`, `fair`, `normal`, `tough`, `hard`, `very_hard`, `expert` (8) |
| CS2 | + `elite` (9) |

Каждый уровень наследует недостающие ключи из `default`. В предоставленных файлах все 9 уровней.

### 🎯 4.1 Параметры прицеливания

| Ключ | Тип | Описание |
|------|-----|----------|
| `aim_target_acquisition_lerp_time` | float | Время «доезда» прицела до новой цели (сек). Больше = медленнее наводится. |
| `aim_target_acquisition_lerp_time_deviation` | float | Случайный разброс времени. |
| `aim_target_acquisition_angle_penalty` | float | Штраф за большой угол до цели. Больше = сильнее замедляется при большом угле. |
| `aim_target_acquisition_angle_penalty_deviation` | float | Разброс штрафа. |
| `aim_target_acquisition_angle_penalty_reduction_ratio` | float | Насколько штраф уменьшается со временем (0–1). |
| `aim_target_acquisition_angle_tolerance` | float | Допуск угла при захвате. Меньше = точнее нужно навестись. |
| `aim_target_acquisition_angle_lerp_bias` | float | Сглаживание угла захвата (0–1). |
| `aim_target_tracking_lerp_time` | float | Скорость сопровождения цели (сек). Больше = медленнее следит. |
| `aim_target_tracking_lerp_time_deviation` | float | Разброс сопровождения. |
| `aim_target_tracking_focus_interval` | float | Интервал «фокуса» при трекинге (сек). |
| `aim_target_tracking_focus_interval_deviation` | float | Разброс интервала. |
| `aim_target_tracking_angle_lerp_bias` | float | Сглаживание трекинга. |
| `aim_ready_angle_tolerance` | float | Угол, при котором `action_attack` считает прицел готовым (флаг `ready`). В CS:GO не документирован. |
| `aim_new_target_angle_tolerance` | float | Угол для переключения на новую цель. |
| `aim_max_duration` | float | Максимальное время прицеливания (сек). |
| `aim_max_duration_deviation` | float | Разброс макс. времени. |
| `aim_punch_angle_reaction_chance` | float | Шанс реакции на отдачу (0–1). |

### 👀 4.2 Параметры осмотра

| Ключ | Описание |
|------|----------|
| `look_around_awareness_yaw_range` | Макс. угол поворота по горизонтали (градусы). |
| `look_around_awareness_pitch_range` | Макс. угол по вертикали. |
| `look_around_focus_interval` | Пауза между поворотами камеры (сек). |
| `look_around_focus_interval_deviation` | Разброс паузы. |
| `look_around_lerp_time` | Время поворота камеры (сек). |
| `look_around_lerp_time_deviation` | Разброс времени. |
| `look_around_lerp_bias` | Сглаживание движения камеры. |

### ⚔️ 4.3 Реакция и бой

| Ключ | Описание |
|------|----------|
| `reaction_time` | Задержка перед реакцией на событие (сек). |
| `combat_crouch_chance` | Шанс присесть в бою (0–1). |
| `combat_dodge_command_duration` | Длительность команды уклонения (сек). |
| `combat_dodge_command_duration_deviation` | Разброс длительности. |

### 🔫 4.4 Таблица `attack`

Каждая строка — 5 чисел: **burst duration, duration deviation, cadence, cooldown, cooldown deviation**.

| Поле | Что означает |
|------|--------------|
| burst duration | Длительность одной очереди (сек). |
| duration deviation | Разброс длительности. |
| cadence | Пауза между выстрелами внутри очереди (0 = без паузы). |
| cooldown | Пауза между очередями (сек). |
| cooldown deviation | Разброс паузы. |

#### 📋 13 строк (Deathmatch/Rush/Practice)

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

#### 📋 20 строк (Co-op/Guardian)

Те же 13 + `FISTS`, `BREACHCHARGE`, `BUMPMINE`, `TABLET`, `MELEE`, `SHIELD`, `WEAPONTYPE_ZONE_REPULSOR` (порядок как в файлах: после `STACKABLEITEM` идут `FISTS`, `BREACHCHARGE`, `BUMPMINE`, `TABLET`, `MELEE`, `SHIELD`, `WEAPONTYPE_ZONE_REPULSOR`, `UNKNOWN`).

#### 📊 Пример `RIFLE`

| Конфиг | Значение | Интерпретация |
|--------|----------|---------------|
| Deathmatch | `[ 0.22, 0.07, 0.0, 0.45, 0.15 ]` | Очередь ~0.22 с (±0.07), без паузы внутри, новая очередь через 0.45 с (±0.15). |
| Rush/Guardian | `[ 0.25, 0.55, 0.0, 0.95, 0.25 ]` | Очередь ~0.25 с (±0.55, очень большой разброс), cooldown 0.95 с. |

---

## 5. Композиты

Композиты — узлы, которые содержат несколько детей и решают, как их выполнять.

| Узел | Ключи | Что делает | Пример |
|------|-------|------------|--------|
| `parallel` | `succeed_after_first` (bool), `children` (node[]) | Запускает **всех детей каждый тик**. Провал любого = провал всего. Успех = все дети успешны (или первый, если `succeed_after_first = 1`). | `parallel[aim, attack]` — бот одновременно целится и стреляет. |
| `selector` | `children` (node[]) | **Список приоритетов.** Проверяет детей с первого каждый тик. Первый прошедший запускается. Если более ранний ребёнок начал проходить — текущий останавливается. | `selector[buy, damage_aim, attack, hunt]` — если нужно купить — покупает, иначе если есть урон — целится, иначе атакует, иначе охотится. |
| `sequencer` | `children` (node[]) | Выполняет детей **по порядку**. Провал любого = стоп. Успех = все прошли. Check-pass смотрит только на текущего ребёнка — пройденные условия не перепроверяются. | `sequencer[move_to, look_at, wait]` — сначала дойди, потом посмотри, потом подожди. |
| `subtree` | `file` (string), `name` (string), `params` ({`key`, `value`}[]) | Запускает **другое дерево** из файла. `params` записывают значения в blackboard до старта. Только **int или string** — float/bool отвергаются. Имена ключей регистронезависимы (`Name` = `name`). | `subtree file="bt_attack.kv3" name="Attack"` — запускает модуль атаки. |

### 📄 Пример subtree с параметрами

```kv3
{
    type = "subtree"
    file = "scripts/ai/mymode/bt_goto.kv3"
    name = "GotoA"
    params =
    [
        { key = "Target" value = "site_a_target" },
        { key = "WaitTime" value = 3 }
    ]
}
```

---

## 6. Декораторы

Декораторы оборачивают **одного ребёнка** через `child`. Почти все поддерживают `negated = 1` (инверсия check-результата).

### 🏗️ 6.1 Корневые сервисы

| Узел | Ключи | Что делает | Где встречается |
|------|-------|------------|-----------------|
| `decorator_bot_service` | `memory_to_expire` ({`key`, `time`, `distance`, `domain`}[]), `tagged_entities_to_expire` (string[]), `input_chatter_enemies` (string), `basic_chatter_enable` (bool), `chatter_outnumbered_threshold` (int), `child` | Корневой сервис. Истекает память (удаляет старые записи), управляет болтовнёй. `memory_to_expire` — список ключей памяти с временем жизни и дистанцией. | Все деревья (Co-op — с болтовнёй) |
| `decorator_buy_service` | `output` (string), `child` | Возвращает массив предметов к покупке. Выход — ключ blackboard (обычно `ShouldBuy`). | Deathmatch, Practice, Guardian (кроме card13) |
| `decorator_hiding_spot_service` | `domain` (string), `output_hiding_spot` (string), `expiration_time` (float), `distance_threshold` (float), `child` | Учитывает укрытия. Боты в одном домене не проверяют уже проверенные укрытия. Выход — ключ с укрытием (обычно `HidingSpot`). | Practice, Guardian, Co-op |

### 🎣 6.2 Пикеры (фильтры сущностей)

Все пикеры имеют `input`, `input_domain`, `output`, `output_domain`, `negated`. Они **сужают** список сущностей.

| Узел | Доп. ключи | Что делает | Пример |
|------|------------|------------|--------|
| `decorator_picker_nearby` | `cutoff_distance` (float) | Убирает сущности дальше `cutoff_distance`. | `bt_memorize_enemies_vision` — 1500. |
| `decorator_picker_visible` | `check_fov` (bool, по умолчанию 1) | Убирает невидимые сущности. `check_fov = 0` — игнорирует FOV (видит на 360°). | `bt_select_nearby_threat` — `check_fov = 0`. |
| `decorator_picker_reaction_time` | — | Задерживает выход на `reaction_time` бота (из конфига). Бот «думает» перед реакцией. | `bt_attack` — `input = "ShortTermAttackMemory" output = "Enemy"`. |
| `decorator_picker_dedup` | `against` (string), `distance_threshold` (float) | Убирает дубликаты — сущности, которые ближе `distance_threshold` к сущностям из `against`. | `bt_memorize_noises` — `against = "LongTermMemory" distance_threshold = 100`. |
| `decorator_picker_random_by_distance` | `distance_min`, `distance_max` | Вероятностный фильтр по дистанции. Ближе к `min` — менее вероятно удаление; ближе к `max` — более вероятно. | `bt_memorize_noises` — 800–3000. |
| `decorator_picker_weight_as_distance` | — | Убирает сущности, чья дистанция больше их «веса». Вес — внутреннее понятие, используется для слуха. | `bt_memorize_noises`. |
| `decorator_picker_grenade_type` | `types[]` (enum) | Фильтр гранат по типу (`FLASH`, `FIRE`, `EXPLOSIVE`, `DECOY`, `SMOKE`). | `bt_memorize_area_damage_grenades` — `["FIRE", "EXPLOSIVE"]`. |
| `decorator_picker_blocked_by_smoke` | — | **Оставляет только сущности, которые закрыты дымом** (за дымом). Убирает тех, кто не за дымом. `distance_threshold` **игнорируется**. | `bt_memorize_damage` — копирует Damage в `DamageThroughSmokeMemory` только если источник за дымом. |
| `decorator_picker_max_score` | — | Берёт **первую** сущность из списка. По умолчанию — по порядку на сервере. Если перед ним стоит `decorator_ranker_dist`, берётся ближайшая. | `bt_attack` — `ranker_dist` сортирует, `max_score` берёт ближайшего. |

### 📊 6.3 Ранкеры

| Узел | Ключи | Что делает | Пример |
|------|-------|------------|--------|
| `decorator_ranker_dist` | `input` (string) | Сортирует сущности по дистанции (ближайшие первыми). | `bt_attack`, `bt_select_nearby_threat`, `bt_pickup_bomb_if_nearby`, `bt_pickup_hostage`, `bt_defuse_bomb_if_nearby`. |

### 📡 6.4 Сенсоры

| Узел | Ключи | Что делает | Пример |
|------|-------|------------|--------|
| `decorator_sensor` | `output` (string), `class_name` (string), `priority` (int), `orphan_only` (bool), `append_items` (bool), `entity_type_filter` (enum), `team_filter` (enum), `attacker_filter` (enum), `shape` ({`type`, `radius`}), `child` | Сканирует мир и записывает найденные сущности в `output`. `shape` — форма сканирования: `sensor_shape_fov` (конус впереди) или `sensor_shape_sphere` (сфера вокруг бота с `radius`). `orphan_only = 1` — только «бесхозные» сущности (бомба, заложник, не занятые другим ботом). `append_items = 1` — добавить к существующему списку, а не заменить. `priority` — приоритет сенсора. | `bt_memorize_enemies_vision` — FOV, PLAYERS, ENEMY → `Vision`. |

### 💾 6.5 Память

| Узел | Ключи | Что делает | Пример |
|------|-------|------------|--------|
| `decorator_memory` | `input` (string), `output` (string), `output_domain` (string), `child` | Копирует сущности из `input` в `output` (память). Если указан `output_domain` — в общую память домена (например, `AllBots`). | `bt_memorize_enemies_vision` — `input = "Vision" output = "ShortTermAttackMemory"` и `output = "LongTermMemory"`. |
| `decorator_set_reaction_time` | `input`, `child` | Задерживает прохождение `input` на `reaction_time` бота. | `bt_memorize_damage` — `input = "Damage"`. |

### 🔀 6.6 Управление потоком

| Узел | Ключи | Что делает | Пример |
|------|-------|------------|--------|
| `decorator_repeat` | `amount` (int), `child` | Перезапускает ребёнка каждый раз, когда он завершается. Без `amount` — вечно. С `amount = N` — N раз и возвращает результат последнего. | `decorator_repeat` вокруг всего дерева — бесконечный цикл. |
| `decorator_maybe` | `chance` (float, 0–1), `child` | Выполняет ребёнка с вероятностью `chance`. 1 = 100%, 0.7 = 70%. | `bt_plant_bomb_if_owned` — 0.5. Co-op guard/roam — 0.7. `bt_throw_utility_if_covered` — 0.15. |
| `decorator_succeed` | `negated` (bool), `child` | Без `negated`: **всегда проходит check-pass**, после ребёнка возвращает **success**. С `negated = 1`: **всегда проходит check-pass**, после ребёнка возвращает **failure**. Используется для side-effect действий, чтобы не останавливать selector. | `bt_clear_threats_within_fov` — `succeed negated=1` (очищает угрозы, но всегда failure). |
| `decorator_invert` | `child` | Инвертирует check-результат ребёнка. Если ребёнок не прошёл — invert проходит. Если прошёл — invert не проходит. Во время работы результаты ребёнка проходят без изменений. | В предоставленных деревьях не встречается — в CS2 вместо него используют `negated = 1`. Требует тестирования. |
| `decorator_run_once` | `domain` (string), `max_attempts` (int), `child` | Ограничивает запуск. Без `domain` — каждый бот выполняет один раз. С `domain` — **только один бот** на сервере выполняет. | Guardian card7_2 — `domain = "'AllBots'"`, `domain = "'SetupGlobalCounters'"`. |
| `decorator_try_lock` | `domain` (string), `child` | Блокирует домен на время выполнения ребёнка. **Только один бот** может выполнять ребёнка одновременно. | Rush — `"'PressAntennaButton'"`. `bt_pickup_bomb_if_nearby` — `"'PickupBombIfNearby'"`. `bt_defuse_bomb_if_nearby` — `"'DefuseBombIfNearby'"`. |
| `decorator_wait_success` | `timeout` (float), `child` | Ждёт, пока ребёнок вернёт success, или пока не истечёт `timeout` секунд. | Guardian card13 — 30. Card7_2 — 4. |
| `decorator_need_healing` | `health_threshold` (int), `child` | Выполняет ребёнка, если HP бота ниже `health_threshold`. | `bt_heal_if_needed` — 60. |
| `decorator_random_int` | `min` (int), `max` (int), `output` (string), `child` | Генерирует случайное целое от `min` до `max` и пишет в `output`. | Deathmatch — `min = 0 max = 1 output = "BombSiteIndex"`. Guardian card1 — `min = 0 max = 0`. |
| `decorator_random_approach_point` | `output` (string), `child` | Генерирует случайную видимую точку в навмеше — «вход» в текущую область. Полезно для осмотра. | `bt_look_around` — `output = "ApproachPoint"`. |

### 🏷️ 6.7 Теги и счётчики

| Узел | Ключи | Что делает | Пример |
|------|-------|------------|--------|
| `decorator_tag_entity` | `input` (string), `output` (string), `expiration_time` (float), `operation_type` (enum), `child` | Помечает первую сущность из `input` в `output`. `operation_type`: `SET` (добавить) или `CLEAR` (убрать). `expiration_time` — через сколько секунд тег снимется. | `bt_attack` — `SET expiration_time=1`. `bt_throw_utility_if_covered` — `CLEAR`. |
| `decorator_tag_threshold` | `entity_input` (string), `tagged_entities_input` (string), `amount` (int), `check_type` (enum), `child` | Сравнивает количество помеченных сущностей. `AT_LEAST` — минимум `amount`; `AT_MOST` — максимум `amount`. | `bt_plant_bomb_if_covered` — `amount = 2 AT_LEAST` (закладка, если в бою ≥2 врага). |
| `decorator_token_service` | `config` ({`tokens[]`, `assignments[][]`, `team_restriction[]`}), `output_token_name` (string), `output_token_domain` (string), `domain` (string), `child` | Раздаёт ботам «токены» (роли). `tokens` — список имён ролей. `assignments` — матрица: сколько ботов получают каждый токен в каждом варианте. `team_restriction` — ограничение по команде (`T`, `CT`). Выход — `GroupId` (имя токена) и `GroupMemberId` (номер в группе). | Practice — 10 токенов, по 1 боту. Guardian blacksite_1 — `tokens=["Parachute"] assignments=[[1000]]`. |
| `decorator_remove` | `input_domain` (string), `input` (string), `remove` (string), `child` | Удаляет сущности из `input` (в домене `input_domain`), которые есть в `remove`. | `bt_clear_threats_within_fov` — `input_domain = "AllBots" input = "Threats" remove = "ThreatsWithinFOV"`. |
| `decorator_remove_key` | `input` (string), `child` | Удаляет ключ blackboard. | Deathmatch — `input = "ShouldBuy"`. `bt_select_nearby_threat` — `input = "NearestThreatMemory"`. |
| `decorator_dec_global_counter` | `input_name` (string), `child` | Уменьшает глобальный счётчик на 1. | В предоставленных деревьях не встречается — требует тестирования. |
| `decorator_find_utility_strat` | много | — | В предоставленных деревьях не встречается — требует тестирования. |
| `decorator_route_service` | много | — | В предоставленных деревьях не встречается — требует тестирования. |
| `decorator_set_barrier` | много | — | В предоставленных деревьях не встречается — требует тестирования. |

---

## 7. Условия

Условия проверяются в **каждом check-pass**. Пока условие истинно — выполняется ребёнок (или просто успех). При провале — провал ветки. Все поддерживают `negated = 1`.

| Узел | Ключи | Что делает | Где встречается |
|------|-------|------------|-----------------|
| `condition_is_empty` | `input` (string), `global` (bool), `negated`, `child` | Проверяет, пуст ли ключ или его нет. `global = 1` — проверяет глобальный флаг. | Deathmatch — `ShouldBuy negated=1`. `bt_plant_bomb_if_owned` — `DontPlantBomb global=1`. |
| `condition_is_equal` | `source` (string), `destination` (int или string), `negated`, `child` | Проверяет равенство `source` и `destination`. | Practice — `TeamName = 'T'`. Guardian — `@mp_plant_c4_anywhere = 1`. `NumberOfAliveTerrorists = 1`. Co-op — `LastCoopSpawnPointName = 'enemy.wave02'`. |
| `condition_is_greater_equal` | `source`, `destination`, `negated`, `child` | Проверяет `source ≥ destination`. | Официальные деревья. |
| `condition_is_less_equal` | `source`, `destination`, `negated`, `child` | Проверяет `source ≤ destination`. | Официальные деревья. |
| `condition_is_greater` | `source`, `destination`, `negated`, `child` | Проверяет `source > destination`. | В предоставленных деревьях не встречается — требует тестирования. |
| `condition_is_less` | `source`, `destination`, `negated`, `child` | Проверяет `source < destination`. | В предоставленных деревьях не встречается — требует тестирования. |
| `condition_is_inv_slot_empty` | `slot` (enum), `negated`, `child` | Проверяет, пуст ли слот (`RIFLE`, `PISTOL`, `KNIFE`, `GRENADES`, `C4`). | В предоставленных деревьях не встречается — требует тестирования. |
| `condition_is_weapon_equipped` | `weapon` (string), `negated`, `child` | Проверяет, что активное оружие — с этим classname. | `bt_attack` — `weapon = "weapon_knife"`. |
| `condition_out_of_ammo` | `negated`, `child` | Проверяет, что в активном оружии нет патронов. | `bt_attack`. |
| `condition_owns_item` | `item` (string), `items_one_of` (string[]), `negated`, `child` | Проверяет, есть ли у бота предмет (по classname). | `bt_plant_bomb_*` — `weapon_c4`. `bt_throw_utility_if_covered` — `weapon_hegrenade`, `weapon_incgrenade`. |
| `condition_is_at_bomb_site` | `negated`, `child` | Проверяет, находится ли бот на бомбсайте. | `bt_plant_bomb_*`. |
| `condition_distance_less` | `input` (string), `distance_threshold_min` (float), `distance_threshold_max` (float), `negated`, `child` | Проверяет, что дистанция до `input` **меньше** случайного порога между `min` и `max`. | Co-op guard — 450–550. |
| `condition_has_parachute` | `negated`, `child` | Проверяет, есть ли у бота парашют. | Guardian blacksite_1. |
| `condition_inactive` | `input` ({`key`}[]), `round_start_threshold_seconds` (float), `sensor_inactivity_threshold_seconds` (float), `negated`, `child` | Проверяет, что в указанных ключах нет активности в течение `sensor_inactivity_threshold_seconds` секунд, и с начала раунда прошло не менее `round_start_threshold_seconds` секунд. | Guardian (все деревья) — 60 с нет активности и прошло 90 с → самоубийство. |
| `condition_is_airborne` | `negated`, `child` | Проверяет, что бот в воздухе. | Guardian blacksite_1 (парашют), county_2 (не атаковать в воздухе). |
| `condition_is_reloading` | `negated`, `child` | Проверяет, что активное оружие перезаряжается. | В предоставленных деревьях не встречается — требует тестирования. |
| `condition_is_weapon_suitable` | `input` (string), `weapon` (string), `negated`, `child` | Проверяет, подходит ли оружие для дистанции до `input`. | В предоставленных деревьях не встречается — требует тестирования. |
| `condition_barrier` | `input_domain` (string), `input_name` (string), `radius` (float), `negated`, `child` | — | В предоставленных деревьях не встречается — требует тестирования. |

---

## 8. Действия

Действия возвращают `running`, пока не завершатся, затем `success` или `failure`.

| Узел | Ключи | Что делает | Где встречается |
|------|-------|------------|-----------------|
| `action_aim` | `input` (position), `ready` (string) | Поворачивает бота к точке `input`. Когда прицел готов — устанавливает флаг `ready`. **Никогда не разскопится со снайперки** до смены оружия. `acquire_only` **игнорируется**. | `bt_attack` — `input = "Enemy" ready = "AimReady"`. |
| `action_attack` | `input` (string), `ready` (string), `output` (string) | Стреляет по `input`, когда `ready` истинно. Если `ready` не указан — стреляет сразу. `output` устанавливается в true, пока идёт атака. | `bt_attack` — `input = "Enemy" output = "Attacking" ready = "AimReady"`. |
| `action_move_to` | `destination` (position), `hiding_spot` (number или ключ), `threat` (string), `auto_look_adjust` (bool), `damaging_areas_penalty_cost` (float), `arrival_epsilon` (float), `additional_arrival_epsilon_2d` (float), `hiding_spot_check_distance_threshold` (float), `nearest_area_distance_threshold` (float), `movement_type` (enum), `route_type` (enum) | Ведёт бота к `destination` по навмешу. `movement_type` — `WALK` или `RUN`. `route_type` — `FASTEST_ROUTE` или `SAFEST_ROUTE`. `hiding_spot` — укрытие для проверки. `threat` — угроза для избегания. `arrival_epsilon` — насколько близко считать прибытием. `nearest_area_distance_threshold` — насколько близко к нав-зоне считать прибытием (20 = почти вплотную). `auto_look_adjust = 0` — не крутить головой при движении. | `bt_plant_bomb_if_owned` — `destination = "TargetArea" hiding_spot = "HidingSpot" threat = "NearestThreatMemory"`. |
| `action_pull_trigger` | — | Нажимает ЛКМ (стреляет). `ratio` **игнорируется** (в файлах 1). | `bt_plant_bomb_if_owned` — `repeat` вокруг (стреляет, пока не закончится закладка). |
| `action_use` | — | Нажимает `+use` (использовать, разминировать, подобрать). | Rush — нажать кнопку. `bt_defuse_bomb_if_nearby` — `repeat` вокруг. |
| `action_look_at` | `input_angles` (string) или `input_location` (position) | Поворачивает бота. Указывать **одно из двух**. `input_angles` — углы (как `setang`). `input_location` — позиция. | `bt_attack` — `input_location = "Enemy"`. Guardian — `input_location = "4363.661621 5653.200195 735.644348"`. |
| `action_wait` | `wait_time_min` (float), `wait_time_max` (float), `wait_modulo_tick_mod` (int), `wait_modulo_tick_add` (int) | Замораживает ИИ на случайное время между `min` и `max`. `wait_modulo_tick_mod`/`add` — для синхронизации по тикам (редко используется). | `bt_attack` — `0.2–0.5`. |
| `action_equip_weapon` | `weapon` (string, classname или `BEST`) | Выбирает оружие. `BEST` — лучшее доступное. | `bt_attack` — `BEST`. Guardian card13 — `weapon_shield`. |
| `action_equip_item` | `item` (string), `items_one_of` (string[]) | Выбирает предмет по classname. `items_one_of` — любой из списка. | `bt_heal_if_needed` — `weapon_healthshot`. `bt_plant_bomb_*` — `weapon_c4`. `bt_throw_utility_if_covered` — `weapon_hegrenade`, `weapon_incgrenade`. |
| `action_buy` | — | Случайная покупка. | Deathmatch/Practice — после `action_wait 3-3`. |
| `action_choose_bomb_site_area` | `input` (number), `output` (string) | Возвращает позицию бомбсайта по индексу. `input` может быть `0`/`1`, ключом blackboard (`BombSiteIndex`) или ConVar (`@mp_guardian_target_site`). | Deathmatch — `input = "BombSiteIndex" output = "HuntAreas"`. Guardian — `input = "@mp_guardian_target_site" output = "BombSiteArea"`. |
| `action_choose_random_waypoint` | `input` (string), `output` (string) | Выбирает случайный тайл навмеша из `input` (нав-зоны) и пишет позицию в `output`. | Deathmatch — `input = "HuntAreas" output = "TargetHuntArea"`. |
| `action_choose_team_spawn_area` | `output` (string), `team` (enum) | Возвращает нав-зону со спавнами команды. | Deathmatch — `output = "HuntAreas"`. |
| `action_choose_random_waypoint_within_radius` | `origin` (position), `radius` (number), `output` (string) | Выбирает случайную точку в радиусе `radius` вокруг `origin`. | Co-op roam — `LastCoopSpawnPointLocation 700`. Co-op rush — `RushCenterLocation RushCenterRadius`. |
| `action_combat_positioning` | `input` (string), `is_attacking` (string) | Уклонение/сближение. `is_attacking` — флаг, установленный `action_attack`. | `bt_attack` — `input = "Enemy" is_attacking = "Attacking"`. |
| `action_flee_area_damage` | `input` (string), `output` (string), `max_search_range` (float), `threat_min_keep_distance` (float) | Находит безопасную точку от урона по площади и пишет в `output`. | `bt_flee_area_damage_threats` — `500`. |
| `action_set_global_flag` | `name` (string), `expiration_time_min` (float), `expiration_time_max` (float) | Устанавливает глобальный флаг на случайное время между `min` и `max`. | `bt_plant_bomb_if_owned` — `'DontPlantBomb' 3–5`. |
| `action_set_global_counter` | `input_name` (string), `input_value` (int) | Устанавливает глобальный счётчик. | Guardian card7_2 — `'PreparingToThrowSmoke'=3`, `'ThrowPalaceFlash'=1`, `'PositioningForTopSmoke'=1`. |
| `action_compare_global_counter` | `input_name` (string), `input_value` (int) | Сравнивает глобальный счётчик с `input_value`. | Guardian card7_2 — `'WaitForCoordinatedBuy'=0`. Card13 — `'RushSite'=1`. |
| `action_set_value_float` | `key` (string), `value` (float) | Записывает float в blackboard. | Co-op `bt_determine_rush_center` — `'RushCenterRadius'`. |
| `action_set_value_vector` | `key` (string), `value` (position) | Записывает позицию в blackboard. | Co-op `bt_determine_rush_center` — `'RushCenterLocation'`. |
| `action_aim_projectile` | `input` (string), `output` (string) | Рассчитывает углы для броска гранаты. **Баг: не учитывает траекторию** — бот бросает неточно. | `bt_throw_utility_if_covered` — `input = "Enemy" output = "UtilityTarget"`. |
| `action_choose_guardian_bomb_plant_location` | `output` (string) | Выбирает позицию закладки бомбы (Guardian). | `bt_plant_bomb_*` — `output = "TargetArea"`. |
| `action_commit_suicide` | — | Убивает бота. | Guardian (все деревья) — если `condition_inactive`. |
| `action_jump` | — | Прыжок. | Co-op rush — если нет `RushCenterLocation`. |
| `action_parachute_positioning` | — | Движение по диагонали (для парашюта). | Guardian blacksite_1. |
| `action_say` | `phrase` (string), `high_priority` (bool) | Радио-команда. | Co-op guard/roam — `"Clear"`. |
| `action_select_areas_within_radius` | `input` (position), `radius` (number или string), `output` (string) | Выбирает нав-зоны в радиусе вокруг `input`. `radius` может быть числом, строкой-литералом (`"300"`) или ключом blackboard. | Guardian insertion2_a/b — `input = "Hostage" radius = "300" output = "HostageSurroundingAreas"`. |
| `action_entity_accept_input` | `name` (string), `input` (string) | Отправляет input энтити по targetname. Используется для нажатия кнопок. | Guardian county_2 — `name = "'AutoInstance7-_03creek_door_button'" input = "'Use'"`. |
| `action_coordinated_buy` | `id_no_purchase` (string), `purchases` ({`items[]`, `id`}[]), `team_filter` (enum), `save_threshold` (int) | Скоординированная покупка. **Баг: один бот заставляет всех покупать.** | Guardian card7_2 — через `bt_coordinated_buy` с `decorator_run_once`. |
| `action_custom_buy` | `item_aliases[]` (string[]) | Покупка по алиасам. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_hide` | `output` (string), `max_range` (float) | Случайное укрытие. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_teleport` | `destination` (position) | Телепорт. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_acquire_items` | `items[]` (string[]), `remove_all_items` (bool) | Выдаёт предметы. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_crouch` | — | Присесть. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_standup` | — | Встать. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_reload` | — | Перезарядка. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_drop_active_weapon` | — | Выбросить активное оружие. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_inspect_current_weapon` | — | Проиграть анимацию осмотра оружия. | В предоставленных деревьях не встречается — требует тестирования. |
| `action_secondary_attack` | — | Вторичная атака (ПКМ). | В предоставленных деревьях не встречается — требует тестирования. |

---

## 9. Модель выполнения

Каждый тик дерево обходится **дважды**:

1. **Check-pass** — сверху вниз. Решает, какие ветки имеют право выполняться. Если корень не проходит — тик «пустой» (ничего не выполняется).
2. **Run-pass** — выполняет выбранные ветки. Здесь бот двигается, целится, стреляет.

Поскольку check-pass повторяется каждый тик, условия перепроверяются постоянно. `selector` может прервать текущую ветку, если более приоритетная начала проходить проверку.

### 🧩 Поведение композитов

| Композит | Check-pass | Run-pass | Прерывание |
|----------|------------|----------|------------|
| `sequencer` | Только текущий ребёнок | По порядку до провала | Нет (условия позади не перепроверяются) |
| `selector` | С первого ребёнка каждый тик | Запускает первого прошедшего | Да — останавливает текущего при появлении более приоритетного |
| `parallel` | Все дети | Все дети каждый тик | Провал любого = стоп всех |

### 🔄 Пример разворачивания

```
sequencer[ move_to(bot_target), look_at(bot_target), wait(2..3) ] в repeat
```

1. Тик 1: `move_to` двигает на 1 тик → running.
2. Тики 2..N: игра идёт прямо в `move_to` — одно непрерывное движение.
3. Прибытие: `move_to` success → `look_at` success.
4. `wait` выбирает время окончания и возвращает running.
5. Все три готовы → sequencer success → repeat заново.

> 💡 **Важно:** узлы не двигают бота напрямую. `action_move_to` управляет pathfinding; `action_use`/`action_reload` «нажимают кнопку» на тик; `action_aim`/`action_attack` рулят встроенным aiming/shooting, пока не завершатся.

---

## 10. Blackboard, домены, память

**Blackboard** — хранилище ключей-значений бота. Сенсор пишет врагов в `output = "Enemy"`, действие атаки читает их через `input = "Enemy"`.

### 📝 Строки

| Тип | Пример | Примечание |
|-----|--------|------------|
| Литерал | `'CT'` | В одинарных кавычках; кавычки отбрасываются. |
| Ключ blackboard | `TeamName` | Должен содержать строку. |

### 🔢 Числа

| Тип | Пример | Примечание |
|-----|--------|------------|
| Литерал | `3`, `0.5` | — |
| ConVar | `@mp_plant_c4_anywhere` | — |
| Ключ blackboard | `RoundNumber` | Если содержит строку — разрешается ещё раз, но только на 1 уровень. |

### 📍 Позиции и имена сущностей

Ключи, ожидающие позицию (`destination`, `input_location`), принимают:

| Тип | Пример | Примечание |
|-----|--------|------------|
| Литерал | `"-1450 2230 2"` | — |
| Ключ blackboard | `TargetArea` | Позиция, строка или список sensed-сущностей (берётся первая). |
| Targetname энтити | `ant.button` | Если ни один ключ так не называется. Используется origin энтити. |

Это позволяет управлять ботами с карты: Rush-боты идут к `ant.button` по имени; скрипт может двигать target-энтити, чтобы перенаправлять ботов.

### 🌐 Домены

Домены — способ делить состояние между ботами. Кавычки вокруг имён обязательны.

| Домен | Где встречается | Назначение |
|-------|-----------------|------------|
| `AllBots` | `memory_to_expire`, `input_domain`, `output_domain`, `decorator_run_once` | Общая память всех ботов; блокировка для run_once |
| `'StratTokens'` | `decorator_token_service` | Токены стратегий (Practice, Guardian) |
| `'PressAntennaButton'` | `decorator_try_lock` | Блокировка нажатия кнопки (Rush) |
| `'DefuseBombIfNearby'` | `decorator_try_lock` | Блокировка разминирования (Practice) |
| `'PickupBombIfNearby'` | `decorator_try_lock` | Блокировка подбора бомбы |
| `'SetupGlobalCounters'` | `decorator_run_once` | Установка счётчиков (Guardian card7_2) |

### 💾 Память

- `decorator_sensor` → `output = "Vision"` (список сущностей).
- `decorator_picker_*` → сужают `"Vision"`.
- `decorator_memory` → копирует в `"ShortTermAttackMemory"`.
- `decorator_bot_service` в корне → истекает память: `{key, time, distance}`.

#### Стандартные имена памяти

| Ключ | Время | Дистанция | Что хранит |
|------|-------|-----------|------------|
| `ShortTermAttackMemory` | 0.7 | 0 | Врагов, виденных недавно |
| `LongTermMemory` | 10 | 500 | Врагов, виденных давно |
| `ShortTermInvestigateMemory` | 3 | 200 (2000 в card13) | События для исследования |
| `ShortTermDamageMemory` | 0.7 | 0 | Источники урона |
| `ShortTermAreaDamageMemory` | 3 | 0 | Урон по площади (гранаты, инферно) |
| `DamageThroughSmokeMemory` | 3 | 0 | Источники урона за дымом |
| `Threats` (AllBots) | 10 | 0 | Глобальные угрозы |

#### Другие ключи blackboard

`Vision`, `Hearing`, `Damage`, `Grenades`, `Infernos`, `AreaDamage`, `FleeSpot`, `TargetArea`, `TargetHuntArea`, `HuntAreas`, `BombSiteArea`, `BombSiteIndex`, `RandomRoamPoint`, `RushPoint`, `RushCenterLocation`, `RushCenterRadius`, `UtilityTarget`, `ApproachPoint`, `HidingSpot`, `ShouldBuy`, `NearestThreat`, `NearestThreatMemory`, `Event`, `VisibleShortTermInvestigateMemory`, `ShortTermInvestigateMemoryWithinReactionTime`, `Bombs`, `PlantedBomb`, `Hostage`, `GroupId`, `GroupMemberId`, `EngagedEntities`, `ThreatsWithinFOV`, `RushSite`, `DontPlantBomb`, `PreparingToThrowSmoke`, `ThrowPalaceFlash`, `PositioningForTopSmoke`, `WaitForCoordinatedBuy`.

### 🚩 Глобальные флаги и счётчики

| Узел | Что делает |
|------|------------|
| `action_set_global_flag` | Устанавливает именованный флаг на случайное время между `expiration_time_min` и `expiration_time_max`. Проверяется через `condition_is_empty` с `global = 1`. |
| `action_set_global_counter` | Устанавливает именованный счётчик. |
| `decorator_dec_global_counter` | Уменьшает счётчик. |
| `action_compare_global_counter` | Сравнивает счётчик с `input_value`. |

### 📦 Параметры subtree

`subtree` может записывать значения в blackboard до старта. Только **int или string** — float/bool отвергаются с ошибкой `[AI BT]: unsupported type for subtree parameter.`

---

## 11. Зарезервированные переменные

Игра сама кладёт их в blackboard. Читать можно, создавать нельзя.

| Переменная | Тип | Откуда | Где используется |
|------------|-----|--------|------------------|
| `NumberOfTerrorists` | int | Движок | `condition_is_equal` в `bt_plant_bomb_if_owned` |
| `NumberOfAliveTerrorists` | int | Движок | `condition_is_equal` в `bt_plant_bomb_if_owned` |
| `NumberOfCounterTerrorists` | int | Движок | — |
| `NumberOfAliveCounterTerrorists` | int | Движок | — |
| `AccountBalance` | int | Движок | — |
| `AmmoCount/<weapon>` | int | Движок | — |
| `AmmoCount/current` | int | Движок | — |
| `TeamName` | string | Движок | `condition_is_equal source="TeamName" destination="'T'"` в Practice |
| `RoundNumber` | int | Движок | Все `bt_t_*.kv3` и `bt_ct_*.kv3` в Practice |
| `BlindnessPercentage` | float | Движок | — |
| `GuardianWaveNumber` | int | Движок (Guardian) | `condition_is_equal` в Guardian county_2 |
| `LastCoopSpawnPointName` | string | Движок (Co-op) | `condition_is_equal` в `bt_determine_rush_center` |
| `LastCoopSpawnPointLocation` | Vector | Движок (Co-op) | `action_move_to` в Co-op guard/roam, `action_choose_random_waypoint_within_radius` в roam |

> 📌 **Примечание:** `TeamName` и `RoundNumber` отсутствуют в списке Source2 Wiki, но активно используются в Practice. Список на вики неполный.

---

## 12. Отладка

| Команда | Назначение | Требования |
|---------|------------|------------|
| `mp_bot_ai_bt "путь"` | Назначить дерево всем ботам | — |
| `mp_bot_ai_bt_clear_cache` | Сбросить кэш после правок | — |
| `cv_bot_ai_bt_debug_target <index>` | Нарисовать дерево бота | `sv_cheats 1`, локальный/приватный сервер |
| `cv_bot_ai_bt_hiding_spot_show` | Показать укрытия | `sv_cheats 1` |
| `cv_bot_ai_bt_moveto_show_next_hiding_spot` | Показать следующее проверяемое укрытие | `sv_cheats 1` |
| `developer 1` | Писать `[AI BT]: Loaded behavior tree '<file>'` при спавне | — |

### 💬 Сообщения

| Сообщение | Причина |
|-----------|---------|
| `[AI BT]: Loaded behavior tree '<file>'` | При `developer 1` |
| `[AI BT] warnings` | Всегда |
| `Blackboard entry 'X' does not exist nor it is a valid entity name` | Ключ не найден |
| `[AI BT]: Config node 'attack' must contain 13 entries` | Неверное число строк |
| `[AI BT]: unsupported type for subtree parameter` | Float/bool в `params` |

---

## 13. CS:GO vs CS2

| Аспект | CS:GO | CS2 |
|--------|-------|-----|
| Уровни скилла | 8 (без `elite`) | 9 (с `elite`) |
| Строк в `attack` | 16 (в старых); в предоставленном файле 20 | 13 для Deathmatch/Rush/Practice, 20 для Co-op/Guardian |
| `aim_ready_angle_tolerance` | Не документирован | Есть в конфиге |
| `TeamName` / `RoundNumber` | Не документированы | Используются в Practice |
| Документация узлов | Неточная, много TODO | Точный список читаемых ключей |
| Узлы вне официальных деревьев | Не отмечены | 36 отмечены, но часть используется в Guardian/Co-op |
| Игнорируемые параметры | Не отмечены | `acquire_only`, `ratio`, `distance_threshold` |
| Модель выполнения | Не описана | Два прохода, детально |
| Subtree params | Не описаны | Только int/string |
| `decorator_picker_*` | Без доменов | С `input_domain`/`output_domain` |
| `decorator_sensor` | Без `append_items` | С `append_items` |
| `action_wait` | 2 ключа | + `wait_modulo_tick_mod`/`add` |

---

## 14. Готовые модули

### 💀 Deathmatch / Rush / Practice

| Модуль | Что делает |
|--------|------------|
| `bt_attack` | Берёт `ShortTermAttackMemory` → reaction time → сортирует по дистанции → берёт ближайшего → если есть враг: помечает его в `EngagedEntities` (SET, 1 с) → параллельно: экипирует `BEST`, повторяет `succeed[out_of_ammo → equip BEST]` (смена оружия при пустом магазине), selector[flee_area_damage, если нож — бежит к врагу, иначе combat_positioning], повторяет `aim Enemy ready="AimReady"`, повторяет `attack Enemy ready="AimReady" output="Attacking"`. |
| `bt_memorize_enemies_vision` | Сенсор FOV, PLAYERS, ENEMY → nearby 1500 → visible → reaction time → `ShortTermAttackMemory` + `LongTermMemory`. |
| `bt_memorize_noises` | Сенсор сфера 3000, NOISE → weight_as_distance → random_by_distance (800–3000) → dedup (100 к `LongTermMemory`) → reaction time → `LongTermMemory`. |
| `bt_memorize_damage` | Сенсор DAMAGE → reaction time → `ShortTermDamageMemory`; blocked_by_smoke (3000, игнор) → `DamageThroughSmokeMemory`. |
| `bt_memorize_area_damage_grenades` | Сенсор GRENADE → grenade_type (FIRE, EXPLOSIVE) → nearby 2500 → visible → reaction time → `ShortTermAreaDamageMemory`. |
| `bt_memorize_area_damage_infernos` | Сенсор CLASSNAME `inferno` → nearby 1000 → reaction time → `ShortTermAreaDamageMemory`. |
| `bt_memorize_area_damage_current` | Сенсор AREA_DAMAGE → `ShortTermAreaDamageMemory`. |
| `bt_memorize_nearest_investigation` | `succeed` → `condition_is_empty ShortTermInvestigateMemory` → если пусто: `memory LongTermMemory → ShortTermInvestigateMemory` → ranker_dist → max_score. Всегда успех, но если память была пуста — заполняет ближайшим событием. |
| `bt_investigate_closest_memorized_event` | Берёт `ShortTermInvestigateMemory` → reaction time → если есть событие: экипирует `BEST`, если видимо — смотрит и ждёт 0.2–0.5 с, затем идёт к событию (`move_to threat="NearestThreatMemory"`). |
| `bt_heal_if_needed` | Если HP < 60: экипирует `weapon_healthshot`, ждёт 1 с, `action_pull_trigger` (ratio игнор). |
| `bt_face_damage_inflictor` | Берёт `ShortTermDamageMemory` → reaction time → если есть: целится (`acquire_only` игнор), параллельно `succeed` → blocked_by_smoke (3000, игнор) → если есть: повторяет `action_attack output="Attacking"` (без input, стреляет по текущему прицелу). |
| `bt_attack_damage_inflictor_smoke` | Берёт `DamageThroughSmokeMemory` → reaction time → сортирует → берёт ближайшего → целится, повторяет `action_attack input="Damage" output="Attacking"`. |
| `bt_flee_area_damage_threats` | Если `ShortTermAreaDamageMemory` не пусто: `action_flee_area_damage` (500) → бежит к `FleeSpot` (`auto_look_adjust=0`). |
| `bt_look_around` | `random_approach_point` → `action_aim` (acquire_only) → ждёт 0.2–0.5 с. |
| `bt_plant_bomb_if_covered` | Если есть `weapon_c4` и в `ShortTermAttackMemory` ≥2 врага: выбирает бомбсайт, идёт, проверяет `@mp_plant_c4_anywhere` или `is_at_bomb_site`, экипирует C4, ждёт 0.5 с, повторяет `action_pull_trigger`. |
| `bt_plant_bomb_if_owned` | Если есть `weapon_c4` и нет флага `DontPlantBomb`: экипирует `BEST`, выбирает бомбсайт, идёт, проверяет site, экипирует C4, ждёт 1 с, параллельно: с шансом 0.5 и если `NumberOfAliveTerrorists=1` ставит флаг `DontPlantBomb` (3–5 с), и повторяет `action_pull_trigger`. Если флаг `DontPlantBomb` — смотрит по сторонам. |
| `bt_pickup_bomb_if_nearby` | Сенсор сфера 3000, CLASSNAME `weapon_c4`, orphan_only → сортирует → берёт ближайшее → `try_lock 'PickupBombIfNearby'` → `action_move_to` (nearest_area_distance_threshold=20). |
| `bt_defuse_bomb_if_nearby` | Сенсор сфера 3000, CLASSNAME `planted_c4`, orphan_only → сортирует → `try_lock 'DefuseBombIfNearby'` → идёт, смотрит, ждёт 0.25 с, повторяет `action_use`. |
| `bt_pickup_hostage` | Сенсор CLASSNAME `hostage_entity`, orphan_only → идёт (arrival_epsilon=35), смотрит, ждёт 1 с, повторяет `action_use`. |
| `bt_pickup_hostage_if_covered` | Берёт `ShortTermAttackMemory` → reaction time → сортирует → если ≥2 врага в `EngagedEntities` (CLEAR, amount=2, AT_LEAST) → `bt_pickup_hostage`. |
| `bt_clear_threats_within_fov` | `succeed negated=1` → если `ShortTermAttackMemory` пусто: берёт `Threats` (AllBots) → reaction time → visible → удаляет из `Threats`. Side-effect: очищает угрозы в FOV. Всегда failure, чтобы не останавливать selector. |
| `bt_select_nearby_threat` | `selector[succeed negated=1 → remove_key NearestThreatMemory; succeed negated=1 → Threats (AllBots) → reaction time → visible (check_fov=0) → ranker_dist → max_score → если есть: memory → NearestThreatMemory]`. Всегда failure, но обновляет `NearestThreatMemory`. |
| `bt_memorize_threats` | Сенсор DAMAGE → reaction time → `Threats` (AllBots). |

### 🤝 Co-op

| Модуль | Что делает |
|--------|------------|
| `bt_basic_player_engagement` | `selector[throw_utility_if_covered, attack, face_damage_inflictor, parallel[flee_area_damage_threats, repeat look_around], attack_damage_inflictor_smoke, clear_threats_within_fov, select_nearby_threat, heal_if_needed]`. |
| `bt_basic_player_memorization` | `parallel[memorize_enemies_vision, memorize_noises, memorize_nearest_investigation, memorize_area_damage_grenades, memorize_area_damage_infernos, memorize_damage, memorize_threats, memorize_area_damage_current]`. |
| `bt_determine_rush_center` | `succeed negated=1` → `selector[condition_is_equal LastCoopSpawnPointName='enemy.waveXX' → parallel[set_value_vector RushCenterLocation, set_value_float RushCenterRadius]]` для каждого wave. Всегда failure, но устанавливает центр раша. |
| `bt_throw_utility_if_covered` | `sequencer[preconditions (reaction time, ≥2 врага в EngagedEntities, есть hegrenade/incgrenade), maybe 0.15 → sequencer[aim_projectile (баг), equip_item, wait 0.5, look_at, pull_trigger, wait 0.5]]`. |
| `bt_guard` | `parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement, Investigate (look at), Investigate (distance_less 450–550), go back to spawn + look around + maybe say "Clear" (0.7) + wait 4–8]]`. |
| `bt_roam` | `parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement, Investigate, roam (choose_random_waypoint_within_radius LastCoopSpawnPointLocation 700, move_to, maybe say "Clear" (0.7), wait 1–3)]]`. |
| `bt_rush` | `parallel[repeat BasicPlayerMemorization, repeat selector[DetermineRushCenter, condition_is_empty RushCenterLocation → action_jump, BasicPlayerEngagement, Investigate, Rush (choose_random_waypoint_within_radius RushCenterLocation RushCenterRadius, move_to)]]`. |
| `bt_sniper` | `parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement]]`. |

### 🛡️ Guardian

| Модуль | Что делает |
|--------|------------|
| `bt_item_strategy` | Покупка/выдача предметов по токенам стратегий. |
| `bt_takeover_strategy` | Захват позиции по токенам стратегий. |
| `bt_attack_with_shield` | Атака со щитом (аналог `bt_attack`, но с `weapon_shield`). |
| `bt_investigate_with_shield` | Исследование со щитом. |
| `bt_coordinated_buy` | Скоординированная покупка (с багом). |
| `bt_strategy_smoke`, `bt_strategy_grenades` | Стратегии дымов/гранат (card13). |

---

## 15. Паттерны

### 🔄 Восприятие + решение через `parallel`

```
parallel
├── repeat → parallel (sensors, memory)
└── repeat → selector (priority list)
```

### 🎯 Selector как список приоритетов

```
selector
├── condition_is_empty ShouldBuy (negated) → sequencer[wait, buy, remove_key]
├── sensor DAMAGE → aim
├── subtree Attack
├── subtree HealIfNeeded
└── subtree Hunt
```

### 🔒 Блокировка действия одним ботом

```
decorator_try_lock domain = "'PressAntennaButton'"
└── sequencer[ move_to ant.button, look_at, wait, use ]
```

### 🎭 Side-effect без остановки selector

```
decorator_succeed negated = 1
└── sequencer[ ... сделать что-то ... ]
```

Selector увидит failure и пойдёт дальше, но side-effect уже выполнен.

### 🧹 Истечение памяти в корне

```
decorator_bot_service
  memory_to_expire =
  [
    { key = "ShortTermAttackMemory"  time = 0.7  distance = 0 }
    { key = "LongTermMemory"         time = 10   distance = 500 }
    { key = "ShortTermInvestigateMemory" time = 3 distance = 200 }
  ]
```

---

## 16. Подводные камни

| ❌ Не делайте | ✅ Делайте вместо этого |
|---------------|------------------------|
| Полагаться на `acquire_only`, `ratio`, `distance_threshold` | Эти параметры игнорируются |
| Использовать `action_coordinated_buy` без `decorator_run_once` | `decorator_run_once` + `action_buy` |
| Забывать `mp_bot_ai_bt_clear_cache` | Всегда чистить кэш после правок |
| Передавать float/bool в subtree params | Только int или string |
| Использовать 20 строк в `attack` для CS2 Deathmatch | Ровно 13, в фиксированном порядке |
| Ожидать, что `action_aim` разскопится | Смените оружие или экипируйте то же заново |
| Полагаться на `condition_is_greater`/`is_less` | Используйте `greater_equal`/`less_equal` |
| Использовать `decorator_invert` в CS2 | Используйте `negated = 1` |
| Забывать, что `action_attack` без `ready` стреляет сразу | Указывайте `ready`, если нужна синхронизация с `action_aim` |
| Ожидать, что `action_aim_projectile` точно бросит гранату | Баг: не учитывает траекторию |
| Использовать `decorator_picker_blocked_by_smoke` без понимания | Он оставляет **только** сущности за дымом |

---

## 📚 Источники

- [Source2 Wiki: Bot behavior trees](https://www.source2.wiki/Scripting/Counter-Strike%202/bot-behavior-trees)
- [Valve Developer Community: CS:GO Bot Behavior Trees](https://developer.valvesoftware.com/wiki/Counter-Strike:_Global_Offensive/Bot_Behavior_Trees)
- Предоставленные KV3-файлы: конфиги (13 и 20 строк), деревья Deathmatch/Rush/Practice/Co-op/Guardian, все модули.
