# Качество воздуха по DALI: IEC 62386-306

Как снять температуру, влажность, eCO₂, eVOC и звук с `zc-pms-da-2sw-r`. Собрано 13.10.2026 из интернета и из текста стандарта `DALI/iec62386-306{ed1.0}b.pdf` (IEC 62386-306:2023).

## Как zencontrol отдаёт эти величины

- Климатические величины — инстансы IEC 62386-306 (general purpose sensor), instance type 6.
- База знаний zencontrol говорит это прямо про `zc-pms-iot-2sw-r`. Для `-da` даташит пишет «IEC62386-3XX Generic input sensor (x4)», стандарт не называет. Считаем, что это тоже 306, пока не проверено опросом.
- zencontrol писали черновик 306 и первыми его внедрили в своих air-quality сенсорах.
- Почему в таблице мануала для `-2sw-r` только 5 инстансов, а у инстанса 2 тип 0 — нигде не объяснено. Проверять опросом.
- Их собственный путь: application controller + TPI Advanced / MQTT / BACnet. Нам не нужен, если есть DALI-мастер.

## Порядок чтения инстанса 306

Команды control device, 24-битные кадры (IEC 62386-103).

1. Найти инстансы: `QUERY NUMBER OF INSTANCES`, затем для каждого `QUERY INSTANCE TYPE`. Ищем тип 6.
2. Разрядность: `QUERY RESOLUTION`. `inputValue` занимает 1 байт при resolution ≤ 8, 2 байта при ≤ 16, 3 байта при ≤ 24. Значимые биты — старшие, хвост дополняется повторением старших бит.
3. Значение: `QUERY INPUT VALUE` даёт старший байт, каждый следующий `QUERY INPUT VALUE LATCH` — следующий байт. Все байты 0xFF = MASK: прогрев или отказ датчика, значение недействительно.
4. Что за величина и масштаб: `DTR0 = n`, затем `QUERY MEASUREMENT VARIABLE` (опкод 0x5F). После каждого вызова DTR0 сам увеличивается на 1, многобайтные значения читаются подряд.
5. Пересчёт: `measuredValue = signal / 10^(magnitude − 127) + K`, где K = 2^(resolution−1) − 1 для знаковых входов (`inputSignalSigned` = 1), иначе 0. Обратно: `signal = (measuredValue − K) · 10^(magnitude − 127)`.
6. События по желанию: `SET EVENT FILTER`, `SET REPORT TIMER`, `SET DEADTIME TIMER`, `SET HYSTERESIS`. Событие measurement несёт 9 старших бит `inputValue`, бит 9 = 1. Периодические отчёты идут с приоритетом 5, по изменению — 4.
7. Ошибки: `instanceError` виден в `QUERY INSTANCE STATUS`; подробности — `QUERY INSTANCE ERROR` (бит 0 — отказ физического датчика, биты 4–7 — ошибки производителя).

## Опкоды команд 306 (таблица 16 стандарта)

Адресация: device + instance, как у остальных инстансных команд 103.

| Опкод | Команда | Примечание |
| --- | --- | --- |
| 0x50 | SET REPORT TIMER (DTR0) | `tReport` |
| 0x51 | SET ALARM REPORT TIMER (DTR0) | `tAlarmReport` |
| 0x52 | SET HYSTERESIS (DTR0) | 0…25, иначе игнор |
| 0x53 | SET DEADTIME TIMER (DTR0) | |
| 0x54 | SET HYSTERESIS MIN (DTR0) | |
| 0x55 | SET ALARM TYPE (DTR0) | |
| 0x56 | SET MAGNITUDE (DTR0) | только в пределах `magnitudePhMin…Max` |
| 0x57 | SET ALARM (DTR2,DTR1,DTR0) | |
| 0x58 | SET ALARM HYSTERESIS (DTR2,DTR1,DTR0) | |
| 0x5A | QUERY HYSTERESIS MIN | |
| 0x5B | QUERY DEADTIME TIMER | |
| 0x5C | QUERY REPORT TIMER | |
| 0x5D | QUERY ALARM REPORT TIMER | |
| 0x5E | QUERY HYSTERESIS | |
| 0x5F | QUERY MEASUREMENT VARIABLE (DTR0) | ответ по таблице ниже |

Опкод 0x59 в таблице не используется.

## QUERY MEASUREMENT VARIABLE: селекторы DTR0 (таблица 17)

