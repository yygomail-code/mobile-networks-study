# 15. QoS в мобильных сетях — подробный конспект

> Кому: инженерам, изучающим качество обслуживания в мобильных сетях (2G/3G/4G/5G/IMS).
> Цель: разобрать QoS «от метрики до радио»: модель bearer'ов в 4G, QoS flows в 5G, параметры
> (QCI/5QI, ARP, AMBR, GBR), управление политиками (PCC), практические настройки для абонентов
> и диагностику проблем.
>
> Связанные файлы: `06_QoS.md` (краткий конспект), `03_Diameter_CCR_CCA.md` (Gx/Gy),
> `09_Interfejsy_LTE_PS_CS_troubleshooting.md` (диагностика), `13_Diameter_detalny_konspekt.md`.

## 15.1 Что такое QoS и зачем он нужен

**QoS (Quality of Service)** — набор механизмов, которые обеспечивают разному трафику разные
условия передачи: приоритет, гарантированную полосу, допустимые задержки и потери.

Зачем:

- голос и видео чувствительны к задержке и джиттеру, а веб — нет;
- сеть перегружена не «вообще», а в конкретный момент и в конкретной соте;
- оператор продаёт разные тарифы и услуги (скорость, приоритет, VIP, IoT);
- без QoS невозможно гарантировать критичные сервисы (экстренные вызовы, MCPTT, телеметрия).

Ключевые метрики:

| Метрика | Что означает | Типичные требования |
|---|---|---|
| Throughput | скорость передачи (DL/UL) | web — «сколько дадут», видео — 5–25 Мбит/с |
| Latency (задержка) | время доставки пакета (one-way/RTT) | голос < 100 мс, игры < 50 мс, URLLC 5–10 мс |
| Jitter (джиттер) | разброс задержки | голос < 30–50 мс |
| Packet loss (потери) | доля потерянных пакетов | голос 10⁻²…10⁻³, данные 10⁻⁶ |
| QoE | субъективное качество | MOS для голоса (1–5), video QoE |

Принцип работы QoS в мобильной сети:

1. **Классификация** — определить, что за трафик (по 5-tuple, приложению, DPI).
2. **Маркировка** — отнести к классу (QCI/5QI) и приоритету (ARP).
3. **Приоритизация и гарантии** — выделить полосу (GBR), ограничить (AMBR/MBR).
4. **Контроль** — admission control, policing/shaping, перегрузка.
5. **Учёт** — тарификация по классам (rating groups).

Модели QoS:

- **4G (EPS)**: QoS привязан к **bearer'ам** (каналам между UE и PGW).
- **5G**: QoS привязан к **QoS flow** внутри PDU-сессии.
- **2G/3G**: классы QoS в PDP-контекстах (conversational/streaming/interactive/background) — историческая модель.
- **Политики (PCC)** — «мозг» QoS: PCRF/PCF решает, какой класс дать трафику.

## 15.2 QoS в 4G: модель EPS bearer

### 15.2.1 Что такое bearer

**EPS bearer** — логический канал с определённым QoS между UE и PGW.
Сквозной путь bearer'а:

```
UE ──(radio bearer)── eNB ──(S1 bearer)── SGW ──(S5/S8 bearer)── PGW
        └──────────── E-RAB = radio + S1 ────────────┘
```

Виды:

- **Default bearer** — создаётся при подключении (attach/PDN connectivity), всегда non-GBR,
  «best effort»; по умолчанию без TFT (весь неклассифицированный трафик).
- **Dedicated bearer** — создаётся при необходимости (VoLTE, игры, GBR-сервисы);
  бывает GBR и non-GBR; имеет **TFT**, чтобы трафик попал именно в него.

### 15.2.2 QCI — класс QoS (4G)

**QCI (QoS Class Identifier)** — номер класса (1–9, 65–95), определяющий приоритет, допустимую
задержку (PDB) и потери (PER). Таблица (TS 23.203):

