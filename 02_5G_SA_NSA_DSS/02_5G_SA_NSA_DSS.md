# 2. 5G: SA, NSA, DSS — отличия и как реализуются

> Цель: понять разницу между Standalone, Non-Standalone и Dynamic Spectrum Sharing,
> увидеть их архитектуру, процедуры и ограничения.

## 2.1 Терминология

- **SA (Standalone)** — 5G NR + 5GC (Option 2). NR самостоятелен: сигнализация и данные через gNB и 5GC.
- **NSA (Non-Standalone)** — NR используется как «усилитель» к LTE (EN-DC, Option 3/3a/3x). Управление — через LTE (MeNB), ядро — EPC.
- **DSS (Dynamic Spectrum Sharing)** — динамическое совместное использование одного спектра LTE и NR; не отдельная архитектура, а RAN-технология.

Варианты 3GPP (для контекста):

- Option 2: NR + 5GC (SA).
- Option 3/3a/3x: LTE + NR + EPC (NSA, EN-DC).
- Option 4/4a: NR + LTE + 5GC (NE-DC).
- Option 7/7a/7x: LTE + NR + 5GC (NGEN-DC).
- Option 5: LTE + 5GC.

## 2.2 NSA (EN-DC, Option 3/3a/3x)

### 2.2.1 Архитектура

- **MeNB** (Master eNB, LTE) — узел-«якорь»: RRC, NAS, S1-MME.
- **SgNB** (Secondary gNB, NR) — добавляется для пользовательской ёмкости/скорости.
- Ядро — **EPC**: MME, SGW, PGW (без изменений).
- **X2-C/X2-U** между MeNB и SgNB.

### 2.2.2 Варианты пользовательской плоскости

| Вариант | S1-U | Разделение bearer | Особенность |
|---|---|---|---|
| Option 3 | SgNB → SGW | нет | весь пользовательский трафик через NR; LTE — только сигнализация |
| Option 3a | MeNB → SGW и SgNB → SGW | нет | два независимых bearer'а (MCG и SCG) |
| Option 3x | SgNB → SGW | split bearer в SgNB (PDCP в SgNB) | гибкое распределение трафика между LTE и NR |

На практике чаще всего используется **Option 3x**.

### 2.2.3 Процедура добавления NR (SgNB Addition)

1. UE в LTE (RRC Connected), eNB настраивает измерения NR (событие B1: «сосед NR лучше порога»).
2. UE → MeNB: Measurement Report (B1).
3. MeNB → SgNB: X2AP SgNB Addition Request (UE capability, E-RAB-контекст, S1 TNL).
4. SgNB: admission control; X2AP SgNB Addition Request Acknowledge (NR RRC-конфигурация, S1-U TNL).
5. MeNB → UE: RRCConnectionReconfiguration (добавление SCG).
6. UE: Random Access к SgNB; MeNB → SgNB: X2AP SgNB Reconfiguration Complete.
7. Обновление пути данных в EPC: MME/SGW Modify Bearer (S1-U теперь на SgNB для Option 3/3x).
8. UE получает агрегированную скорость LTE + NR.

### 2.2.4 Ограничения NSA

- Нет 5GC: нет network slicing, URLLC, MEC-интеграции как в SA.
- Зависимость от покрытия LTE (якорь).
- Голос — VoLTE/CSFB по LTE; NR голос не несёт (в NSA).
- Сигнальная нагрузка на LTE RRC.
- Добавление/смена SgNB — задержки и сложность mobility (anchor MeNB).

## 2.3 SA (Option 2)

### 2.3.1 Архитектура 5GC (SBA)

| NF | Функция |
|---|---|
| AMF | доступ и мобильность, NAS, координация аутентификации |
| SMF | управление PDU-сессиями, IP-адресация, выбор UPF |
| UPF | пользовательская плоскость, шлюз к DN, DPI/steering |
| UDM | данные абонента (аналог HSS) |
| AUSF | аутентификация |
| PCF | политики (аналог PCRF) |
| NRF | реестр NF (service discovery) |
| NSSF | выбор сетевого слайса |
| NEF | exposure API для внешних систем |
| AF | приложения (в т.ч. IMS) |
| CHF | charging (конвергентная тарификация) |

Основные интерфейсы:

| Интерфейс | Между | Протокол |
|---|---|---|
| N1 | UE ↔ AMF | NAS (через N2) |
| N2 | gNB ↔ AMF | NGAP |
| N3 | gNB ↔ UPF | GTP-U |
| N4 | SMF ↔ UPF | PFCP |
| N6 | UPF ↔ DN | IP |
| N8 | AMF ↔ UDM | SBI |
| N10 | SMF ↔ UDM | SBI |
| N11 | AMF ↔ SMF | SBI |
| N12 | AMF ↔ AUSF | SBI |
| N15 | AMF ↔ PCF | SBI |
| N22 | AMF ↔ NSSF | SBI |
| N7 | SMF ↔ PCF | SBI |
| N5 | PCF ↔ AF | SBI |
| N26 | AMF ↔ MME | GTPv2-C |
| N32 | SEPP ↔ SEPP | SBI (roaming) |

### 2.3.2 Ключевые возможности SA

- Network slicing (S-NSSAI), изоляция ресурсов.
- QoS flows (5QI/QFI), reflective QoS.
- URLLC, mMTC, eMBB.
- VoNR; до готовности — EPS fallback в LTE.
- MEC: ULCL/BP, локальная разгрузка.
- SBA: HTTP/2, REST, service discovery.

## 2.4 DSS

### 2.4.1 Принцип

Один и тот же несущий (полоса) одновременно используется LTE и NR; распределение ресурсов динамическое, по нагрузке/приоритету. Альтернатива — **refarming** (выделение полосы только под NR).

### 2.4.2 Как реализуется

- LTE и NR — на одном сайте, одной полосе, часто одном BBU (единый вендор).
- NR SSB передаётся в MBSFN-субкадрах LTE (чтобы не конфликтовать с LTE CRS).
- NR PDSCH rate-matching вокруг LTE CRS (параметр LTE-CRS-ToMatchAround).
- LTE PDSCH rate-matching вокруг NR SSB/CSI-RS.
- Планировщики LTE и NR координируются (общий scheduler или быстрый обмен).
- Требования: жёсткая фазовая синхронизация, поддержка UE, единый вендор, часто FDD-диапазоны (1800/2100/2600).

### 2.4.3 Плюсы и минусы

| Плюсы | Минусы |
|---|---|
| Быстрый запуск 5G без нового спектра | Снижение пиковой ёмкости LTE и NR |
| Плавный переход, покрытие 5G на существующих сайтах | Сложность планирования и оптимизации |
| Работает и с NSA, и с SA | Зависимость от вендора, ограничения по UE |
| Не нужен refarming | Эффективность ниже, чем у выделенной полосы |

## 2.5 Сводная таблица

| | NSA (EN-DC) | SA | DSS |
|---|---|---|---|
| Ядро | EPC | 5GC | зависит (NSA или SA) |
| Управление | LTE (MeNB) | NR (gNB) | зависит |
| NR-интерфейсы | X2, S1-U (SgNB) | N1/N2/N3/N4... | RAN-функция |
| Слайсинг | нет | да | нет/зависит от SA |
| URLLC/MEC | нет | да | зависит |
| Голос | VoLTE/CSFB | VoNR/EPS fallback | зависит |
| Спектр | отдельный NR или DSS | отдельный/DSS | общий LTE/NR |
| Зрелость | высокая (запуск 5G) | целевая | тактика запуска |

## 2.6 Вопросы для самопроверки

1. Чем SA отличается от NSA на уровне ядра и управления?
2. Что означают Option 3, 3a, 3x и где разделяется пользовательская плоскость?
3. Какие процедуры выполняются при добавлении SgNB?
4. Почему в NSA нельзя использовать slicing?
5. Назовите NF в 5GC и их функции.
6. Что такое DSS и как он реализуется на уровне ресурсов?
7. Почему DSS снижает пиковую ёмкость?
8. Как голос работает в NSA, SA и при DSS?
9. Какие интерфейсы 5GC нужны для VoNR?
10. Когда выбирают NSA, а когда SA?

## 2.7 Спецификации

- TS 23.501 — 5G System architecture
- TS 23.502 — 5G procedures
- TS 37.340 — Multi-connectivity (EN-DC, NGEN-DC, NE-DC)
- TS 38.300 — NR overall
- TS 38.331 — NR RRC
- TS 38.211/38.213 — физический уровень (в т.ч. для DSS)
- TS 36.300/36.331 — LTE (для DSS/EN-DC)
