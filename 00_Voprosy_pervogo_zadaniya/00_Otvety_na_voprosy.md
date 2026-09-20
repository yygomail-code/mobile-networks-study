# Ответы на вопросы первого задания

> Исходное задание (сохранено дословно):
>
> 1. Какие интерфейсы используются для работы процедуры CSFB, eSRVCC, Handover.
> 2. В чем отличие работы 5g для схем SA, NSA, DSS как реализуется.
> 3. Какие данные передаются и для чего в сообщения diameter CCA/CCR.
> 4. Какие узлы используются в IMS и их связь с PS Core.
> 5. Какие типа фильтрации трафика на мобильных сетях вы знаете.
> 6. Что такое QOS какие возможности есть по его настроек и использования на сети для абонентов мобильной связи.
> 7. Какие существуют виды и возможности оптимизации сети RAN есть с т.з. Core. Приведите примеры из работы.
>
> Этот файл — сводка-навигатор: краткий ответ по каждому вопросу и где лежат подробные
> материалы (конспекты, схемы, видеолекции). Все 7 тем закрыты подробно.

## Карта: вопрос → материалы

| № | Вопрос | Папка с материалами | Видео |
|---|---|---|---|
| 1 | CSFB, eSRVCC, Handover | `01_CSFB_eSRVCC_Handover/` | 12:34 |
| 2 | 5G: SA, NSA, DSS | `02_5G_SA_NSA_DSS/` | 10:19 + 11:35 |
| 3 | Diameter CCA/CCR | `03_Diameter/` | 11:56 + 20:37 |
| 4 | IMS и PS Core | `05_IMS/`, `04_PS_Core/` | 11:09 и 17:05 |
| 5 | Фильтрация трафика | `07_Filtraciya/` | 10:43 |
| 6 | QoS | `06_QoS/` | 10:09 + 12:43 |
| 7 | Оптимизация RAN (Core) | `08_Optimizaciya_RAN/` | 17:51 + 12:10 |

---

## 1. Интерфейсы CSFB, eSRVCC, Handover

**CSFB (голос до установления вызова):**

- **SGs** — MME ↔ MSC (SGsAP, SCTP 29118): combined attach/TAU, paging CS, SMS; обязателен
  TAC↔LAC mapping.
- Перевод в 2G/3G: **PS HO** (S3 MME↔SGSN, S4 SGW↔SGSN) или **redirect** (RRC Release с redirect info);
  возврат — fast return.
- KPI: CSFB SR, setup time (цель ≤ 6 с), SGs paging SR.

**eSRVCC (перевод активного голоса):**

- **Sv** — MME ↔ MSC (Diameter, TS 29.280): PS-to-CS перевод.
- IMS-якорь: **SCC AS** + **ATCF/ATGW** (непрерывность медиа, TS 23.237);
  interruption time < 300 мс.

**Handover:**

- Intra-LTE: **X2** (X2AP, SCTP 36422) и **S1** (S1AP, SCTP 36412); S10 при смене MME,
  Path Switch к SGW (S11/S5-S8).
- 5G: **Xn** (XnAP, SCTP 38422), **N2** (NGAP, SCTP 38412); LTE↔5G — **N26** (AMF↔MME).
- Inter-RAT LTE↔2G/3G: S3/S4; голос — SGs/Sv.
- Ошибки MRO: too early / too late / wrong cell; KPI — HO SR по типам.

**Подробно:** `01_CSFB_eSRVCC_Handover/27_...md` (+ PDF), схемы `28` (CSFB+SRVCC) и `29` (карта
переходов), видео `video/27_CSFB_modul1.mp4`.

## 2. 5G: отличие SA, NSA, DSS

- **SA (Option 2)**: радио NR + ядро 5GC (сервисная архитектура SBA: AMF/SMF/UPF/PCF/UDM);
  якорь — NR; доступны слайсинг (S-NSSAI), URLLC, VoNR, MEC; интерфейсы N1/N2/N3/N4/N7.
- **NSA (EN-DC, Option 3x)**: якорь — LTE (eNB, MN), NR — вторичный узел (gNB, SN), ядро — EPC;
  управление через LTE, данные — LTE+NR (X2-C/U, split bearer), голос — VoLTE.
  Быстрый запуск, но нет слайсинга/VoNR; UE добавляется в NR по событию B1 (SgNB Addition).
- **DSS**: LTE и NR делят одну несущую; NR обходит LTE CRS (rate matching), SSB NR —
  в MBSFN-субкадрах LTE, SCS 15 кГц; плюс — покрытие 5G без нового спектра, минус — потеря
  ёмкости и ограничение скорости.
