# RTK GPS (PX4 інтеграція)

[Реальний кінематичний час](https://en.wikipedia.org/wiki/Real_Time_Kinematic) (RTK) забезпечує точність GPS на рівні сантиметрів.
Ця сторінка пояснює, як RTK інтегрується в PX4.

:::tip
Інструкції щодо _використання_ RTK GPS наведені в розділі [Периферійне обладнання > RTK GPS](../gps_compass/rtk_gps.md).
:::

## Загальний огляд

RTK використовує виміри фази несучої хвилі сигналу, а не інформаційний вміст сигналу.
Він покладається на одну вихідну станцію для надання корекцій у реальному часі, які можуть працювати з кількома мобільними станціями.

Для налаштування RTK з PX4 потрібні два модулі RTK GPS та даталінк.
Закріплена на землі GPS-одиниця називається Базою, а повітряна одиниця - Ровером_.
Базова одиниця підключається до _QGroundControl_ (через USB) та використовує даталінк для передачі поправок RTCM до транспортного засобу (за допомогою повідомлення MAVLink [GPS_RTCM_DATA](https://mavlink.io/en/messages/common.html#GPS_RTCM_DATA)). На автопілоті пакети MAVLink розпаковуються та надсилаються до пристрою Rover, де вони обробляються для отримання рішення RTK.
На автопілоті пакети MAVLink розпаковуються та надсилаються до пристрою Rover, де вони обробляються для отримання рішення RTK.

Канал передачі даних зазвичай має підтримувати швидкість висхідної лінії зв’язку 300 байт на секунду (додаткову інформацію див. у розділі [Швидкість передачі даних у висхідній лінії зв’язку](#uplink-datarate)).

## Підтримувані GPS-модулі RTK

Список пристроїв, які ми тестували, можна знайти [в посібнику користувача](../gps_compass/rtk_gps.md#supported-rtk-devices).

:::info
Більшість пристроїв поставляється з двома варіантами: базою та ровером.
Переконайтеся, що ви обрали правильний варіант.
:::

## Автоматична конфігурація

Стек GPS PX4 автоматично налаштовує модулі GPS для надсилання та отримання правильних повідомлень через UART або USB в залежності від того, куди підключений модуль (до QGroundControl або автопілота).

Як тільки автопілот отримує повідомлення MAVLink `GPS_RTCM_DATA`, він автоматично пересилає дані RTCM до приєднаного модуля GPS через існуючі канали передачі даних (для даних корекції не потрібен окремий канал).

:::info
Інструмент конфігурації модуля RTK U-Center від u-blox не потрібний/не використовується!
:::

:::info
Both _QGroundControl_ and the autopilot firmware share the same [PX4 GPS driver stack](https://github.com/PX4/PX4-GPSDrivers).
На практиці це означає, що підтримку нових протоколів і/або повідомлень потрібно додавати лише в одному місці.
:::

### RTCM повідомлення

QGroundControl налаштовує базову станцію RTK на вивід наступних рамок RTCM3.2, кожну з частотою 1 Гц, якщо не вказано інше:

- \*\*1005 - Координати станції XYZ для точки антени (Базова позиція), 0.2 Гц.
- \*\*1077 - Повні псевдодальності GPS, фази несучої, Доплер та сила сигналу (висока роздільна здатність).
- \*\*1087 - Повні псевдодальності ГЛОНАСС, фази несущої, Доплер і сила сигналу (висока роздільна здатність).
- **1230** - Зміщення фаз коду ГЛОНАСС.
- Повні псевдодальності GLONASS, фази несущої, Доплер і сила сигналу (висока роздільна здатність)
- **1127** - Повні псевдодальності BeiDou, фази несущої, Доплер і сила сигналу (висока роздільна здатність)

## Uplink datarate(Швидкість передачі даних вгору)

The raw RTCM messages from the base are packed into a MAVLink `GPS_RTCM_DATA` message and sent over the datalink.
Максимальна довжина кожного повідомлення MAVLink становить 182 байти. Залежно від RTCM-повідомлення, повідомлення MAVLink майже ніколи не заповнюється повністю.

Повідомлення про базову позицію RTCM (1005) має довжину 22 байти, тоді як інші всі мають змінну довжину, залежно від кількості видимих супутників та кількості сигналів від супутника (лише 1 для однодіапазонних пристроїв, наприклад, M8P).
Оскільки в будь-який момент часу _максимальна_ кількість видимих супутників з будь-якої констеляції становить 12, то за реальних умов теоретично достатньо максимальної швидкості передачі даних в 300 байт/с.

Якщо використовується _MAVLink 1_, для кожного повідомлення RTCM, незалежно від його довжини, надсилається повідомлення `GPS_RTCM_DATA` розміром 182 байти.
Внаслідок цього приблизна вимога до пропускної здатності каналу зв'язку складає близько 700 байт на секунду.
Це може призвести до насичення каналу зв'язку на модулях телеметрії з низькою пропускною здатністю напівдуплексного режиму (наприклад, телеметричні радіозв'язки 3DR).

Якщо використовується _MAVLink 2_, будь-який порожній простір у повідомленні `GPS_RTCM_DATA` видаляється.
Отримана вимога до пропускної здатності для передачі вгору становить приблизно ту ж величину, що і теоретичне значення (близько 300 байт на секунду).

:::tip
PX4 автоматично переключається на MAVLink 2, якщо контрольна станція та модулі телеметрії підтримують це.
:::

MAVLink 2 слід використовувати на каналах з низькою пропускною здатністю для хорошої продуктивності RTK. Необхідно вдбати про те, щоб впевнитися, що ланцюжок телеметрії використовує MAVLink 2 на всіх етапах передачі даних.
Ви можете перевірити версію протоколу, використовуючи команду `mavlink status` у системній консолі:

```sh
nsh> mavlink status
instance #0:
        GCS heartbeat:  593486 us ago
        mavlink chan: #0
        type:           3DR RADIO
        rssi:           219
        remote rssi:    219
        txbuf:          94
        noise:          61
        remote noise:   58
        rx errors:      0
        fixed:          0
        flow control:   ON
        rates:
        tx: 1.285 kB/s
        txerr: 0.000 kB/s
        rx: 0.021 kB/s
        rate mult: 0.366
        accepting commands: YES
        MAVLink version: 2
        transport protocol: serial (/dev/ttyS1 @57600)
```