| QCI | Ресурс | Приоритет | PDB (задержка) | Потери | Пример |
|---|---|---|---|---|---|
| 1 | GBR | 2 | 100 мс | 10⁻² | голос (VoLTE) |
| 2 | GBR | 4 | 150 мс | 10⁻³ | видео-вызов |
| 3 | GBR | 3 | 50 мс | 10⁻³ | игры, V2X |
| 4 | GBR | 5 | 300 мс | 10⁻⁶ | потоковое видео |
| 5 | non-GBR | 1 | 100 мс | 10⁻⁶ | IMS-сигнализация |
| 6 | non-GBR | 6 | 300 мс | 10⁻⁶ | видео (буферизация), TCP |
| 7 | non-GBR | 7 | 100 мс | 10⁻³ | голос/видео/игры (non-GBR) |
| 8 | non-GBR | 8 | 300 мс | 10⁻⁶ | web, e-mail, чат |
| 9 | non-GBR | 9 | 300 мс | 10⁻⁶ | default bearer |
| 65 | GBR | 0,7 | 75 мс | 10⁻² | MCPTT голос |
| 66 | GBR | 2 | 100 мс | 10⁻² | mission critical (non-MCPTT) |
| 67 | GBR | 1,5 | 100 мс | 10⁻³ | mission critical video |
| 69 | GBR | 0,5 | 60 мс | 10⁻³ | mission critical delay-sensitive |
| 70 | GBR | 5,5 | 200 мс | 10⁻⁶ | mission critical data |
| 75 | GBR | 2,5 | 50 мс | 10⁻² | V2X |
| 79 | GBR | 6,5 | 50 мс | 10⁻² | low latency eMBB |
| 80 | GBR | 6,8 | 10 мс | 10⁻⁶ | low latency eMBB (UDP) |

Чем меньше приоритет числом — тем «важнее» класс (1 — самый высокий у non-GBR; у GBR есть
ещё более высокие: 0,5 и 0,7).

### 15.2.3 ARP — приоритет и вытеснение

**ARP (Allocation and Retention Priority)** — три параметра:

- **Priority Level** (1–15; 1 — высший): используется при admission control и перегрузке;
- **Pre-emption Capability**: может ли bearer вытеснить другие (may/must not pre-empt);
- **Pre-emption Vulnerability**: можно ли этот bearer вытеснить.

ARP не влияет на полосу и задержку — он решает, **кого обслуживать и кем жертвовать** при нехватке
ресурсов. Пример: экстренный вызов (QCI 1) с высоким ARP вытесняет обычный трафик.

### 15.2.4 Полоса: GBR, MBR, AMBR

| Параметр | Где | Смысл |
|---|---|---|
| GBR | per bearer (GBR) | гарантированная полоса |
| MBR | per bearer | максимальная полоса bearer'а |
| APN-AMBR | per APN (PGW) | суммарный лимит non-GBR по APN (UL/DL) |
| UE-AMBR | per UE (RAN) | суммарный лимит non-GBR по всем APN абонента |

Важно: **AMBR не ограничивает GBR-трафик** — голос может идти поверх лимита AMBR.
Именно APN-AMBR чаще всего «режет» скорость в тарифах («до 100 Мбит/с»).

### 15.2.5 TFT — как трафик попадает в bearer

**TFT (Traffic Flow Template)** — набор фильтров (packet filters) в UE для **uplink**:

- фильтр = 5-tuple (src/dst IP, порты, протокол) + опции (DSCP/TOS, Flow Label, SPI);
- у фильтра есть **precedence** — порядок применения;
- UE направляет пакет в тот bearer, чей фильтр совпал.

В PGW для **downlink** используются SDF-фильтры из PCC-правил (Flow-Information).
Пример TFT для VoLTE: `permit out 17 from 10.0.0.0/8 1000-2000 to any 5060` (RTP).

### 15.2.6 Процедуры управления bearer'ами

- **Активация dedicated bearer** (сеть → UE): NAS Activate Dedicated EPS Bearer Context Request
  (QCI, TFT, EPS Bearer ID, LBI — связь с default);
- **Модификация** (QCI/GBR/TFT) — Modify EPS Bearer Context;
- **Деактивация** — Deactivate EPS Bearer Context (Regular deactivation, 36).
- UE-инициированное изменение: Bearer Resource Allocation/Modification Request (редко используется).
- В RAN: **E-RAB Setup/Modify** (S1AP) с параметрами E-RAB Level QoS Parameters
  (QCI, ARP, GBR/MBR); на радио — DRB с соответствующим QoS.

### 15.2.7 Где применяются параметры