| DTR0 | Переменная |
| --- | --- |
| 0x00 | `alarmIsActivated` |
| 0x01 | `alarmType` |
| 0x02 | бит 0 = `inputSignalSigned` |
| 0x03–0x0E | `alarm0…alarm3`, по 3 байта, MSB первый |
| 0x0F–0x1A | `alarm0Hysteresis…alarm3Hysteresis`, по 3 байта |
| 0x1B | `unitOfMeasurement` |
| 0x1C, 0x1D | `unitOfMeasurementExtended` LSB, MSB (смысл вендорский) |
| 0x1E | `quantityName` |
| 0x1F, 0x20 | `quantityNameExtended` LSB, MSB (смысл вендорский) |
| 0x21 | `magnitude` |
| 0x22 | `magnitudePhMax` |
| 0x23 | `magnitudePhMin` |
| 0x24–0x27 | `maxMeasuredValue`, 4 байта, MSB первый; чтение байта 0 защёлкивает, чтение последнего значимого байта сбрасывает в 0 |
| 0x28–0x2B | `minMeasuredValue`, то же, сброс в MASK |

Многобайтные значения выровнены по старшим битам, неиспользуемые биты = 0. На прочие DTR0 ответа нет, но DTR0 всё равно инкрементируется.

## Коды единиц и величин, которые нам нужны (Annex A)

`unitOfMeasurement`: 0 — вендорская, 1 — безразмерная, 8 — K, 9 — °C, 10 — °F, 16 — Pa, 49 — dB, 50 — %, 51 — ‰, 53 — ppm, 32 — lx. Код ppb в таблице отсутствует: для eVOC ждём ppm с magnitude или вендорский код 0.

`quantityName`: 0 — вендорская, 20 — уровень звукового давления, 26 — температура, 27 — температура по влажному термометру, 28 — абсолютная влажность, 29 — относительная влажность, 30 — точка росы, 31 — давление, 34 — CO₂, 36 — VOC, 46 — индекс качества воздуха IAQ, 44/45 — PM10/PM2.5.

Полные таблицы A.1 (0…53) и A.2 (0…58) — в стандарте, страницы 33–34.

## Что остаётся неизвестным до опроса

- Есть ли инстансы типа 6 на `-da` вообще. Подтверждение 306 есть только для `-iot`; для `-da` это гипотеза по даташиту.
- Номера инстансов, resolution, magnitude и единицы именно для этого корпуса. Стандарт разрешает `unitOfMeasurement` = 0 и `quantityName` = 0 с вендорским смыслом — тогда сопоставлять по физике (дыхание — eCO₂/eVOC/влажность, нагрев — температура, шум — звук).
- Даташит обещает 4 generic-инстанса, а величин пять. Либо одной нет на DALI у `-da`, либо она идёт через extended-коды. Узнаем по `quantityName`.

## План опроса

1. Стенд: сенсор на L/N, внешний БП шины DALI, DALI-мастер с поддержкой 24-битных кадров и чтением ответов на DA/DA.
2. Найти control device, назначить короткий адрес.
3. `QUERY NUMBER OF INSTANCES`, по каждому инстансу `QUERY INSTANCE TYPE`, `QUERY RESOLUTION`, `QUERY INSTANCE STATUS`. Повторить при других режимах ECD, если список короче ожидаемого.
4. Для инстансов типа 6: `QUERY MEASUREMENT VARIABLE` с DTR0 = 0x02, 0x1B…0x23; затем `QUERY INPUT VALUE` + `LATCH`.
5. Если тип 6 не найден: читать `QUERY INPUT VALUE` со всех инстансов и смотреть, что меняется при дыхании, нагреве, шуме.
6. Дамп кадров и итоговую таблицу инстансов положить сюда же.

## Источники

- `DALI/iec62386-306{ed1.0}b.pdf` — IEC 62386-306:2023, таблицы 16, 17, A.1, A.2, разделы 9.3, 9.4.3, 9.5.9, 9.5.10, 9.6. Лицензия однопользовательская, сюда вынесены только нужные значения.
- `DALI/IEC 62386-103-2014.pdf` — базовые инстансные команды.
- https://zencontrol.com/2025/03/05/dali-306-standard/
- https://lumenation.jitbit.com/KB/View/83142246-introduction-to-the-zencontrol-environmental-multisensor
- https://lumenation.jitbit.com/helpdesk/KB/View/83191503-introduction-to-iec-62386-306-expanding-data-input-capabilities-in-dali-systems-within-zencontrol
- https://zencontrol.com/wp-content/uploads/products/datasheets/zc-pms-da_Leaflet_Datasheet_w.pdf
- https://infosys.beckhoff.com/content/1033/tcplclib_tc3_dali/16002952843.html — Tc3_DALI, Part 306 (селекторы совпадают с таблицей 17)
- https://github.com/sde1000/python-dali — реализация 103/301/303 на Python, 306 там нет
