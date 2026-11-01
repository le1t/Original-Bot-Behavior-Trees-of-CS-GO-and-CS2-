# 🤖 Behavior Trees для ботов в CS:GO и CS2

## 🏷️ Легенда статусов

| Значок | Смысл |
| --- | --- |
| ✅ | Подтверждено дампами и/или документацией |
| 📄 | Описано в документации (Source2 Wiki / VDC), но в дампах не встречается |
| 🔬 | Вывод из анализа дампов; официально не документировано |
| ⚠️ | Не подтверждено / под вопросом (в т.ч. пометки [confirm] в VDC) |
| ❌ | Ключ игнорируется движком (мёртвый) |

## 📋 Содержание

> Ссылки работают на GitHub: якоря формируются из заголовков автоматически (строчные буквы, пробелы → дефисы, знаки препинания и эмодзи отбрасываются). Если переименуете заголовок — обновите якорь.

**[Часть I. Основы](#часть-i-основы)**

1. [Что такое Behavior Tree](#1-что-такое-behavior-tree)
2. [Быстрый старт](#2-быстрый-старт)
3. [Формат KV3](#3-формат-kv3)
4. [Конфиг бота](#4-конфиг-бота)

**[Часть II. Узлы](#часть-ii-узлы)**

5. [Композиты](#5-композиты)
6. [Декораторы](#6-декораторы)
7. [Условия](#7-условия)
8. [Действия](#8-действия)

**[Часть III. Выполнение](#часть-iii-выполнение)**

9. [Модель выполнения](#9-модель-выполнения)
10. [Blackboard, домены, память](#10-blackboard-домены-память)
11. [Зарезервированные переменные](#11-зарезервированные-переменные)

**[Часть IV. Практика](#часть-iv-практика)**

12. [Отладка](#12-отладка)
13. [CS:GO vs CS2](#13-csgo-vs-cs2)
14. [Готовые модули](#14-готовые-модули)
15. [Паттерны](#15-паттерны)
16. [Подводные камни](#16-подводные-камни)

**[Часть V. Справочники](#часть-v-справочники)**

17. [Скриптовые установки гранат](#17-скриптовые-установки-гранат)
18. [Координация через глобальные счётчики](#18-координация-через-глобальные-счётчики)
19. [Паттерны-предохранители](#19-паттерны-предохранители)
20. [Парашют (Blacksite, CS:GO Guardian)](#20-парашют-blacksite-csgo-guardian)
21. [Карты Guardian (CS:GO-дампы)](#21-карты-guardian-csgo-дампы)
22. [Practice: полная таблица раундов](#22-practice-полная-таблица-раундов)
23. [Полный список доменов](#23-полный-список-доменов)
24. [Быстрый справочник: aim_projectile vs throw_utility](#24-быстрый-справочник-aim_projectile-vs-throw_utility)

**[Часть VI. Приложения](#часть-vi-приложения)**

25. [Инвентарь узлов и частоты](#25-инвентарь-узлов-и-частоты) — [25.1 объёмы](#251-объёмы-дампов) · [25.2 композиты](#252-композиты-и-subtree) · [25.3 декораторы](#253-декораторы) · [25.4 условия](#254-условия) · [25.5 действия](#255-действия) · [25.6 особые счётчики](#256-особые-счётчики)
26. [Мёртвые ключи и неиспользуемые узлы](#26-мёртвые-ключи-и-неиспользуемые-узлы) — [26.1 мёртвые ключи](#261-мёртвые-ключи-есть-в-файлахдокументации-но-движком-не-читаются) · [26.2 нулевые узлы](#262-узлы-не-встречающиеся-ни-разу-0-в-544--57-файлах) · [26.3 «unused» по Source2 Wiki](#263-unused-по-source2-wiki-но-реально-используются-в-guardianco-op)
27. [Анти-паттерны](#27-анти-паттерны)
28. [Известные баги движка](#28-известные-баги-движка)

---

# Часть I. Основы

## 1. Что такое Behavior Tree

**1.1.** Behavior Tree (BT) — KV3-файл, описывающий поведение бота как дерево узлов. Игра обходит дерево каждый тик: проверяет условия, выполняет действия. Бот не «думает» — он следует дереву. Если mp_bot_ai_bt пуст — боты используют классический встроенный ИИ (hardcoded classic bot AI), без дерева.

**1.2.** Где лежат файлы:

| Что | Где |
| --- | --- |
| Деревья и модули | scripts/ai/... внутри game/csgo/pak01_dir.vpk, извлекаются Source2 Viewer |
| Своё дерево | scripts/ai/mymode/bt_default.kv3 |
| Конфиг | scripts/ai/<mode>/bt_config.kv3 |
| Per-spawn дерево (CS:GO Co-op Strike) | ключ behavior_tree_file на info_enemy_terrorist_spawn — назначает дерево конкретной спавн-энтити |

**1.2.1.** Инвентарь дампов (что где лежит):

| Игра | Содержимое |
| --- | --- |
| CS:GO (544 файла) | guardian — 519 (51 корень + модули op09/op10/op11), coop/autumn — 9, modules — 14, deathmatch — 2 |
| CS2 (57 файлов) | armsrace — 2 (bt_config + bt_default), deathmatch — 2, modules — 14, **practice — 37**, rush — 2 |

CS2 practice = ровно 37 файлов: bt_config, bt_default, newplayer_dust2_01 + 10 общих practice-модулей + 24 newplayer-модуля (2+2+14+37+2 = 57; «плюс конфиги» уже включены в эти числа).

> ⚠️ CS2 All-in-One-дамп (57 файлов) **неполный**: в нём нет Guardian/Co-op-контента (action_throw_utility, парашюта, большинства op11_-стратегий и т.д.). Отсутствие узла в CS2-дампе **не означает** отсутствия узла в CS2 — такие узлы подтверждены только CS:GO-дампами.

Все 51 guardian-корень ссылаются на scripts/ai/guardian/bt_config.kv3; coop — на scripts/ai/coop/bt_config.kv3. Полное дерево папок — в §29.

**1.3.** В предоставленных дампах файлы названы .txt, но в игре используются .kv3. Внутри — KV3-разметка. Как читать «склеенные» All-in-One-дампы — см. §29.

**1.4.** Как включить:

```
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
mp_restartgame 1
```

**1.5.** С developer 1 боты при спавне пишут:

```
[AI BT]: Loaded behavior tree '<file>'
```

Ошибки — всегда с префиксом [AI BT].

**1.6.** Официальные режимы:

| Режим | Игра | Дерево | Строк в attack |
| --- | --- | --- | --- |
| Deathmatch / DM Freeforall | CS2 и CS:GO | scripts/ai/deathmatch/bt_default.kv3 | 13 (CS2) / 20 (CS:GO) |
| Arms Race | только CS2 | scripts/ai/armsrace/bt_default.kv3 | 13 |
| Rush | только CS2 | scripts/ai/rush/bt_default.kv3 | 13 |
| New User Training (Practice) | только CS2 | scripts/ai/practice/newplayer_dust2_01.kv3 | 13 (отдельный конфиг practice/bt_config.kv3) |
| Co-op (Guard/Roam/Rush/Sniper) | только CS:GO | scripts/ai/coop/bt_*.kv3 | 20 |
| Guardian | только CS:GO | scripts/ai/guardian/bt_op*.kv3 | 20 |

> **Important**
>
> CS2: все официальные конфиги (armsrace, deathmatch, practice, rush) байт-идентичны (совпадает md5) и содержат ровно 13 строк в attack. Guardian/Co-op-деревьев и конфигов в CS2-дампе нет; если такие режимы есть в игре, их параметры дампами не подтверждены. 20-строчный конфиг в CS2 просто не загрузится:
>
> `[AI BT]: Config node 'attack' must contain 13 entries`
>
> CS:GO: во всех трёх конфигах (coop, deathmatch, guardian) — 20 строк. В CS:GO BT использовались только в Co-op Strike, Guardian и Deathmatch (Arms Race, Rush и Practice — чисто CS2-режимы, отдельных CS:GO-деревьев для них в дампах нет).

Детали, подтверждённые дампами:

- ⚠️ **armsrace/bt_default.kv3 ссылается на scripts/ai/deathmatch/bt_config.kv3**, а не на собственный конфиг (хотя сам armsrace-конфиг существует и байт-идентичен остальным).
- ⚠️ В practice **два дерева**: newplayer_dust2_01.kv3 (тренировочное — назначается gamemode_new_user_training.cfg) и **bt_default.kv3** — почти точная копия CS2 Deathmatch-дерева без hostage-сенсора, с живой buy-веткой (см. §14.4.2).
- ⚠️ У Practice отдельный конфиг scripts/ai/practice/bt_config.kv3 — в CS2 он байт-идентичен остальным трём CS2-конфигам.
- ⚠️ CS:GO-конфиги не идентичны между собой: coop и deathmatch различаются только look_around_* (для low: 0.65/0.1/0.5 против 0.9/0.4/0.6), а guardian — отдельный «замедленный» конфиг: в default отличается только строка RIFLE, в low/fair/normal — почти всеми aim/look/reaction-ключами, в tough…elite — только reaction_time и строкой RIFLE (см. §4.6.4).

**1.6.1.** Официальные режимы выставляют mp_bot_ai_bt в своих конфигах режимов:

| Конфиг | Дерево |
| --- | --- |
| gamemode_deathmatch.cfg, gamemode_dm_freeforall.cfg | scripts/ai/deathmatch/bt_default.kv3 |
| gamemode_armsrace.cfg | scripts/ai/armsrace/bt_default.kv3 |
| gamemode_rush.cfg | scripts/ai/rush/bt_default.kv3 |
| gamemode_new_user_training.cfg | scripts/ai/practice/newplayer_dust2_01.kv3 |

Выставить mp_bot_ai_bt можно из cfg-файла или из cs_script через Instance.ServerCommand.

## 2. Быстрый старт

**2.1.** Создайте scripts/ai/mymode/bt_default.kv3:

```
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

**2.2.** Что делает дерево:

| Узел | Действие |
| --- | --- |
| decorator_repeat | Бесконечный цикл |
| sequencer | Выполняет детей по порядку |
| action_move_to | Бот бежит к bot_target. Если bot_target нет в blackboard, игра попробует найти энтити с таким targetname и взять её origin |
| action_look_at | Смотрит на bot_target |
| action_wait | Ждёт 2–3 секунды |

**2.3.** Как проверить:

```
mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3"
mp_bot_ai_bt_clear_cache
mp_restartgame 1
```

Поставьте на карту info_target с targetname bot_target. Бот будет бежать к ней.

> **Tip**
>
> В practice/newplayer_dust2_01.kv3 Valve оставила готовую строку-комментарий: `// mp_bot_ai_bt "scripts/ai/practice/newplayer_dust2_01.kv3"`. Удобно копировать как шаблон запуска.

## 3. Формат KV3

**3.1.** Каждый файл начинается с обязательного заголовка:

```
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
```

**3.2.** Далее — объект с двумя ключами:

| Ключ | Обязателен | Что означает |
| --- | --- | --- |
| config | ✅ для корневого дерева | Путь к конфигу |
| root | ✅ | Корневой узел |

**3.3.** Узел — таблица с ключом type. Декораторы и условия имеют одного ребёнка child. Композиты — массив children. Subtree-файлы содержат один узел верхнего уровня без config и root.

**3.4.** Синтаксис:

| Элемент | Пример | Примечание |
| --- | --- | --- |
| Комментарии | // текст | Однострочные; поддерживаются и многострочные /* ... */ (по VDC) |
| Ключи | type, input, destination | Регистронезависимы: Name = name встречается |
| Строки | "scripts/ai/..." | В файлах Valve часто domain = "'PressAntennaButton'" — двойная строка с одинарными кавычками внутри |
| Числа | 3, 0.5 | — |
| Массивы | [ ... ] | — |
| Объекты | { ... } | — |

> **Warning**
>
> Не «исправляйте» domain = "'PressAntennaButton'" на domain = "PressAntennaButton" без проверки. В предоставленных файлах для decorator_try_lock, decorator_token_service, decorator_run_once, action_set_global_counter и т.п. используются именно строки с внутренними одинарными кавычками. При этом decorator_hiding_spot_service domain = "AllBots" и memory_to_expire { domain = "AllBots" } — без внутренних кавычек. Единственное исключение из правила — decorator_run_once domain = "'AllBots'" (с кавычками!) в op11_county_2, волна 1, перед action_entity_accept_input для дверей: так только один бот нажимает все кнопки дверей Creek, а не все пять одновременно.

> **Warning**
>
> ⚠️ Движок молча игнорирует любые параметры, не принадлежащие узлу, — без ошибок и предупреждений. Реально читаются только специфичные для данного type ключи. Из-за этого «правдоподобные» имена (source / input / input_location — действителен только один из них) и опечатки почти неуловимы при тестировании — сверяйте ключи с таблицами §6–8. (Подтверждено VDC.)

**3.5.** Пример subtree-файла:

```
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
{
    type = "decorator_picker_reaction_time"
    input = "ShortTermAttackMemory"
    output = "Enemy"
    child = { ... }
}
```

**3.6.** Сводная таблица значений enum (по Source2 Wiki):

| Ключ | Значения |
| --- | --- |
| entity_type_filter | ALL, PLAYERS, HUMAN_PLAYERS, NOISE, DAMAGE, AREA_DAMAGE, CLASSNAME, GRENADE |
| team_filter, attacker_filter | ANY, CT, TERRORIST, SAME, OPPOSITE, ENEMY |
| team | ANY, CT, TERRORIST, OPPOSITE |
| team_restriction | ANY, CT, T |
| shape type | sensor_shape_fov, sensor_shape_sphere (с radius) |
| movement_type | BT_ACTION_MOVETO_WALK, BT_ACTION_MOVETO_RUN |
| route_type | BT_ACTION_MOVETO_FASTEST_ROUTE, BT_ACTION_MOVETO_SAFEST_ROUTE |
| slot | RIFLE, PISTOL, KNIFE, GRENADES, C4 |
| types (гранаты) | **CS2-enum: FLASH, FIRE, EXPLOSIVE, DECOY, SMOKE.** SENSOR и SNOWBALL встречаются только в старой CS:GO-вике — в CS2-дампах и документации их нет, не используйте их в CS2 |
| operation_type | BT_DECORATOR_TAG_ENTITY_SET, BT_DECORATOR_TAG_ENTITY_CLEAR |
| check_type | BT_DECORATOR_TAG_THRESHOLD_AT_LEAST, BT_DECORATOR_TAG_THRESHOLD_AT_MOST |
| weapon | BEST или classname (например, weapon_knife) |

## 4. Конфиг бота

**4.1.** Конфиг задаёт параметры действий: скорость прицеливания, реакцию, стрельбу. Он не управляет логикой — только «мышечной памятью».

**4.2.** Уровни скилла

| Версия | Уровни |
| --- | --- |
| CS:GO (дампы coop/deathmatch/guardian) | default, low, fair, normal, tough, hard, very_hard, expert, elite (9) |
| CS2 | default, low, fair, normal, tough, hard, very_hard, expert, elite (9) |

**4.2.1.** Каждый уровень наследует недостающие ключи из default. В предоставленных файлах есть все уровни. Утверждение из старых версий вики «в CS:GO 8 уровней без elite» для этих дампов неверно: elite присутствует во всех трёх конфигах CS:GO.

**4.3.** Параметры прицеливания

| Ключ | Тип | Описание |
| --- | --- | --- |
| aim_target_acquisition_lerp_time | float | Время «доезда» прицела до новой цели. Больше = медленнее наводится |
| aim_target_acquisition_lerp_time_deviation | float | Случайный разброс времени |
| aim_target_acquisition_angle_penalty | float | Штраф за большой угол до цели |
| aim_target_acquisition_angle_penalty_deviation | float | Разброс штрафа |
| aim_target_acquisition_angle_penalty_reduction_ratio | float | Насколько штраф уменьшается со временем |
| aim_target_acquisition_angle_tolerance | float | Допуск угла при захвате |
| aim_target_acquisition_angle_lerp_bias | float | Сглаживание угла захвата |
| aim_target_tracking_lerp_time | float | Скорость сопровождения цели |
| aim_target_tracking_lerp_time_deviation | float | Разброс сопровождения |
| aim_target_tracking_focus_interval | float | Интервал «фокуса» при трекинге |
| aim_target_tracking_focus_interval_deviation | float | Разброс интервала |
| aim_target_tracking_angle_lerp_bias | float | Сглаживание трекинга |
| aim_ready_angle_tolerance | float | Угол, при котором action_attack считает прицел готовым |
| aim_new_target_angle_tolerance | float | Угол для переключения на новую цель |
| aim_max_duration | float | Максимальное время прицеливания |
| aim_max_duration_deviation | float | Разброс макс. времени |
| aim_punch_angle_reaction_chance | float | Шанс реакции на отдачу |

> **Note**
>
> aim_ready_angle_tolerance отсутствует в старой CS:GO-вики, но есть во всех CS:GO-конфиг-файлах: 3.0 в coop/deathmatch; в guardian по уровням: default 3.0, low 5.0, fair 4.5, normal 4.5, tough и выше — 3.0. Это не «CS2-новинка», а просто пробел старой документации.

**4.4.** Параметры осмотра

| Ключ | Описание |
| --- | --- |
| look_around_awareness_yaw_range | Макс. угол поворота по горизонтали |
| look_around_awareness_pitch_range | Макс. угол по вертикали |
| look_around_focus_interval | Пауза между поворотами камеры |
| look_around_focus_interval_deviation | Разброс паузы |
| look_around_lerp_time | Время поворота камеры |
| look_around_lerp_time_deviation | Разброс времени |
| look_around_lerp_bias | Сглаживание движения камеры |

**4.5.** Реакция и бой

| Ключ | Описание |
| --- | --- |
| reaction_time | Задержка перед реакцией на событие |
| combat_crouch_chance | Шанс присесть в бою |
| combat_dodge_command_duration | Длительность команды уклонения |
| combat_dodge_command_duration_deviation | Разброс длительности |

### 4.6. Таблица attack

**4.6.1.** Каждая строка — 5 чисел: burst duration, duration deviation, cadence, cooldown, cooldown deviation.

| Поле | Что означает |
| --- | --- |
| burst duration | Длительность одной очереди |
| duration deviation | Разброс длительности |
| cadence | Пауза между выстрелами внутри очереди; 0 = без паузы |
| cooldown | Пауза между очередями |
| cooldown deviation | Разброс паузы |

**4.6.2.** 13 строк — все конфиги CS2 (Deathmatch / Rush / Practice / Arms Race; все четыре байт-идентичны). Строгий порядок:

| # | Тип |
| --- | --- |
| 1 | KNIFE |
| 2 | PISTOL |
| 3 | SUBMACHINEGUN |
| 4 | RIFLE |
| 5 | SHOTGUN |
| 6 | SNIPER_RIFLE |
| 7 | MACHINEGUN |
| 8 | C4 |
| 9 | TASER |
| 10 | GRENADE |
| 11 | EQUIPMENT |
| 12 | STACKABLEITEM |
| 13 | UNKNOWN |

**4.6.3.** 20 строк — все три конфига CS:GO (coop, deathmatch, guardian). Ровно 20; порядок: 12 базовых типов CS2 (KNIFE…STACKABLEITEM) + 7 специфичных для старых режимов + UNKNOWN последней, на 20-й позиции:

| # | Тип |
| --- | --- |
| 1–12 | KNIFE … STACKABLEITEM (как в CS2, но без UNKNOWN) |
| 13 | FISTS |
| 14 | BREACHCHARGE |
| 15 | BUMPMINE |
| 16 | TABLET |
| 17 | MELEE |
| 18 | SHIELD |
| 19 | WEAPONTYPE_ZONE_REPULSOR |
| 20 | UNKNOWN |

> **Note**
>
> В CS2 UNKNOWN — 13-я строка; в CS:GO она сдвинута на 20-ю позицию. Итог везде ровно 20 строк — никакой «фактически 21-й» нет.

**4.6.4.** Guardian — отдельный «замедленный» конфиг, но важно, **на каком уровне** смотреть различия:

- В блоке **default** guardian отличается от deathmatch/coop **практически только строкой RIFLE** (0.25/0.55/0.0/0.95/0.25 против 0.22/0.07/0.0/0.45/0.15) — остальные ключи совпадают (проверено построчным diff).
- В блоках **low/fair/normal** отличий много — почти все aim/look/reaction-ключи (Guardian «замедлен»: acquisition lerp 0.8/0.7/0.6 против 0.5/0.45/0.4, tracking lerp 1.0/0.9/0.85 против 0.65/0.55/0.5, reaction_time 0.85/0.8/0.65 против 0.5/0.5/0.4).
- В блоках **tough/hard/very_hard/expert/elite** отличий снова меньше — ровно два: **reaction_time** (Guardian 0.5/0.45/0.35/0.3/0.25 против 0.35/0.3/0.25/0.2/0.12 у Co-op/DM) и **строка RIFLE** (длительность и разброс чуть выше, cooldown совпадает — см. §4.6.5). Остальные aim/look-ключи на этих уровнях совпадают.

Значения в таблице ниже — **уровень low** (в старых редакциях вики они подавались как общие, без указания уровня):

| Ключ | Co-op / Deathmatch (CS:GO, low) | Guardian (CS:GO, low) |
| --- | --- | --- |
| aim_target_acquisition_lerp_time | 0.5 | 0.8 |
| aim_target_acquisition_angle_penalty | 5 | 10 |
| aim_ready_angle_tolerance | 3.0 | 5.0 |
| aim_new_target_angle_tolerance | 30 | 45 |
| aim_max_duration | 1.1 | 1.2 |
| reaction_time | 0.5 | 0.85 |
| look_around_* | значения deathmatch (low: 0.9/0.4/0.6) | отличия по всем ключам (low: 0.75/…/0.5) |
| строка RIFLE | [ 0.22, 0.07, 0.0, 0.45, 0.15 ] | по уровням — см. §4.6.5 |

> **Note**
>
> aim_ready_angle_tolerance в guardian по уровням: default = 3.0, low = 5.0, fair = 4.5, normal = 4.5, tough/hard/very_hard/expert/elite = 3.0. То есть 4.5 — это fair/normal, а не hard.

**4.6.5.** Пример RIFLE:

| Конфиг | Уровень | Значение | Интерпретация |
| --- | --- | --- | --- |
| CS2 (все 4 конфига идентичны) | все | [ 0.22, 0.07, 0.0, 0.45, 0.15 ] | Очередь ~0.22 с, без паузы внутри, новая очередь через 0.45 с |
| CS:GO Co-op / Deathmatch | все уровни | [ 0.22, 0.07, 0.0, 0.45, 0.15 ] (default–normal); [ 0.225, 0.075, 0.0, 0.32, 0.12 ] (tough); [ 0.225, 0.075, 0.0, 0.25, 0.05 ] (hard); … [ 0.155, 0.035, 0.0, 0.15, 0.05 ] (elite) | Идентично CS2-конфигу в default |
| CS:GO Guardian | default | [ 0.25, 0.55, 0.0, 0.95, 0.25 ] | Очередь ~0.25 с, очень большой разброс, cooldown 0.95 с |
| CS:GO Guardian | low | [ 0.65, 0.55, 0.0, 0.95, 0.25 ] | Длинная очередь — «полив» на низком скилле |
| CS:GO Guardian | fair | [ 0.55, 0.55, 0.0, 0.95, 0.25 ] | — |
| CS:GO Guardian | normal | [ 0.30, 0.55, 0.0, 0.75, 0.25 ] | — |
| CS:GO Guardian | tough | [ 0.25, 0.45, 0.0, 0.32, 0.12 ] | Cooldown как у Co-op, но длительность/разброс выше |
| CS:GO Guardian | hard | [ 0.30, 0.2, 0.0, 0.25, 0.05 ] | — |
| CS:GO Guardian | very_hard | [ 0.30, 0.2, 0.0, 0.2, 0.05 ] | — |
| CS:GO Guardian | expert | [ 0.30, 0.2, 0.0, 0.15, 0.05 ] | — |
| CS:GO Guardian | elite | [ 0.25, 0.20, 0.0, 0.15, 0.05 ] | — |

> **Note**
>
> Из старой вики: «Rush/Guardian — [ 0.25, 0.55, 0.0, 0.95, 0.25 ]» — это только CS:GO Guardian (default). CS2 Rush/Deathmatch байт-идентичны друг другу.

---

# Часть II. Узлы

## 5. Композиты

| Узел | Ключи | Что делает |
| --- | --- | --- |
| parallel | succeed_after_first (bool), children | Запускает всех детей каждый тик. Провал любого = провал всего. Успех = все дети успешны, либо первый успешный при succeed_after_first = 1. succeed_after_first = 1 реально используется — единственное место в дампах: op09_grunts_card2/bt_strat_hold_position (2×) |
| selector | children | Список приоритетов. Проверяет детей с первого каждый тик. Первый прошедший запускается. Если более ранний ребёнок начал проходить — текущий останавливается |
| sequencer | children | Выполняет детей по порядку. Провал любого = стоп. Успех = все прошли. Check-pass смотрит только на текущего ребёнка; пройденные условия не перепроверяются |
| subtree | file, name, params | Запускает другое дерево из файла. params пишут значения в blackboard до старта. Только int или string; float/bool отвергаются. Имена ключей регистронезависимы |

**5.1.** subtree с параметрами — это не мёртвый синтаксис: в CS:GO-дампе ровно 15 использований params, все в op09_grunts_card2 — bt_strat_lobby (×6), bt_strat_heaven (×3), bt_strat_outside (×6) передают параметры в bt_strat_hold_position.kv3:

| Ключ параметра | Тип значения | Пример |
| --- | --- | --- |
| StratCounter | int | 1 |
| HoldDestination | vector строкой | "85.5 -749.55 -351.91" |
| HoldApproachPoint | vector строкой | аналогично |
| HoldAngles | vector строкой | аналогично |
| IntermediateWaypoint | vector строкой | аналогично |
| ThrowUtility | строка-литерал | "'LobbyRushSite'" |

Схематически (по дампу card2):

```
type = "subtree"
file = ".../op09_grunts_card2/bt_strat_hold_position.kv3"
name = "..."
params =
[
    { key = "StratCounter" value = 1 }
    { key = "HoldDestination" value = "85.5 -749.55 -351.91" }
    { key = "ThrowUtility" value = "'LobbyRushSite'" }
]
```

Это целый официальный паттерн Valve — «параметризованное удержание позиции»: одна стратегия bt_strat_hold_position переиспользуется с разными позициями/углами/точками подхода для lobby, heaven и outside.

> **Warning**
>
> В params можно передавать только int и string (векторы — строкой). float/bool дают ошибку:
>
> `[AI BT]: unsupported type for subtree parameter`
>
> В CS2-дампе params не встречается вовсе; официальный образец — только CS:GO Guardian (op09_grunts_card2).

## 6. Декораторы

**6.1.** Почти все поддерживают negated = 1 — инверсия check-результата.

### 6.2. Корневые сервисы

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_bot_service | memory_to_expire ({key, time, distance, domain}[]), tagged_entities_to_expire (string[]), input_chatter_enemies (string), basic_chatter_enable (bool), chatter_outnumbered_threshold (int), child | Корневой сервис. Истекает память, управляет болтовнёй | ✅ кроме chatter_outnumbered_threshold ⚠️ |
| decorator_buy_service | output (string), child | Возвращает массив предметов к покупке. Выход — ключ blackboard, обычно ShouldBuy | ✅ |
| decorator_hiding_spot_service | domain (string), output_hiding_spot (string), expiration_time (float), distance_threshold (float), child | Учитывает укрытия. Боты в одном домене не проверяют уже проверенные укрытия | ✅ |

**6.2.1.** decorator_bot_service — что делают tagged_entities_to_expire и basic_chatter_enable:

| Ключ | Смысл |
| --- | --- |
| tagged_entities_to_expire = [ "EngagedEntities" ] | При истечении памяти снимает теги с EngagedEntities. Без этого теги висели бы до expiration_time (1 с), но с этим — снимаются сразу при истечении памяти |
| basic_chatter_enable = 1 | Включает радио-болтовню |
| input_chatter_enemies = "ShortTermAttackMemory" | Какой ключ памяти использовать для «враг замечен» |

**6.2.2.** decorator_hiding_spot_service — что реально делает:

| Параметр | Смысл |
| --- | --- |
| domain = "AllBots" | Все боты делят состояние укрытий |
| output_hiding_spot = "HidingSpot" | Ключ blackboard, куда пишется выбранное укрытие |
| distance_threshold = 800 | Максимальная дистанция до укрытия |
| expiration_time | Сколько секунд укрытие считается «занятым/проверенным». 10 в Co-op и почти всех корнях Guardian, 15 — ровно в трёх корнях (см. §21.2) |

action_move_to hiding_spot = "HidingSpot" заставляет бота двигаться от укрытия к укрытию (cover-to-cover), а не по прямой. Это ключевой паттерн для Co-op/Guardian.

### 6.3. Пикеры

> **Note**
>
> У **всех** decorator_picker_* есть общие ключи: **input, input_domain, output, output_domain, negated, child**. В таблице ниже перечислены только специфичные ключи; «—» означает «специфичных ключей нет», а не «у узла нет ключей» (например, decorator_picker_reaction_time реально использует input/output — см. §3.5).

| Узел | Доп. ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_picker_nearby | cutoff_distance (float) | Убирает сущности дальше cutoff_distance | ✅ |
| decorator_picker_visible | check_fov (bool, по умолчанию 1) | Убирает невидимые сущности. check_fov = 0 — игнорирует FOV | ✅ |
| decorator_picker_reaction_time | — | Задерживает выход на reaction_time бота | ✅ |
| decorator_picker_dedup | against (string), distance_threshold (float) | Убирает дубликаты — сущности ближе distance_threshold к сущностям из against | ✅ |
| decorator_picker_random_by_distance | distance_min (float), distance_max (float) | Вероятностный фильтр по дистанции: сущности ближе к distance_min удаляются реже, ближе к distance_max — чаще. Не жёсткий порог. Важно для bt_memorize_noises. 🔬 Вероятностная модель — вывод из дампов, официально не документирована | ✅ |
| decorator_picker_weight_as_distance | — | Убирает сущности, чья дистанция больше их «веса» (weight — внутренняя метрика слышимости). По VDC использовать только при сенсинге NOISE | ✅ |
| decorator_picker_grenade_type | types[] (в CS2: FLASH, FIRE, EXPLOSIVE, DECOY, SMOKE; SENSOR и SNOWBALL — только старая CS:GO-вики, в CS2 их нет) | Фильтр гранат по типу | ✅ |
| decorator_picker_blocked_by_smoke | distance_threshold (float) | Оставляет только сущности за дымом | ⚠️ distance_threshold в файлах есть, но по вики игнорируется |
| decorator_picker_max_score | — | Берёт сущность с максимальным score. Если перед ним decorator_ranker_dist — score обратно пропорционален дистанции, т.е. берётся ближайшая. По VDC **без ранкера** берётся первая сущность массива (порядок появления на сервере); конкретное значение score официально не документировано — результат не «случайный» и не «лучший». Частая ошибка новичков | ✅ |

> **Note**
>
> decorator_picker_max_score выбирает сущность с наивысшим score, который задаётся предыдущими пикерами/ранкерами. decorator_ranker_dist присваивает score обратно пропорционально дистанции (ближе = выше score). 🔬 Формула обратной зависимости выведена из наблюдений и официально не документирована. Всегда ставьте ранкер перед max_score, если нужна осмысленная выборка.

### 6.4. Ранкеры

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_ranker_dist | input (string) | Сортирует сущности по дистанции, присваивает score | ✅ |

### 6.5. Сенсоры

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_sensor | output (string), class_name (string), priority (int), orphan_only (bool), append_items (bool), entity_type_filter (enum), team_filter (enum), attacker_filter (enum), shape ({type, radius}), child | Сканирует мир и записывает найденные сущности в output. shape: sensor_shape_fov или sensor_shape_sphere с radius. orphan_only = 1 — только «бесхозные». append_items = 1 — добавить к списку | ✅ кроме attacker_filter ⚠️; append_items — только CS2 |

append_items: ключ перечислен Source2 Wiki и встречается только в CS2 — например, в deathmatch hunt-ветке: `decorator_sensor entity_type_filter="CLASSNAME" output="HuntAreas" append_items=1 class_name="hostage_entity"`. В CS:GO-дампах не встречается.

> **Warning**
>
> Известный баг: при фильтрации гранат с team_filter = "ENEMY" сенсор ошибочно возвращает гранаты T-команды независимо от команды бота. Обходной путь (подтверждено Valve Developer Community) — использовать OPPOSITE.

### 6.6. Память

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_memory | input (string), output (string), output_domain (string), child | Копирует сущности из input в output. Если указан output_domain — в общую память домена | ✅ |
| decorator_set_reaction_time | input (string), child | Задерживает прохождение input на reaction_time бота | ✅ |

### 6.7. Управление потоком

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_repeat | amount (int), child | Перезапускает ребёнка каждый раз, когда он завершается, и возвращает running. Без amount — вечно. С amount = N — N раз, затем возвращает результат последнего запуска. ⚠️ Расхождение источников: Source2 Wiki не описывает остановку по failure; по VDC узел останавливается, если ребёнок вернул failure. В дампах amount не встречается, поэтому поведение с amount + failure дампами не подтверждено | ✅ amount ⚠️ не встречается ни разу (все 7 вхождений amount в дампах — это decorator_tag_threshold amount = 2) |
| decorator_maybe | chance (float, 0–1), child | Выполняет ребёнка с вероятностью chance | ✅ |
| decorator_succeed | negated (bool), child | Без negated: всегда проходит check-pass, после ребёнка возвращает success. С negated = 1: всегда проходит check-pass, после ребёнка возвращает failure | ✅ |
| decorator_invert | child | Инвертирует только check-результат (running-результаты ребёнка проходят насквозь без изменений). В CS2 вместо него обычно используют negated = 1. ⚠️ По VDC — единственный способ инверсии для action-узлов, которые не поддерживают negated (обходной путь) | ⚠️ не встречен |
| decorator_run_once | domain (string), max_attempts (int), child | Ограничивает запуск. Без domain — каждый бот один раз. С domain — только один бот на сервере | ✅ |
| decorator_try_lock | domain (string), child | Блокирует домен на время выполнения ребёнка. Только один бот одновременно | ✅ |
| decorator_wait_success | timeout (float), child | Ждёт success от ребёнка или timeout. В CS2-официальных деревьях не встречается (Source2 Wiki помечает «unused» именно поэтому); в CS:GO Guardian используется активно — ×9 в 8 файлах (координация, §18; например op09_grunts_card7_2 и bt_strat_hold_position) | ✅ (CS:GO Guardian) |
| decorator_need_healing | health_threshold (int), child | Выполняет ребёнка, если HP ниже порога | ✅ |
| decorator_random_int | min (int), max (int), output (string), child | Пишет случайное целое в output | ✅ |
| decorator_random_approach_point | output (string), child | Пишет случайную точку на навмеше — «вход» в область. Видимость точки источниками не подтверждена — не рассчитывайте на неё | ✅ |

**6.7.1.** decorator_run_once — разница между «с доменом» и «без»:

| Конфигурация | Смысл |
| --- | --- |
| decorator_run_once без domain | Каждый бот выполняет один раз |
| decorator_run_once domain = "'CoordinatedBuy'" | Только один бот на сервере выполняет; остальные видят, что домен занят |
| decorator_run_once max_attempts = 3 | До 3 попыток |

Пример из bt_coordinated_buy.kv3:

```
decorator_run_once
domain = "'CoordinatedBuy'"
child = sequencer[ set_global_counter, wait 3, coordinated_buy, ... ]
```

Только один бот запускает координацию покупки. Остальные видят, что домен занят, и идут в condition_is_empty WaitForCoordinatedBuy — ждут.

### 6.8. Теги и счётчики

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_tag_entity | input (string), output (string), expiration_time (float), operation_type (enum), child | Помечает первую сущность из input в output. operation_type: BT_DECORATOR_TAG_ENTITY_SET / BT_DECORATOR_TAG_ENTITY_CLEAR | ✅ |
| decorator_tag_threshold | entity_input (string), tagged_entities_input (string), amount (int), check_type (enum), child | Сравнивает количество помеченных. BT_DECORATOR_TAG_THRESHOLD_AT_LEAST / BT_DECORATOR_TAG_THRESHOLD_AT_MOST | ✅ |
| decorator_token_service | config ({tokens[], assignments[][], team_restriction[]}), output_token_name (string), output_token_domain (string), domain (string), child | Раздаёт ботам «токены»-роли. Выход — GroupId и GroupMemberId | ✅ |
| decorator_remove | input_domain (string), input (string), remove (string), child | Удаляет сущности из input, которые есть в remove | ✅ |
| decorator_remove_key | input (string), child | Удаляет ключ blackboard | ✅ |
| decorator_dec_global_counter | input_name (string), child | Уменьшает глобальный счётчик на 1 | ✅ |

**6.8.1.** decorator_tag_entity + expiration_time — что это:

```
decorator_tag_entity
input = "Enemy"
output = "EngagedEntities"
operation_type = "BT_DECORATOR_TAG_ENTITY_SET"
expiration_time = 1
```

Тег живёт 1 секунду, потом автоматически снимается. Это позволяет bt_plant_bomb_if_covered проверять «есть ли ≥2 врага, с которыми мы реально воюем прямо сейчас». Без expiration_time тег висел бы вечно — см. анти-паттерн «утечка тегов» в §27.

**6.8.2.** decorator_tag_entity + BT_DECORATOR_TAG_ENTITY_CLEAR — что это:

```
decorator_tag_entity
input = "Enemy"
output = "EngagedEntities"
operation_type = "BT_DECORATOR_TAG_ENTITY_CLEAR"
child = decorator_tag_threshold ...
```

Перед проверкой AT_LEAST теги сначала очищаются. Это важно: bt_attack ставит теги с expiration_time = 1, но bt_pickup_hostage_if_covered дополнительно чистит их, чтобы не полагаться на таймер.

**6.8.3.** decorator_token_service — assignments и team_restriction:

```
tokens = [ "StratLobby", "StratOutside", "StratHeaven", "NoStratSqueaky", "NoStartLobby" ]
assignments =
[
    [ 3, 3 ],       // 3 бота получают токен 1, 3 — токен 2
    [ 6 ],          // все 6 получают токен 1
    [ 0, 6 ],       // 0 получают токен 1, 6 — токен 2
    [ 0, 3, 3 ],    // 3 получают токен 2, 3 — токен 3
    ...
]
```

| Поле | Смысл |
| --- | --- |
| assignments | Массив вариантов раздачи. Движок перебирает массив сверху вниз и берёт первый вариант, у которого сумма чисел совпадает с количеством ботов в команде и соблюдены team_restriction. Если подходящего варианта нет (например, на сервере 3 бота, а варианты только на 5 и 6) — токены не выдаются, GroupId остаётся пустым |
| team_restriction | Массив, параллельный tokens. team_restriction[i] ограничивает tokens[i] указанной командой |

> **Note**
>
> Источник примера выше — op09_grunts_card2 (не card12). В op09_grunts_card12 другие токены: StratAtrium, StratLobby, StratUpOffice (3 токена, 6 вариантов раздачи). Пример с 9 токенами — op10_ancient_a (CoverSmoke … LongAwp) × 5 вариантов раздачи. В Practice — assignments = [[1,1,1,1,1,1,1,1,1,1]] (по одному боту на токен).

Пример team_restriction из newplayer_dust2_01.kv3:

```
tokens = [ "t_1", "t_2", "t_3", "t_4", "t_5", "ct_1", "ct_2", "ct_3", "ct_4", "ct_5" ]
team_restriction = [ "T", "T", "T", "T", "T", "CT", "CT", "CT", "CT", "CT" ]
```

### 6.9. Прочие декораторы

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| decorator_find_utility_strat | distance_threshold (float), input_location, output_location, output_angles, output_weapon, utilities ({x, y, z, ang_pitch, ang_yaw, ang_roll, weapon}[]), child | Предположительно ищет позицию для утилиты. Не используется | ⚠️ не встречен |
| decorator_route_service | config ({routes: waypoint[][], strategies: int[][]}), output_waypoint, output_waypoint_name, output_domain, child | Предположительно управляет маршрутами. Не используется | ⚠️ не встречен |
| decorator_set_barrier | input_domain (string), input_name (string), input_location (position) — по VDC | Предположительно ставит «барьер» (запретную зону) для pathfinding. Не используется | ⚠️ не встречен |
| decorator_game_event | — | Узел из старой CS:GO-вики (с пометкой самого сообщества «не существует в коде?»). В дампах отсутствует; приведён для полноты | ⚠️ существование под вопросом |

## 7. Условия

**7.1.** Все поддерживают negated = 1.

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| condition_is_empty | input (string), global (bool), negated, child | Проверяет, пуст ли ключ или его нет. global = 1 — глобальный флаг | ✅ |
| condition_is_equal | source (string), destination (int или string), domain (string), negated, child | Проверяет равенство. destination может быть числом или строкой, включая "'ct_1'". Незадокументированный ключ domain — см. §21.2.1 | ✅ |
| condition_is_greater_equal | source, destination, negated, child | source >= destination. Примеры: GuardianWaveNumber >= 5 (op11_county_2/bt_bot_1..5), RoundNumber >= 4 (practice) | ✅ |
| condition_is_less_equal | source, destination, negated, child | source <= destination | ✅ |
| condition_is_greater | source, destination, negated, child | source > destination | ⚠️ не встречен |
| condition_is_less | source, destination, negated, child | source < destination | ⚠️ не встречен |
| condition_is_inv_slot_empty | slot (enum: RIFLE, PISTOL, KNIFE, GRENADES, C4), negated, child | Проверяет пустоту слота | ✅ |
| condition_is_weapon_equipped | weapon (string), negated, child | Активное оружие — с этим classname | ✅ |
| condition_out_of_ammo | negated, child | В активном оружии нет патронов | ✅ |
| condition_owns_item | item (string), items_one_of (string[]), negated, child | Есть ли предмет | ✅ |
| condition_is_at_bomb_site | negated, child | Бот на бомбсайте | ✅ |
| condition_distance_less | input (string), distance_threshold_min (float), distance_threshold_max (float), negated, child | Дистанция до input меньше случайного порога между min и max. При negated = 1 — «дистанция НЕ меньше порога», т.е. враг далеко | ✅ |
| condition_has_parachute | negated, child | Есть ли парашют | ✅ (только CS:GO Guardian, Blacksite) |
| condition_inactive | input ({key, domain?}[]), round_start_threshold_seconds (float), sensor_inactivity_threshold_seconds (float), negated, child | Нет активности в указанных ключах заданное время и с начала раунда прошло не меньше порога. По Source2 Wiki элемент массива input может содержать domain; в дампах используется только key | ✅ |
| condition_is_airborne | negated, child | Бот в воздухе | ✅ |
| condition_is_reloading | negated, child | Оружие перезаряжается | ⚠️ не встречен |
| condition_is_weapon_suitable | input, weapon, negated, child | Подходит ли оружие для дистанции до input. Задумано так: снайперки «не подходят» ближе 160 юнитов, дробовики — дальше 600 юнитов (VDC). ⚠️ По старой VDC фактически работает (с багом) как condition_is_weapon_equipped, но сам источник ставит пометку [confirm] — статус: не подтверждено, требует самостоятельной проверки в игре | ⚠️ не встречен |
| condition_barrier | input_domain, input_name, radius (по Source2 Wiki) / input_location (по VDC), negated, child | Источники расходятся в наборе ключей | ⚠️ не встречен |

> **Note**
>
> Незадокументированный ключ domain достоверно используется у condition_is_equal (40× в bt_parachute_1; domain отсутствует в Source2 Wiki). По утверждению сообщества он работает и у condition_is_greater_equal, condition_is_less_equal, condition_is_empty — дампами это не подтверждено. Если domain указан, узел ищет переменную в blackboard конкретного домена, а не в локальной доске бота. **Не переносите domain на другие условия без проверки.**

**7.2.** condition_distance_less с negated = 1 — семантика:

Из bt_attack_with_shield.kv3:

```
condition_distance_less
input = "Enemy"
distance_threshold_min = 120
distance_threshold_max = 550
negated = 1
```

Это значит: «дистанция до врага НЕ меньше случайного порога 120–550», т.е. враг далеко. Логика: со щитом бот идёт в атаку только на дальней дистанции (щит — для сближения), а вблизи щит бесполезен.

## 8. Действия

**8.1.** Действия возвращают running, пока не завершатся, затем success или failure.

### 8.2. Движение и позиционирование

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| action_move_to | destination, hiding_spot, threat, auto_look_adjust, damaging_areas_penalty_cost, arrival_epsilon, additional_arrival_epsilon_2d, hiding_spot_check_distance_threshold, nearest_area_distance_threshold, movement_type, route_type | Ведёт бота к destination. movement_type: WALK/RUN. route_type: FASTEST_ROUTE. destination — позиция, ключ blackboard или targetname | ✅ |
| action_combat_positioning | input (string), is_attacking (string) | Уклонение/сближение. Читает is_attacking (флаг из action_attack output="Attacking") и не даёт боту стоять на месте во время перестрелки. Единственный узел «микро» в бою, кроме combat_crouch_chance и combat_dodge_command_duration из конфига | ✅ |
| action_flee_area_damage | input (string), output (string), max_search_range (float), threat_min_keep_distance (float) | Ищет безопасную точку от урона по площади. threat_min_keep_distance — минимальная дистанция, на которую нужно отбежать от угрозы, а не от зоны урона | ✅ |
| action_jump | — | Прыжок. Помимо fallback в Co-op rush, ядро jump-throw паттерна: 25 активных вхождений в 22 CS:GO-файлах (утилити-стратегии op10/op11 — см. §15.12) | ✅ |
| action_crouch | — | Присесть | ✅ (op09_grunts_card2, op10_engage_b/bt_boost_molly) |
| action_teleport | destination (position) | Телепорт. ⚠️ Не сбрасывает auto_look_adjust и не влияет на камеру — для точных углов после телепорта ставьте явный action_look_at | ✅ |
| action_parachute_positioning | — | Движение по диагонали для парашюта; периодически меняет направление (каждые несколько секунд) | ✅ (только CS:GO Guardian, Blacksite) |
| action_hide | output (string), max_range (float) | Случайное укрытие. Не встречен. ⚠️ По VDC не проверяет, занято ли укрытие. Используйте decorator_hiding_spot_service + action_move_to hiding_spot | ⚠️ не встречен |
| action_standup | — | Встать | ⚠️ не встречен |

**8.2.1.** hiding_spot и threat у action_move_to — влияние на pathfinding:

| Ключ | Смысл |
| --- | --- |
| hiding_spot = "HidingSpot" | Бот строит маршрут от укрытия к укрытию (cover-to-cover), а не по прямой линии — критично для выживания. В дампах значение — ключ blackboard "HidingSpot"; Source2 Wiki типизирует параметр как number, т.е. blackboard-ключ/число |
| threat = "NearestThreatMemory" | Алгоритм pathfinding активно избегает нав-тайлов, близких к указанной угрозе (если это возможно без значительного удлинения маршрута) |

> **Note**
>
> BT_ACTION_MOVETO_SAFEST_ROUTE не используется ни разу — только BT_ACTION_MOVETO_FASTEST_ROUTE. Движение — только RUN / WALK.

damaging_areas_penalty_cost, additional_arrival_epsilon_2d, hiding_spot_check_distance_threshold встречаются только в CS:GO-дампах (Guardian); в CS2-дампе файлов с ними нет (примеры: op10_ancient_a/bt_cover_smoke, op09_grunts_card7_2/bt_a_setup_smoke_only_1, op09_grunts_card6_2/bt_takeover_strategy — все это CS:GO Guardian).

> ⚠️ Баг: после action_jump тип движения сбрасывается на RUN, даже если был WALK (подтверждено Valve Developer Community).

> ⚠️ Баг: при включённом столкновении с тиммейтами бот не перестраивает маршрут вокруг товарища, перегородившего путь, а продолжает «проталкиваться» сквозь него, пока тот не уйдёт с дороги (подтверждено VDC).

### 8.3. Прицеливание и стрельба

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| action_aim | input (position), ready (string) | Поворачивает бота к точке input. Когда прицел готов — ставит флаг ready. acquire_only в файлах передаётся, но по вики игнорируется. Не разскопится со снайперки до смены оружия | ✅ |
| action_attack | input (string), ready (string), output (string) | Стреляет по input, когда ready истинно. Если ready не указан — стреляет сразу. output = true, пока идёт атака | ✅ |
| action_pull_trigger | ratio (в файлах встречается один раз: ratio = 1 в bt_heal_if_needed; по вики не читается) | Нажимает ЛКМ | ✅ ratio ❌ |
| action_look_at | input_angles (string) или input_location (string) | Поворачивает бота. Указывать одно из двух | ✅ |
| action_secondary_attack | — | Вторичная атака (ПКМ) | ⚠️ не встречен |
| action_reload | — | Перезарядка | ⚠️ не встречен |

> **Important**
>
> action_aim не разскопится со снайперки до смены оружия — это известный баг. Официальный workaround: вызвать action_equip_weapon с тем же оружием, которое уже в руках (или с "BEST", если оно уже лучшее) — это сбрасывает состояние прицеливания (unscope). Не полагайтесь на acquire_only, чтобы заставить бота смотреть, но не стрелять — используйте action_look_at.

### 8.4. Гранаты

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| action_aim_projectile | input (position), output (string) | Рассчитывает углы для броска гранаты. Известный баг: полностью игнорирует гравитацию и физику полёта — просто берёт вектор направления от бота к цели и записывает его в углы. output пишет углы, а не позицию. Работает только потому, что action_look_at input_location принимает и углы, и позиции | ✅ |
| action_throw_utility | name (string), location (position), velocity (position) | Прямой бросок гранаты со спавном в заданной мировой позиции. location — мировая позиция, куда спавнится граната (не позиция бота). velocity — начальная скорость. Не использует action_aim_projectile, не зависит от того, куда смотрит бот | ✅ (по дампам — только CS:GO Guardian, op11_county_2; в CS2-дампе 0 вхождений, но дамп неполный) |

**8.4.1.** action_throw_utility — это не просто «бросить гранату», а скриптовая установка. См. §17.

**8.4.2.** action_aim_projectile — где используется и как обходится:

| Где | Что |
| --- | --- |
| coop/autumn/modules/bt_throw_utility_if_covered.kv3 (CS:GO) | action_aim_projectile input = "Enemy" output = "UtilityTarget" |
| Обход в Guardian (CS:GO) | action_throw_utility с точной позицией и скоростью (только op11_county_2) либо скриптовый паттерн move_to → look_at(input_angles) → equip → wait → pull_trigger (sirocco_2/4/5) |

### 8.5. Предметы и инвентарь

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| action_equip_weapon | weapon (string, classname или BEST) | Выбирает оружие. Вызов с уже активным оружием сбрасывает прицел (unscope) — см. workaround в §8.3 | ✅ |
| action_equip_item | item (string), items_one_of (string[]) | Выбирает предмет по classname | ✅ |
| action_acquire_items | items (string[]), remove_all_items (bool) | Выдаёт предметы. remove_all_items = 1 — предварительно очистить инвентарь | ✅ |
| action_buy | — | Случайная покупка | ✅ |
| action_custom_buy | item_aliases (string[]) | Покупка по алиасам | ⚠️ не встречен |
| action_drop_active_weapon | — | Выбросить активное оружие | ⚠️ не встречен |
| action_inspect_current_weapon | — | Анимация осмотра оружия | ⚠️ не встречен |

### 8.6. Координация покупок

**8.6.1.** action_coordinated_buy — структура:

```
action_coordinated_buy
team_filter = "SAME"
save_threshold = 2000
id_no_purchase = "DidNotPurchase"
purchases =
[
    { items = [ "weapon_flashbang" ] id = "PalaceExit1" }
    { items = [ "weapon_smokegrenade" "weapon_flashbang" ] id = "ASetupSmokeAndFlashbang1" }
]
```

| Поле | Смысл |
| --- | --- |
| items | Массив classname |
| id | Строковый идентификатор покупки, проверяется через condition_is_empty (например, PalaceExit1 в bt_palace_exit_1.kv3) |
| id_no_purchase | ID, который получает бот, если не купил (например, DidNotPurchase) |
| save_threshold | Если у бота меньше денег — пропускает покупку |
| team_filter | SAME / ENEMY / ANY / CT / TERRORIST / OPPOSITE (полный список Source2 Wiki; в дампах используется SAME) |

Узел возвращает failure и пропускается, если ботов, подходящих под team_filter, не хватает на все записи purchases — покупка целиком не выполняется (по VDC).

**8.6.2.** Известных бага два плюс одно важное предупреждение (Valve Developer Community):

| № | Тип | Суть |
| --- | --- | --- |
| — | Предупреждение (штатная механика, не баг) | Один вызов узла инициирует покупку для ВСЕХ ботов, подходящих под team_filter. Это задуманное поведение — поэтому VDC рекомендует всегда вызывать узел строго внутри decorator_run_once с уникальным domain |
| 1 | Баг — наследование предметов | Боты, стоящие ниже в массиве purchases, наследуют предметы от ботов выше: внутренний буфер покупок не очищается между итерациями по ботам |
| 2 | Баг — id_no_purchase | Боты, не подходящие под team_filter, всё равно ошибочно получают id_no_purchase — не используйте его как фильтр по команде |

Решение: decorator_run_once domain = "'CoordinatedBuy'" — выполняет только один бот-инициатор; остальные ждут через decorator_wait_success + action_compare_global_counter (см. §18).

### 8.7. Стратегии и координация

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| action_choose_bomb_site_area | input (number), output (string) | Позиция бомбсайта по индексу. input может быть 0/1, ключом blackboard или ConVar @mp_guardian_target_site (47 использований, все — CS:GO Guardian) | ✅ |
| action_choose_random_waypoint | input (string), output (string) | Случайный тайл навмеша из input | ✅ |
| action_choose_team_spawn_area | output (string), team (enum) | Нав-зона со спавнами команды | ✅ team ⚠️ не встречается |
| action_choose_random_waypoint_within_radius | origin (position), radius (number), output (string) | Случайная точка в радиусе. radius может быть числом, строкой-литералом или ключом blackboard | ✅ |
| action_choose_guardian_bomb_plant_location | output (string) | Выбирает позицию закладки бомбы Guardian. Работает, только если @mp_plant_c4_anywhere = 1 (18 использований: 14 CS:GO Guardian — 10 корней op11 + 4 в guardian/modules bt_plant_bomb_if_owned/if_covered — и 4 CS2 Practice в practice/modules). Пишет нав-тайл (позицию), совместимый с action_move_to | ✅ |
| action_select_areas_within_radius | input (position), radius (number), output (string) | Выбирает нав-зоны в радиусе. radius — число, строка или ключ | ✅ |
| action_assign_guardian_loadout | input_token_name (string), assignments ({token, loadout[]}[]) | Раздаёт лодауты Guardian по токенам и волнам (wave_numbers / wave_number_min / wave_number_max / weight) | ✅ |

**8.7.1.** action_assign_guardian_loadout — структура loadout:

```
{
    token = "CoverSmoke"
    loadout =
    [
        { items = [ "weapon_glock" ] wave_numbers = [ 1 ] weight = 8 }
        { items = [ "item_assaultsuit" "weapon_ak47" ] wave_number_min = 3 weight = 8 }
    ]
}
```

| Поле | Смысл |
| --- | --- |
| items | Массив classname |
| wave_numbers | Массив номеров волн, в которых доступен этот loadout |
| wave_number_min / wave_number_max | Альтернатива wave_numbers — диапазон волн: «начиная с этой» / «…по эту» |
| weight | Вес при случайном выборе среди подходящих |

Пример — op10_apollo («пистолетные раунды»): все лодауты — пистолеты, раздаваемые по волнам: usp/hkp2000 → elite/tec9/fiveseven/cz75a → revolver/deagle (волны 1–4).

### 8.8. Служебные действия

| Узел | Ключи | Что делает | Статус |
| --- | --- | --- | --- |
| action_wait | wait_time_min (number), wait_time_max (number), wait_modulo_tick_mod (number), wait_modulo_tick_add (number) | Замораживает ИИ на случайное время | ✅ кроме modulo ⚠️ |
| action_use | — | Нажимает +use: использовать, разминировать, подобрать | ✅ |
| action_say | phrase (string), high_priority (bool) | Радио-команда. Список фраз — в botchatter.db | ✅ |
| action_commit_suicide | — | Убивает бота | ✅ |
| action_entity_accept_input | name (string), input (string) | Отправляет input энтити по targetname | ✅ |
| action_set_global_flag | name (string), expiration_time_min (float), expiration_time_max (float) | Ставит глобальный флаг на случайное время | ✅ |
| action_set_global_counter | input_name (string), input_value (int) | Ставит глобальный счётчик | ✅ |
| action_compare_global_counter | input_name (string), input_value (int) | Сравнивает глобальный счётчик с числом | ✅ |
| action_set_value_float | key (string), value (float) | Пишет float в blackboard | ✅ |
| action_set_value_vector | key (string), value (position) | Пишет позицию в blackboard | ✅ |

**8.8.1.** action_entity_accept_input — что реально делает:

```
action_entity_accept_input
name = "'AutoInstance7-_03creek_door_button'"
input = "'Use'"
```

| Поле | Смысл |
| --- | --- |
| name | targetname энтити (обратите внимание: строка с внутренними одинарными кавычками, как domain) |
| input | имя input-порта энтити ('Use', 'Open', 'Trigger' и т.д.) |

Позволяет открывать двери/нажимать кнопки на карте без pathfinding-движения к ним. В bt_op11_county_2 используется в волне 1 (6 вхождений), чтобы один бот открыл все двери на Creek — в паре с decorator_run_once domain = "'AllBots'" (единственное место с кавычками у AllBots).

**8.8.2.** action_wait — про modulo:

Modulo-параметры (wait_modulo_tick_mod/add) в дампах не встречаются, но описаны в Source2 Wiki. В bt_default.kv3 (CS2 Deathmatch) и bt_strat_hold_position.kv3 — только wait_time_min / wait_time_max.

---

# Часть III. Выполнение

## 9. Модель выполнения

**9.1.** Дерево оценивается каждый тик — примерно 64 раза в секунду, и каждый узел помнит, на чём остановился. Каждый тик дерево обходится дважды:

| Проход | Что делает |
| --- | --- |
| Check-pass | Сверху вниз. Решает, какие ветки имеют право выполняться. Если корень не проходит — тик «пустой» |
| Run-pass | Выполняет выбранные ветки. Здесь бот двигается, целится, стреляет |

**9.2.** Поскольку check-pass повторяется каждый тик, условия перепроверяются постоянно. selector может прервать текущую ветку, если более приоритетная начала проходить проверку.

**9.3.** Поведение композитов

| Композит | Check-pass | Run-pass | Прерывание |
| --- | --- | --- | --- |
| sequencer | Только текущий ребёнок | По порядку до провала | Нет; условия позади не перепроверяются |
| selector | С первого ребёнка каждый тик | Запускает первого прошедшего | Да — останавливает текущего при появлении более приоритетного |
| parallel | Все дети | Все дети каждый тик | Провал любого = стоп всех |

**9.4.** Пример разворачивания sequencer[ move_to(bot_target), look_at(bot_target), wait(2..3) ] в repeat:

| Тик | Что происходит |
| --- | --- |
| 1 | move_to двигает на 1 тик → running |
| 2..N | Игра идёт прямо в move_to — одно непрерывное движение |
| Прибытие | move_to success → look_at success |
| Далее | wait выбирает время окончания и возвращает running |
| Все три готовы | sequencer success → repeat заново |

> **Tip**
>
> Узлы не двигают бота напрямую. action_move_to управляет pathfinding; action_use/action_reload «нажимают кнопку» на тик; action_aim/action_attack рулят встроенным aiming/shooting, пока не завершатся.

> **Important**
>
> Из §9.4 следует: **пока action_move_to возвращает running, sequencer остаётся на этом узле** — бот не «останавливается после одного шага». decorator_repeat вокруг действия нужен только когда требуется повторно запускать уже завершившееся действие (например, цикл выбора новой цели), а не для самого движения.

## 10. Blackboard, домены, память

**10.1.** Blackboard — хранилище ключей-значений бота. Сенсор пишет врагов в output = "Enemy", действие атаки читает их через input = "Enemy".

**10.2.** Строки

| Тип | Пример | Примечание |
| --- | --- | --- |
| Литерал | "'CT'" | В файлах Valve часто двойная строка с внутренними одинарными кавычками |
| Ключ blackboard | TeamName | Должен содержать строку |

**10.3.** Числа

| Тип | Пример | Примечание |
| --- | --- | --- |
| Литерал | 3, 0.5 | — |
| ConVar | @mp_plant_c4_anywhere | — |
| Ключ blackboard | RoundNumber | Если содержит строку — разрешается ещё раз, но только на 1 уровень |

**10.4.** Позиции и имена сущностей

Ключи, ожидающие позицию (destination, input_location), принимают:

| Тип | Пример | Примечание |
| --- | --- | --- |
| Литерал | "-1450 2230 2" | — |
| Ключ blackboard | TargetArea | Позиция, строка или список sensed-сущностей; берётся первая |
| Targetname энтити | ant.button | Если ни один ключ так не называется, берётся origin энтити |

Это позволяет управлять ботами с карты: Rush-боты идут к ant.button по имени; скрипт может двигать target-энтити, чтобы перенаправлять ботов.

**10.5.** Домены

| Домен | Где встречается | Назначение |
| --- | --- | --- |
| AllBots | memory_to_expire, input_domain, output_domain, decorator_run_once | Общая память всех ботов; блокировка для run_once |
| 'StratTokens' | decorator_token_service | Токены стратегий |
| 'PressAntennaButton' | decorator_try_lock | Блокировка нажатия кнопки |
| 'DefuseBombIfNearby' | decorator_try_lock | Блокировка разминирования |
| 'PickupBombIfNearby' | decorator_try_lock | Блокировка подбора бомбы |
| 'SetupGlobalCounters' | decorator_run_once | Установка счётчиков |

Полный список — в §23.

**10.6.** Память

**10.6.1.** Схема:

```
decorator_sensor → output = "Vision"
decorator_picker_* → сужают "Vision"
decorator_memory → копирует в "ShortTermAttackMemory"
decorator_bot_service в корне → истекает память: {key, time, distance}
```

**10.6.2.** Стандартные имена памяти:

| Ключ | Время | Дистанция | Что хранит |
| --- | --- | --- | --- |
| ShortTermAttackMemory | 0.7 | 0 | Врагов, виденных недавно |
| LongTermMemory | 10 | 500 | Враг, виденных давно |
| ShortTermInvestigateMemory | 3 | 200 (2000 в card13) | События для исследования |
| ShortTermDamageMemory | 0.7 | 0 | Источники урона |
| ShortTermAreaDamageMemory | 3 | 0 | Урон по площади |
| DamageThroughSmokeMemory | 3 | 0 | Источники урона за дымом |
| Threats (AllBots) | 10 | 0 | Глобальные угрозы |

**10.6.3.** Другие ключи blackboard:

| Ключ | Что хранит | Кто пишет | Кто читает |
| --- | --- | --- | --- |
| Vision | Список врагов, видимых сенсору FOV | decorator_sensor в bt_memorize_enemies_vision | decorator_picker_visible, decorator_memory |
| Hearing | Список услышанных шумов | decorator_sensor (NOISE) в bt_memorize_noises | decorator_picker_weight_as_distance, decorator_picker_random_by_distance, decorator_memory |
| Damage | Сущности, нанёсшие урон | decorator_sensor (DAMAGE) в bt_memorize_damage | bt_face_damage_inflictor, bt_memorize_threats |
| Grenades | Список гранат в поле зрения | decorator_sensor (GRENADE) в bt_memorize_area_damage_grenades | decorator_picker_grenade_type |
| Infernos | Список inferno-энтити | decorator_sensor (CLASSNAME inferno) в bt_memorize_area_damage_infernos | decorator_picker_nearby, decorator_memory |
| AreaDamage | Список зон урона | decorator_sensor (AREA_DAMAGE) в bt_memorize_area_damage_current | decorator_memory → ShortTermAreaDamageMemory |
| FleeSpot | Позиция безопасной точки | action_flee_area_damage | action_move_to в bt_flee_area_damage_threats |
| TargetArea | Целевая позиция (нав-тайл) | action_choose_random_waypoint, action_choose_guardian_bomb_plant_location, action_select_areas_within_radius | action_move_to |
| TargetHuntArea | Позиция для охоты | action_choose_random_waypoint в ветке «hunt» | action_move_to |
| HuntAreas | Нав-зоны для охоты | action_choose_team_spawn_area / action_choose_bomb_site_area | action_choose_random_waypoint |
| BombSiteArea | Нав-зона бомбсайта | action_choose_bomb_site_area | action_choose_random_waypoint |
| BombSiteIndex | Индекс бомбсайта (0/1) | decorator_random_int | action_choose_bomb_site_area |
| RandomRoamPoint | Случайная точка для bt_roam | action_choose_random_waypoint_within_radius | action_move_to |
| RushPoint | Случайная точка для bt_rush | action_choose_random_waypoint_within_radius | action_move_to |
| RushCenterLocation | Центр раша | action_set_value_vector в bt_determine_rush_center | action_choose_random_waypoint_within_radius в bt_rush |
| RushCenterRadius | Радиус раша | action_set_value_float в bt_determine_rush_center | action_choose_random_waypoint_within_radius в bt_rush |
| UtilityTarget | Углы для броска гранаты | action_aim_projectile | action_look_at в bt_throw_utility_if_covered |
| ApproachPoint | Случайная точка на навмеше для осмотра | decorator_random_approach_point в bt_look_around | action_aim |
| HidingSpot | Позиция выбранного укрытия | decorator_hiding_spot_service | action_move_to (hiding_spot = "HidingSpot") |
| ShouldBuy | Массив предметов к покупке | decorator_buy_service | condition_is_empty, action_buy (читается в deathmatch, rush и practice/bt_default; мёртвый — в armsrace и newplayer_dust2_01) |
| NearestThreat | Ближайшая видимая угроза | decorator_picker_reaction_time в bt_select_nearby_threat | decorator_memory |
| NearestThreatMemory | Копия ближайшей угрозы | decorator_memory в bt_select_nearby_threat | action_move_to (threat = "NearestThreatMemory") |
| Event | Событие для исследования | decorator_picker_reaction_time в bt_investigate_closest_memorized_event | action_look_at, action_move_to |
| VisibleShortTermInvestigateMemory | Видимое событие | decorator_picker_visible в bt_investigate_closest_memorized_event | action_look_at |
| ShortTermInvestigateMemoryWithinReactionTime | Событие после reaction time | decorator_picker_reaction_time в bt_guard | condition_distance_less, decorator_ranker_dist |
| Bombs | Список weapon_c4 рядом | decorator_sensor (CLASSNAME weapon_c4) в bt_pickup_bomb_if_nearby | action_move_to |
| PlantedBomb | planted_c4 рядом | decorator_sensor (CLASSNAME planted_c4) в bt_defuse_bomb_if_nearby | action_move_to, action_use |
| Hostage | hostage_entity рядом | decorator_sensor (CLASSNAME hostage_entity) в bt_pickup_hostage | action_move_to |
| HostageSurroundingAreas | Нав-зоны вокруг хостажа | action_select_areas_within_radius | action_choose_random_waypoint |
| GroupId | Идентификатор группы токенов | decorator_token_service | condition_is_equal |
| GroupMemberId | Идентификатор участника группы | decorator_token_service | condition_is_equal (в т.ч. 40× 'ParachuteN' в Blacksite) |
| EngagedEntities | Помеченные враги | decorator_tag_entity | decorator_tag_threshold |
| ThreatsWithinFOV | Угрозы в FOV | decorator_picker_reaction_time в bt_clear_threats_within_fov | decorator_picker_visible, decorator_remove |
| RushSite | Глобальный счётчик «сайт раша» | action_set_global_counter в bt_coordinated_buy / bt_strategy_smoke | action_compare_global_counter |
| DontPlantBomb | Глобальный флаг | action_set_global_flag в bt_plant_bomb_if_owned | condition_is_empty (global = 1) |
| PreparingToThrowSmoke | Глобальный счётчик | action_set_global_counter | decorator_dec_global_counter |
| ThrowPalaceFlash | Глобальный счётчик | action_set_global_counter | decorator_dec_global_counter |
| PositioningForTopSmoke | Глобальный счётчик | action_set_global_counter | decorator_dec_global_counter |
| WaitForCoordinatedBuy | Глобальный счётчик | action_set_global_counter в bt_coordinated_buy | action_compare_global_counter |
| DidNotPurchase | ID бота, который не купил | action_coordinated_buy | condition_is_empty |
| StratCounter | Счётчик стратегии (int) | subtree params (card2) | bt_strat_hold_position |
| HoldDestination | Позиция удержания | subtree params (card2) | action_move_to в bt_strat_hold_position |
| HoldApproachPoint | Точка подхода при удержании | subtree params (card2) | action_move_to |
| HoldAngles | Углы взгляда при удержании | subtree params (card2) | action_look_at |
| IntermediateWaypoint | Промежуточная точка маршрута | subtree params (card2) | action_move_to |
| ThrowUtility | Имя утилити-подстратегии | subtree params (card2) | selector стратегии броска |

**10.7.** Глобальные флаги и счётчики

| Узел | Что делает |
| --- | --- |
| action_set_global_flag | Ставит именованный флаг на случайное время между expiration_time_min и expiration_time_max. Проверяется через condition_is_empty с global = 1 |
| action_set_global_counter | Ставит именованный счётчик |
| decorator_dec_global_counter | Уменьшает счётчик |
| action_compare_global_counter | Сравнивает счётчик с input_value |

Это три разных механизма: глобальные флаги (условные, с временем жизни), глобальные счётчики (числовые) и домены (блокировки). Не смешивайте их.

**10.8.** Параметры subtree

subtree может записывать значения в blackboard до старта. Только int или string (векторы — строкой). float/bool дают ошибку:

```
[AI BT]: unsupported type for subtree parameter
```

> **Note**
>
> params официально используется Valve: 15 вхождений в CS:GO-дампе, все в op09_grunts_card2 (bt_strat_lobby ×6, bt_strat_heaven ×3, bt_strat_outside ×6 → параметры для bt_strat_hold_position). В CS2-дампе не встречается. См. §5.1.

## 11. Зарезервированные переменные

**11.1.** Игра сама кладёт их в blackboard. Читать можно, создавать нельзя.

| Переменная | Тип | Откуда | Использование в дампах |
| --- | --- | --- | --- |
| NumberOfTerrorists | int | Движок | — |
| NumberOfAliveTerrorists | int | Движок | ×1 в каждой версии (CS:GO guardian/modules/bt_plant_bomb_if_owned и CS2 practice-версия) |
| NumberOfCounterTerrorists | int | Движок | — |
| NumberOfAliveCounterTerrorists | int | Движок | — |
| AccountBalance | int | Движок | вики-источник, использования не найдено |
| AmmoCount/<weapon> | int | Движок | вики-источник, использования не найдено |
| AmmoCount/current | int | Движок | вики-источник, использования не найдено |
| TeamName | string | Движок | ×4 (Practice: 3× condition_is_equal destination="'T'" и 1× destination="'CT'") |
| RoundNumber | int | Движок | ×138 (Practice) |
| BlindnessPercentage | float | Движок | вики-источник, использования не найдено |
| GuardianWaveNumber | int | Движок (Guardian) | активно (county_2: >= 5 в bt_bot_1..5; ancient и т.д.) |
| BombIsBeingDefused | — | Движок (из CS:GO-вики) | вики-источник, в дампах не встречено |
| LastCoopSpawnPointName | string | Движок (Co-op) | Co-op bt_determine_rush_center |
| LastCoopSpawnPointLocation | Vector | Движок (Co-op) | Co-op guard/roam/rush |

> **Note**
>
> TeamName и RoundNumber отсутствуют в списке Source2 Wiki, но активно используются в Practice. Список на вики неполный — как и список зарезервированных переменных в целом. «Нельзя создавать» относится к объявлению новых ключей с этими именами; перезапись значений существующих зарезервированных ключей через ваши действия формально не описана — не полагайтесь на неё.

---

# Часть IV. Практика

## 12. Отладка

**12.1.** Команды

| Команда | Назначение | Требования |
| --- | --- | --- |
| mp_bot_ai_bt "путь" | Назначить дерево всем ботам на сервере (обеим командам — и T, и CT) | — |
| mp_bot_ai_bt_clear_cache | Сбросить кэш после правок | — |
| cv_bot_ai_bt_debug_target <index> | Нарисовать дерево бота | sv_cheats 1, локальный/приватный сервер |
| cv_bot_ai_bt_hiding_spot_show | Показать укрытия | sv_cheats 1 |
| cv_bot_ai_bt_moveto_show_next_hiding_spot | Показать следующее проверяемое укрытие | sv_cheats 1 |
| developer 1 | Писать [AI BT]: Loaded behavior tree '<file>' при спавне | — |
| Instance.ServerCommand (cs_script) | Выставить mp_bot_ai_bt скриптом карты (альтернатива cfg) | — |

> **Warning**
>
> По Source2 Wiki cheat-квары отладки (cv_bot_ai_bt_*) не имеют release-флага и могут не работать в релизной сборке игры — только в локальных/приватных сессиях с sv_cheats 1.

**12.2.** Сообщения

| Сообщение | Причина |
| --- | --- |
| [AI BT]: Loaded behavior tree '<file>' | При developer 1 |
| [AI BT] warnings | Всегда |
| Blackboard entry 'X' does not exist nor it is a valid entity name | Ключ не найден |
| [AI BT]: Config node 'attack' must contain 13 entries | Неверное число строк (CS2) |
| [AI BT]: unsupported type for subtree parameter | float/bool в params |

## 13. CS:GO vs CS2

| Аспект | CS:GO | CS2 |
| --- | --- | --- |
| Уровни скилла | 9 (включая elite) | 9 (с elite) |
| Строк в attack | 20 во всех конфигах (coop, deathmatch, guardian) | 13 во всех конфигах (armsrace, deathmatch, practice, rush — байт-идентичны) |
| Набор режимов с деревьями | Co-op Strike, Guardian, Deathmatch | Deathmatch, Arms Race, Rush, Practice |
| Согласованность конфигов | coop ≠ deathmatch (look_around_*), guardian — отдельный «замедленный» конфиг (default — только RIFLE; low/fair/normal — почти всем; tough…elite — только reaction_time + RIFLE; см. §4.6.4) | 4 конфига байт-идентичны (совпадающий md5, проверено); armsrace-дерево ссылается на deathmatch-конфиг |
| aim_ready_angle_tolerance | Есть во всех конфигах (3.0; в guardian по уровням 3.0/5.0/4.5/4.5/3.0…) — отсутствовал только в старой вики | Есть в конфиге |
| TeamName / RoundNumber | Не документированы | Используются в Practice |
| Документация узлов | Неточная, много TODO | Точный список читаемых ключей |
| Узлы вне официальных деревьев | Не отмечены | 36 отмечены, но часть используется в Guardian (см. §26.3) |
| Игнорируемые параметры | Не отмечены | acquire_only, ratio, distance_threshold и др. (см. §26.1) |
| Модель выполнения | Не описана | Два прохода |
| Subtree params | Используются (op09_grunts_card2, 15 вхождений); только int/string | Не встречаются |
| decorator_picker_* | С input_domain = "AllBots" в bt_clear_threats_within_fov и bt_select_nearby_threat (как в CS2) | С input_domain/output_domain |
| decorator_sensor | Без append_items | С append_items |
| action_wait | 2 ключа | + wait_modulo_tick_mod/add (не используются) |
| action_move_to доп. параметры | damaging_areas_penalty_cost, additional_arrival_epsilon_2d, hiding_spot_check_distance_threshold — только здесь (Guardian) | — |
| Deathmatch-деревья | База | CS2 = CS:GO + ровно один блок: hostage-сенсор с append_items в hunt-ветке (diff в 8 строк) — боты подбирают заложников в DM, если они есть на карте |
| Общие модули Guardian/Practice (clear_threats_within_fov, memorize_threats, pickup_bomb_if_nearby, pickup_hostage, select_nearby_threat) | Идентичны CS2-версиям | — |
| Диффы модулей | plant_bomb_if_covered/owned, pickup_hostage_if_covered (~4–15 строк) | — |
| 14 общих модулей (modules/) | 13 из 14 байт-идентичны CS2-версиям; bt_memorize_noises отличается только одной пустой строкой (форматирование, не семантика) | — |
| Practice-деревья | — | Два: newplayer_dust2_01.kv3 (тренировочное) и bt_default.kv3 — копия CS2 DM-дерева без hostage-сенсора, с живой buy-веткой (§14.4.2) |
| Инвентарь дампов | 544 файла: 519 guardian + 9 coop + 14 modules + 2 deathmatch | 57 файлов: armsrace — 2, deathmatch — 2, modules — 14, practice — 37, rush — 2 |

## 14. Готовые модули

### 14.1. Deathmatch / Rush / Practice

| Модуль | Что делает |
| --- | --- |
| bt_attack | Берёт ShortTermAttackMemory → reaction time → сортирует по дистанции (decorator_ranker_dist + decorator_picker_max_score — берётся ближайший, см. §15.9) → если есть враг: помечает в EngagedEntities (SET, 1 с) → параллельно: экипирует BEST, повторяет succeed[out_of_ammo → equip BEST], selector[flee_area_damage, если нож — бежит к врагу, иначе combat_positioning], повторяет aim Enemy ready="AimReady", повторяет attack Enemy ready="AimReady" output="Attacking" |
| bt_memorize_enemies_vision | Сенсор FOV, PLAYERS, ENEMY → nearby 1500 → visible → reaction time → ShortTermAttackMemory + LongTermMemory. В op11 — локальные override-ы (см. §21.3) |
| bt_memorize_noises | Сенсор сфера 3000, NOISE → weight_as_distance → random_by_distance 800–3000 (вероятностный фильтр) → dedup 100 к LongTermMemory → reaction time → LongTermMemory |
| bt_memorize_damage | Сенсор DAMAGE → reaction time → ShortTermDamageMemory; blocked_by_smoke 3000 → DamageThroughSmokeMemory |
| bt_memorize_area_damage_grenades | Сенсор GRENADE → grenade_type (FIRE, EXPLOSIVE) → nearby 2500 → visible → reaction time → ShortTermAreaDamageMemory |
| bt_memorize_area_damage_infernos | Сенсор CLASSNAME inferno → nearby 1000 → reaction time → ShortTermAreaDamageMemory |
| bt_memorize_area_damage_current | Сенсор AREA_DAMAGE → ShortTermAreaDamageMemory |
| bt_memorize_nearest_investigation | succeed → condition_is_empty ShortTermInvestigateMemory → если пусто: memory LongTermMemory → ShortTermInvestigateMemory → ranker_dist → max_score |
| bt_investigate_closest_memorized_event | Берёт ShortTermInvestigateMemory → reaction time → если есть событие: экипирует BEST, если видимо — смотрит и ждёт 0.2–0.5 с, затем идёт к событию (move_to threat="NearestThreatMemory") |
| bt_heal_if_needed | Если HP < 60: экипирует weapon_healthshot, ждёт 1 с, action_pull_trigger (с ratio = 1 — единственное использование ratio в дампах) |
| bt_face_damage_inflictor | Берёт ShortTermDamageMemory → reaction time → если есть: целится (acquire_only), параллельно succeed → blocked_by_smoke 3000 → если есть: повторяет action_attack output="Attacking" |
| bt_attack_damage_inflictor_smoke | Берёт DamageThroughSmokeMemory → reaction time → сортирует → берёт ближайшего → целится, повторяет action_attack input="Damage" output="Attacking" |
| bt_flee_area_damage_threats | Если ShortTermAreaDamageMemory не пусто: action_flee_area_damage 500 → бежит к FleeSpot (auto_look_adjust=0) |
| bt_look_around | random_approach_point → action_aim (acquire_only) → ждёт 0.2–0.5 с |
| bt_plant_bomb_if_covered | Если есть weapon_c4 и в ShortTermAttackMemory ≥2 врага: выбирает бомбсайт, идёт, проверяет @mp_plant_c4_anywhere или is_at_bomb_site, экипирует C4, ждёт 0.5 с, повторяет action_pull_trigger |
| bt_plant_bomb_if_owned | Если есть weapon_c4 и нет флага DontPlantBomb: экипирует BEST, выбирает бомбсайт, идёт, проверяет site, экипирует C4, ждёт 1 с, параллельно: с шансом 0.5 и если NumberOfAliveTerrorists=1 ставит флаг DontPlantBomb (3–5 с) — предохранитель, чтобы последний T не закладывал слишком рано (см. §15.7), и повторяет action_pull_trigger. Если флаг DontPlantBomb — смотрит по сторонам |
| bt_pickup_bomb_if_nearby | Сенсор сфера 3000, CLASSNAME weapon_c4, orphan_only → сортирует → берёт ближайшее → try_lock 'PickupBombIfNearby' → action_move_to (nearest_area_distance_threshold=20) |
| bt_defuse_bomb_if_nearby | Сенсор сфера 3000, CLASSNAME planted_c4, orphan_only → сортирует → try_lock 'DefuseBombIfNearby' → идёт (arrival_epsilon = 35), смотрит, ждёт 0.25 с, повторяет action_use |
| bt_pickup_hostage | Сенсор CLASSNAME hostage_entity, orphan_only → идёт (arrival_epsilon=35), смотрит, ждёт 1 с, повторяет action_use |
| bt_pickup_hostage_if_covered | Берёт ShortTermAttackMemory → reaction time → сортирует → если ≥2 врага в EngagedEntities (CLEAR, amount=2, AT_LEAST) → bt_pickup_hostage |
| bt_clear_threats_within_fov | succeed negated=1 → если ShortTermAttackMemory пусто: берёт Threats (AllBots, input_domain = "AllBots" — и в CS:GO, и в CS2) → reaction time → visible → удаляет из Threats. Side-effect, всегда failure |
| bt_select_nearby_threat | selector[succeed negated=1 → remove_key NearestThreatMemory; succeed negated=1 → Threats (AllBots) → reaction time → visible (check_fov=0) → ranker_dist → max_score → если есть: memory → NearestThreatMemory]. Всегда failure, но обновляет NearestThreatMemory |
| bt_memorize_threats | Сенсор DAMAGE → reaction time → Threats (AllBots) |

**14.1.1.** CS2-деревья с живой buy-веткой — **Deathmatch и Rush** (а также practice/bt_default — см. §14.4.2):

| Дерево | Ветки |
| --- | --- |
| deathmatch/bt_default.kv3 | Buy: decorator_buy_service output = "ShouldBuy" → condition_is_empty ShouldBuy (negated) → action_buy → decorator_remove_key ShouldBuy (ShouldBuy реально читается) |
| rush/bt_default.kv3 | Buy: та же цепочка (ShouldBuy читается); Antenna: decorator_try_lock domain = "'PressAntennaButton'" → движение к ant.button по targetname → нажатие; Hunt: выбор зоны охоты + hostage-сенсор с append_items = 1 |
| practice/bt_default.kv3 | Та же buy-цепочка; от CS2 Deathmatch отличается только отсутствием hostage-сенсора |

### 14.2. Co-op (CS:GO)

| Модуль | Что делает |
| --- | --- |
| bt_basic_player_engagement | selector[throw_utility_if_covered, attack, face_damage_inflictor, parallel[flee_area_damage_threats, repeat look_around], attack_damage_inflictor_smoke, clear_threats_within_fov, select_nearby_threat, heal_if_needed] |
| bt_basic_player_memorization | parallel[memorize_enemies_vision, memorize_noises, memorize_nearest_investigation, memorize_area_damage_grenades, memorize_area_damage_infernos, memorize_damage, memorize_threats, memorize_area_damage_current] |
| bt_determine_rush_center | succeed negated=1 → selector[condition_is_equal LastCoopSpawnPointName='enemy.waveXX' → parallel[set_value_vector RushCenterLocation, set_value_float RushCenterRadius]]. Всегда failure, но ставит центр раша |
| bt_throw_utility_if_covered | sequencer[preconditions (reaction time, ≥2 врага в EngagedEntities, есть hegrenade/incgrenade), maybe 0.15 → sequencer[aim_projectile, equip_item, wait 0.5, look_at, pull_trigger, wait 0.5]] |
| bt_guard | parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement, Investigate (look at), Investigate (distance_less 450–550), go back to spawn + look around + maybe say "Clear" (0.7) + wait 4–8]] |
| bt_roam | parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement, Investigate, roam (choose_random_waypoint_within_radius LastCoopSpawnPointLocation 700, move_to, maybe say "Clear" (0.7), wait 1–3)]] |
| bt_rush | parallel[repeat BasicPlayerMemorization, repeat selector[DetermineRushCenter, condition_is_empty RushCenterLocation → action_jump, BasicPlayerEngagement, Investigate, Rush (choose_random_waypoint_within_radius RushCenterLocation RushCenterRadius, move_to)]] |
| bt_sniper | parallel[repeat BasicPlayerMemorization, repeat selector[BasicPlayerEngagement]] |

**14.2.1.** Co-op — сравнение деревьев:

| Дерево | Investigate | Движение | Радио |
| --- | --- | --- | --- |
| bt_guard | Да, только если событие в 450–550 юнитах; иначе смотрит на событие (если видимо) | Возврат на LastCoopSpawnPointLocation | maybe say "Clear" (0.7), wait 4–8 |
| bt_roam | Да | choose_random_waypoint_within_radius LastCoopSpawnPointLocation 700 | maybe say "Clear" (0.7), wait 1–3 |
| bt_rush | Да | choose_random_waypoint_within_radius RushCenterLocation RushCenterRadius; если RushCenterLocation пуст — action_jump (fallback) | — |
| bt_sniper | Нет | Нет (стоит на месте) | — |

### 14.3. Guardian (CS:GO)

| Модуль | Что делает |
| --- | --- |
| bt_item_strategy | Покупка/выдача предметов по токенам стратегий |
| bt_takeover_strategy | Захват позиции по токенам стратегий. В Practice — селектор из 10 subtree (t_1…t_5, ct_1…ct_5) |
| bt_attack_with_shield | Атака со щитом (условие дальности — см. §7.2) |
| bt_investigate_with_shield | Исследование со щитом |
| bt_coordinated_buy | Скоординированная покупка (два известных бага + одно предупреждение, см. §8.6.2) |
| bt_strategy_smoke, bt_strategy_grenades | Стратегии дымов/гранат |
| bt_strat_hold_position | Параметризованное удержание позиции — единственный получатель subtree params (см. §5.1) |
| action_assign_guardian_loadout | Раздача лодаутов по волнам в item_strategy |
| action_throw_utility | Прямые броски дымов/молотовых — только в op11_county_2 (39 вызовов); в sirocco_2/4/5 вместо него паттерн move_to → look_at(input_angles) → equip → wait → pull_trigger |

### 14.4. Practice (newplayer_dust2_01, CS2)

| Модуль | Что делает |
| --- | --- |
| bt_ct_1 … bt_ct_5 | Стратегии CT по группам (GroupId = 'ct_1' … 'ct_5'). В раунде 1 все идут на B, в раундах 2–12 — разные позиции в зависимости от группы |
| bt_t_1 … bt_t_5 | Стратегии T по группам. Раунд 1 — long doors, раунд 2 — long, раунд 3 — cat, раунд 4–5 — B, и т.д. — полная таблица в §22.4 |
| bt_takeover_strategy | Селектор из 10 subtree: t_1…t_5, ct_1…ct_5 (диспетчер по GroupId) |
| bt_goto_a_site, bt_goto_b_site_from_ct, bt_goto_a_corner, bt_goto_mid_b, bt_goto_a_car, bt_goto_b_car_from_ct, bt_goto_b_site_from_t | Точки назначения (координаты вшиты) |
| bt_wait | action_wait 4.5–5.5 с |
| bt_acquire_t_weapons / bt_acquire_ct_weapons | Round-based loadout (см. §22) |
| bt_item_strategy | decorator_run_once → action_buy |

**14.4.1.** Practice-корень newplayer_dust2_01 (полная схема):

```
decorator_hiding_spot_service (обёртка — как в Co-op!)
└── parallel
    ├── token_service: 10 токенов (t_1..t_5, ct_1..ct_5),
    │   assignments = [[1,1,1,1,1,1,1,1,1,1]], team_restriction T/CT
    ├── buy_service (ShouldBuy пишется, но НЕ читается — покупка через item_strategy!)
    ├── repeat → memorization (8 модулей)
    └── repeat → selector
        ├── item_strategy (run_once → action_buy)
        ├── attack
        ├── face_damage
        ├── attack_damage_inflictor_smoke
        ├── clear_threats
        ├── select_nearby_threat
        ├── takeover_strategy (10 subtree по GroupId)
        ├── по TeamName:
        │   T  → plant_if_covered / plant_if_owned / pickup_bomb
        │   CT → defuse
        └── investigate …
```

**14.4.2.** practice/bt_default.kv3 — второе Practice-дерево (в предыдущих редакциях вики не упоминалось):

```
decorator_bot_service
└── buy_service
└── parallel
    ├── repeat → memorization (3 модуля)
    └── repeat → selector
        ├── buy: ShouldBuy (negated) → wait → action_buy → remove_key ShouldBuy  ← ЖИВАЯ ветка!
        ├── damage / attack / heal_if_needed
        ├── investigate
        └── hunt
```

Фактически это CS2 Deathmatch-дерево без hostage-сенсора (единственный содержательный diff). Отличие от newplayer_dust2_01: там ShouldBuy пишется, но не читается (покупка через item_strategy), здесь buy-ветка живая, как в deathmatch/rush.

### 14.5. Дополнительные модули Guardian (CS:GO)

| Модуль | Где | Что делает |
| --- | --- | --- |
| bt_bot_1 … bt_bot_5 | op11_county_2 | Пер-ботовые волновые стратегии: GroupId = 'Bot1'…'Bot5' + GuardianWaveNumber >= 5 → телепорты и subtree (Firebomb, SmokeHouseRush, …). Токены Bot1…Bot5 раздаёт decorator_token_service корня |
| bt_attack_example / bt_utility_example | blacksite_1/3, county_2, sirocco_5 | Шаблоны-заготовки: селектор по GroupId = 'AttackExample', decorator_run_once max_attempts = 1, teleport + wait. Идеальный образец пер-ботовой стратегии |
| bt_parachute_1 | op11_blacksite_1 | 40 condition_is_equal с GroupMemberId = 'ParachuteN' и domain = "'ParachuteN'". См. §20 |

## 15. Паттерны

**15.1.** Восприятие + решение через parallel

```
parallel
├── repeat → parallel (sensors, memory)
└── repeat → selector (priority list)
```

**15.2.** Selector как список приоритетов

```
selector
├── condition_is_empty ShouldBuy (negated) → sequencer[wait, buy, remove_key]
├── sensor DAMAGE → aim
├── subtree Attack
├── subtree HealIfNeeded
└── subtree Hunt
```

**15.3.** Блокировка действия одним ботом

```
decorator_try_lock
domain = "'PressAntennaButton'"
child = sequencer[ move_to ant.button, look_at, wait, use ]
```

Полный список доменов — в §23.

**15.4.** Side-effect без остановки selector

```
decorator_succeed
negated = 1
child = sequencer[ ... сделать что-то ... ]
```

Selector увидит failure и пойдёт дальше, но side-effect уже выполнен.

**15.4.1.** Полный список модулей, построенных на этом паттерне:

| Модуль | Что делает | Всегда возвращает |
| --- | --- | --- |
| bt_clear_threats_within_fov | Удаляет из Threats видимые угрозы | failure (decorator_succeed negated=1) |
| bt_select_nearby_threat | Обновляет NearestThreatMemory | failure |
| bt_memorize_nearest_investigation | Заполняет ShortTermInvestigateMemory | success (decorator_succeed без negated) |
| bt_memorize_threats | Пишет урон в Threats (AllBots) | success |
| bt_determine_rush_center | Пишет RushCenterLocation/RushCenterRadius | failure (decorator_succeed negated=1) |

**15.5.** Истечение памяти в корне

```
decorator_bot_service
memory_to_expire =
[
    { key = "ShortTermAttackMemory" time = 0.7 distance = 0 }
    { key = "LongTermMemory" time = 10 distance = 500 }
    { key = "ShortTermInvestigateMemory" time = 3 distance = 200 }
]
```

**15.6.** Переэкипировка без остановки атаки

Из bt_attack.kv3:

```
decorator_repeat
child = decorator_succeed
child = condition_out_of_ammo
child = action_equip_weapon weapon = "BEST"
```

«Если патроны кончились — переэкипировать BEST, и всегда возвращать success, чтобы decorator_repeat не останавливался».

**15.7.** decorator_maybe + NumberOfAliveTerrorists

Из bt_plant_bomb_if_owned.kv3:

```
decorator_maybe
chance = 0.5
child = decorator_succeed
child = condition_is_equal
source = "NumberOfAliveTerrorists"
destination = 1
child = sequencer[ wait 0.2–0.3, action_set_global_flag DontPlantBomb 3–5 ]
```

«Если я последний живой T, с шансом 50% не закладывать бомбу, а подождать».

**15.8.** Точная расстановка: teleport → move_to → look_at

Из op10_ancient_a/bt_cross_smoke.kv3 (не cover_smoke — там другие координаты):

```
action_teleport destination = "-1803.901367 -271.996948 72.083061"
action_move_to destination = "-1906.750732 169.041382 76.031250" ... auto_look_adjust = 1
action_wait 1
action_look_at input_angles = "-2.191546 136.888290 0.000000"
action_equip_weapon weapon = "weapon_smokegrenade"
action_wait 1.5
action_pull_trigger
action_wait 0.5
action_move_to destination = "-1881.802856 1085.127563 52.031250" ... damaging_areas_penalty_cost = 0.0
```

damaging_areas_penalty_cost = 0.0 — бот идёт сквозь молотов, чтобы успеть на позицию.

В Guardian для юнит-стратегий типичен паттерн с телепортом: сначала action_move_to до опорной точки (иногда с arrival_epsilon = 0.1–0.2), затем action_teleport в точную позицию (гасит погрешность навигации), затем action_look_at input_angles → equip → wait → pull_trigger с нужных углов. action_teleport используется для точной расстановки ботов перед стратегией ×449 в CS:GO Guardian. После телепорта всегда ставьте явный action_look_at — auto_look_adjust сам не сбрасывается, и телепорт не влияет на камеру.

**15.9.** decorator_picker_max_score + decorator_ranker_dist

Связка:

```
decorator_ranker_dist
input = "Enemy"
child = decorator_picker_max_score
input = "Enemy"
```

decorator_ranker_dist присваивает score обратно пропорционально дистанции. decorator_picker_max_score берёт максимальный score → ближайшую сущность. Без ранкера max_score берёт первую сущность массива (порядок появления на сервере; значение score не документировано) — не забывайте ранкер.

**15.10.** Диагональное движение с парашютом

См. §20.

**15.11.** Готовый образец пер-ботовой стратегии

bt_attack_example / bt_utility_example (blacksite_1/3, county_2, sirocco_5) — минимальный шаблон:

```
selector
├── condition_is_equal GroupId = 'AttackExample'
└── sequencer[
      decorator_run_once max_attempts = 1,
      teleport,
      wait
    ]
```

Это официальный образец того, как привязать действие к конкретному боту через GroupId/GroupMemberId из токен-сервиса.

**15.12.** Jump-throw паттерн (Guardian, CS:GO)

action_jump — 25 активных вхождений в 22 CS:GO-файлах (плюс 10 закомментированных), и почти все — утилити-стратегии op10/op11: jump-throw молотовых и смоков, которые нельзя кинуть с земли (ancient, mirage, nuke_b, overpass, train, vertigo, sandwich_molly, blacksite molly_one/two/smoke_two). Схема:

```
sequencer[
    move_to (точная позиция, иногда teleport),
    look_at (input_angles),
    equip (граната),
    wait,
    action_jump,        // прыжок
    pull_trigger        // бросок в воздухе — граната летит дальше/выше
]
```

## 16. Подводные камни

| ❌ Не делайте | ✅ Делайте вместо этого |
| --- | --- |
| Полагаться на acquire_only, ratio, distance_threshold (blocked_by_smoke) | Эти параметры движком не читаются (см. §26.1) |
| Использовать action_coordinated_buy без decorator_run_once | decorator_run_once domain = "'CoordinatedBuy'" + ожидание через decorator_wait_success (два известных бага + предупреждение — §8.6.2) |
| Забывать mp_bot_ai_bt_clear_cache | Всегда чистить кэш после правок |
| Передавать float/bool в subtree params | Только int/string (векторы — строкой); официальный образец — op09_grunts_card2 |
| Использовать 20 строк в attack для CS2 | Ровно 13, в фиксированном порядке; 20-строчный конфиг в CS2 не загрузится |
| Использовать 13 строк в attack для CS:GO | Ровно 20 во всех CS:GO-конфигах (coop, deathmatch, guardian) |
| Ожидать, что action_aim разскопится | Смените оружие или экипируйте то же заново (action_equip_weapon сбрасывает прицел) |
| Полагаться на condition_is_greater/is_less | Используйте greater_equal/less_equal с negated |
| Использовать decorator_invert в CS2 | Используйте negated = 1 |
| Забывать, что action_attack без ready стреляет сразу | Указывайте ready, если нужна синхронизация с action_aim |
| Ожидать, что action_aim_projectile точно бросит гранату | Баг: полностью игнорирует гравитацию/физику. Используйте action_throw_utility или скриптовый look_at-паттерн |
| Использовать decorator_picker_blocked_by_smoke без понимания | Он оставляет только сущности за дымом |
| decorator_picker_max_score без ранкера | Вернёт первую сущность массива, а не «лучшую»; ставьте decorator_ranker_dist перед ним |
| decorator_sensor team_filter = "ENEMY" для гранат | Баг: вернёт гранаты T независимо от команды; используйте OPPOSITE |
| Думать, что action_acquire_items не работает | ✅ Работает; встречается в Practice/Guardian |
| Думать, что action_teleport не работает | ✅ Работает; ×449 в CS:GO Guardian. Но не сбрасывает камеру — ставьте action_look_at |
| Думать, что action_crouch не работает | ✅ Работает; встречается в op09_grunts_card2 и op10_engage_b (bt_boost_molly) |
| Игнорировать decorator_dec_global_counter | ✅ Встречается в op09_grunts_card7_2 |
| Писать domain = PressAntennaButton без кавычек | В файлах Valve: domain = "'PressAntennaButton'" (исключение: "'AllBots'" в op11_county_2) |
| Унифицировать все domain под "AllBots" | Смотрите конкретный узел: где-то "AllBots", где-то "'StratTokens'" |
| Использовать decorator_find_utility_strat, decorator_route_service, decorator_set_barrier | Эти узлы не встречаются в официальных деревьях, поведение не подтверждено |
| Путать action_hide и decorator_hiding_spot_service | «Укрыться» = decorator_hiding_spot_service + action_move_to hiding_spot = "HidingSpot" |
| Ожидать BT_ACTION_MOVETO_SAFEST_ROUTE | Используется только BT_ACTION_MOVETO_FASTEST_ROUTE; движение — RUN/WALK |
| action_jump при movement_type = WALK | Баг: после прыжка тип движения сбрасывается на RUN — переустанавливайте WALK после |
| Считать, что decorator_buy_service всегда влияет на поведение | В Arms Race и newplayer_dust2_01 ShouldBuy пишется, но никогда не читается (см. §16.5); читается — в deathmatch, rush и practice/bt_default |
| Надеяться, что опечатка в параметре вызовет ошибку | Движок молча игнорирует чужие для узла параметры, без предупреждений (см. Warning в §3.4) — сверяйте имена ключей с §6–8 |
| Копировать закомментированные блоки из op10/11-корней | В корнях 62 закомментированных subtree (в т.ч. parallel[FleeAreaDamageThreats, LookAround] после Item Strategy — активный только в Co-op autumn); проверяйте, что копируете |
| Оборачивать action_move_to в decorator_repeat «на всякий случай» | Не нужно: пока бот не дошёл, узел возвращает running, и sequencer остаётся на нём (§9.4). repeat — только для повторного запуска завершившегося действия |

**16.5.** Мёртвый ShouldBuy в Arms Race и newplayer_dust2_01

В armsrace/bt_default.kv3 и в practice/newplayer_dust2_01.kv3 есть decorator_buy_service output = "ShouldBuy", но ветки покупки в selector нет. ShouldBuy вычисляется и никогда не читается — в newplayer_dust2_01 покупка идёт через bt_item_strategy = run_once → action_buy. ShouldBuy реально читается в deathmatch, rush и practice/bt_default (§14.4.2).

Это отличный пример того, что не всякий side-effect в дереве означает работающую механику. Копируя дерево как шаблон, проверяйте, что выход сервиса действительно читается ниже.

---

# Часть V. Справочники

## 17. Скриптовые установки гранат

> **Note**
>
> Раздел основан на CS:GO Guardian-дампах. В CS2 All-in-One action_throw_utility отсутствует (0 вхождений) — но CS2-дамп неполный (Guardian/Co-op-контента в нём нет), поэтому это не доказательство отсутствия механики в CS2. «0 в CS2-дампе» ≠ «нет в CS2».

**17.1.** action_throw_utility — одно из самых мощных действий в Guardian (CS:GO). Оно позволяет спавнить гранату в произвольной мировой позиции с заданной скоростью, независимо от того, где стоит бот.

**17.2.** Синтаксис:

```
action_throw_utility
name = "weapon_incgrenade"
location = "2333 -4365 -1465"
velocity = "0 0 -600"
```

| Поле | Смысл |
| --- | --- |
| name | classname гранаты (weapon_incgrenade, weapon_smokegrenade, weapon_hegrenade и т.д.) |
| location | Мировая позиция, куда спавнится граната (не позиция бота) |
| velocity | Начальная скорость гранаты. "0 0 -600" — падение вниз |

**17.3.** Пример: стена огня из 7 incendiary-гранат (bt_firebomb.kv3):

```
type = "sequencer"
children =
[
    { type = "action_throw_utility" name = "weapon_incgrenade" location = "2333 -4365 -1465" velocity = "0 0 -600" }
    { type = "action_wait" wait_time_min = 0.15 wait_time_max = 0.25 }
    { type = "action_throw_utility" name = "weapon_incgrenade" location = "2000 -4365 -1465" velocity = "0 0 -600" }
    { type = "action_wait" wait_time_min = 0.15 wait_time_max = 0.25 }
    { type = "action_throw_utility" name = "weapon_incgrenade" location = "1666 -4365 -1465" velocity = "0 0 -600" }
    ...
]
```

**17.4.** Пример: тройная стена огня (bt_firebomb_wide.kv3) — 18 incendiary, не 21. Это 6 X-позиций (2333…600; X=250 из обычного firebomb сюда не входит) × 3 Y-значения (−4365/−4650/−4150). Обычный bt_firebomb = 7 ✓.

**17.5.** Пример: кольцо дымов (bt_smoke_ring.kv3) — 8 дымов по кругу вокруг центра ≈ (1390 -4405) на высоте Z = 850; кольцо неровное (радиус ~100–215 юнитов).

**17.6.** Где используется — исключительно op11_county_2, 39 вызовов в 5 файлах:

| Файл | Вызовов | Что |
| --- | --- | --- |
| op11_county_2/bt_firebomb.kv3 | 7 | Стена огня из 7 incendiary |
| op11_county_2/bt_firebomb_wide.kv3 | 18 | Тройная стена огня |
| op11_county_2/bt_smoke_ring.kv3 | 8 | Кольцо дымов |
| op11_county_2/bt_smoke_house_rush.kv3 | 3 | 3 дыма для раша дома |
| op11_county_2/bt_smoke_houses.kv3 | 3 | 3 дыма для домов |

**17.7.** Это скриптовые «арт-установки», а не обычный бросок. В Co-op используется action_aim_projectile (с багом — игнорирует гравитацию), в Guardian — action_throw_utility. При этом в sirocco_2/4/5 (Guardian) throw_utility не используется — там те же результаты получают «вручную»: move_to в точную позицию → look_at(input_angles) → equip → wait → pull_trigger.

## 18. Координация через глобальные счётчики

> **Note**
>
> Пример — CS:GO Guardian (op09_grunts_card7_2).

**18.1.** Полный пример из op09_grunts_card7_2/bt_coordinated_buy.kv3:

```
decorator_run_once
domain = "'CoordinatedBuy'"
child = sequencer
children =
[
    { type = "action_set_global_counter" input_name = "'WaitForCoordinatedBuy'" input_value = 1 }
    { type = "action_set_global_counter" input_name = "'RushSite'" input_value = 1 }
    { type = "action_wait" wait_time_min = 3 wait_time_max = 3 }
    { type = "decorator_succeed" child = sequencer
        children =
        [
            { type = "action_coordinated_buy" team_filter = "SAME" save_threshold = 2000 id_no_purchase = "DidNotPurchase" purchases = [ ... ] }
            { type = "action_set_global_counter" input_name = "'RushSite'" input_value = 0 }
        ]
    }
    { type = "action_set_global_counter" input_name = "'WaitForCoordinatedBuy'" input_value = 0 }
]
```

**18.2.** Как это работает:

| Шаг | Что происходит |
| --- | --- |
| 1 | Первый бот берёт decorator_run_once (домен 'CoordinatedBuy' занят) |
| 2 | Устанавливает WaitForCoordinatedBuy = 1 |
| 3 | Устанавливает RushSite = 1 |
| 4 | Ждёт 3 секунды |
| 5 | Выполняет action_coordinated_buy |
| 6 | Сбрасывает RushSite = 0 |
| 7 | Сбрасывает WaitForCoordinatedBuy = 0 |
| 8 | Остальные боты видят WaitForCoordinatedBuy = 1 и идут в decorator_wait_success timeout = 4 → action_compare_global_counter WaitForCoordinatedBuy = 0 |

**18.3.** bt_strategy_smoke.kv3 (card13) использует ту же схему с счётчиками PreparingToThrowSmoke, ThrowPalaceFlash, PositioningForTopSmoke.

**18.4.** decorator_wait_success + action_compare_global_counter — стандартная пара для «ждать, пока другой бот что-то сделает». Связка run_once (domain) → set_global_counter → wait_success → compare_global_counter → dec_global_counter — универсальный каркас координации, формализованный здесь полностью.

## 19. Паттерны-предохранители

**19.1.** condition_inactive + action_commit_suicide

Из Guardian (CS:GO) — по 50 вхождений каждого узла, все в Guardian-корнях:

```
condition_inactive
input = [ { key = "ShortTermAttackMemory" }, { key = "ShortTermAreaDamageMemory" } ]
round_start_threshold_seconds = 90
sensor_inactivity_threshold_seconds = 60
child = action_commit_suicide
```

«Если 90 секунд прошло с начала раунда и 60 секунд не было ни атаки, ни урона по площади — бот убивает себя». Предохранитель от зависших раундов.

**19.2.** Флаг DontPlantBomb

Из bt_plant_bomb_if_owned.kv3:

```
action_set_global_flag
name = "'DontPlantBomb'"
expiration_time_min = 3
expiration_time_max = 5
```

Бот ставит флаг, чтобы другие боты не закладывали бомбу 3–5 секунд. Проверяется через condition_is_empty global = 1 input = "DontPlantBomb".

**19.3.** decorator_maybe + NumberOfAliveTerrorists

См. §15.7.

**19.4.** decorator_succeed вокруг action_coordinated_buy

В bt_coordinated_buy.kv3 action_coordinated_buy обёрнут в decorator_succeed — даже если покупка провалилась, дерево продолжится. Покупка не должна ломать дерево. Тот же приём стоит вокруг любых «рискованных» действий, чей провал не должен ронять ветку.

## 20. Парашют (Blacksite, CS:GO Guardian)

**20.1.** Уникальная механика Guardian на карте Blacksite (в файлах папка называется op11_blacksite_* — «blacksite», не «blacksire»). Боты телепортируются в воздух, получают парашют, и action_parachute_positioning рулит диагональным движением. В CS2-дампе этого контента нет (дамп неполный).

**20.2.** Условия и действия:

| Узел | Что делает |
| --- | --- |
| condition_has_parachute | Есть ли парашют |
| condition_is_airborne | Бот в воздухе |
| action_parachute_positioning | Движение по диагонали для парашюта; периодически меняет направление (каждые несколько секунд) |
| action_acquire_items items = [ "parachute" ] | Выдать парашют |

**20.3.** Пример из op11_blacksite_1/bt_default.kv3:

```
condition_has_parachute
child = condition_is_airborne
child = parallel
children =
[
    { type = "action_parachute_positioning" }
    { type = "action_look_at" input_location = "4363.661621 5653.200195 735.644348" }
]
```

**20.4.** action_look_at input_location фиксирует взгляд на точке X (в дампе — с комментарием // also look towards the bomb X mark). Это нужно, чтобы бот приземлился в нужном месте.

**20.5.** Токен-сервис парашюта. В корне bt_op11_blacksite_1 токен один:

```
tokens = [ "Parachute" ]
assignments = [ [ 1000 ] ]
```

Число 1000 означает, что движок попытается выдать этот токен до 1000 ботам — то есть всем доступным. Это стандартный трюк Valve для режима «всем подряд».

Никаких 40 токенов нет. Parachute1…Parachute40 — это значения GroupMemberId. 🔬 Судя по дампам, GroupMemberId формируется движком как <имя токена><N>: токен один, assignments = [[1000]], а в bt_parachute_1.kv3 проверяются GroupMemberId = 'Parachute1'…'Parachute40' (40 condition_is_equal). ⚠️ Официально механика генерации GroupMemberId не документирована — это вывод из дампов, а не задокументированный факт.

## 21. Карты Guardian (CS:GO-дампы)

**21.1.** Полный список карт в Guardian-дампах (CS:GO; список модулей частичный — полная таблица по 519 файлам выходит за рамки документа):

| Карта | Модули | Особенности |
| --- | --- | --- |
| op09_grunts_card1, card1_2, card2, card4, card6_1, card6_2, card7_1, card7_2, card8, card9, card10, card11, card12, card13, card14, card15 | Базовые grunts-стратегии | Токен-сервисы, action_teleport, action_crouch; card2 — subtree params; card1 — BombSiteArea |
| op10_ancient_a/b | Ancient A/B | 7 смоков/молотовых + LongAwp; в ancient_b — LeftSmoke/RightSmoke/TunRush; в bt_long_awp — закомментированный // type = "action_RIGHT_CLICK?" |
| op10_apollo | Apollo | Пистолетные раунды: assign_guardian_loadout по волнам; все лодауты — пистолеты (usp/hkp2000 → elite/tec9/fiveseven/cz75a → revolver/deagle, волны 1–4) |
| op10_dust2_a/b | Dust2 A/B | a: Goose, CatSmoke, CrossSmoke, car_molly, goose_molly; b: Car/Doors/Plat/Window, car_smoke, doors_smoke, plat_smoke, window_molly/smoke |
| op10_engage_a/b | Engage A/B | 5+5 молотовых + rush; в engage_a — floor_molly_1..5, top_molly_1..5, top_rush_1..5; bt_boost_molly использует action_crouch |
| op10_inferno_a/b | Inferno A/B | a: ASiteFlash, BalconySmoke, DefaultASmoke, MotoSmoke, PitSmoke, CarMolly, SandwichMolly, BananaHE, BananaFlash, SmokeCoffins, SmokeCT, rush_apts/patio/arch (+delay); b: strat_b*, ct_church, ct_spawn, no_strat_b |
| op10_mirage_a/b | Mirage A/B | StairsSmoke, CTSmoke, JungleSmoke (×2), TripleSmoke, SiteSmoke, MainFlash, PalaceFlash, BenchSmoke, MarketSmoke, MarketWindowSmoke, RightArchSmoke, ArchMolly; в mirage_a — DarkSmoke, DarkMolly, SandwichMolly + rush-модули |
| op10_nuke_a/b | Nuke A/B | HeavenSmoke, HutFlash, MiniFlash, HutMolly, ControlSmoke; в nuke_a — RampSmokeRight, в nuke_b — RampSmokeLeft, dark_molly, ramp_flash |
| op10_overpass_a/b | Overpass A/B | a: AFlash, LongFlash, BankSmoke, DumpsterSmoke, LongSmokeTeleport, MiddleASmoke, TruckMolly, TruckSmoke; b: BarrelsMolly, BSiteFlash, BSiteSmoke, ConstructionSmoke(water), HeavenSmoke, MonsterFlash, MonsterSmoke |
| op10_train_a/b | Train A/B | LaneSmoke, RightRedSmoke, EBoxSmoke, IvySmoke, SafeSmoke, ConnSmoke, EarlyBSmoke, LeftBSiteSmoke, LowerBSmoke, BPopFlash, UpperPopFlash, UpperSafeSmoke |
| op10_vertigo_a/b | Vertigo A/B | AEntrySmoke, AFlash, LeftASmoke, RightASmoke, BShortFlash, CrossBSmoke, LeftBSmoke, MidPopFlash; в vertigo_a — HeadshotMolly |
| op11_basalt_a/b | Basalt A/B | CTSmoke, PlatSmoke, SiteMolly, CubbyMolly, EntryFlash; в basalt_b — DoorSmoke, SiteSmoke1/2, CornerMolly1/2, CubbyFlash |
| op11_blacksite_1 | Blacksite 1 | Парашют (Parachute1…Parachute40 через GroupMemberId); override зрения: сфера 25000 (см. §21.3) |
| op11_blacksite_2 | Blacksite 2 | EastSmoke, EastSmoke2, WestSmoke, WestSmoke2, WestFlash, EastFlash, WestMolly, RushWest; override зрения 1500→1850 |
| op11_blacksite_3 | Blacksite 3 | MollyStrat, HEStrat, TunnelStrat, BehindStrat (домены на 5 ботов, см. §23) |
| op11_blacksite_4 | Blacksite 4 | DivebombMolly, MollyOne, MollyTwo, SmokeTwo, TouristStairwell, TouristGround, TouristMarket, ConstructionGreen, ConstructionBlue, Echo1, DivebombBridge, DivebombStreet, Divebomb1, Divebomb2; override зрения 1500→1850 |
| op11_county_1 | County 1 | NorthShield, SouthShield, EastShield, NorthFlash, NorthFlash2, SouthFlash, SouthFlash2, WestFlash, WestFlash2, EastFlash |
| op11_county_2 | County 2 | Скриптовые установки гранат (firebomb, firebomb_wide, smoke_ring, smoke_house_rush, smoke_houses) + action_entity_accept_input; bt_bot_1..5; run_once domain = "'AllBots'" (исключение кавычек); override зрения 1500→4000 |
| op11_extraction | Extraction | CTSmoke, RoomSmoke, SiteMolly, EntryNade, EntryFlash |
| op11_insertion2_a/b | Insertion 2 A/B | EntryFlash, EntryFlash2, BackFlash, EntrySmoke, SideSmoke, DockSmoke, BackSmoke, RushEntry, RushSide, RushDock |
| op11_ravine | Ravine | CTSmoke, SiteSmoke, SiteMolly, CubbyMolly, EntryFlash, EntryFlash2 |
| op11_sirocco_1 | Sirocco 1 | RushNorth, RushEast, RushSouth, RushWest, EastSmoke, WestSmoke, WestNade, SouthFlash, RushIvy |
| op11_sirocco_2 | Sirocco 2 | NorthSmoke, SouthSmoke, NorthMolly, SouthMolly, RushBeach, RushOil, RushHill, RushApc; override зрения 1500→1850 |
| op11_sirocco_4 | Sirocco 4 | AttackLowerEast/North/West, AttackMidEast/West, AttackUpperEast/North/West, SmokeMidWest, SmokeUpperNorth, MollyUpperEast, FlashMidWest; override зрения 1500→4000 |
| op11_sirocco_5 | Sirocco 5 | AttackUpperS, AttackUpperSW, AttackMidS, AttackMidSE, AttackMidSW, AttackLowerS, AttackLowerSE, AttackLowerSW, FlashSite, SmokeBarrels, SmokeBarrels2, MollyBox; override зрения 1500→4000 |

**21.2.** decorator_hiding_spot_service — expiration_time:

| Значение | Где |
| --- | --- |
| 10 | Co-op (op11_insertion2_a/b, все Co-op-корни), CS2 practice-корни и остальные 48 Guardian-корней |
| 15 | Ровно три корня: bt_grunts_card13.txt (без префикса op09!), bt_op09_grunts_card1_2.txt, bt_op09_grunts_card6_1.txt |

**21.2.1.** condition_is_equal и domain:

condition_is_equal поддерживает незадокументированный ключ domain (string) — его нет в Source2 Wiki. В bt_parachute_1.kv3 ровно **41** условие condition_is_equal: одна проверка GroupId = "'Parachute'" без domain и **40** проверок GroupMemberId с domain = "'ParachuteN'" (N = 1…40). 🔬 Судя по семантике (совпадает с decorator_try_lock), это индивидуальный лок на каждого бота. ⚠️ Назначение не подтверждено официальной документацией; не переносите domain на другие условия без проверки.

**21.3.** Локальные override-ы bt_memorize_enemies_vision в op11:

Целый недокументированный паттерн «настройки зрения под карту». Общий модуль видит на 1500, локальные копии — дальше:

| Карта | Отличие от общего модуля |
| --- | --- |
| blacksite_2, blacksite_4, sirocco_2 | cutoff_distance 1500 → 1850 |
| county_2, sirocco_4, sirocco_5 | cutoff_distance 1500 → 4000 |
| blacksite_1 | sensor_shape_sphere radius = 25000 + cutoff_distance = 25000 — сенсор «видит» всю карту; LOS-фильтр остаётся в picker_visible |

**21.4.** Токен-сервисы Guardian/Practice:

| Карта | Токены | Assignments |
| --- | --- | --- |
| op10_ancient_a | 9 (CoverSmoke…LongAwp) | 5 вариантов раздачи |
| op09_grunts_card2 | 5 (StratLobby, StratOutside, StratHeaven, NoStratSqueaky, NoStartLobby) | См. §6.8.3; здесь же 15 использований subtree params |
| op09_grunts_card12 | 3 (StratAtrium, StratLobby, StratUpOffice) | 6 вариантов раздачи |
| practice/newplayer_dust2_01 | 10 (t_1..t_5, ct_1..ct_5) | [[1,1,1,1,1,1,1,1,1,1]] + team_restriction T/CT |
| op11_blacksite_1 | 1 (Parachute) | [[1000]] (см. §20.5) |
| op11_county_2 | Bot1…Bot5 (в корне) | См. §14.5 |

## 22. Practice: полная таблица раундов

**22.1.** bt_acquire_t_weapons и bt_acquire_ct_weapons — round-based loadout.

**22.2.** CT loadout (bt_acquire_ct_weapons.kv3):

| Раунд | Что покупается |
| --- | --- |
| 1 | weapon_hkp2000 |
| 2 | item_kevlar, weapon_p250 |
| 3 | item_kevlar, weapon_mp9 |
| 4–6 | item_assaultsuit, weapon_p90 |
| 7 | weapon_hkp2000 (с remove_all_items = 1 — сброс!) |
| 8 | item_kevlar, weapon_p250 |
| 9 | item_kevlar, weapon_mp9 |
| 10+ | item_assaultsuit, weapon_p90 |
| Fallback | Если слот PISTOL пуст — weapon_p2000 (с remove_all_items = 0) |

**22.3.** T loadout (bt_acquire_t_weapons.kv3):

| Раунд | Что покупается |
| --- | --- |
| 1 | weapon_glock |
| 2 | item_kevlar, weapon_p250 |
| 3 | item_kevlar, weapon_mac10 |
| 4–6 | item_assaultsuit, weapon_p90 |
| 7 | weapon_glock (с remove_all_items = 1 — сброс!) |
| 8 | item_kevlar, weapon_p250 |
| 9 | item_kevlar, weapon_mac10 |
| 10+ | item_assaultsuit, weapon_p90 |
| Fallback | Если слот PISTOL пуст — weapon_glock (с remove_all_items = 0) |

**22.4.** T стратегии по раундам (исправлено по дампам bt_t_1…bt_t_5):

| Раунд | t_1, t_2 | t_3 | t_4, t_5 |
| --- | --- | --- | --- |
| 1 | bt_wait → A-бомбсайт через long doors | то же | то же |
| 2 | bt_wait → A через long doors | то же | то же |
| 3 | bt_wait → A через cat | то же | то же |
| 4–5 | B site | B site | B site |
| 6–8 | long | cat | B |
| 9 | long | long | long |
| 10 | cat | cat | cat |
| 11 | B | B | B |
| 12 | long | cat | B |

> **Note**
>
> Структура каждого bt_t_N (по дампам): condition_is_equal GroupId = "'t_N'" → decorator_run_once (max_attempts = 1) → selector из 12 ветвей condition_is_equal RoundNumber = 1…12. В раундах 1–3 перед стратегией стоит bt_wait, с раунда 4 — сразу goto.
>
> bt_t_1 и bt_t_2 во **всех** раундах имеют одинаковые стратегии — отличаются только значением GroupId (обе ведут на long). В раунде 9 все пять групп идут через long; в раундах 6–8 и 12 t_3 идёт cat, t_4/t_5 — B.

**22.5.** CT стратегии по раундам:

| Раунд | Группы CT | Что делают |
| --- | --- | --- |
| 1 | ct_1–ct_5 | bt_goto_b_site_from_ct → bt_wait × 2 |
| 2–12 | ct_1 | bt_goto_a_site |
| 2–12 | ct_2 | bt_goto_b_site_from_ct |
| 2–12 | ct_3 | bt_goto_mid_b |
| 2–12 | ct_4 | bt_goto_a_corner |
| 2–12 | ct_5 | bt_goto_b_car_from_ct |

**22.6.** bt_wait — action_wait 4.5–5.5 с.

**22.7.** bt_item_strategy — decorator_run_once → action_buy.

## 23. Полный список доменов

| Домен | Игра | Где | Что блокирует |
| --- | --- | --- | --- |
| "AllBots" (без внутренних кавычек) | обе | memory_to_expire, input_domain, output_domain, decorator_run_once | Общая память всех ботов; блокировка для run_once |
| "'AllBots'" (с внутренними кавычками — исключение!) | CS:GO | op11_county_2, волна 1, decorator_run_once перед action_entity_accept_input для дверей | Единственный случай с кавычками для AllBots; только один бот жмёт все кнопки дверей |
| "'StratTokens'" | обе | decorator_token_service | Токены стратегий |
| "'PressAntennaButton'" | CS2 | rush/bt_default | Нажатие кнопки антенны |
| "'DefuseBombIfNearby'" | CS2 | practice/modules/bt_defuse_bomb_if_nearby | Разминирование |
| "'PickupBombIfNearby'" | обе | practice/modules/bt_pickup_bomb_if_nearby | Подбор бомбы |
| "'PickupHostage'" | CS:GO | op11_insertion2_a/b/bt_pickup_hostage | Подбор хостажа |
| "'CoordinatedBuy'" | CS:GO | op09_grunts_card7_2/bt_coordinated_buy | Координированная покупка |
| "'SetupGlobalCounters'" | CS:GO | decorator_run_once в card7_2 | Установка счётчиков |
| "'StrategySmoke'" | CS:GO | op09_grunts_card13/bt_strategy_smoke | Стратегический дым |
| "'StrategyGrenade1/2/3'" | CS:GO | op09_grunts_card13/bt_strategy_grenades | Стратегические гранаты |
| "'HoldBackOnFirstWaveBot1/2/3'" | CS:GO | op09_grunts_card6_2 | Задержка ботов на первой волне |
| "'TunnelStrat1'"…"'TunnelStrat5'" | CS:GO | op11_blacksite_3 | Пер-ботовый туннель (по домену на каждого из 5 ботов) |
| "'MollyStrat1'"…"'MollyStrat5'" | CS:GO | op11_blacksite_3 | Пер-ботовая молотов (5 ботов) |
| "'HEStrat1'"…"'HEStrat5'" | CS:GO | op11_blacksite_3 | Пер-ботовая HE (5 ботов) |
| "'BehindStrat1'"…"'BehindStrat5'" | CS:GO | op11_blacksite_3 | Пер-ботовый заход с тыла (5 ботов) |
| "'ParachuteN'" (N = 1..40) | CS:GO | bt_parachute_1.kv3 — condition_is_equal domain, не try_lock! | Индивидуальный «лок» на бота; 🔬 семантика не подтверждена |

> **Warning**
>
> AllBots без внутренних одинарных кавычек в memory_to_expire, input_domain, output_domain, decorator_run_once (кроме одного исключения в op11_county_2). 'StratTokens', 'PressAntennaButton' и т.д. — с ними. Не унифицируйте.

## 24. Быстрый справочник: aim_projectile vs throw_utility

| | action_aim_projectile | action_throw_utility |
| --- | --- | --- |
| Использует позицию бота | Да | Нет (спавнит в location) |
| Учитывает траекторию | Нет — полностью игнорирует гравитацию и физику (баг) | Задаётся вручную через velocity |
| Где используется | Co-op bt_throw_utility_if_covered (CS:GO) | Только op11_county_2 (CS:GO Guardian), 39 вызовов |
| Точность | Низкая | Высокая (скриптовая) |
| Пишет в blackboard | Углы (output) | Ничего |
| Читает из blackboard | input (позиция) | — |
| Когда использовать | Нужен «естественный» бросок от бота | Нужна точная установка дыма/огня/молотова |

> **Note**
>
> Третий путь — «ручной» скриптовый паттерн из sirocco_2/4/5 (Guardian): move_to в точную позицию → look_at(input_angles) → equip → wait → pull_trigger. Даёт точность throw_utility без спавна гранаты «из воздуха».

---

# Часть VI. Приложения

## 25. Инвентарь узлов и частоты

Все счётчики получены программным подсчётом по самим дампам (awk по строкам `type = "..."`; закомментированные строки отфильтрованы и посчитаны отдельно). Формат: **активные** (+ закомментированные) вхождения.

### 25.1. Объёмы дампов

| Дамп | Файлов | Узлов (активных) | Уникальных типов | Закомментированных |
| --- | --- | --- | --- | --- |
| BT CSGO All-in-One | 544 (coop 9, deathmatch 2, guardian 519, modules 14) | 8974 | 82 | 143 |
| BT CS2 All-in-One | 57 (armsrace 2, deathmatch 2, modules 14, practice 37, rush 2) | 876 | 60 | 0 |

### 25.2. Композиты и subtree

| Узел | CS:GO | CS2 |
| --- | --- | --- |
| parallel | ×359 (+31 закомм.) | ×16 |
| selector | ×477 | ×37 |
| sequencer | ×585 | ×156 |
| subtree | ×1420 (+62 закомм.) | ×200 |

### 25.3. Декораторы

| Узел | CS:GO | CS2 |
| --- | --- | --- |
| decorator_bot_service | ×56 | ×5 |
| decorator_buy_service | ×51 | ×5 |
| decorator_dec_global_counter | ×5 (все — op09_grunts_card7_2) | 0 |
| decorator_hiding_spot_service | ×55 | ×1 |
| decorator_maybe | ×4 | ×1 |
| decorator_memory | ×27 | ×11 |
| decorator_need_healing | ×1 | ×1 |
| decorator_picker_blocked_by_smoke | ×2 | ×2 |
| decorator_picker_dedup | ×1 | ×1 |
| decorator_picker_grenade_type | ×2 | ×1 |
| decorator_picker_max_score | ×12 | ×8 |
| decorator_picker_nearby | ×12 | ×3 |
| decorator_picker_random_by_distance | ×1 | ×1 |
| decorator_picker_reaction_time | ×17 | ×8 |
| decorator_picker_visible | ×16 | ×5 |
| decorator_picker_weight_as_distance | ×1 | ×1 |
| decorator_random_approach_point | ×1 | ×2 |
| decorator_random_int | ×3 | ×5 |
| decorator_ranker_dist | ×12 | ×8 |
| decorator_remove | ×1 | ×1 |
| decorator_remove_key | ×16 | ×4 |
| decorator_repeat | ×153 (+31 закомм.) | ×22 |
| decorator_run_once | ×949 | ×11 |
| decorator_sensor | ×23 | ×16 |
| decorator_set_reaction_time | ×15 | ×6 |
| decorator_succeed | ×523 | ×16 |
| decorator_tag_entity | ×6 | ×3 |
| decorator_tag_threshold | ×5 | ×2 |
| decorator_token_service | ×46 | ×1 |
| decorator_try_lock | ×3 (bt_pickup_bomb_if_nearby + bt_pickup_hostage a/b) | ×3 |
| decorator_wait_success | ×9 (в 8 файлах) | 0 |

### 25.4. Условия

| Узел | CS:GO | CS2 |
| --- | --- | --- |
| condition_distance_less | ×3 (coop bt_guard; bt_attack_with_shield ×2 — card13, county_1) | 0 |
| condition_has_parachute | ×1 | 0 |
| condition_inactive | ×50 | 0 |
| condition_is_airborne | ×2 | 0 |
| condition_is_at_bomb_site | ×2 | ×2 |
| condition_is_empty | ×55 | ×24 |
| condition_is_equal | ×507 | ×151 |
| condition_is_greater_equal | ×5 | ×4 |
| condition_is_inv_slot_empty | ×33 | ×2 |
| condition_is_less_equal | 0 (!) | ×2 |
| condition_is_weapon_equipped | ×1 | ×1 |
| condition_out_of_ammo | ×1 | ×1 |
| condition_owns_item | ×5 | ×2 |

### 25.5. Действия

| Узел | CS:GO | CS2 |
| --- | --- | --- |
| action_acquire_items | ×277 | ×18 |
| action_aim | ×7 | ×8 |
| action_aim_projectile | ×1 (coop bt_throw_utility_if_covered) | 0 |
| action_assign_guardian_loadout | ×35 | 0 |
| action_attack | ×3 | ×3 |
| action_buy | ×16 | ×4 |
| action_choose_bomb_site_area | ×49 | ×5 |
| action_choose_guardian_bomb_plant_location | ×12 (10 корней op11 + bt_plant_bomb_if_owned/if_covered) | ×2 (practice/modules/bt_plant_bomb_if_*) |
| action_choose_random_waypoint | ×51 | ×6 |
| action_choose_random_waypoint_within_radius | ×2 (coop bt_roam, bt_rush) | 0 |
| action_choose_team_spawn_area | ×1 | ×3 |
| action_combat_positioning | ×1 | ×1 |
| action_commit_suicide | ×50 | 0 |
| action_compare_global_counter | ×9 | 0 |
| action_coordinated_buy | ×1 (op09_grunts_card7_2) | 0 |
| action_crouch | ×2 (bt_strat_hold_position, bt_boost_molly) | 0 |
| action_entity_accept_input | ×6 (op11_county_2) | 0 |
| action_equip_item | ×7 | ×3 |
| action_equip_weapon | ×324 | ×7 |
| action_flee_area_damage | ×1 | ×1 |
| action_jump | ×25 в 22 файлах (+10 закомм.) | 0 |
| action_look_at | ×309 | ×4 |
| action_move_to | ×557 | ×39 |
| action_parachute_positioning | ×1 | 0 |
| action_pull_trigger | ×289 | ×3 |
| action_say | ×2 (только coop: bt_guard, bt_roam) | 0 |
| action_select_areas_within_radius | ×2 (op11_insertion2_a/b) | 0 |
| action_set_global_counter | ×12 | 0 |
| action_set_global_flag | ×1 | ×1 |
| action_set_value_float | ×13 (все — coop bt_determine_rush_center) | 0 |
| action_set_value_vector | ×13 (все — coop bt_determine_rush_center) | 0 |
| action_teleport | ×449 | 0 |
| action_throw_utility | ×39 (только op11_county_2: firebomb 7, firebomb_wide 18, smoke_ring 8, smoke_house_rush 3, smoke_houses 3) | 0 |
| action_use | ×3 | ×3 |
| action_wait | ×870 (+8 закомм.) | ×13 |

> **Note**
>
> condition_is_less_equal — единственный из «неравенств», встречающийся **только в CS2** (practice), а condition_is_greater/is_less не встречаются нигде (§26.2). Глобальные счётчики (action_set_global_counter, action_compare_global_counter, decorator_dec_global_counter) в CS2-дампе отсутствуют полностью — механика подтверждена только CS:GO Guardian.

### 25.6. Особые счётчики

| Счётчик | CS:GO | CS2 |
| --- | --- | --- |
| succeed_after_first = 1 | ×2 (op09_grunts_card2/bt_strat_hold_position) | 0 |
| subtree params | ×15 (bt_strat_heaven 3, bt_strat_lobby 6, bt_strat_outside 6) | 0 |
| ratio = 1 (action_pull_trigger) | ×1 (bt_heal_if_needed) | ×1 (bt_heal_if_needed) |
| acquire_only (action_aim) | ×6 | ×7 |
| amount | ×5 (все — decorator_tag_threshold amount = 2) | ×2 |
| @mp_guardian_target_site | ×47 | 0 |
| @mp_plant_c4_anywhere | ×14 (10 корней op11 + 4 в guardian/modules) | ×4 (practice/modules) |
| RoundNumber | — | ×138 (practice) |
| TeamName | — | ×4 (3× "'T'", 1× "'CT'") |
| domain = "'AllBots'" | ×1 (op11_county_2) | 0 |
| Закомментированные subtree в корнях op10/11 | 62 | — |
| append_items (decorator_sensor) | 0 | ×2 (armsrace + deathmatch bt_default) |

## 26. Мёртвые ключи и неиспользуемые узлы

### 26.1. Мёртвые ключи (есть в файлах/документации, но движком не читаются)

| Ключ | Узел | Статус в дампах |
| --- | --- | --- |
| acquire_only | action_aim | Передаётся в bt_face_damage_inflictor, bt_look_around; игнорируется |
| ratio | action_pull_trigger | ×1 в bt_heal_if_needed; игнорируется. Предположительно задумывался как длительность удержания ЛКМ (0.0–1.0), но по Source2 Wiki не читается |
| distance_threshold | decorator_picker_blocked_by_smoke | В файлах есть, игнорируется |
| chatter_outnumbered_threshold | decorator_bot_service | Не встречается |
| attacker_filter | decorator_sensor | Не встречается |
| wait_modulo_tick_mod/add | action_wait | Не встречаются |
| team | action_choose_team_spawn_area | Не встречается |
| amount | decorator_repeat | Не встречается ни разу — все 5 вхождений amount в дампах это decorator_tag_threshold amount = 2 (ровно по числу этих узлов) |
| BT_ACTION_MOVETO_SAFEST_ROUTE | action_move_to route_type | 0 использований — только FASTEST |

### 26.2. Узлы, не встречающиеся ни разу (0 в 544 + 57 файлах)

| Условия | Декораторы | Действия |
| --- | --- | --- |
| condition_is_greater | decorator_invert | action_hide |
| condition_is_less | decorator_find_utility_strat | action_standup |
| condition_is_reloading | decorator_route_service | action_secondary_attack |
| condition_is_weapon_suitable (⚠️ по VDC работает как condition_is_weapon_equipped — не подтверждено, источник ставит [confirm]) | decorator_set_barrier | action_reload |
| condition_barrier | — | action_custom_buy |
| — | — | action_drop_active_weapon |
| — | — | action_inspect_current_weapon |

### 26.3. «Unused» по Source2 Wiki, но реально используются в Guardian/Co-op

Source2 Wiki помечает ~36 узлов как «существуют, но не используются в официальных деревьях», потому что смотрит **только на CS2-деревья** deathmatch/armsrace/rush/practice. Для CS:GO Guardian/Co-op наличие узла подтверждается дампами, даже если Source2 Wiki его не перечисляет или помечает «unused» (в т.ч. action_throw_utility, action_assign_guardian_loadout, action_choose_guardian_bomb_plant_location, action_entity_accept_input, decorator_wait_success, condition_has_parachute, action_parachute_positioning, condition_inactive, action_commit_suicide). Поэтому ваша вики здесь точнее официальной.

Обратное тоже верно: **0 вхождений в неполном CS2-дампе не означает отсутствия узла в CS2** — Guardian/Co-op-контента в CS2 All-in-One просто нет.

| Узел | Где реально используется |
| --- | --- |
| action_aim_projectile | Co-op bt_throw_utility_if_covered |
| action_throw_utility | op11_county_2 (×39) |
| action_parachute_positioning + condition_has_parachute + condition_is_airborne | op11_blacksite_1 (парашют, §20) |
| action_coordinated_buy | op09_grunts_card7_2 |
| decorator_wait_success | Guardian (координация, §18; ×9 в 8 файлах) |
| condition_inactive + action_commit_suicide | Guardian (предохранители, ×50) |
| action_teleport | Guardian (×449) |
| action_set_value_vector / action_set_value_float | Co-op bt_determine_rush_center (×13 + ×13, все — только этот модуль) |
| action_entity_accept_input | op11_county_2 (×6) |
| action_assign_guardian_loadout | Guardian (×35, лодауты по волнам) |
| action_choose_guardian_bomb_plant_location | Guardian ×12 (10 корней op11 + модули bt_plant_bomb_if_owned/if_covered) **и CS2 Practice ×2** (practice/modules/bt_plant_bomb_if_covered, bt_plant_bomb_if_owned) — «unused» неверно даже для CS2-деревьев |
| action_select_areas_within_radius | op11_insertion2_a/b (×2) |
| action_say | Co-op autumn (bt_guard, bt_roam; ×2) |
| condition_distance_less | Co-op bt_guard + bt_attack_with_shield (card13, county_1; ×3) |
| action_crouch | bt_strat_hold_position (card2) + bt_boost_molly (op10_engage_b); ×2 |
| action_choose_random_waypoint_within_radius | Co-op bt_roam, bt_rush; ×2 |
| глобальные счётчики (action_set_global_counter / action_compare_global_counter / decorator_dec_global_counter) | CS:GO Guardian: card7_2 (dec ×5, set ×12, compare ×9); в CS2-дампе — 0 |

## 27. Анти-паттерны

| ❌ Как НЕ надо | Почему ломается | ✅ Как надо |
| --- | --- | --- |
| decorator_repeat вокруг action_move_to «для надёжности» | Не нужен: пока бот не дошёл, action_move_to возвращает running, и sequencer остаётся на этом узле до достижения цели (§9.4) | repeat — только когда нужно повторно запускать уже завершившееся действие (цикл выбора новой цели и т.п.) |
| action_attack без синхронизации с action_aim (флаг ready) | Бот может стрелять до наводки — вниз/в сторону, пока крутится | Рекомендуемый шаблон: sequencer[repeat aim Enemy ready="AimReady", repeat attack Enemy ready="AimReady"] — как в bt_attack |
| decorator_tag_entity без expiration_time и без последующего BT_DECORATOR_TAG_ENTITY_CLEAR | Утечка тегов: бот навсегда считает врага «помеченным», ломается логика bt_plant_bomb_if_covered и подобных | expiration_time = 1 и/или CLEAR перед AT_LEAST — как в bt_pickup_hostage_if_covered |
| decorator_picker_max_score без ранкера | Вернёт первую сущность массива, а не ближайшую/лучшую | decorator_ranker_dist перед max_score |
| action_coordinated_buy вне run_once | Два бага (наследование предметов, id_no_purchase вне team_filter) + покупка всей подходящей команды одним вызовом | run_once domain = "'CoordinatedBuy'" + wait_success (§18) |
| Копирование дерева целиком без проверки веток | Мёртвые side-effect'ы: ShouldBuy в Arms Race/newplayer_dust2_01 пишется, но не читается | Проверять, что выход сервиса реально читается ниже |
| Копипаст закомментированных блоков из op10/11-корней | 62 закомментированных subtree — мусор в дереве и ложное чувство функции | Разкомментировывать осознанно (активный parallel[Flee…, LookAround] — только в Co-op autumn) |

## 28. Известные баги движка

| Баг | Детали | Обходной путь |
| --- | --- | --- |
| action_aim не разскопируется со снайперки | Прицел «залипает» до смены оружия | action_equip_weapon с тем же оружием / BEST сбрасывает прицел (unscope) |
| action_aim_projectile игнорирует гравитацию и физику полёта | Просто пишет вектор направления бот→цель в углы; граната летит неточно | action_throw_utility (§17) или скриптовый look_at-паттерн (sirocco) |
| action_coordinated_buy — покупка всей командой | Не баг, а штатная механика: один вызов инициирует покупку всех ботов, подходящих под team_filter (VDC) | Строго внутри run_once с уникальным domain |
| action_coordinated_buy — баг 1 (наследование предметов) | Внутренний буфер покупок не очищается между ботами: боты ниже по массиву наследуют предметы от ботов выше | Тот же run_once-паттерн + wait_success для остальных |
| action_coordinated_buy — баг 2 (id_no_purchase) | Боты, не подходящие под team_filter, всё равно ошибочно получают id_no_purchase (VDC) | Не использовать id_no_purchase как фильтр по команде |
| decorator_sensor с team_filter = "ENEMY" для гранат | Возвращает гранаты T-команды независимо от команды бота | OPPOSITE вместо ENEMY (подтверждено VDC) |
| action_jump сбрасывает movement_type | WALK превращается в RUN после прыжка | Переустанавливать movement_type = "BT_ACTION_MOVETO_WALK" после action_jump |
| action_move_to не обходит тиммейтов | При включённой коллизии бот не перестраивает маршрут вокруг товарища, а «проталкивается» сквозь него, пока тот не уйдёт (VDC) | Не строить точные «узкие» маршруты впритык друг к другу |
| Мёртвый ShouldBuy | В Arms Race и newplayer_dust2_01 пишется, но не читается | Покупка через item_strategy (run_once → action_buy), как у Valve; либо живая buy-ветка как в deathmatch/rush/practice/bt_default |
| condition_is_weapon_suitable работает как condition_is_weapon_equipped | ⚠️ Не подтверждено: по старой VDC — баг движка, но сам источник ставит пометку [confirm] | Не использовать для проверки «подходит ли оружие по дистанции» без самостоятельной проверки в игре |
| // type = "action_RIGHT_CLICK?" | В op10_ancient_a/bt_long_awp Valve хотела ПКМ-действие (зум AWP), но закомментировала | Тривия; используйте action_equip_weapon для скоупа |
