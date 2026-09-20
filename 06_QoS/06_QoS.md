# 6. QoS: что это, как настраивается и используется

> Цель: понять модель QoS в 4G и 5G, ключевые параметры (QCI/5QI, ARP, AMBR, GBR),
> механизмы управления и практические настройки для абонентов.

## 6.1 Зачем QoS

Сеть передаёт разнородный трафик (голос, видео, игры, web, IoT). QoS:

- выделяет ресурсы и приоритеты;
- обеспечивает предсказуемость (задержка, потери, джиттер);
- позволяет тарифицировать и сегментировать услуги;
- управляет перегрузкой.

## 6.2 Модель EPS (4G)

- EPS bearer — «канал» с определённым QoS между UE и PGW.
- **Default bearer**: создаётся при attach/PDN-подключении; non-GBR.
- **Dedicated bearer**: создаётся при необходимости (VoLTE, игры); может быть GBR; имеет TFT для привязки трафика.
- **GBR** (Guaranteed Bit Rate) — гарантированная полоса; non-GBR — «best effort».
- Параметры: QCI, ARP, GBR/MBR (для GBR), APN-AMBR, UE-AMBR.
- E-RAB = EPS bearer + S1 bearer; radio bearer — на радио.

## 6.3 QCI (QoS Class Identifier)

| QCI | Ресурс | Приоритет | PDB (задержка) | Потери | Пример |
|---|---|---|---|---|---|
| 1 | GBR | 2 | 100 мс | 10⁻² | голос (VoLTE) |
| 2 | GBR | 4 | 150 мс | 10⁻³ | видео-вызов |
| 3 | GBR | 3 | 50 мс | 10⁻³ | игры, V2X |
| 4 | GBR | 5 | 300 мс | 10⁻⁶ | потоковое видео |
| 5 | non-GBR | 1 | 100 мс | 10⁻⁶ | IMS-сигнализация |
| 6 | non-GBR | 6 | 300 мс | 10⁻⁶ | видео (буферизация) |
| 7 | non-GBR | 7 | 100 мс | 10⁻³ | голос/видео/игры |
| 8 | non-GBR | 8 | 300 мс | 10⁻⁶ | web, e-mail |
| 9 | non-GBR | 9 | 300 мс | 10⁻⁶ | default bearer |
| 65 | GBR | 0,7 | 75 мс | 10⁻² | MCPTT голос |
| 66 | GBR | 2 | 100 мс | 10⁻² | mission critical |
| 69 | GBR | 0,5 | 60 мс | 10⁻³ | mission critical delay-sensitive |
| 70 | GBR | 5,5 | 200 мс | 10⁻⁶ | mission critical data |
| 75 | GBR | 2,5 | 50 мс | 10⁻² | V2X |
| 79 | GBR | 6,5 | 50 мс | 10⁻² | low latency eMBB |

## 6.4 ARP и AMBR

- **ARP** (Allocation and Retention Priority): priority (1–15, 1 — высший), pre-emption capability, pre-emption vulnerability. Используется при admission control и перегрузке.
- **APN-AMBR**: агрегатный лимит non-GBR по APN (UL/DL).
- **UE-AMBR**: агрегатный лимит non-GBR по UE (сумма по всем APN).
- **MBR/GBR**: per bearer.
- Speed caps для абонента обычно реализуются через APN-AMBR.

## 6.5 Управление QoS в 4G

Источники политик:

- HSS: подписка (QCI/ARP default bearer, APN-AMBR).
- PCRF: динамические PCC-правила; Gx к PCEF (PGW).
- AF (P-CSCF) → Rx: медиа-параметры.
- Локальная конфигурация PGW (если PCC не используется).

Цепочка установки dedicated bearer:

AF → Rx AAR → PCRF → Gx CCA (Charging-Rule-Install) → PGW → GTPv2-C Create Bearer Request → SGW → MME → S1AP E-RAB Setup → eNB → RRC → UE (NAS Activate Dedicated EPS Bearer, TFT).

## 6.6 QoS в 5G

- PDU-сессия; внутри — QoS flows (QFI, 6 бит).
- QoS flow: GBR / non-GBR / Delay-Critical GBR.
- **5QI** — аналог QCI; параметры: resource type, priority, PDB, PER, MDBV (maximum data burst volume), averaging window.
- **QoS rule** (в UE): QRI (precedence), packet filter set, QFI.
- **SDAP** (Service Data Adaptation Protocol): маппинг QoS flow → DRB.
- Session-AMBR.
- **Reflective QoS** (RQI): UE «отражает» DL-классификацию для UL без явных правил.
- **QoS Notification Control (QNC)**: сеть уведомляет, если GBR не выполняется.
- **Alternative QoS Profiles (AQP)**: запасные профили для GBR-потоков.

