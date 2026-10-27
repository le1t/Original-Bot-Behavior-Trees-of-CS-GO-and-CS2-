# Original-Bot-Behavior-Trees-of-CS-GO-and-CS2

Википедия по ботам CS:GO и CS2 на Behavior Trees
Введение
Behavior Tree (BT) — это декларативная система описания поведения ботов, используемая в Counter-Strike: Global Offensive (CS:GO) и Counter-Strike 2 (CS2). Вместо императивного C++ кода (как в слитом NextBot) поведение задаётся в KV3-файлах как дерево узлов, которое игра вычисляет каждый тик. Деревья используются в режимах Deathmatch, Arms Race, Rush, обучении (Practice) и Guardian. Любой сервер или карта может указать собственное дерево через ConVar mp_bot_ai_bt.

Важно: слитый C++ код NextBot относится к архитектуре CS:GO/Left 4 Dead и не используется в CS2. Для создания ботов в CS2 рабочим инструментом являются KV3-файлы и понимание модели выполнения узлов. C++ код может быть полезен только для понимания общих идей (приоритеты, память), но не для копирования структур.

Выбор дерева и конфигурация
ConVar mp_bot_ai_bt задаёт путь к файлу .kv3 (например, scripts/ai/deathmatch/bt_default.kv3). Если пуст, используется классический ИИ.

После редактирования дерева необходимо выполнить mp_bot_ai_bt_clear_cache и переспавнить ботов (перезапустить раунд).

Официальные режимы задают свои деревья в конфигах:

Deathmatch / DM Freeforall: scripts/ai/deathmatch/bt_default.kv3

Arms Race: scripts/ai/armsrace/bt_default.kv3

Rush: scripts/ai/rush/bt_default.kv3

New User Training: scripts/ai/practice/newplayer_dust2_01.kv3

Деревья и модули поставляются в game/csgo/pak01_dir.vpk и могут быть извлечены через Source2 Viewer.

Пользовательское дерево можно разместить в scripts/ai/ (например, scripts/ai/mymode/bt_default.kv3) и указать в mp_bot_ai_bt.

Формат файлов KV3
Файл дерева имеет заголовок:

text
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
Далее идёт объект с ключами config (путь к конфигу) и root (корневой узел). Пример:

kv3
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
Каждый узел — таблица с ключом type. Декораторы и условия оборачивают одного ребёнка через child; композиты содержат массив children.

Имена ключей регистронезависимы, комментарии // разрешены.

Subtree-файлы содержат один узел верхнего уровня без config и root.

Конфигурационный файл (bt_config.kv3)
Конфиг задаёт «мышечную память» бота: скорость прицеливания, реакцию, стрельбу. Он ортогонален логике дерева.

Уровни скилла
CS:GO: default, low, fair, normal, tough, hard, very_hard, expert (8 уровней).

CS2: добавлен elite (9 уровней). В предоставленных файлах присутствуют все 9.

Каждый уровень наследует недостающие ключи из default.

Параметры прицеливания
Параметр	Тип	Назначение
aim_target_acquisition_lerp_time	float	Время «доезда» прицела до новой цели
aim_target_acquisition_lerp_time_deviation	float	Разброс этого времени
aim_target_acquisition_angle_penalty	float	Штраф за угловое расстояние до цели
aim_target_acquisition_angle_penalty_deviation	float	Разброс штрафа
aim_target_acquisition_angle_penalty_reduction_ratio	float	Насколько штраф уменьшается со временем
aim_target_acquisition_angle_tolerance	float	Допуск угла при захвате
aim_target_acquisition_angle_lerp_bias	float	Сглаживание угла захвата
aim_target_tracking_lerp_time	float	Скорость сопровождения цели
aim_target_tracking_lerp_time_deviation	float	Разброс сопровождения
aim_target_tracking_focus_interval	float	Интервал «фокуса» при трекинге
aim_target_tracking_focus_interval_deviation	float	Разброс интервала
aim_target_tracking_angle_lerp_bias	float	Сглаживание трекинга
aim_ready_angle_tolerance	float	Угол, при котором action_attack считает прицел готовым (в CS:GO не документирован)
aim_new_target_angle_tolerance	float	Угол для переключения на новую цель
aim_max_duration	float	Максимальное время прицеливания
aim_max_duration_deviation	float	Разброс макс. времени
aim_punch_angle_reaction_chance	float	Шанс реакции на punch angle (отдача)
Параметры осмотра
Параметр	Назначение
look_around_awareness_yaw_range	Макс. угол поворота по горизонтали (градусы)
look_around_awareness_pitch_range	Макс. угол по вертикали
look_around_focus_interval	Пауза между поворотами камеры
look_around_focus_interval_deviation	Разброс паузы
look_around_lerp_time	Время поворота камеры
look_around_lerp_time_deviation	Разброс времени
look_around_lerp_bias	Сглаживание движения камеры
Реакция и бой
Параметр	Назначение
reaction_time	Задержка перед реакцией на событие (сек)
combat_crouch_chance	Шанс присесть в бою (0–1)
combat_dodge_command_duration	Длительность команды уклонения
combat_dodge_command_duration_deviation	Разброс длительности
Таблица attack
Каждая строка — 5 чисел: burst duration, duration deviation, cadence, cooldown, cooldown deviation.
Порядок строк:

CS:GO (16 строк): KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN (фактически 16? В предоставленном CS:GO конфиге 16 строк: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN — это 20? Проверим: в файле bt_config.txt (CS:GO) строки: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN. Это 20 строк? Но в вики CS:GO сказано 16 строк. Возможно, в предоставленном файле 16 строк, просто комментарии сгруппированы. Посмотрим: в CS:GO bt_config.txt после STACKABLEITEM идут FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN — это 8 дополнительных, итого 12+8=20. Но в вики CS:GO указано 16 строк: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN — это 20. Возможно, в вики ошибка или в файле лишние. В CS2 ровно 13 строк: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, UNKNOWN.

CS2: 13 строк в фиксированном порядке: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, UNKNOWN. Если строк не 13, конфиг отвергается с ошибкой [AI BT]: Config node 'attack' must contain 13 entries.

Пример для RIFLE в default:
[ 0.22, 0.07, 0.0, 0.45, 0.15 ] — очередь ~0.22 с (±0.07), без паузы внутри (cadence 0), перезарядка/новая очередь через 0.45 с (±0.15).

Узлы (Nodes)
Типы значений
Тип	Значение
bool	0 или 1
int, float	Литеральное число
number	Число, ConVar (@mp_...) или ключ blackboard
position	Литерал ("x y z"), ключ blackboard или targetname энтити
string	Литерал в '...' или ключ blackboard
enum	Значение из таблицы Enum
node, node[]	Дочерний узел / массив узлов
Enum-значения
Ключ	Значения
entity_type_filter	ALL, PLAYERS, HUMAN_PLAYERS, NOISE, DAMAGE, AREA_DAMAGE, CLASSNAME, GRENADE
team_filter, attacker_filter	ANY, CT, TERRORIST, SAME, OPPOSITE, ENEMY
team	ANY, CT, TERRORIST, OPPOSITE
team_restriction	ANY, CT, T
shape type	sensor_shape_fov, sensor_shape_sphere (с radius)
movement_type	BT_ACTION_MOVETO_WALK, BT_ACTION_MOVETO_RUN
route_type	BT_ACTION_MOVETO_FASTEST_ROUTE, BT_ACTION_MOVETO_SAFEST_ROUTE
slot	RIFLE, PISTOL, KNIFE, GRENADES, C4
types (гранаты)	FLASH, FIRE, EXPLOSIVE, DECOY, SMOKE
operation_type	BT_DECORATOR_TAG_ENTITY_SET, BT_DECORATOR_TAG_ENTITY_CLEAR
check_type	BT_DECORATOR_TAG_THRESHOLD_AT_LEAST, BT_DECORATOR_TAG_THRESHOLD_AT_MOST
weapon	BEST или classname (например, weapon_knife)
Композиты
Узел	Ключи	Поведение	Статус
parallel	succeed_after_first (bool), children	Все дети каждый тик. Провал любого = провал. Успех = все (или первый при флаге)	Официальный
selector	children	Список приоритетов. Проверка с первого ребёнка каждый тик. Более ранний прошедший — останавливает текущего и запускается	Официальный
sequencer	children	По порядку. Провал = стоп. Check-pass смотрит только на текущего ребёнка — пройденные условия не перепроверяются	Официальный
subtree	file (string), name (string), params ({key, value}[])	Запуск другого дерева. params пишут значения в blackboard до старта. Только int или string — float/bool отвергаются	Официальный
Декораторы
Декораторы оборачивают одного ребёнка через child. Почти все поддерживают negated = 1.

Узел	Ключи	Назначение	Статус в CS2
decorator_bot_service	memory_to_expire[], tagged_entities_to_expire[], input_chatter_enemies, basic_chatter_enable, chatter_outnumbered_threshold, child	Корневой сервис: истечение памяти, болтовня	Официальный
decorator_buy_service	output, child	Возвращает массив к покупке	Официальный
decorator_hiding_spot_service	domain, output_hiding_spot, expiration_time, distance_threshold, child	Учёт укрытий; боты в одном домене не проверяют уже проверенные	Официальный
decorator_maybe	chance (float), child	Выполняет ребёнка с шансом (1 = 100%)	Официальный
decorator_memory	input, output, output_domain, child	Копирует сущности в память	Официальный
decorator_need_healing	health_threshold, child	Ребёнок, если HP ниже порога	Официальный
decorator_picker_blocked_by_smoke	input, input_domain, output, output_domain, negated, child	Убирает сущности, не покрытые дымом (параметр distance_threshold игнорируется)	Официальный
decorator_picker_dedup	input, input_domain, output, output_domain, against, distance_threshold, negated, child	Убирает дубликаты по дистанции до against	Официальный
decorator_picker_grenade_type	input, input_domain, output, output_domain, types[], negated, child	Фильтр гранат по типу	Официальный
decorator_picker_max_score	input, input_domain, output, output_domain, negated, child	Берёт первую сущность (по умолчанию — по порядку на сервере)	Официальный
decorator_picker_nearby	input, input_domain, output, output_domain, cutoff_distance, negated, child	Убирает дальше порога	Официальный
decorator_picker_random_by_distance	input, input_domain, output, output_domain, distance_min, distance_max, negated, child	Вероятностный фильтр по дистанции	Официальный
decorator_picker_reaction_time	input, input_domain, output, output_domain, negated, child	Задержка по реакции бота	Официальный
decorator_picker_visible	input, input_domain, output, output_domain, check_fov (bool), negated, child	Убирает невидимые. check_fov = 0 игнорирует FOV	Официальный
decorator_picker_weight_as_distance	input, input_domain, output, output_domain, negated, child	Фильтр по «весу» (для слуха)	Официальный
decorator_random_approach_point	output, child	Случайная видимая точка-«вход» в область	Официальный
decorator_random_int	min, max, output, child	Случайное целое	Официальный
decorator_ranker_dist	input, child	Сортирует по дистанции	Официальный
decorator_remove	input_domain, input, remove, child	Удаляет сущности из памяти	Официальный
decorator_remove_key	input, child	Удаляет ключ blackboard	Официальный
decorator_repeat	amount (int), child	Цикл. Без amount — вечно; с N — N раз	Официальный
decorator_run_once	domain, max_attempts, child	Ограничивает запуск. domain — только один бот выполнит	Официальный
decorator_sensor	output, class_name, priority, orphan_only, append_items, entity_type_filter, team_filter, attacker_filter, shape({type, radius}), child	Заполняет output sensed-сущностями	Официальный
decorator_set_reaction_time	input, child	Задержка по реакции	Официальный
decorator_succeed	negated, child	Всегда проходит check-pass; после ребёнка — success (или failure при negated=1)	Официальный
decorator_tag_entity	input, output, expiration_time, operation_type, child	Помечает первую сущность из input	Официальный
decorator_tag_threshold	entity_input, tagged_entities_input, amount, check_type, child	Сравнивает количество помеченных	Официальный
decorator_token_service	config({tokens[], assignments[][], team_restriction[]}), output_token_name, output_token_domain, domain, child	Группы ботов (StratTokens)	Официальный
decorator_try_lock	domain, child	Блокировка домена — только один бот одновременно	Официальный
decorator_invert	child	Инверсия check-результата ребёнка	Не используется в официальных деревьях CS2 (в CS:GO мог быть workaround)
decorator_dec_global_counter	input_name, child	Уменьшает счётчик	Не используется
decorator_find_utility_strat	много	—	Не используется
decorator_route_service	много	—	Не используется
decorator_set_barrier	много	—	Не используется
decorator_wait_success	timeout, child	Ждёт успеха ребёнка или таймаута	Не используется в основных деревьях CS2, но встречается в Guardian
decorator_game_event	—	Упоминается в патчноутах, но отсутствует в коде	Не существует
Важно: Source2 Wiki указывает, что параметры acquire_only у action_aim, ratio у action_pull_trigger и distance_threshold у decorator_picker_blocked_by_smoke не читаются игрой. Однако в предоставленных официальных файлах они присутствуют (acquire_only = 1, ratio = 1, distance_threshold = 3000) — вероятно, наследие старых версий; не полагайтесь на них.

Условия
Узел	Ключи	Назначение	Статус
condition_is_empty	input, global (bool), negated, child	Пусто/не существует. global = 1 — глобальный флаг	Официальный
condition_is_equal	source, destination (int или string), negated, child	Равенство	Официальный
condition_is_greater_equal	source, destination, negated, child	≥	Официальный
condition_is_less_equal	source, destination, negated, child	≤	Официальный
condition_is_inv_slot_empty	slot (enum), negated, child	Слот пуст	Официальный
condition_is_weapon_equipped	weapon, negated, child	Активное оружие с этим classname	Официальный
condition_out_of_ammo	negated, child	Нет патронов	Официальный
condition_owns_item	item, items_one_of[], negated, child	Есть предмет	Официальный
condition_is_at_bomb_site	negated, child	На бомбсайте	Официальный
condition_distance_less	input, distance_threshold_min, distance_threshold_max, negated, child	Дистанция меньше порога	Не используется в основных деревьях
condition_has_parachute	negated, child	Есть парашют	Не используется (но встречается в Guardian)
condition_inactive	input[], round_start_threshold_seconds, sensor_inactivity_threshold_seconds, negated, child	Нет активности	Не используется в основных деревьях, но встречается в Guardian
condition_is_airborne	negated, child	В воздухе	Не используется в основных деревьях, но встречается в Guardian
condition_is_greater	source, destination, negated, child	>	Не используется
condition_is_less	source, destination, negated, child	<	Не используется
condition_is_reloading	negated, child	Перезарядка	Не используется
condition_is_weapon_suitable	input, weapon, negated, child	Подходит ли оружие для дистанции	Не используется
condition_barrier	input_domain, input_name, radius, negated, child	—	Не используется
Действия
Узел	Ключи	Назначение	Статус
action_acquire_items	items[], remove_all_items	Выдать предметы	Официальный
action_aim	input, ready (string)	Повернуть к точке. Никогда не разскопится со снайперки до смены оружия	Официальный
action_attack	input, ready, output	Стрелять по input, когда ready истинно	Официальный
action_buy	—	Случайная покупка	Официальный
action_choose_bomb_site_area	input (number), output	Позиция бомбсайта по индексу	Официальный
action_choose_random_waypoint	input, output	Случайный тайл навмеша	Официальный
action_choose_team_spawn_area	output, team (enum)	Navzone со спавнами	Официальный
action_combat_positioning	input, is_attacking (string)	Уклонение/сближение	Официальный
action_equip_item	item, items_one_of[]	Выбрать предмет	Официальный
action_equip_weapon	weapon (classname или BEST)	Выбрать оружие	Официальный
action_flee_area_damage	input, output, max_search_range, threat_min_keep_distance	Безопасная точка от урона по площади	Официальный
action_look_at	input_angles или input_location	Повернуть бота. Указывать одно из двух	Официальный
action_move_to	destination, hiding_spot, threat, auto_look_adjust, damaging_areas_penalty_cost, arrival_epsilon, additional_arrival_epsilon_2d, hiding_spot_check_distance_threshold, nearest_area_distance_threshold, movement_type, route_type	Двигаться к точке	Официальный
action_pull_trigger	—	Нажать ЛКМ	Официальный
action_use	—	Нажать +use	Официальный
action_wait	wait_time_min, wait_time_max, wait_modulo_tick_mod, wait_modulo_tick_add	Заморозить ИИ	Официальный
action_set_global_flag	name, expiration_time_min, expiration_time_max	Глобальный флаг на случайное время	Официальный
action_aim_projectile	input, output	Рассчитать углы броска	Не используется (баг: не учитывает траекторию)
action_choose_guardian_bomb_plant_location	output	Guardian: позиция закладки	Не используется в основных деревьях, но используется в Guardian
action_choose_random_waypoint_within_radius	origin, radius, output	Случайная точка в радиусе	Не используется в основных деревьях, но используется в Co-op
action_commit_suicide	—	Самоубийство	Не используется в основных деревьях, но используется в Guardian
action_compare_global_counter	input_name, input_value	Сравнить счётчик	Не используется в основных деревьях, но используется в Guardian
action_coordinated_buy	id_no_purchase, purchases[], team_filter, save_threshold	Скоординированная покупка	Не используется (баг: один бот заставляет всех)
action_crouch	—	Присесть	Не используется в основных деревьях, но используется в Guardian
action_custom_buy	item_aliases[]	—	Не используется
action_drop_active_weapon	—	—	Не используется
action_hide	output, max_range	Случайное укрытие	Не используется
action_inspect_current_weapon	—	—	Не используется
action_jump	—	Прыжок	Не используется в основных деревьях, но используется в Co-op
action_parachute_positioning	—	Движение по диагонали	Не используется в основных деревьях, но используется в Guardian
action_reload	—	Перезарядка	Не используется
action_say	phrase, high_priority	Радио-команда	Не используется в основных деревьях, но используется в Co-op
action_secondary_attack	—	—	Не используется
action_select_areas_within_radius	input, radius, output	Navzone в радиусе	Не используется в основных деревьях, но используется в Guardian
action_set_global_counter	input_name, input_value	Установить счётчик	Не используется в основных деревьях, но используется в Guardian
action_set_value_float	key, value	Записать float	Не используется в основных деревьях, но используется в Guardian
action_set_value_vector	key, value	Записать позицию	Не используется в основных деревьях, но используется в Guardian
action_standup	—	Встать	Не используется
action_teleport	destination	Телепорт	Не используется
action_entity_accept_input	name, input	Отправить input энтити (встречается в Guardian)	Не документирован в вики, но используется
Примечание: Source2 Wiki отмечает 36 узлов, которые существуют, но не используются в официальных деревьях (Deathmatch/Arms Race/Rush/Practice). Однако в деревьях Guardian и Co-op некоторые из них применяются. Поэтому статус «не используется» не означает, что узел не работает.

Модель выполнения
Дерево вычисляется каждый тик (~64 раза/сек) в два прохода:

Check-pass — сверху вниз. Решает, какие ветки имеют право выполняться. Если корень не проходит — тик «пустой».

Run-pass — выполняет выбранные ветки. Здесь двигаются, целятся, стреляют.

Поскольку check-pass повторяется каждый тик:

Условия перепроверяются постоянно.

selector может прервать текущую ветку, если более приоритетная начала проходить проверку.

Поведение композитов
sequencer — check-pass смотрит только текущего ребёнка; run-pass по порядку до провала. Условия позади не перепроверяются.

selector — check-pass с первого ребёнка каждый тик; запускает первого прошедшего; останавливает текущего при появлении более приоритетного.

parallel — все дети каждый тик; провал любого = стоп всех; успех = все (или первый при succeed_after_first = 1).

decorator_repeat — перезапускает ребёнка каждый раз, когда он завершается; с amount = N останавливается после N запусков и возвращает результат последнего.

decorator_succeed — всегда проходит check-pass; после ребёнка возвращает success (или failure при negated=1).

decorator_invert — инвертирует check-результат ребёнка; во время работы результаты ребёнка проходят без изменений.

condition_* — проверяется в каждом check-pass; пока условие истинно, выполняет ребёнка (или просто успех, если ребёнка нет); при провале — провал ветки. Все условия поддерживают negated = 1.

Пример разворачивания дерева
text
sequencer[ move_to(bot_target), look_at(bot_target), wait(2..3) ] в repeat
Тик 1: sequencer запускает move_to, тот двигает на 1 тик → running.

Тики 2..N: игра идёт прямо в move_to — это одно непрерывное движение, а не новое решение.

Прибытие: move_to success → look_at → success.

wait выбирает время окончания и возвращает running.

Все три готовы → sequencer success → repeat заново.

Ключевой вывод: узлы не двигают бота напрямую. action_move_to управляет встроенным pathfinding; action_use/action_reload «нажимают кнопку» на тик; action_aim/action_attack рулят встроенным aiming/shooting, пока не завершатся.

Значения и Blackboard
Большинство значений в дереве — это ключи blackboard (хранилище бота), которые узлы читают и пишут. Например, сенсор пишет врагов в output = "Enemy", а действие атаки читает их через input = "Enemy".

Строки
Литерал в одинарных кавычках: 'CT' — кавычки отбрасываются.

Иначе — имя ключа blackboard, который должен содержать строку.

Числа
Литеральное число.

ConVar: @mp_plant_c4_anywhere.

Ключ blackboard. Если ключ содержит строку, она разрешается ещё раз, но только на 1 уровень.

Позиции и имена сущностей
Ключи, ожидающие позицию (destination, input_location), принимают:

Литерал: "-1450 2230 2".

Ключ blackboard (позиция, строка или список sensed-сущностей — берётся первая).

Targetname энтити на карте, если ни один ключ так не называется. Используется origin энтити.

Это позволяет управлять ботами с карты: Rush-боты идут к ant.button по имени; скрипт может двигать target-энтити, чтобы перенаправлять ботов.

Домены (Domains)
Домены — способ делить состояние между ботами. Читаются по-разному в разных узлах, поэтому официальные деревья — лучший справочник.

Домен	Где встречается	Назначение
AllBots	memory_to_expire, input_domain, output_domain	Общая память всех ботов
'StratTokens'	decorator_token_service	Токены стратегий (Practice, Guardian)
'PressAntennaButton'	decorator_try_lock	Блокировка нажатия кнопки (Rush)
'DefuseBombIfNearby'	decorator_try_lock	Блокировка разминирования (Practice)
Важно: кавычки вокруг имён доменов обязательны — это литералы.

Память
decorator_sensor → output = "Vision" (список сущностей).

decorator_picker_* → сужают "Vision".

decorator_memory → копирует в "ShortTermAttackMemory".

decorator_bot_service в корне → истекает память: {key, time, distance}.

Стандартные имена памяти в официальных деревьях:

ShortTermAttackMemory (0.7 с, 0 дистанция)

LongTermMemory (10 с, 500)

ShortTermInvestigateMemory (3 с, 200)

ShortTermDamageMemory (Practice, Guardian)

ShortTermAreaDamageMemory (Practice, Guardian)

DamageThroughSmokeMemory (Practice, Guardian)

Threats (Practice, Guardian, домен AllBots)

Глобальные флаги и счётчики
action_set_global_flag устанавливает именованный флаг на случайное время между expiration_time_min и expiration_time_max. Официальные деревья проверяют его через condition_is_empty с global = 1.

action_set_global_counter, decorator_dec_global_counter, action_compare_global_counter — именованные счётчики.

Параметры subtree
subtree может записывать значения в blackboard до старта:

kv3
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
Значения должны быть целыми или строками. Дробные и булевы отвергаются с ошибкой [AI BT]: unsupported type for subtree parameter.

Отладка
Команда	Назначение	Требования
mp_bot_ai_bt "путь"	Назначить дерево всем ботам	—
mp_bot_ai_bt_clear_cache	Сбросить кэш после правок	—
cv_bot_ai_bt_debug_target <index>	Нарисовать дерево бота	sv_cheats 1, локальный/приватный сервер
cv_bot_ai_bt_hiding_spot_show	Показать укрытия	sv_cheats 1
cv_bot_ai_bt_moveto_show_next_hiding_spot	Показать следующее проверяемое укрытие	sv_cheats 1
Что искать в консоли:

[AI BT]: Loaded behavior tree '<file>' — при developer 1.

[AI BT] warnings — всегда, даже без developer.

Blackboard entry 'X' does not exist nor it is a valid entity name — ключ не найден.

[AI BT]: Config node 'attack' must contain 13 entries — неверное число строк.

[AI BT]: unsupported type for subtree parameter — float/bool в params.

Зарезервированные переменные
Это переменные, которые игра сама кладёт в blackboard. Вы можете их читать, но не можете создавать.

Переменная	Тип	Откуда берётся	Где используется
NumberOfTerrorists	int	Движок	condition_is_equal в bt_plant_bomb_if_owned
NumberOfAliveTerrorists	int	Движок	condition_is_equal в bt_plant_bomb_if_owned
NumberOfCounterTerrorists	int	Движок	—
NumberOfAliveCounterTerrorists	int	Движок	—
AccountBalance	int	Движок	—
AmmoCount/<weapon>	int	Движок	—
AmmoCount/current	int	Движок	—
TeamName	string	Движок	condition_is_equal source="TeamName" destination="'T'" в Practice
RoundNumber	int	Движок	Все bt_t_*.kv3 и bt_ct_*.kv3 в Practice
BlindnessPercentage	float	Движок	—
GuardianWaveNumber	int	Движок (Guardian)	—
LastCoopSpawnPointName	string	Движок (Co-op)	—
LastCoopSpawnPointLocation	Vector	Движок (Co-op)	—
Примечание: TeamName и RoundNumber отсутствуют в списке зарезервированных переменных на Source2 Wiki, но активно используются в официальном Practice-дереве. Это означает, что список на вики неполный, и они доступны как минимум в режиме обучения (вероятно, и в других).

Различия между CS:GO и CS2
Аспект	CS:GO	CS2
Уровни скилла	8 (без elite)	9 (с elite)
Строк в attack	16 (включая WEAPONTYPE_ZONE_REPULSOR)	13 (ровно, иначе ошибка)
aim_ready_angle_tolerance	Не документирован	Есть в конфиге
TeamName / RoundNumber	Не документированы	Используются в Practice
Документация узлов	Неточная, много TODO	Точный список читаемых ключей
Неиспользуемые узлы	Не отмечены	36 отмечены (но некоторые используются в Guardian/Co-op)
Игнорируемые параметры	Не отмечены	acquire_only, ratio, distance_threshold
Модель выполнения	Не описана	Два прохода, детально
Subtree params	Не описаны	Только int/string
decorator_picker_*	Без доменов	С input_domain/output_domain
decorator_sensor	Без append_items	С append_items
action_wait	2 ключа	+ wait_modulo_tick_mod/add
Баг с гранатами (ENEMY)	Есть	Не упоминается
Примеры деревьев
Простое дерево (движение, взгляд, ожидание)
kv3
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
Паттерны, проверенные официальными деревьями
1. Восприятие + решение через parallel:

text
parallel
├── repeat → parallel (sensors, memory)
└── repeat → selector (priority list)
2. Selector как список приоритетов:

text
selector
├── condition_is_empty ShouldBuy (negated) → sequencer[wait, buy, remove_key]
├── sensor DAMAGE → aim
├── subtree Attack
├── subtree HealIfNeeded
└── subtree Hunt
3. Блокировка действия одним ботом:

text
decorator_try_lock domain = "'PressAntennaButton'"
└── sequencer[ move_to ant.button, look_at, wait, use ]
4. Скоординированная покупка (не используйте action_coordinated_buy — он сломан):

text
decorator_run_once domain = "'BuyPhase'"
└── action_buy
5. Истечение памяти в корне:

text
decorator_bot_service
  memory_to_expire =
  [
    { key = "ShortTermAttackMemory"  time = 0.7  distance = 0 }
    { key = "LongTermMemory"         time = 10   distance = 500 }
    { key = "ShortTermInvestigateMemory" time = 3 distance = 200 }
  ]
Официальные деревья и модули (из предоставленных файлов)
Deathmatch / DM Freeforall: scripts/ai/deathmatch/bt_default.kv3

Корень: decorator_bot_service → decorator_buy_service → parallel (восприятие + selector).

Selector: покупка, реакция на урон, атака, лечение, исследование, охота.

Arms Race: scripts/ai/armsrace/bt_default.kv3 (аналогично, но с другими модулями).

Rush: scripts/ai/rush/bt_default.kv3

Дополнительно: decorator_try_lock на 'PressAntennaButton', движение к ant.button, использование.

Practice (New User Training): scripts/ai/practice/newplayer_dust2_01.kv3

decorator_hiding_spot_service → decorator_token_service ('StratTokens') → decorator_bot_service.

В selector: ItemStrategy, Attack, FaceDamageInflictor, AttackDamageInflictorSmoke, ClearThreatsWithinFov, SelectNearbyThreat, TakeoverStrategy, проверки TeamName для закладки/разминирования, исследование, охота.

Co-op (Guard, Roam, Rush, Sniper): scripts/ai/coop/bt_guard.kv3 и др.

decorator_hiding_spot_service → decorator_bot_service (с basic_chatter_enable, input_chatter_enemies).

BasicPlayerMemorization, BasicPlayerEngagement, исследование, спецповедение (охрана точки, рой, снайпер).

Guardian (пример op11_sirocco_2.kv3):

decorator_hiding_spot_service → decorator_token_service ('StratTokens' с токенами стратегий) → decorator_bot_service.

Selector: condition_inactive (самоубийство при неактивности), ItemStrategy, Attack, FaceDamageInflictor, AttackDamageInflictorSmoke, ClearThreatsWithinFov, SelectNearbyThreat, TakeoverStrategy, PlantIfCovered, PlantIfOwned, PickupBombIfNearby, HealIfNeeded, исследование, охота.

Модули: bt_item_strategy, bt_takeover_strategy, bt_attack_with_shield, bt_investigate_with_shield, bt_memorize_threats, bt_clear_threats_within_fov, bt_select_nearby_threat и др.

Модули (общие):

bt_attack.kv3 — выбор цели, экипировка, позиционирование, прицеливание, атака.

bt_memorize_enemies_vision.kv3 — сенсор зрения (FOV, PLAYERS, ENEMY, nearby 1500, visible, реакция, память).

bt_memorize_noises.kv3 — сенсор слуха (сфера 3000, NOISE, weight_as_distance, random_by_distance, dedup, реакция, память).

bt_memorize_damage.kv3 — сенсор DAMAGE → ShortTermDamageMemory и DamageThroughSmokeMemory.

bt_memorize_area_damage_*.kv3 — гранаты, инферно, текущий урон.

bt_memorize_nearest_investigation.kv3 — выбирает ближайшее событие из LongTermMemory в ShortTermInvestigateMemory.

bt_investigate_closest_memorized_event.kv3 — идёт к событию, смотрит, ждёт.

bt_heal_if_needed.kv3 — лечение при HP < 60.

bt_face_damage_inflictor.kv3 — поворот к источнику урона, атака (если не в дыму).

bt_attack_damage_inflictor_smoke.kv3 — атака источника урона сквозь дым.

bt_flee_area_damage_threats.kv3 — бегство из зоны урона.

bt_look_around.kv3 — случайный обзор.

bt_plant_bomb_if_covered.kv3 — закладка при прикрытии.

bt_plant_bomb_if_owned.kv3 — закладка, если есть бомба (с флагом DontPlantBomb).

bt_pickup_bomb_if_nearby.kv3 — подбор бомбы.

bt_defuse_bomb_if_nearby.kv3 — разминирование.

bt_pickup_hostage.kv3, bt_pickup_hostage_if_covered.kv3 — захват заложника.

bt_clear_threats_within_fov.kv3 — удаление угроз из общего списка, если они в FOV.

bt_select_nearby_threat.kv3 — выбор ближайшей видимой угрозы в NearestThreatMemory.

bt_memorize_threats.kv3 — добавление источников урона в глобальный Threats.

bt_determine_rush_center.kv3 — определение центра раша по имени спавна (Co-op).

bt_throw_utility_if_covered.kv3 — бросок гранаты при прикрытии.

bt_basic_player_memorization.kv3, bt_basic_player_engagement.kv3 — базовые модули Co-op.

Практические рекомендации
С чего начать
Извлеките официальные деревья через Source2 Viewer из pak01_dir.vpk. Изучите bt_default.kv3 (Deathmatch) и bt_practice/newplayer_dust2_01.kv3 — это лучшие образцы.

Не пытайтесь «портировать» C++ логику из слива. Думайте в терминах узлов, blackboard и приоритетов.

Начните с простого дерева и итеративно усложняйте.

Шаблон минимального дерева
kv3
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
                { type = "action_move_to"  destination = "bot_target"  movement_type = "BT_ACTION_MOVETO_RUN"  route_type = "BT_ACTION_MOVETO_FASTEST_ROUTE" }
                { type = "action_look_at"  input_location = "bot_target" }
                { type = "action_wait"     wait_time_min = 2  wait_time_max = 3 }
            ]
        }
    }
}
Чего избегать
❌ Не делайте	✅ Делайте вместо этого
Полагаться на acquire_only, ratio, distance_threshold у picker_blocked_by_smoke	Используйте узлы без этих параметров — они игнорируются
Использовать action_coordinated_buy	decorator_run_once + action_buy
Забывать mp_bot_ai_bt_clear_cache	Всегда чистить кэш после правок
Передавать float/bool в subtree params	Только int или string
Использовать 16 строк в attack (CS2)	Ровно 13, в фиксированном порядке
Ожидать, что action_aim разскопится	Смените оружие или экипируйте то же заново
Полагаться на condition_is_greater/is_less	Используйте greater_equal/less_equal — они используются в официальных деревьях
Проверка гипотез о «мёртвых» параметрах
Если хотите проверить, действительно ли acquire_only не работает:

Создайте два дерева: с acquire_only = 1 и без.

Запустите cv_bot_ai_bt_debug_target на боте.

Сравните поведение прицеливания. Если разницы нет — параметр мёртв.

Сводная таблица различий: слитый C++ код vs CS2 KV3
Концепт в C++ (слив)	Аналог в CS2 KV3	Можно ли перенести?
Action<Actor>::OnStart	OnStart узла (неявно)	❌ Нет прямого аналога
ActionResult (CONTINUE/CHANGE_TO/SUSPEND_FOR/DONE)	success/failure/running	⚠️ Концептуально похоже, но API другой
Behavior<Actor>	root + дерево	❌
INextBotComponent::Update	Тик дерева	❌
IVision::IsAbleToSee	decorator_picker_visible	⚠️ Только концептуально
CKnownEntity	decorator_memory + ключ blackboard	❌
GetPrimaryKnownThreat	selector + decorator_ranker_dist + picker_max_score	⚠️ Логику можно воспроизвести, код — нет
PathFollower	action_move_to	❌
ILocomotion::Approach	action_move_to	❌
Главный вывод: слитый C++ код — это музейная ценность для понимания эволюции ИИ Valve, но не рабочий инструмент для CS2. Ваш рабочий инструмент — KV3-файлы, Source2 Wiki и официальные деревья из вашего архива.