| Узел | Что настраивает/контролирует |
|---|---|
| HSS/UDM | подписка: QCI/ARP default bearer, APN-AMBR, разрешённые QCI |
| MME/AMF | передача QoS в RAN, выбор bearer'ов, ARP при перегрузке |
| PCRF/PCF | PCC-правила: QCI, GBR/MBR, ARP, flow-фильтры, AMBR |
| PGW/UPF | enforcement: gate, policing/shaping, bearer binding, учёт |
| SGW | транспорт bearer'ов (S5/S8↔S1), relay QoS |
| eNB/gNB | admission control, планировщик, DRB, приоритеты |
| UE | TFT/QoS rules, маркировка UL-трафика |

## 15.3 Управление QoS: PCC (Policy and Charging Control)

### 15.3.1 Архитектура PCC

```
AF (P-CSCF, приложения) ──Rx──► PCRF ──Gx──► PCEF (PGW)
                                  │  └─Gxx─► BBERF (SGW, PMIP)
                                  ├─Sy──► OCS (лимиты)
                                  ├─Sd──► TDF (DPI)
                                  └─Sp──► SPR (данные абонента)
```

- **PCRF** — принимает решения (правила), **PCEF** — исполняет (PGW), **BBERF** — для PMIP (SGW).
- **AF** — приложение (P-CSCF для VoLTE), запрашивает ресурсы.
- **OCS** — тарификация и лимиты (Gy, Sy).
- **TDF** — DPI-детект приложений (Sd).

### 15.3.2 Gx: правила от PCRF к PGW

Основные AVP (имена): Charging-Rule-Install/Remove, QoS-Information (QCI, GBR/MBR, ARP),
Default-EPS-Bearer-QoS, APN-Aggregate-Max-Bitrate-DL/UL, Flow-Information (фильтры),
Event-Trigger, Revalidation-Time, Session-Release-Cause.

Сценарии:

- установка/изменение правил при attach, смене RAT/локации, запросе услуги;
- PCRF-инициированное изменение (RAR/RAA);
- снятие правил при завершении.

### 15.3.3 Rx: запрос ресурсов под медиа (VoLTE)

P-CSCF (AF) отправляет AAR с Media-Component-Description:

- Media-Component-Number, Media-Type (audio/video);
- Max-Requested-Bandwidth-UL/DL (по кодекам: EVS/AMR-WB ~ 24–128 кбит/с);
- Flow-Description (IP/порты RTP/RTCP);
- AF-Charging-Identifier, Specific-Action.

PCRF преобразует это в PCC-правило (QCI 1/2 + GBR + flow-фильтры) и через Gx передаёт PGW.

### 15.3.4 Сквозной пример: VoLTE (dedicated bearer QCI 1)

1. UE: SIP INVITE (Gm) → P-CSCF.
2. P-CSCF → PCRF: Rx AAR (медиа: аудио, полосы, flow).
3. PCRF → PGW: Gx CCA (Charging-Rule-Install: QCI 1, GBR, ARP, Flow-Information).
4. PGW → SGW → MME: GTPv2-C Create Bearer Request.
5. MME → eNB: S1AP E-RAB Setup Request (QoS: QCI 1, GBR, ARP).
6. eNB → UE: RRC + NAS Activate Dedicated EPS Bearer (TFT).
7. Медиа идёт по bearer'у QCI 1; сигнализация — по QCI 5.
8. Завершение: Delete Bearer (PCRF/Gx → PGW → …).

### 15.3.5 Пример: default bearer при attach

1. UE Attach → MME → HSS (S6a): подписка (default QCI/ARP, APN-AMBR).
2. MME → SGW → PGW: Create Session; PGW запрашивает PCRF (Gx CCR INITIAL).
3. PCRF отвечает CCA (Default-EPS-Bearer-QoS, APN-AMBR, правила).
4. MME → eNB: Initial Context Setup (E-RAB QoS); UE получает default bearer (QCI 9/8/6).

## 15.4 QoS в 5G: QoS flows

### 15.4.1 Модель

В 5G нет bearer'ов уровня EPS: внутри **PDU-сессии** создаются **QoS flow** — потоки с
одинаковым качеством. Каждый поток имеет:

- **QFI (QoS Flow Identifier)** — номер потока (6 бит);
- **5QI** — класс качества (аналог QCI);
- **QoS rule** в UE: QRI (precedence), packet filter set, QFI;
- маппинг на DRB через **SDAP** (Service Data Adaptation Protocol).

Виды потоков:

- **Default QoS flow** — при создании PDU-сессии, non-GBR;
- **Signalling flow** — 5QI 5 (IMS-сигнализация);
- **GBR / non-GBR / Delay-Critical GBR** — по типу ресурса.

