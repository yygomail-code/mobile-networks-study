# 7. Оптимизация RAN с точки зрения Core

> Цель: понять, какие механизмы оптимизации RAN существуют и какую роль в них играет Core
> (MME/AMF/SMF/PCF/UPF), а также разобрать практические примеры.

## 7.1 Что значит «со стороны Core»

Core (MME/AMF/SMF/PCF/UPF) не управляет радиоресурсами напрямую, но:

- передаёт сигнализацию, влияющую на решения RAN (SON, HO, paging);
- предоставляет данные для оптимизации (MDT, trace, KPI);
- применяет политики QoS/трафика (PCRF/PCF, RCAF);
- участвует в path switch, bearer'ах, overload control.

## 7.2 SON и участие Core

| Функция | Что делает | Роль Core |
|---|---|---|
| ANR | находит соседей, ведёт neighbour relations | MME передаёт eNB Configuration Transfer (SON info) между eNB, если нет X2 |
| MRO | исправляет параметры HO (too early/late/wrong cell) | Core видит HO-сбои, RLF, KPI; MME — транспорт сигнализации |
| MLB | балансирует нагрузку между сотами | X2/S1 HO; MME участвует в S1 HO |
| RACH opt | подбирает параметры доступа | Косвенно через KPI |
| PCI opt | устраняет конфликты PCI | Косвенно |
| Energy saving | выключение capacity-сот | Core видит доступность сот, влияет на пейджинг |

Пример ANR: eNB обнаруживает неизвестный PCI; UE считывает ECGI; eNB формирует S1AP eNB Configuration Transfer (SON Configuration Transfer) → MME → целевой eNB; целевой отвечает своим TNL-адресом; поднимается X2. MME здесь — «релей» между eNB.

## 7.3 Мобильность и нагрузка

- **MME pool / AMF set**: распределение абонентов, снижение меж-MME TAU (S10/N14), отказоустойчивость.
- **Overload control**: S1AP Overload Start/Stop (MME → eNB); eNB отклоняет/ограничивает нагрузку; NAS back-off timer.
- **Paging optimization**: TAI list, повторные пейджинги, пейджинг по подзонам; баланс между TAU-нагрузкой и зоной пейджинга.
- **TA list optimization**: MME назначает TAI list; слишком широкий — много пейджинга, слишком узкий — много TAU.

## 7.4 Idle mode: eDRX и PSM

- **eDRX** (extended DRX): увеличенный цикл «сна» UE; параметры согласуются через NAS (MME/AMF), поддерживаются RAN.
- **PSM** (Power Saving Mode): UE «замирает» на длительное время; таймеры T3324/T3412 управляются Core.
- Применение: IoT (счётчики, датчики) — экономия батареи и сигнализации; минус — недоступность для MT-трафика.

## 7.5 Покрытие: MDT и trace

- **MDT** (Minimization of Drive Tests, TS 37.320):
  - signaling-based: активация через Core (HSS/MME) для выбранного абонента;
  - management-based: через OSS на eNB.
- Собираются: RSRP/RSRQ, RLF-отчёты, throughput (M1), задержки/потери (M2).
- Используется для оптимизации покрытия/ёмкости без drive-тестов.
- **Trace** (TS 32.422): общая трассировка сессий/абонентов, включая MDT.

## 7.6 Перегрузка: RCAF и UPCON

- **RCAF** (RAN Congestion Awareness Function): получает данные о перегрузке user plane и передаёт в PCRF по Np (Diameter, TS 29.217).
- PCRF применяет политики: throttling, gating, снижение качества, уведомление AF.
- **UPCON** (User Plane Congestion): управление перегрузкой в пользовательской плоскости.
- Пример: вечерняя перегрузка соты видео-трафиком → RCAF сигнализирует PCRF → PCRF снижает AMBR для видео-потоков / откладывает загрузки.

## 7.7 Traffic steering

- PCRF/PCF: правила маршрутизации/приоритизации.
- **ATSSS** (5G, TS 24.193): Access Traffic Steering, Switching and Splitting — управление трафиком между 3GPP и non-3GPP (Wi-Fi):
  - режимы: active-standby, smallest delay, load balancing;
  - технологии: MPTCP, ATSSS-LL;
  - правила ATSSS в UE, UPF как anchor.