Источники
Source2 Wiki: Bot behavior trees

Valve Developer Community: CS:GO Bot Behavior Trees

Предоставленные KV3-файлы: bt_config.kv3, bt_default.kv3 (deathmatch, armsrace, rush, practice, coop, guardian), модули (bt_attack.kv3, bt_memorize_*.kv3, bt_plant_bomb_*.kv3 и др.).

Слитый C++ код NextBot (использован только для сравнения, исключён из практических рекомендаций).

Этот ответ сгенерирован AI, только для справки.
Блять, ты можешь написать все красиво? Ебаный ты карась. Зачем ты пишешь про слитый C++, если он здесь не нужне. ПРОЙДИСЬ ПО ВСЕМ ИСХОДНЫМ ФАЙЛАМ и составь понятную википедию (можешь ее переделать). Добавь, если надо, каких-то уточнений, пояснений. Сделай так, чтобы было понятно. Сделай так, чтобы по этой википедии можно было написать свой скрипт ботов блять.

СДЕЛАЛ ДАВАЙ БЫСТРО. Удали мусорную иннформацию если нужно (или переделай понятно) и пришли красивый понятный вики блять
Пользователь хочет получить единую, понятную вики по ботам CS2/CS:GO на Behavior Trees, основанную на всех загруженных файлах. Нужно убрать упоминания слитого C++ кода, убрать мусор, сделать структурированно и практично, чтобы можно было написать свой скрипт. Нужно пройтись по всем исходным файлам: bt_config (CS:GO и CS2), bt_default (deathmatch, armsrace, rush, practice, coop, guardian), модули, а также вики Source2 и CS:GO. Объединить, уточнить, сделать понятным.