- Переходы: **N26** (AMF↔MME), **EPS fallback** для голоса, промежуточная опция 7x
  (LTE-якорь + 5GC).

**Подробно:** `02_5G_SA_NSA_DSS/18_...md` (+ PDF), схемы `19` (NSA/SA) и `20` (DSS),
видео `video/18_5G_modul1.mp4`, `video/18_5G_modul2.mp4`.

## 3. Данные в сообщениях Diameter CCA/CCR

- CCR/CCA — приложение **Credit-Control** (RFC 4006/8506), код команды **272**.
- Ключевые AVP: **CC-Request-Type** (INITIAL=1 / UPDATE=2 / TERMINATION=3 / EVENT=4),
  **CC-Request-Number**, **Subscription-Id** (IMSI/MSISDN),
  **Multiple-Services-Credit-Control (MSCC)**: Rating-Group, Service-Identifier,
  **Requested-Service-Unit** (запрос квоты), **Used-Service-Unit** (отчёт о расходе),
  **Granted-Service-Unit** (выданная квота), Final-Unit-Indication.
- **Зачем:** онлайн-контроль расхода: выдача квот, списание, блокировка при исчерпании.
  Цикл на Gy/Ro: INITIAL → CCA(GSU) → периодические UPDATE (USU) → TERMINATION;
  коды 4010/4011/4012 — credit-limit reached / no credit.
- **На Gx** (PCRF↔PGW): CCR — запрос/изменение правил, CCA несёт Charging-Rule-Install/Remove,
  QoS-Information (QCI, GBR/MBR, ARP), APN-AMBR, Default-EPS-Bearer-QoS, Event-Trigger,
  Revalidation-Time — это то, как политики QoS и тарификации доезжают до шлюза.
- **Зачем инженеру:** FUP, zero-rating, тарифные пороги, изменения при роуминге (S9),
  диагностика (Result-Code, Experimental-Result-Code).

**Подробно:** `03_Diameter/13_...md` (+ PDF), схема `14`, видео `video/13_Diameter_modul1.mp4`,
`video/13_Diameter_modul2.mp4`; связка с QoS — `06_QoS/15_...md`.

## 4. Узлы IMS и их связь с PS Core

- **Узлы IMS:** P-CSCF (первый контакт, SIP-прокси), I-CSCF (вход, выбор S-CSCF), S-CSCF
  (сессии, триггеры услуг), HSS/SLF (профиль), AS (MMTel, SCC, IP-SM-GW), MRF (медиа-ресурсы),
  BGCF/MGCF/MGW (стык с PSTN), IBCF/TrGW (граница сетей), DNS/ENUM, SEG.
- **Связь с PS Core:** PS-домен — транспорт для IMS:
  - доступ UE к P-CSCF — через **SGi** (PGW/UPF) по bearer'у APN/DNN `ims`;
  - QoS медиа — **Rx/N5** (P-CSCF → PCRF/PCF) → **Gx/N7** → PGW/UPF → выделенный
    bearer **QCI 1** / поток **5QI 1** (сигнализация — QCI 5);
  - Diameter **Cx/Sh** к HSS (в 5G — UDM/PCF через SBI, N5/N7);
  - голос: **VoLTE** = IMS + QCI 1; **VoNR** — та же IMS в 5GC; без VoLTE — CSFB/SRVCC (SGs/Sv);
    **EPS fallback** (N26) — голос из SA в LTE; роуминг — S8HR/SEPP.
- Кодеки: EVS/AMR-WB; метрики: setup < 2 с, MOS ≥ 3,5–4,0, RTP loss < 1%, jitter < 30–50 мс.

**Подробно:** `05_IMS/30_...md` (+ PDF), схемы `31` (архитектура) и `32` (VoLTE e2e),
видео `video/30_IMS_modul1.mp4`; PS-часть — `04_PS_Core/21_...md`, схемы `22/23`,
видео `video/21_PS_Core_modul1.mp4`.

## 5. Типы фильтрации трафика

- **Уровни:** UE (TFT в 4G / QoS rules в 5G — исходящий трафик), Core PGW/UPF (SDF/PDR —
  входящий; FAR/QER/URR — действия/QoS/учёт; ACL/NAT), TDF/DPI (приложения, URL, категории; Sd),
  граница SGi/N6 (firewall, anti-DDoS), DNS-фильтрация (домены/категории).
- **По чему фильтруют:** 5-tuple (IP/порты/протокол), DSCP, приложение (сигнатуры DPI),
  URL/категории, домены, SPI; precedence решает порядок правил.