- **ULCL/BP** (5G): локальная разгрузка на UPF (MEC), выбор пути SMF.
- AF influence (traffic influence) через NEF: приложение влияет на маршрутизацию.

## 7.8 Энергосбережение

- RAN: выключение capacity-сот в часы низкой нагрузки, координация через X2/OSS; umbrella-соты сохраняют покрытие.
- Core: видит доступность сот; MME/AMF корректирует пейджинг; при выключении соты абоненты переходят на соседние.
- Эффект: снижение энергопотребления без потери покрытия.

## 7.9 Слайсинг

- **S-NSSAI**: идентификатор слайса; NSSF выбирает слайс; AMF/SMF обеспечивают.
- RAN: slice-aware scheduling, изоляция ресурсов, admission control.
- Core: оркестрация, политики (PCF), QoS по слайсам.
- Пример: слайс для критичной связи (приоритет) и слайс для массового IoT (низкий приоритет) на одной инфраструктуре.

## 7.10 RAN sharing

- **MORAN**: общая RAN, отдельные частоты/PLMN.
- **MOCN**: общая RAN и частоты; несколько PLMN в broadcast; Core выбирается по PLMN (MME pool/AMF set каждого оператора).
- MOCN Gateway (для 2G/3G).
- Core: маршрутизация по PLMN, отдельные политики/тарификация.

## 7.11 CA/DC и участие Core

- Carrier Aggregation — RAN-функция, Core не участвует напрямую.
- Dual Connectivity (EN-DC/NGEN-DC): добавление SgNB/SeNB меняет путь S1-U/N3:
  - eNB → MME: S1AP E-RAB Modification Indication;
  - MME → SGW: GTPv2-C Modify Bearer Request (новый TEID/адрес);
  - в 5G: NGAP Path Switch / PDU Session Modification.
- Core должен поддерживать смену пути; иначе DC/CA-сценарии не работают.

## 7.12 Примеры из практики

1. ANR в новом районе: MME-релей SON Configuration Transfer → X2 поднят → HO работает.
2. MLB в центре города: eNB передаёт часть абонентов на соседнюю соту → снижение перегрузки.
3. MRO: частые «too late» HO → корректировка CIO → меньше RLF.
4. RCAF: вечерняя перегрузка видео → PCRF снижает AMBR видео → сеть стабильна.
5. IoT: eDRX/PSM для счётчиков → батарея 10 лет, минимум сигнализации.
6. Массовое мероприятие: Overload Start + ACB → защита Core/RAN.
7. MEC: ULCL для AR/VR на стадионе → задержка < 20 мс.
8. Слайсинг: отдельный слайс для скорой помощи → приоритет даже при перегрузке.
9. MOCN: два оператора на одной RAN → экономия, Core отдельно.
10. ATSSS: переключение на Wi-Fi при деградации 5G → непрерывность.

## 7.13 Вопросы для самопроверки

1. Как MME участвует в работе ANR?
2. Чем MRO отличается от MLB?
3. Как работает Overload Start?
4. Что такое TAI list и как он оптимизируется?
5. Зачем eDRX/PSM и кто управляет параметрами?
6. Как MDT помогает оптимизировать покрытие?
7. Что такое RCAF и через какой интерфейс он работает?
8. Как реализуется ATSSS?
9. Как Core участвует в Dual Connectivity?
10. Что такое MOCN/MORAN?

## 7.14 Спецификации

- TS 36.300/36.413/36.423 — LTE/E-UTRAN, S1AP, X2AP
- TS 38.300/38.413/38.423 — NR, NGAP, XnAP
- TS 36.331, TS 38.331 — RRC
- TS 37.320 — MDT; TS 32.422 — Trace
- TS 23.203, TS 29.217 — PCC, RCAF/Np
- TS 23.501/23.502 — 5G; TS 24.193/24.526 — ATSSS, URSP
- TS 23.251 — Network sharing
- TS 23.401 — eDRX/PSM в EPS