### 15.4.2 5QI — классы QoS (5G)

| 5QI | Ресурс | Приоритет | PDB | Потери | Пример |
|---|---|---|---|---|---|
| 1 | GBR | 20 | 100 мс | 10⁻² | голос |
| 2 | GBR | 40 | 150 мс | 10⁻³ | видео-вызов |
| 3 | GBR | 30 | 50 мс | 10⁻³ | игры, V2X |
| 4 | GBR | 50 | 300 мс | 10⁻⁶ | потоковое видео |
| 5 | non-GBR | 10 | 100 мс | 10⁻⁶ | IMS-сигнализация |
| 6 | non-GBR | 60 | 300 мс | 10⁻⁶ | видео, web (TCP) |
| 7 | non-GBR | 70 | 100 мс | 10⁻³ | голос/видео/игры |
| 8 | non-GBR | 80 | 300 мс | 10⁻⁶ | web, e-mail |
| 9 | non-GBR | 90 | 300 мс | 10⁻⁶ | default |
| 65 | GBR | 7 | 75 мс | 10⁻² | MCPTT голос |
| 66 | GBR | 20 | 100 мс | 10⁻² | mission critical |
| 67 | GBR | 15 | 100 мс | 10⁻³ | mission critical video |
| 69 | GBR | 5 | 60 мс | 10⁻³ | delay-sensitive signalling |
| 70 | GBR | 55 | 200 мс | 10⁻⁶ | mission critical data |
| 75 | GBR | 25 | 50 мс | 10⁻² | V2X |
| 79 | non-GBR | 65 | 50 мс | 10⁻² | low latency eMBB |
| 80 | non-GBR | 68 | 10 мс | 10⁻⁶ | low latency eMBB (UDP) |
| 82–85 | Delay-Critical GBR | 19–24 | 5–30 мс | 10⁻⁴…10⁻⁵ | автоматизация, транспорт, энергетика |

Дополнительные параметры 5QI: **MDBV** (Maximum Data Burst Volume) и **averaging window** —
для delay-critical GBR; полная таблица — TS 23.501, табл. 5.7.4-1.

### 15.4.3 Параметры и механизмы

- **Session-AMBR** — лимит non-GBR на всю PDU-сессию (аналог APN-AMBR);
- **Reflective QoS (RQI)** — UE выводит UL-правила из DL-классификации (без явных правил);
- **QNC (QoS Notification Control)** — RAN уведомляет SMF, если GBR не выполняется;
- **Alternative QoS Profiles (AQP)** — запасные профили (например, сниженная полоса) для GBR;
- **UL/DL packet filters** в QoS rules; маркировка UL через SDAP.

### 15.4.4 Процедуры

- **PDU Session Establishment**: SMF выбирает 5QI, формирует QoS rules, через N4 (PFCP) программирует
  UPF, через NGAP (N2) — gNB (PDU Session Resource Setup, QoS flows c 5QI/QFI).
- **PDU Session Modification**: изменение 5QI/AMBR, добавление потоков (по запросу UE, AF/N5 или SMF).
- **PDU Session Release**: удаление потоков.
- При handover QoS flows переносятся на целевой gNB (N2/Xn), при interworking с EPS — конверсия
  5QI↔QCI.

### 15.4.5 5QI ↔ QCI (interworking)

Для переходов 5GS↔EPS классы конвертируются: 5QI 1↔QCI 1, 2↔2, 3↔3, 4↔4, 5↔5, 6↔6, 7↔7,
8↔8, 9↔9, 65↔65, 66↔66, 69↔69, 70↔70, 75↔75, 79↔79, 80↔80 (TS 23.501). При отсутствии
соответствия поток может быть отброшен или понижен — это важно при переходах LTE↔5G.

### 15.4.6 URLLC и TSN (кратко)

- **URLLC** — 5QI 82–85: PDB 5–30 мс, надёжность 10⁻⁴…10⁻⁵, MDBV ограничивает размер «всплеска».
- **TSC (Time Sensitive Communication)** — синхронный трафик (заводы, роботы): TSCAI
  (Time Sensitive Communication Assistance Information), hold-and-forward в RAN.
- Для критичных сервисов QoS сочетается со слайсингом (S-NSSAI) и MEC.

## 15.5 QoS в IMS: VoLTE и VoNR