Основные разделы:

Введение: что такое BT в CS2, как работает, где лежат файлы.

Быстрый старт: как создать своё дерево, команды.

Формат KV3: синтаксис, структура файла.

Конфиг (bt_config.kv3): уровни скилла, параметры, таблица attack (13 строк в CS2, 16 в CS:GO).

Узлы: полный справочник с ключами, типами, описанием, статусом (официальный/не используется). Разделить на композиты, декораторы, условия, действия.

Модель выполнения: два прохода, поведение композитов, пример.

Значения и Blackboard: строки, числа, позиции, домены, память, глобальные флаги, subtree params.

Зарезервированные переменные.

Отладка.

Различия CS:GO vs CS2.

Примеры деревьев и модулей (из предоставленных файлов).

Практические рекомендации и типичные ошибки.

Нужно убрать всё про C++ (слитый код), убрать дублирование, сделать таблицы читаемыми. Уточнить, что в CS2 13 строк attack, в CS:GO 16 (но в предоставленном CS:GO конфиге 20 строк? Перепроверить: в файле bt_config.txt (CS:GO) строки: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN — это 20. В вики CS:GO сказано 16 строк. Возможно, в вики ошибка или в файле лишние. Но в CS2 ровно 13: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, UNKNOWN. В предоставленных CS2 конфигах именно 13 строк. В CS:GO конфиге из файла — 20 строк. В вики CS:GO указано 16 строк, но перечислены: KNIFE, PISTOL, SUBMACHINEGUN, RIFLE, SHOTGUN, SNIPER_RIFLE, MACHINEGUN, C4, TASER, GRENADE, EQUIPMENT, STACKABLEITEM, FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD, WEAPONTYPE_ZONE_REPULSOR, UNKNOWN — это 20. Значит, в вики CS:GO, вероятно, ошибка (16 строк — это старые данные). В CS2 точно 13. Укажем: CS:GO — 16 строк в старых версиях, но в современных (и в предоставленном файле) может быть больше; CS2 — строго 13.

Нужно сделать вики самодостаточной: включить все узлы, которые есть в файлах, с пометками, где используются. Убрать C++.

Сделаю структурированный документ с заголовками, таблицами, примерами. Без лишней воды. Пользователь торопится, поэтому нужно быстро и чётко.

План ответа:

Заголовок.

Введение (что такое BT, где лежат, как включить).

Быстрый старт (3 шага).

Формат KV3 (синтаксис, пример).

Конфиг (уровни, параметры, attack table).

Узлы (полный справочник с таблицами).

Модель выполнения (два прохода, пример).

Значения и Blackboard.

Зарезервированные переменные.

Отладка.

Различия CS:GO/CS2.

Примеры (минимальное дерево, паттерны, официальные модули).

Рекомендации и ошибки.

Убрать всё про C++. Уточнить, что в CS2 13 строк attack, в CS:GO 16 (но в файле 20 — возможно, кастом). Сделать акцент на CS2, так как пользователь спрашивает про CS2. Но упомянуть CS:GO для контекста.

Сделаю таблицы с ключами и описанием. Для узлов укажу, какие ключи обязательны, какие опциональны. Где возможно, дам примеры.

Проверю все файлы: bt_config (CS2 и CS:GO), bt_default (deathmatch, armsrace, rush, practice, coop, guardian), модули (bt_attack, bt_memorize_, bt_plant_bomb_, bt_pickup_, bt_defuse_, bt_heal_if_needed, bt_flee_area_damage_threats, bt_look_around, bt_face_damage_inflictor, bt_attack_damage_inflictor_smoke, bt_clear_threats_within_fov, bt_select_nearby_threat, bt_memorize_threats, bt_determine_rush_center, bt_throw_utility_if_covered, bt_basic_player_memorization, bt_basic_player_engagement, bt_pickup_hostage, bt_pickup_hostage_if_covered, bt_plant_bomb_if_covered, bt_plant_bomb_if_owned). Также новые узлы из Guardian: action_entity_accept_input, condition_has_parachute, condition_is_airborne, condition_inactive, action_commit_suicide, action_compare_global_counter, action_coordinated_buy, action_crouch, action_jump, action_parachute_positioning, action_select_areas_within_radius, action_set_global_counter, action_set_value_float, action_set_value_vector. Их тоже включить с пометкой "используется в Guardian/Co-op".

Убрать дублирование. Сделать единый справочник.

Начну писать.

Behavior Trees ботов CS2 и CS:GO — полная вики
Содержание
Что такое Behavior Tree

Быстрый старт

Формат KV3

Конфиг бота (bt_config.kv3)

Узлы (Nodes)

Модель выполнения

Значения и Blackboard

Зарезервированные переменные

Отладка

Различия CS:GO и CS2

Примеры и паттерны

Практические рекомендации

1. Что такое Behavior Tree
Behavior Tree (BT) — это KV3-файл, описывающий поведение бота как дерево узлов, которое игра вычисляет каждый тик (~64 раза в секунду). Дерево используется в режимах Deathmatch, Arms Race, Rush, New User Training (Practice), Guardian и Co-op.

Файлы лежат в game/csgo/pak01_dir.vpk (извлекаются через Source2 Viewer) по путям scripts/ai/....

Включение: ConVar mp_bot_ai_bt задаёт путь к дереву. Если пуст — классический ИИ.

После правок: mp_bot_ai_bt_clear_cache, затем переспавн ботов (перезапуск раунда).

Своё дерево: положите .kv3 в scripts/ai/ (например, scripts/ai/mymode/bt_default.kv3) и укажите в mp_bot_ai_bt.

Официальные режимы задают деревья в конфигах:

Режим	Путь
Deathmatch / DM Freeforall	scripts/ai/deathmatch/bt_default.kv3
Arms Race	scripts/ai/armsrace/bt_default.kv3
Rush	scripts/ai/rush/bt_default.kv3
New User Training	scripts/ai/practice/newplayer_dust2_01.kv3
2. Быстрый старт
Создайте файл scripts/ai/mymode/bt_default.kv3.

Вставьте минимальное дерево (см. ниже) и настройте config на нужный конфиг.

В консоли сервера: mp_bot_ai_bt "scripts/ai/mymode/bt_default.kv3", затем mp_bot_ai_bt_clear_cache и перезапустите раунд.

С developer 1 боты при спавне пишут [AI BT]: Loaded behavior tree '<file>'. Ошибки пишутся всегда с префиксом [AI BT].

Минимальное дерево:

kv3
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
Это дерево заставляет бота бежать к энтити bot_target, смотреть на неё и ждать 2–3 секунды, повторяя цикл.

3. Формат KV3
Каждый файл начинается с заголовка:

text
<!-- kv3 encoding:text:version{e21c7f3c-8a33-41c5-9977-a76d3a32aa0d} format:generic:version{7412167c-06e9-4698-aff2-e63eb59037e7} -->
Далее — объект:

config — путь к конфигу (обязателен для корневого дерева).

root — корневой узел (обязателен).

Узел — таблица с ключом type. Декораторы и условия имеют одного ребёнка child; композиты — массив children. Subtree-файлы содержат один узел верхнего уровня без config и root.

Синтаксис:

Комментарии //.

Имена ключей регистронезависимы.

Строки-литералы в одинарных кавычках: 'CT'.

Числа, массивы [ ... ], вложенные объекты { ... }.

4. Конфиг бота (bt_config.kv3)
Конфиг задаёт «мышечную память»: скорость прицеливания, реакцию, стрельбу. Уровни скилла:

CS:GO: default, low, fair, normal, tough, hard, very_hard, expert (8).

CS2: добавлен elite (9).

Каждый уровень наследует недостающие ключи из default.

4.1 Параметры прицеливания
Ключ	Тип	Описание
aim_target_acquisition_lerp_time	float	Время «доезда» прицела до новой цели
aim_target_acquisition_lerp_time_deviation	float	Разброс времени
aim_target_acquisition_angle_penalty	float	Штраф за угол до цели
aim_target_acquisition_angle_penalty_deviation	float	Разброс штрафа
aim_target_acquisition_angle_penalty_reduction_ratio	float	Уменьшение штрафа со временем
aim_target_acquisition_angle_tolerance	float	Допуск угла при захвате
aim_target_acquisition_angle_lerp_bias	float	Сглаживание угла захвата
aim_target_tracking_lerp_time	float	Скорость сопровождения
aim_target_tracking_lerp_time_deviation	float	Разброс сопровождения
aim_target_tracking_focus_interval	float	Интервал «фокуса» при трекинге
aim_target_tracking_focus_interval_deviation	float	Разброс интервала
aim_target_tracking_angle_lerp_bias	float	Сглаживание трекинга
aim_ready_angle_tolerance	float	Угол, при котором action_attack считает прицел готовым (в CS:GO не документирован)
aim_new_target_angle_tolerance	float	Угол для переключения на новую цель
aim_max_duration	float	Максимальное время прицеливания
aim_max_duration_deviation	float	Разброс макс. времени
aim_punch_angle_reaction_chance	float	Шанс реакции на отдачу
4.2 Осмотр
Ключ	Описание
look_around_awareness_yaw_range	Макс. угол по горизонтали
look_around_awareness_pitch_range	Макс. угол по вертикали
look_around_focus_interval	Пауза между поворотами
look_around_focus_interval_deviation	Разброс паузы
look_around_lerp_time	Время поворота
look_around_lerp_time_deviation	Разброс времени
look_around_lerp_bias	Сглаживание
4.3 Реакция и бой
Ключ	Описание
reaction_time	Задержка реакции на событие (сек)
combat_crouch_chance	Шанс присесть в бою (0–1)
combat_dodge_command_duration	Длительность уклонения
combat_dodge_command_duration_deviation	Разброс длительности
4.4 Таблица attack
Каждая строка — 5 чисел: burst duration, duration deviation, cadence, cooldown, cooldown deviation.

CS2 — ровно 13 строк, фиксированный порядок:

KNIFE

PISTOL

SUBMACHINEGUN

RIFLE

SHOTGUN

SNIPER_RIFLE

MACHINEGUN

C4

TASER

GRENADE

EQUIPMENT

STACKABLEITEM

UNKNOWN

Если строк не 13 — конфиг отвергается с ошибкой [AI BT]: Config node 'attack' must contain 13 entries.

CS:GO — в старых версиях 16 строк (включая WEAPONTYPE_ZONE_REPULSOR), но в предоставленном файле 20 строк (добавлены FISTS, BREACHCHARGE, BUMPMINE, TABLET, MELEE, SHIELD). Для CS2 используйте только 13.

Пример для RIFLE в default (CS2): [ 0.22, 0.07, 0.0, 0.45, 0.15 ] — очередь ~0.22 с (±0.07), без паузы внутри (cadence 0), новая очередь через 0.45 с (±0.15).

5. Узлы (Nodes)
5.1 Типы значений
Тип	Значение
bool	0 или 1
int, float	Литеральное число
number	Число, ConVar (@mp_...) или ключ blackboard
position	Литерал "x y z", ключ blackboard или targetname энтити
string	Литерал в '...' или ключ blackboard
enum	Значение из таблицы Enum
node, node[]	Дочерний узел / массив узлов
5.2 Enum-значения
Ключ	Значения
entity_type_filter	ALL, PLAYERS, HUMAN_PLAYERS, NOISE, DAMAGE, AREA_DAMAGE, CLASSNAME, GRENADE
team_filter, attacker_filter	ANY, CT, TERRORIST, SAME, OPPOSITE, ENEMY
team	ANY, CT, TERRORIST, OPPOSITE
team_restriction	ANY, CT, T
shape type	sensor_shape_fov, sensor_shape_sphere (с radius)
movement_type	BT_ACTION_MOVETO_WALK, BT_ACTION_MOVETO_RUN
route_type	BT_ACTION_MOVETO_FASTEST_ROUTE, BT_ACTION_MOVETO_SAFEST_ROUTE
slot	RIFLE, PISTOL, KNIFE, GRENADES, C4
types (гранаты)	FLASH, FIRE, EXPLOSIVE, DECOY, SMOKE
operation_type	BT_DECORATOR_TAG_ENTITY_SET, BT_DECORATOR_TAG_ENTITY_CLEAR
check_type	BT_DECORATOR_TAG_THRESHOLD_AT_LEAST, BT_DECORATOR_TAG_THRESHOLD_AT_MOST
weapon	BEST или classname (например, weapon_knife)
5.3 Композиты
Узел	Ключи	Поведение
parallel	succeed_after_first (bool), children	Все дети каждый тик. Провал любого = провал. Успех = все (или первый при succeed_after_first=1).
selector	children	Список приоритетов. Проверка с первого ребёнка каждый тик. Более ранний прошедший останавливает текущего и запускается.
sequencer	children	По порядку. Провал = стоп. Check-pass смотрит только на текущего ребёнка — пройденные условия не перепроверяются.
subtree	file (string), name (string), params ({key, value}[])	Запуск другого дерева. params пишут значения в blackboard до старта. Только int или string — float/bool отвергаются.
5.4 Декораторы
Оборачивают одного ребёнка через child. Почти все поддерживают negated = 1.

Узел	Ключи	Назначение
decorator_bot_service	memory_to_expire[], tagged_entities_to_expire[], input_chatter_enemies, basic_chatter_enable, chatter_outnumbered_threshold, child	Корневой сервис: истечение памяти, болтовня.
decorator_buy_service	output, child	Возвращает массив к покупке.
decorator_hiding_spot_service	domain, output_hiding_spot, expiration_time, distance_threshold, child	Учёт укрытий; боты в одном домене не проверяют уже проверенные.
decorator_maybe	chance (float), child	Выполняет ребёнка с шансом (1 = 100%).
decorator_memory	input, output, output_domain, child	Копирует сущности в память.
decorator_need_healing	health_threshold, child	Ребёнок, если HP ниже порога.
decorator_picker_blocked_by_smoke	input, input_domain, output, output_domain, negated, child	Убирает сущности, не покрытые дымом. distance_threshold игнорируется.
decorator_picker_dedup	input, input_domain, output, output_domain, against, distance_threshold, negated, child	Убирает дубликаты по дистанции до against.
decorator_picker_grenade_type	input, input_domain, output, output_domain, types[], negated, child	Фильтр гранат по типу.
decorator_picker_max_score	input, input_domain, output, output_domain, negated, child	Берёт первую сущность (по умолчанию — по порядку на сервере).
decorator_picker_nearby	input, input_domain, output, output_domain, cutoff_distance, negated, child	Убирает дальше порога.
decorator_picker_random_by_distance	input, input_domain, output, output_domain, distance_min, distance_max, negated, child	Вероятностный фильтр по дистанции.
decorator_picker_reaction_time	input, input_domain, output, output_domain, negated, child	Задержка по реакции бота.
decorator_picker_visible	input, input_domain, output, output_domain, check_fov (bool), negated, child	Убирает невидимые. check_fov = 0 игнорирует FOV.
decorator_picker_weight_as_distance	input, input_domain, output, output_domain, negated, child	Фильтр по «весу» (для слуха).
decorator_random_approach_point	output, child	Случайная видимая точка-«вход» в область.
decorator_random_int	min, max, output, child	Случайное целое.
decorator_ranker_dist	input, child	Сортирует по дистанции.
decorator_remove	input_domain, input, remove, child	Удаляет сущности из памяти.
decorator_remove_key	input, child	Удаляет ключ blackboard.
decorator_repeat	amount (int), child	Цикл. Без amount — вечно; с N — N раз.
decorator_run_once	domain, max_attempts, child	Ограничивает запуск. domain — только один бот выполнит.
decorator_sensor	output, class_name, priority, orphan_only, append_items, entity_type_filter, team_filter, attacker_filter, shape({type, radius}), child	Заполняет output sensed-сущностями.
decorator_set_reaction_time	input, child	Задержка по реакции.
decorator_succeed	negated, child	Всегда проходит check-pass; после ребёнка — success (или failure при negated=1).
decorator_tag_entity	input, output, expiration_time, operation_type, child	Помечает первую сущность из input.
decorator_tag_threshold	entity_input, tagged_entities_input, amount, check_type, child	Сравнивает количество помеченных.
decorator_token_service	config({tokens[], assignments[][], team_restriction[]}), output_token_name, output_token_domain, domain, child	Группы ботов (StratTokens).
decorator_try_lock	domain, child	Блокировка домена — только один бот одновременно.
decorator_invert	child	Инверсия check-результата ребёнка. В официальных деревьях CS2 не используется (в CS:GO — workaround).
decorator_wait_success	timeout, child	Ждёт успеха ребёнка или таймаута. Используется в Guardian.
decorator_dec_global_counter	input_name, child	Уменьшает счётчик. Не используется.
decorator_find_utility_strat	много	Не используется.
decorator_route_service	много	Не используется.
decorator_set_barrier	много	Не используется.
Важно: параметры acquire_only у action_aim, ratio у action_pull_trigger и distance_threshold у decorator_picker_blocked_by_smoke игнорируются игрой. В официальных файлах они присутствуют как наследие — не полагайтесь на них.

5.5 Условия
Узел	Ключи	Назначение
condition_is_empty	input, global (bool), negated, child	Пусто/не существует. global = 1 — глобальный флаг.
condition_is_equal	source, destination (int или string), negated, child	Равенство.
condition_is_greater_equal	source, destination, negated, child	≥.
condition_is_less_equal	source, destination, negated, child	≤.
condition_is_inv_slot_empty	slot (enum), negated, child	Слот пуст.
condition_is_weapon_equipped	weapon, negated, child	Активное оружие с этим classname.
condition_out_of_ammo	negated, child	Нет патронов.
condition_owns_item	item, items_one_of[], negated, child	Есть предмет.
condition_is_at_bomb_site	negated, child	На бомбсайте.
condition_distance_less	input, distance_threshold_min, distance_threshold_max, negated, child	Дистанция меньше порога. Не используется в основных деревьях.
condition_has_parachute	negated, child	Есть парашют. Используется в Guardian.
condition_inactive	input[], round_start_threshold_seconds, sensor_inactivity_threshold_seconds, negated, child	Нет активности. Используется в Guardian.
condition_is_airborne	negated, child	В воздухе. Используется в Guardian.
condition_is_greater	source, destination, negated, child	>. Не используется.
condition_is_less	source, destination, negated, child	<. Не используется.
condition_is_reloading	negated, child	Перезарядка. Не используется.
condition_is_weapon_suitable	input, weapon, negated, child	Подходит ли оружие для дистанции. Не используется.
condition_barrier	input_domain, input_name, radius, negated, child	Не используется.
5.6 Действия
Узел	Ключи	Назначение
action_acquire_items	items[], remove_all_items	Выдать предметы.
action_aim	input, ready (string)	Повернуть к точке. Никогда не разскопится со снайперки до смены оружия.
action_attack	input, ready, output	Стрелять по input, когда ready истинно.
action_buy	—	Случайная покупка.
action_choose_bomb_site_area	input (number), output	Позиция бомбсайта по индексу.
action_choose_random_waypoint	input, output	Случайный тайл навмеша.
action_choose_team_spawn_area	output, team (enum)	Navzone со спавнами.
action_combat_positioning	input, is_attacking (string)	Уклонение/сближение.
action_equip_item	item, items_one_of[]	Выбрать предмет.
action_equip_weapon	weapon (classname или BEST)	Выбрать оружие.
action_flee_area_damage	input, output, max_search_range, threat_min_keep_distance	Безопасная точка от урона по площади.
action_look_at	input_angles или input_location	Повернуть бота. Указывать одно из двух.
action_move_to	destination, hiding_spot, threat, auto_look_adjust, damaging_areas_penalty_cost, arrival_epsilon, additional_arrival_epsilon_2d, hiding_spot_check_distance_threshold, nearest_area_distance_threshold, movement_type, route_type	Двигаться к точке.
action_pull_trigger	—	Нажать ЛКМ.
action_use	—	Нажать +use.
action_wait	wait_time_min, wait_time_max, wait_modulo_tick_mod, wait_modulo_tick_add	Заморозить ИИ.
action_set_global_flag	name, expiration_time_min, expiration_time_max	Глобальный флаг на случайное время.
action_aim_projectile	input, output	Рассчитать углы броска. Баг: не учитывает траекторию.
action_choose_guardian_bomb_plant_location	output	Guardian: позиция закладки.
action_choose_random_waypoint_within_radius	origin, radius, output	Случайная точка в радиусе. Используется в Co-op.
action_commit_suicide	—	Самоубийство. Используется в Guardian.
action_compare_global_counter	input_name, input_value	Сравнить счётчик. Используется в Guardian.
action_coordinated_buy	id_no_purchase, purchases[], team_filter, save_threshold	Скоординированная покупка. Баг: один бот заставляет всех. Не используйте.
action_crouch	—	Присесть. Используется в Guardian.
action_custom_buy	item_aliases[]	Покупка по алиасам.
action_drop_active_weapon	—	—
action_hide	output, max_range	Случайное укрытие.
action_inspect_current_weapon	—	—
action_jump	—	Прыжок. Используется в Co-op.
action_parachute_positioning	—	Движение по диагонали. Используется в Guardian.
action_reload	—	Перезарядка.
action_say	phrase, high_priority	Радио-команда. Используется в Co-op.
action_secondary_attack	—	—
action_select_areas_within_radius	input, radius, output	Navzone в радиусе. Используется в Guardian.
action_set_global_counter	input_name, input_value	Установить счётчик. Используется в Guardian.
action_set_value_float	key, value	Записать float. Используется в Guardian.
action_set_value_vector	key, value	Записать позицию. Используется в Guardian.
action_standup	—	Встать.
action_teleport	destination	Телепорт.
action_entity_accept_input	name, input	Отправить input энтити. Используется в Guardian.
6. Модель выполнения
Дерево вычисляется каждый тик в два прохода:

Check-pass — сверху вниз. Решает, какие ветки имеют право выполняться. Если корень не проходит — тик «пустой».

Run-pass — выполняет выбранные ветки. Здесь двигаются, целятся, стреляют.

Поскольку check-pass повторяется каждый тик, условия перепроверяются постоянно, а selector может прервать текущую ветку, если более приоритетная начала проходить проверку.

Поведение композитов
sequencer — check-pass смотрит только текущего ребёнка; run-pass по порядку до провала. Условия позади не перепроверяются.

selector — check-pass с первого ребёнка каждый тик; запускает первого прошедшего; останавливает текущего при появлении более приоритетного.

parallel — все дети каждый тик; провал любого = стоп всех; успех = все (или первый при succeed_after_first = 1).

decorator_repeat — перезапускает ребёнка каждый раз, когда он завершается; с amount = N останавливается после N запусков и возвращает результат последнего.

decorator_succeed — всегда проходит check-pass; после ребёнка возвращает success (или failure при negated=1).

decorator_invert — инвертирует check-результат ребёнка.

condition_* — проверяется в каждом check-pass; пока условие истинно, выполняет ребёнка (или просто успех, если ребёнка нет); при провале — провал ветки. Все условия поддерживают negated = 1.

Пример разворачивания
text
sequencer[ move_to(bot_target), look_at(bot_target), wait(2..3) ] в repeat
Тик 1: move_to двигает на 1 тик → running.

Тики 2..N: игра идёт прямо в move_to — одно непрерывное движение.

Прибытие: move_to success → look_at success.

wait выбирает время окончания и возвращает running.

Все три готовы → sequencer success → repeat заново.

Узлы не двигают бота напрямую. action_move_to управляет pathfinding; action_use/action_reload «нажимают кнопку» на тик; action_aim/action_attack рулят встроенным aiming/shooting, пока не завершатся.

7. Значения и Blackboard
Большинство значений — ключи blackboard (хранилище бота). Сенсор пишет врагов в output = "Enemy", действие атаки читает их через input = "Enemy".

Строки
Литерал в одинарных кавычках: 'CT' — кавычки отбрасываются.

Иначе — имя ключа blackboard, который должен содержать строку.

Числа
Литеральное число.

ConVar: @mp_plant_c4_anywhere.

Ключ blackboard. Если ключ содержит строку, она разрешается ещё раз, но только на 1 уровень.

Позиции и имена сущностей
Ключи, ожидающие позицию (destination, input_location), принимают:

Литерал: "-1450 2230 2".

Ключ blackboard (позиция, строка или список sensed-сущностей — берётся первая).

Targetname энтити на карте, если ни один ключ так не называется. Используется origin энтити.

Это позволяет управлять ботами с карты: Rush-боты идут к ant.button по имени; скрипт может двигать target-энтити, чтобы перенаправлять ботов.

Домены
Способ делить состояние между ботами. Кавычки вокруг имён обязательны.

Домен	Где встречается	Назначение
AllBots	memory_to_expire, input_domain, output_domain	Общая память всех ботов
'StratTokens'	decorator_token_service	Токены стратегий (Practice, Guardian)
'PressAntennaButton'	decorator_try_lock	Блокировка нажатия кнопки (Rush)
'DefuseBombIfNearby'	decorator_try_lock	Блокировка разминирования (Practice)
Память
decorator_sensor → output = "Vision" (список сущностей).

decorator_picker_* → сужают "Vision".

decorator_memory → копирует в "ShortTermAttackMemory".

decorator_bot_service в корне → истекает память: {key, time, distance}.

Стандартные имена памяти:

ShortTermAttackMemory (0.7 с, 0 дистанция)

LongTermMemory (10 с, 500)

ShortTermInvestigateMemory (3 с, 200)

ShortTermDamageMemory (Practice, Guardian)

ShortTermAreaDamageMemory (Practice, Guardian)

DamageThroughSmokeMemory (Practice, Guardian)

Threats (Practice, Guardian, домен AllBots)

Глобальные флаги и счётчики
action_set_global_flag устанавливает именованный флаг на случайное время между expiration_time_min и expiration_time_max. Проверяется через condition_is_empty с global = 1.

action_set_global_counter, decorator_dec_global_counter, action_compare_global_counter — именованные счётчики.

Параметры subtree
subtree может записывать значения в blackboard до старта:

kv3
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
Значения — только целые или строки. Дробные и булевы отвергаются с ошибкой [AI BT]: unsupported type for subtree parameter.

8. Зарезервированные переменные
Игра сама кладёт их в blackboard. Читать можно, создавать нельзя.

Переменная	Тип	Откуда	Где используется
NumberOfTerrorists	int	Движок	condition_is_equal в bt_plant_bomb_if_owned
NumberOfAliveTerrorists	int	Движок	condition_is_equal в bt_plant_bomb_if_owned
NumberOfCounterTerrorists	int	Движок	—
NumberOfAliveCounterTerrorists	int	Движок	—
AccountBalance	int	Движок	—
AmmoCount/<weapon>	int	Движок	—
AmmoCount/current	int	Движок	—
TeamName	string	Движок	condition_is_equal source="TeamName" destination="'T'" в Practice
RoundNumber	int	Движок	Все bt_t_*.kv3 и bt_ct_*.kv3 в Practice
BlindnessPercentage	float	Движок	—
GuardianWaveNumber	int	Движок (Guardian)	—
LastCoopSpawnPointName	string	Движок (Co-op)	—
LastCoopSpawnPointLocation	Vector	Движок (Co-op)	—
Примечание: TeamName и RoundNumber отсутствуют в списке Source2 Wiki, но активно используются в Practice. Список на вики неполный.

9. Отладка
Команда	Назначение	Требования
mp_bot_ai_bt "путь"	Назначить дерево всем ботам	—
mp_bot_ai_bt_clear_cache	Сбросить кэш после правок	—
cv_bot_ai_bt_debug_target <index>	Нарисовать дерево бота	sv_cheats 1, локальный/приватный сервер
cv_bot_ai_bt_hiding_spot_show	Показать укрытия	sv_cheats 1
cv_bot_ai_bt_moveto_show_next_hiding_spot	Показать следующее проверяемое укрытие	sv_cheats 1
Что искать в консоли:

[AI BT]: Loaded behavior tree '<file>' — при developer 1.

[AI BT] warnings — всегда.

Blackboard entry 'X' does not exist nor it is a valid entity name — ключ не найден.

[AI BT]: Config node 'attack' must contain 13 entries — неверное число строк.

[AI BT]: unsupported type for subtree parameter — float/bool в params.

10. Различия CS:GO и CS2
Аспект	CS:GO	CS2
Уровни скилла	8 (без elite)	9 (с elite)
Строк в attack	16 (в старых версиях; в современных может быть больше)	13 (ровно, иначе ошибка)
aim_ready_angle_tolerance	Не документирован	Есть в конфиге
TeamName / RoundNumber	Не документированы	Используются в Practice
Документация узлов	Неточная, много TODO	Точный список читаемых ключей
Неиспользуемые узлы	Не отмечены	36 отмечены (но некоторые используются в Guardian/Co-op)
Игнорируемые параметры	Не отмечены	acquire_only, ratio, distance_threshold
Модель выполнения	Не описана	Два прохода, детально
Subtree params	Не описаны	Только int/string
decorator_picker_*	Без доменов	С input_domain/output_domain
decorator_sensor	Без append_items	С append_items
action_wait	2 ключа	+ wait_modulo_tick_mod/add
11. Примеры и паттерны
11.1 Простое дерево (движение, взгляд, ожидание)
kv3
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
11.2 Паттерны из официальных деревьев
Восприятие + решение через parallel:

text
parallel
├── repeat → parallel (sensors, memory)
└── repeat → selector (priority list)
Selector как список приоритетов:

text
selector
├── condition_is_empty ShouldBuy (negated) → sequencer[wait, buy, remove_key]
├── sensor DAMAGE → aim
├── subtree Attack
├── subtree HealIfNeeded
└── subtree Hunt
Блокировка действия одним ботом:

text
decorator_try_lock domain = "'PressAntennaButton'"
└── sequencer[ move_to ant.button, look_at, wait, use ]
Скоординированная покупка (не используйте action_coordinated_buy — он сломан):

text
decorator_run_once domain = "'BuyPhase'"
└── action_buy
Истечение памяти в корне:

text
decorator_bot_service
  memory_to_expire =
  [
    { key = "ShortTermAttackMemory"  time = 0.7  distance = 0 }
    { key = "LongTermMemory"         time = 10   distance = 500 }
    { key = "ShortTermInvestigateMemory" time = 3 distance = 200 }
  ]
11.3 Официальные деревья и модули
Deathmatch / DM Freeforall — scripts/ai/deathmatch/bt_default.kv3

Корень: decorator_bot_service → decorator_buy_service → parallel (восприятие + selector).

Selector: покупка, реакция на урон, атака, лечение, исследование, охота.

Arms Race — scripts/ai/armsrace/bt_default.kv3 (аналогично).

Rush — scripts/ai/rush/bt_default.kv3

Дополнительно: decorator_try_lock на 'PressAntennaButton', движение к ant.button, использование.

Practice (New User Training) — scripts/ai/practice/newplayer_dust2_01.kv3

decorator_hiding_spot_service → decorator_token_service ('StratTokens') → decorator_bot_service.

В selector: ItemStrategy, Attack, FaceDamageInflictor, AttackDamageInflictorSmoke, ClearThreatsWithinFov, SelectNearbyThreat, TakeoverStrategy, проверки TeamName для закладки/разминирования, исследование, охота.

Co-op (Guard, Roam, Rush, Sniper) — scripts/ai/coop/bt_guard.kv3 и др.

decorator_hiding_spot_service → decorator_bot_service (с basic_chatter_enable, input_chatter_enemies).

BasicPlayerMemorization, BasicPlayerEngagement, исследование, спецповедение (охрана точки, рой, снайпер).

Guardian (пример op11_sirocco_2.kv3)

decorator_hiding_spot_service → decorator_token_service ('StratTokens' с токенами стратегий) → decorator_bot_service.

Selector: condition_inactive (самоубийство при неактивности), ItemStrategy, Attack, FaceDamageInflictor, AttackDamageInflictorSmoke, ClearThreatsWithinFov, SelectNearbyThreat, TakeoverStrategy, PlantIfCovered, PlantIfOwned, PickupBombIfNearby, HealIfNeeded, исследование, охота.

Модули: bt_item_strategy, bt_takeover_strategy, bt_attack_with_shield, bt_investigate_with_shield, bt_memorize_threats, bt_clear_threats_within_fov, bt_select_nearby_threat и др.

Общие модули

bt_attack.kv3 — выбор цели, экипировка, позиционирование, прицеливание, атака.

bt_memorize_enemies_vision.kv3 — сенсор зрения (FOV, PLAYERS, ENEMY, nearby 1500, visible, реакция, память).

bt_memorize_noises.kv3 — сенсор слуха (сфера 3000, NOISE, weight_as_distance, random_by_distance, dedup, реакция, память).

bt_memorize_damage.kv3 — сенсор DAMAGE → ShortTermDamageMemory и DamageThroughSmokeMemory.

bt_memorize_area_damage_*.kv3 — гранаты, инферно, текущий урон.

bt_memorize_nearest_investigation.kv3 — выбирает ближайшее событие из LongTermMemory в ShortTermInvestigateMemory.

bt_investigate_closest_memorized_event.kv3 — идёт к событию, смотрит, ждёт.

bt_heal_if_needed.kv3 — лечение при HP < 60.

bt_face_damage_inflictor.kv3 — поворот к источнику урона, атака (если не в дыму).

bt_attack_damage_inflictor_smoke.kv3 — атака источника урона сквозь дым.

bt_flee_area_damage_threats.kv3 — бегство из зоны урона.

bt_look_around.kv3 — случайный обзор.

bt_plant_bomb_if_covered.kv3 — закладка при прикрытии.

bt_plant_bomb_if_owned.kv3 — закладка, если есть бомба (с флагом DontPlantBomb).

bt_pickup_bomb_if_nearby.kv3 — подбор бомбы.

bt_defuse_bomb_if_nearby.kv3 — разминирование.

bt_pickup_hostage.kv3, bt_pickup_hostage_if_covered.kv3 — захват заложника.

bt_clear_threats_within_fov.kv3 — удаление угроз из общего списка, если они в FOV.

bt_select_nearby_threat.kv3 — выбор ближайшей видимой угрозы в NearestThreatMemory.

bt_memorize_threats.kv3 — добавление источников урона в глобальный Threats.

bt_determine_rush_center.kv3 — определение центра раша по имени спавна (Co-op).

bt_throw_utility_if_covered.kv3 — бросок гранаты при прикрытии.

bt_basic_player_memorization.kv3, bt_basic_player_engagement.kv3 — базовые модули Co-op.

12. Практические рекомендации
12.1 С чего начать
Извлеките официальные деревья через Source2 Viewer из pak01_dir.vpk. Изучите bt_default.kv3 (Deathmatch) и bt_practice/newplayer_dust2_01.kv3 — это лучшие образцы.

Не пытайтесь «портировать» C++ логику. Думайте в терминах узлов, blackboard и приоритетов.

Начните с простого дерева и итеративно усложняйте.

12.2 Чего избегать
❌ Не делайте	✅ Делайте вместо этого
Полагаться на acquire_only, ratio, distance_threshold	Эти параметры игнорируются
Использовать action_coordinated_buy	decorator_run_once + action_buy
Забывать mp_bot_ai_bt_clear_cache	Всегда чистить кэш после правок
Передавать float/bool в subtree params	Только int или string
Использовать 16 строк в attack (CS2)	Ровно 13, в фиксированном порядке
Ожидать, что action_aim разскопится	Смените оружие или экипируйте то же заново
Полагаться на condition_is_greater/is_less	Используйте greater_equal/less_equal
12.3 Проверка «мёртвых» параметров
Создайте два дерева: с параметром и без.

Запустите cv_bot_ai_bt_debug_target на боте.

Сравните поведение. Если разницы нет — параметр мёртв.

Источники
Source2 Wiki: Bot behavior trees

Valve Developer Community: CS:GO Bot Behavior Trees

Предоставленные KV3-файлы: конфиги (CS:GO и CS2), деревья Deathmatch, Arms Race, Rush, Practice, Co-op, Guardian, модули.