- **4G:** TFT (UL) + SDF/PCC-правила (DL) + TDF; **5G:** PDR/FAR/QER/URR в UPF (N4/PFCP) +
  QoS rules в UE + reflective QoS.
- **Применение:** zero-rating, FUP, родительский контроль, корпоративные APN, роуминг-политики,
  anti-fraud/DDoS, LI.
- **Диагностика:** UL и DL настраиваются раздельно (частая причина «в одну сторону не работает»),
  проверять Gx/N7-правила, N4-сессии (PFCP), счётчики правил/URR; шифрование (TLS 1.3/QUIC)
  снижает точность DPI.

**Подробно:** `07_Filtraciya/33_...md` (+ PDF), схемы `34` (карта уровней) и `35` (конвейер UPF),
видео `video/33_Filter_modul1.mp4`.

## 6. QoS: что это и какие возможности настройки для абонентов

- **QoS** — приоритет, гарантированная полоса, задержки и потери для разных типов трафика;
  метрики: throughput, latency, jitter, loss, QoE (MOS).
- **4G:** bearer'ы (default/dedicated), **QCI** (1–9, 65–95), **ARP** (1–15 + pre-emption),
  **GBR/MBR**, **APN-AMBR**, **UE-AMBR** (в eNB), **TFT**. **5G:** QoS flows (QFI, **5QI**),
  session-AMBR, QNC/AQP, reflective QoS.
- **Управление:** PCC — PCRF/PCF (правила Gx/N7, запросы Rx/N5), подписка HSS/UDM;
  исполнение — PGW/UPF и eNB/gNB (admission control, планировщик).
- **Возможности для абонентов:** тарифные скорости (AMBR), «турбо-кнопка» (временное повышение),
  FUP (снижение/блокировка после лимита), VIP/корпоратив (ARP/GBR), игры (QCI 3), видео (4/6),
  VoLTE (QCI 1), IoT (QCI 9), экстренные/MPS (ARP + pre-emption).
- **Диагностика:** S1AP/NGAP (QCI/5QI, ARP, GBR), Gx/N7 (правила), TFT/QoS rules (классификация);
  типовые ошибки: нет правила, TFT mismatch, низкий AMBR, ARP-конфликт.

**Подробно:** `06_QoS/15_...md` (+ PDF), схемы `16` (4G) и `17` (5G), видео
`video/15_QoS_modul1.mp4`, `video/15_QoS_modul2.mp4`.

## 7. Оптимизация RAN с точки зрения Core + примеры

- **Рычаги Core:** QoS (QCI/5QI, ARP, GBR/AMBR), политики PCC, мобильность (TA/TAI, **RFSP/SPID**,
  пулы MME/SGW/PGW, S10/N26), paging (TA list, DRX/eDRX, T3412), перегрузка (overload,
  backoff T3346), голос (VoLTE/CSFB/EPS fallback/SRVCC), транспорт (GTP-U, DSCP, MTU,
  размещение UPF/MEC), данные (PM-счётчики, Trace, MDT, RCAF, SON-политики).
- **KPI:** RRC SR, E-RAB SR, drop rate, HO SR, paging SR, throughput, latency, MOS;
  формулы «успех/попытки», спецификации TS 32.425/32.450/28.552.
- **Процесс:** baseline → один фактор → guard-KPI → наблюдение 24–72 ч → откат/закрепление;
  счётчики Core и RAN считаются независимо — расхождение указывает на стык/транспорт.
- **Примеры (кейсы из конспекта):** просадка VoLTE MOS (QCI1/транспорт), paging storm у IoT
  (T3412/eDRX), падение HO SR после пересмотра пулов, задержка CSFB (TAC-LAC/redirect),
  просадка LTE после DSS, EPS fallback не срабатывает (N26/IMS-флаг), перегрузка MME
  (overload/backoff) и др. — всего 10 кейсов + чек-листы «симптом → где смотреть → действие».
- Для защиты: в конспекте отдельный раздел «Перекрёстные вопросы и ответы» (20 шт) и
  пример расчёта KPI (99,5% → разбор отказов по cause → 99,9%).

**Подробно:** `08_Optimizaciya_RAN/24_...md` (+ PDF), схемы `25` (рычаги) и `26` (петля
оптимизации), видео `video/24_RAN_modul1.mp4`, `video/24_RAN_modul2.mp4`.

---

## Как пользоваться

1. Быстрый проход — этот файл (или PDF рядом).
2. Углубление — папка вопроса: подробный конспект (md/PDF) + схемы.
3. Закрепление — видеолекция из папки `video/`.
4. Самопроверка — разделы «Вопросы для самопроверки» / «Вопросы для защиты» в конспектах.