- Сигнализация IMS — **QCI/5QI 5** (default bearer APN/DNN ims).
- Голос — **QCI/5QI 1** (GBR), видео — **QCI/5QI 2**.
- Полосы из SDP (кодеки: EVS, AMR-WB; видео H.264/HEVC) → Rx AAR → PCRF → Gx → dedicated bearer.
- **ARP голоса**: обычно высокий приоритет; экстренные вызовы — максимальный, с pre-emption.
- **MPS (Multimedia Priority Service)** — приоритетные вызовы для спецслужб/экстренных служб.
- **VoNR**: голос как QoS flow 5QI 1 в NR; до покрытия — EPS fallback (перевод в LTE).
- **Wi-Fi calling (ePDG)**: QoS-классы переносятся в non-3GPP доступ (SWu), приоритизация в Wi-Fi;
  возможны отличия от сотового QoS.

## 15.6 Практические настройки для абонентов

| Задача | Механизм | Где настраивается |
|---|---|---|
| Ограничение скорости по тарифу | APN-AMBR / session-AMBR | HSS/UDM, PCRF/PCF |
| «Турбо-кнопка» | временное повышение AMBR | PCRF/PCF + OCS (Gx/CCR) |
| FUP (снижение после лимита) | AMBR ↓ или блокировка | OCS (Gy) → PCRF → Gx |
| VIP/корпоратив | ARP priority, dedicated GBR | PCRF/PCF, подписка |
| Игры | QCI/5QI 3, низкий PDB | PCRF/PCF, AF |
| Видео-стриминг | QCI 4/6, ограничение битрейта | PCRF/PCF, TDF (Sd) |
| Экстренные/MCPTT | QCI/5QI 1, 65–70, pre-emption | подписка, PCRF/PCF, RAN |
| IoT | QCI/5QI 9, eDRX/PSM | подписка, MME/AMF |
| Слайсинг | S-NSSAI + 5QI + SLA | 5GC (NSSF/PCF) |
| Роуминг | политики H-PLMN (S8HR) | PCRF/PCF, IPX |

Важно не путать:

- **QoS** — качество передачи (скорость, задержка, приоритет);
- **тарификация** — сколько стоит (Gy/N40, rating groups);
- **zero-rating** — не QoS, а нулевая цена (классификация DPI/TDF).

## 15.7 Диагностика QoS

### 15.7.1 Симптомы

- голос «квакает», эхо, обрывы — нет bearer'а QCI 1/5QI 1 или плохое радио;
- видео буферизуется — QCI 4/6, не хватает полосы, перегрузка;
- скорость ниже тарифа — APN-AMBR/session-AMBR, FUP, перегрузка;
- приоритет не работает — ARP/QCI не применились, правила не установлены;
- после лимита скорость не снижается — OCS/PCRF не сработали (Gy/Sy).

### 15.7.2 Где смотреть (по слоям)

| Слой | Что проверять |
|---|---|
| UE / NAS | какой bearer/QoS flow, TFT/QoS rules, 5QI/QFI, причины отказа |
| S1AP / NGAP | E-RAB/QoS flow: QCI/5QI, ARP, GBR/MBR, cause |
| S11 / N4 | Create/Modify Bearer (QoS), PFCP-правила, cause |
| Gx / N7 | PCC-правила: Charging-Rule-Install, QoS-Information, APN-AMBR |
| Rx / N5 | медиа-компоненты для VoLTE (полосы, flow) |
| Gy / N40 | тарификация и лимиты (не путать с QoS) |
| RAN | планировщик, PRB, перегрузка, admission control, QNC |
| SGi / N6 | DSCP, shaping, лимиты на границе |

### 15.7.3 Типовые причины

1. **Нет PCC-правила** (Gx/N7) → нет dedicated bearer → голос без гарантий.
2. **TFT/QoS rule mismatch** — трафик не попадает в нужный bearer/поток.
3. **AMBR слишком низкий** — «медленно» при хорошем радио.
4. **ARP-конфликт** — вытеснение или отказ в admission control.
5. **5QI↔QCI mapping** — при переходах LTE↔5G класс теряется/понижается.
6. **Session-AMBR** ограничивает все потоки PDU-сессии.
7. **Перегрузка RAN** — QoS есть, но радио не тянет (смотреть QNC, PRB).
8. **QNC/AQP** — RAN сообщил о невыполнении GBR, применён альтернативный профиль.

