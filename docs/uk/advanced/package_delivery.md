# Доставка пакунків у місіях

<Badge type="tip" text="PX4 v1.14" />

Місія доставки пакунка - це розширення операції з шляховою точкою, де користувач може планувати призначення пакету в якості шляхової точки.

Ця тема пояснює архітектуру функції доставки пакету.
Він призначений для розробників, які працюють над розширенням архітектури, наприклад, для підтримки нових механізмів доставки вантажу.

:::info
Зараз лише [Grippers](../peripherals/gripper.md) може бути використано для доставки пакету.
Лебідки поки що не підтримуються.
:::

:::info
Детальну документацію з налаштування плану місії доставки пакетів можна знайти [тут](../flying/package_delivery_mission.md).
Підготовка для модуля `payload_deliverer` описана у документації для механізму доставки, наприклад, [Gripper](../peripherals/gripper.md#px4-configuration).
:::

## Діаграма Архітектури доставки пакетів

![Package delivery architecture overview](../../assets/advanced_config/payload_delivery_mission_architecture.png)

Функціонал доставки пакетів зосереджений навколо повідомлень [VehicleCommand](../msg_docs/VehicleCommand.md) та [VehicleCommandAck](../msg_docs/VehicleCommandAck.md).

Основна ідея полягає в наявності сутності, яка обробляє команду транспортного засобу `DO_GRIPPER` або `DO_WINCH`, виконує її і надсилає підтвердження, коли успішна доставка підтверджена.

Оскільки PX4 автоматично транслює повідомлення uORB `VehicleCommand` до UART-порту, налаштованого на комунікацію у форматі MAVLink як повідомлення [`COMMAND_LONG`](https://mavlink.io/en/messages/common.html#COMMAND_LONG), зовнішній навантаження може отримати команду і виконати її.

Аналогічно, оскільки PX4 автоматично перекладає повідомлення [`COMMAND_ACK`](https://mavlink.io/en/messages/common.html#COMMAND_ACK), що надходить зовнішнім джерелом через порт UART, налаштований на MAVLink, в uORB-повідомлення `vehicle_command_ack`, підтвердження зовнішнього навантаження про успішне розгортання пакета може бути отримано модулем `navigator` PX4.

Нижче є пояснено кожен об'єкт, що бере участь в архітектурі доставки пакету.

## Навігатор

Навігатор обробляє приймання команди ТЗ (описано нижче).
Після отримання повідомлення про успішне розгортання воно встановлює прапорець на рівні блоку місії, щоб сигналізувати про успішне розгортання вантажу.

Це дозволяє місії перейти до наступного пункту (наприклад, Waypoint) безпечно, оскільки ми впевнені у підтвердженні успішного виконання розгортання.

## Транспортний Командний ACK

Ми чекаємо на підтвердження (ACK), яке може прийти як внутрішнє (через модуль `payload_deliverer`), так і зовнішнє (зовнішня сутність відправляє повідомлення MAVLink `COMMAND_ACK`), щоб визначити, чи була успішною дія доставки пакету (або `DO_GRIPPER`, або `DO_WINCH`).

## Місія

Команда «Захват/лебідка» розміщується як `предмет місії`.
Це можливо, оскільки всі пункти місії мають команду `MAV_CMD` для виконання (наприклад, Land, Takeoff, Waypoint і т. д.), яку можна встановити на `DO_GRIPPER` або `DO_WINCH`.

У логіці місії (зелена рамка вище), якщо досягнуто будь-який пункт місії Gripper/Winch, вона використовує функціональність brake_for_hold (яка встановлює прапорець `valid` наступного пункту місії зупинки на значення `false`) для вертольотів (наприклад, багтротика), щоб транспортний засіб утримував своє положення, поки виконується розгортання.

Для фіксованих крил та інших транспортних засобів не розглядається жодна особлива умова зупинки.
Для літаків з фіксованими крилами та інших типів транспортних засобів не передбачено жодних спеціальних умов щодо зупинки.

## Блок Місії

`MissionBlock` є батьківським класом `Mission`, який відповідає за частину "Чи завершена місія?".

Все це виконується у функції `is_mission_item_reached_or_completed`, щоб обробляти затримку часу / перехід до наступного пункту місії.

Також вона реалізує функцію issue_command, яка видасть команду транспортному засобу, відповідну команді `MAV_CMD` пункту місії, яку потім отримає зовнішній вантаж або модуль `payload_deliverer` внутрішньо.

## Доставщик Вантажу

Це спеціалізований модуль, який відповідає за підтримку захопника / лебідки, який використовується для стандартного [плану місії доставки пакетів](../flying/package_delivery_mission.md).

Налаштування для модуля `payload_deliverer` описано у документації щодо налаштування реального механізму випуску пакету, такого як [Gripper](../peripherals/gripper.md#px4-configuration).