## 6.7 5QI (основные)

| 5QI | Ресурс | PDB | Пример |
|---|---|---|---|
| 1 | GBR | 100 мс | голос |
| 2 | GBR | 150 мс | видео-вызов |
| 3 | GBR | 50 мс | игры, V2X |
| 4 | GBR | 300 мс | потоковое видео |
| 5 | non-GBR | 100 мс | IMS-сигнализация |
| 6 | non-GBR | 300 мс | видео, web |
| 7 | non-GBR | 100 мс | голос/видео/игры |
| 8 | non-GBR | 300 мс | web, e-mail |
| 9 | non-GBR | 300 мс | default |
| 65–70 | GBR | 60–200 мс | mission critical |
| 75 | GBR | 50 мс | V2X |
| 79 | GBR | 50 мс | low latency eMBB |
| 80 | GBR | 10 мс | low latency eMBB |
| 82–85 | Delay-Critical GBR | 5–30 мс | автоматизация, транспорт |

## 6.8 Возможности настройки для абонентов

- Профиль в HSS/UDM: default QCI/5QI, ARP, AMBR (базовые ограничения).
- PCRF/PCF-политики: повышение/понижение QoS по событию.
- Speed caps: APN-AMBR / session-AMBR (тарифные «до 100 Мбит/с»).
- «Турбо-кнопка»: временное повышение AMBR/квоты.
- VIP-приоритет: низкий ARP priority, pre-emption capability.
- Выделенные bearer'ы: GBR для enterprise-сервисов, игр, видеонаблюдения.
- Голос: QCI/5QI 5 для сигнализации, 1 для голоса, 2 для видео.
- Emergency: 5QI/QCI 1 с высоким приоритетом и pre-emption.
- FUP: снижение AMBR при превышении порога.
- Zero-rating: тарификация, не QoS.
- Роуминг: политики домашней сети (H-PCRF/PCF).

## 6.9 QoS в IMS

- Сигнализация — QCI/5QI 5 (default bearer).
- Голос — QCI/5QI 1 (GBR).
- Видео — QCI/5QI 2.
- Управление — Rx (4G) / N5 (5G): Media-Component-Description (тип медиа, полосы, flow descriptions).
- PCRF/PCF создаёт PCC-правила → dedicated bearer / QoS flow.

## 6.10 Мониторинг и оптимизация

- KPI: задержка, джиттер, потери, throughput, доля установленных bearer'ов.
- QoE: MOS для голоса (PESQ/POLQA), video QoE.
- Перегрузка: RCAF (Np) → PCRF; UPCON.
- Наблюдение за ARP-конфликтами, pre-emption, admission control.

## 6.11 Практические примеры

1. VoLTE: dedicated bearer QCI=1, сигнализация QCI=5.
2. Игровой трафик: QCI=3/5QI=3 (если оператор предоставляет).
3. Enterprise APN: GBR для критичных приложений, приоритетный ARP.
4. Массовое мероприятие: pre-emption для emergency/VIP, ACB.
5. FUP: после 30 ГБ AMBR снижается до 1 Мбит/с.
6. Турбо: повышение AMBR на 24 часа.
7. IoT: eDRX/PSM + низкий приоритет, small data.

## 6.12 Вопросы для самопроверки

1. Чем GBR отличается от non-GBR?
2. Что такое ARP и его три параметра?
3. Чем APN-AMBR отличается от UE-AMBR?
4. Как устанавливается dedicated bearer для VoLTE?
5. Какие QCI используются для голоса и сигнализации?
6. Что такое 5QI и QFI в 5G?
7. Как работает reflective QoS?
8. Как реализовать «турбо-кнопку»?
9. Что такое QNC?
10. Как оператор ограничивает скорость абонента?

## 6.13 Спецификации

- TS 23.401, TS 23.203 — EPS, PCC
- TS 29.212, TS 29.213, TS 29.214 — Gx/Rx
- TS 23.501, TS 23.502 — 5G QoS
- TS 24.501 — NAS 5G
- TS 29.244 — PFCP
- TS 23.207 — end-to-end QoS (общая модель)