### 15.7.4 Инструменты

- Wireshark/tshark: `s1ap` (E-RAB QoS), `ngap`, `gtpv2`, `pfcp`, `diameter` (Gx/Rx), `nas-eps`;
- трассировка по IMSI на MME/AMF, SGW/PGW/UPF, PCRF/PCF;
- счётчики: setup success rate bearer'ов, drop rate, QCI-распределение, PRB utilization;
- QoE-системы: MOS для голоса, video QoE.

### 15.7.5 Мини-кейсы

1. **VoLTE без QCI 1**: в Rx/Gx нет правила → INVITE проходит, но bearer не создан → «кваканье».
2. **Турбо не сработал**: PCRF не получил команду от OCS (Sy/Gy) → AMBR не изменился.
3. **FUP не режет**: OCS вернул квоту, PCRF не обновил AMBR (Gx) → скорость прежняя.
4. **Игра тормозит**: QCI 3 выдан, но ARP низкий → при перегрузке вытесняется.
5. **При переходе в 5G голос пропал**: 5QI 1 не смаппился/QoS flow не создан (N2/N4).

## 15.8 Сравнение 4G и 5G QoS

| | 4G (EPS) | 5G |
|---|---|---|
| Единица QoS | EPS bearer | QoS flow |
| Идентификатор | QCI + EPS Bearer ID | 5QI + QFI |
| Классы | QCI 1–9, 65–95 | 5QI 1–9, 65–86 |
| Агрегатный лимит | APN-AMBR, UE-AMBR | session-AMBR |
| Классификация | TFT (UL), SDF (DL) | QoS rules, reflective QoS |
| Транспорт QoS | DRB (SDAP отсутствует) | DRB + SDAP |
| Уведомления RAN | — | QNC, AQP |
| Управление | PCRF (Gx/Rx) | PCF (N7/N5) |
| Тарификация | OCS (Gy) | CHF (N40) |

## 15.9 Мини-словарь

- **QoS** — качество обслуживания.
- **QCI / 5QI** — класс QoS в 4G / 5G.
- **ARP** — приоритет и вытеснение bearer'а.
- **GBR / MBR** — гарантированная / максимальная полоса bearer'а.
- **AMBR** — агрегатный лимит (APN/UE/session).
- **TFT / QoS rule** — фильтры трафика в UE.
- **SDF** — service data flow (классифицированный поток).
- **PCC** — policy and charging control.
- **PCEF / BBERF** — исполнители политик (PGW / SGW).
- **PDB / PER** — допустимая задержка / потери.
- **QFI / SDAP** — идентификатор потока / маппинг на DRB в 5G.
- **QNC / AQP** — уведомление о невыполнении QoS / альтернативные профили.
- **MDBV** — максимальный объём «всплеска» для delay-critical GBR.

## 15.10 Вопросы для самопроверки

1. Чем GBR отличается от non-GBR и что ограничивает AMBR?
2. Что такое ARP и как он работает при перегрузке?
3. Чем default bearer отличается от dedicated?
4. Как трафик попадает в нужный bearer (TFT) и что будет при ошибке фильтра?
5. Какие интерфейсы участвуют в создании VoLTE-bearer'а?
6. Что такое 5QI, QFI и QoS rule в 5G?
7. Как работает reflective QoS?
8. Что произойдёт при переходе LTE↔5G с несмаппированным 5QI?
9. Как оператор реализует «турбо-кнопку» и FUP?
10. Какие слои проверять, если «голос квакает», а радио хорошее?

## 15.11 Спецификации

- TS 23.203 — PCC архитектура (QCI, ARP, PCC-правила)
- TS 23.401 — EPS (bearer'ы, AMBR, процедуры)
- TS 29.212 — Gx; TS 29.213 — PCC flows; TS 29.214 — Rx
- TS 24.301 — NAS EPS (bearer-процедуры, TFT)
- TS 23.501, TS 23.502 — 5G QoS (5QI, QFI, QoS rules, SDAP, QNC)
- TS 24.501 — NAS 5G; TS 29.244 — PFCP; TS 29.281 — GTP-U
- TS 23.228, TS 24.229 — IMS (VoLTE/VoNR, Rx)
- TS 22.153 / TS 23.153 — MPS; TS 23.179 — MCPTT
- TS 36.413/36.423, TS 38.413/38.423 — S1AP/X2AP, NGAP/XnAP (QoS в RAN)
