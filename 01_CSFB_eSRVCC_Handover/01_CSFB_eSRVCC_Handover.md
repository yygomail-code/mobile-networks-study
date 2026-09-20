# 1. CSFB, eSRVCC, Handover — интерфейсы и процедуры

> Цель: разобраться, какие интерфейсы и протоколы задействованы при голосовых сценариях LTE
> (CSFB, SRVCC/eSRVCC) и при мобильности (Handover), включая inter-RAT и 5G.

## 1.1 Контекст

- E-UTRAN (LTE) — сеть только с пакетной коммутацией (PS). CS-домена в LTE нет.
- Голос в LTE обеспечивается одним из способов:
  - **CSFB** (Circuit Switched Fallback) — UE уходит в 2G/3G для CS-вызова;
  - **VoLTE** (IMS over LTE) — голос как VoIP в PS-домене;
  - **SRVCC/eSRVCC** — передача уже установленного VoLTE-вызова в CS-домен 2G/3G при потере покрытия LTE.
- В 5G: **VoNR** (IMS over NR); на этапе запуска — **EPS fallback** (перевод в LTE для VoLTE).

Ключевая идея:

- CSFB — «уйти из LTE до вызова»;
- SRVCC — «уйти из LTE во время вызова без разрыва».

## 1.2 CSFB

### 1.2.1 Интерфейсы

| Интерфейс | Между узлами | Протокол | Назначение |
|---|---|---|---|
| SGs | MME ↔ MSC/VLR | SGsAP (SCTP) | «вынос» CS-услуг в LTE: регистрация, пейджинг, MO/MT вызовы, SMS |
| S1-MME | eNB ↔ MME | S1AP (SCTP) | NAS-транспорт, пейджинг, инициирование fallback |
| Uu | UE ↔ eNB | RRC / NAS | радиоинтерфейс |
| A | MSC ↔ BSS (GERAN) | BSSAP | CS-доступ после fallback |
| Iu-CS | MSC ↔ RNS (UTRAN) | RANAP | CS-доступ после fallback |
| S3 | MME ↔ SGSN | GTPv2-C | PS handover при inter-RAT переходе |
| S4 | SGSN ↔ SGW | GTPv2-C | пользовательская плоскость 2G/3G в EPC |
| S11 | MME ↔ SGW | GTPv2-C | управление bearer'ами |
| RIM | eNB/MME ↔ RAN | RAN Information Management | обмен RAN-информацией между RAT (опционально) |

Важно: **SGs — это не Diameter**, это SGsAP поверх SCTP. Diameter используется на Sv (SRVCC), S6a, Gx/Rx и т.д.

### 1.2.2 Основные сообщения SGsAP

| Сообщение | Направление | Когда |
|---|---|---|
| LOCATION-UPDATE-REQUEST / ACCEPT / REJECT | MME → MSC | combined attach/TAU, обновление SGs-ассоциации |
| SERVICE-REQUEST / ACCEPT / REJECT | MME → MSC | MO CSFB (UE хочет позвонить) |
| PAGING-REQUEST / REJECT | MSC → MME | MT CSFB (входящий вызов) |
| DOWNLINK-UNITDATA / UPLINK-UNITDATA | оба | SMS over SGs |
| TMSI-REALLOCATION-COMMAND | MSC → MME | смена TMSI |
| DETACH-INDICATION / ACK, EPS-DETACH-INDICATION, IMSI-DETACH-INDICATION | MME → MSC | отсоединение |
| ALERT-REQUEST / ACK | MME → MSC | UE ответил на пейджинг |
| RESET / RESET-ACK | оба | восстановление после сбоя |
| RELEASE-REQUEST | MSC → MME | освобождение SGs-ассоциации |

### 1.2.3 Combined attach (подготовка к CSFB)

1. UE отправляет Attach Request с типом «combined EPS/IMSI attach» (через eNB → MME).
2. MME выбирает MSC/VLR и отправляет SGsAP-LOCATION-UPDATE-REQUEST (IMSI, TAI/ECGI).
3. MSC создаёт SGs-ассоциацию, возвращает LOCATION-UPDATE-ACCEPT (LAI, TMSI).
4. MME отвечает UE Attach Accept с признаком combined attach, LAI и TMSI.
5. UE и сеть считают, что UE «виден» и в CS-домене (для пейджинга и SMS).

Аналогично выполняется combined TAU при смене Tracking Area.

### 1.2.4 MO CSFB (исходящий вызов)

1. UE → MME: Extended Service Request (service type = mobile originating CS fallback).
2. MME → MSC: SGsAP-SERVICE-REQUEST.
3. MSC → MME: SERVICE-ACCEPT (или REJECT).
4. MME инициирует перевод UE в 2G/3G:
   - **PS Handover** (предпочтительно) — полноценный HO в целевую RAT;
   - **Redirection** — eNB отправляет RRC Connection Release с redirectedCarrierInfo (UTRAN/GERAN); UE сам переходит и при необходимости выполняет LAU/RAU.
5. UE в 2G/3G: CM Service Request → MSC устанавливает CS-вызов.
6. По завершении вызова UE возвращается в LTE (reselection + combined TAU).

### 1.2.5 MT CSFB (входящий вызов)

1. MSC получает входящий вызов и по SGs-ассоциации отправляет SGsAP-PAGING-REQUEST (TMSI/IMSI).
2. MME страничит UE через S1AP Paging (CN domain = CS).
3. UE отвечает Extended Service Request (mobile terminating CS fallback).
4. MME → MSC: SGsAP-SERVICE-REQUEST; MSC → MME: SERVICE-ACCEPT.
5. Далее — перевод в 2G/3G (как в MO); UE отвечает на CS-пейджинг и принимает вызов.

### 1.2.6 SMS over SGs

SMS не требует fallback: сообщения передаются NAS-сигнализацией UE ↔ MME ↔ MSC через SGsAP UPLINK/DOWNLINK-UNITDATA.

### 1.2.7 Способы перевода в CS-домен

| Способ | Как | Плюсы | Минусы |
|---|---|---|---|
| PS Handover | Полноценный HO в целевую RAT | Быстро, контекст сохраняется | Требует поддержки в RAN/MME/UE |
| Redirection | RRC Connection Release с redirectedCarrierInfo | Просто, работает везде | UE сам ищет соту, LAU, задержка выше |
| CCO (Cell Change Order) | Переход в GERAN по команде сети | Для GERAN | Используется редко |

## 1.3 SRVCC / eSRVCC

### 1.3.1 Идея

VoLTE-вызов установлен в PS-домене (IMS). UE теряет покрытие LTE — вызов нужно передать в CS-домен 2G/3G, не разрывая сессию. SRVCC = Single Radio Voice Call Continuity (у UE один радиопередатчик, одновременно LTE и 2G/3G он не держит).

### 1.3.2 Интерфейсы

| Интерфейс | Между узлами | Протокол | Назначение |
|---|---|---|---|
| Sv | MME ↔ MSC Server (enhanced for SRVCC) | Diameter (TS 29.280) | запрос/ответ на PS→CS transfer |
| S1-MME | eNB ↔ MME | S1AP | Handover Required/Command |
| Iu-CS / A | MSC ↔ RNS/BSS | RANAP/BSSAP | целевой CS-доступ |
| Mw | MSC Server ↔ ATCF / IMS | SIP | запрос переноса сессии (STN-SR) |
| I2 | ATCF ↔ SCC AS | SIP | управление transfer в IMS |
| Iq | ATCF ↔ ATGW | внутренний | управление медиашлюзом |
| Gm | UE ↔ P-CSCF/ATCF | SIP | сигнализация UE в IMS |
| Rx | P-CSCF ↔ PCRF | Diameter | авторизация медиа-ресурсов |
| Gx | PCRF ↔ PGW | Diameter | PCC-правила |
| S6a | MME ↔ HSS | Diameter | STN-SR, C-MSISDN (передаются при attach) |
| SGs | — | — | для SRVCC не используется |

### 1.3.3 Sv-сообщения (Diameter)

| Сообщение | Направление | Смысл |
|---|---|---|
| SRVCC PS to CS Request | MME → MSC | начать transfer: IMSI, STN-SR, C-MSISDN, target ID, контейнер |
| SRVCC PS to CS Response | MSC → MME | результат + target-to-source контейнер (CS HO command) |
| SRVCC PS to CS Complete Notification | MME → MSC | UE успешно перешло в CS |
| SRVCC PS to CS Complete Acknowledge | MSC → MME | подтверждение |
| SRVCC PS to CS Cancel Notification / Response | MME → MSC | отмена (если HO не состоялся) |

### 1.3.4 Процедура eSRVCC (упрощённо)

1. eNB по измерениям UE решает, что нужен HO в UTRAN/GERAN (CS).
2. eNB → MME: S1AP Handover Required с признаком SRVCC.
3. MME → MSC Server: Sv SRVCC PS to CS Request.
4. MSC Server выделяет CS-ресурсы и инициирует перенос сессии в IMS:
   - отправляет SIP INVITE с STN-SR;
   - в eSRVCC — сначала в ATCF (Mw), ATCF через I2 взаимодействует с SCC AS; ATGW анкорит медиа;
   - SCC AS обновляет удалённую ногу сессии (remote leg), голос переключается на CS.
5. MSC → MME: Sv SRVCC PS to CS Response (target-to-source контейнер).
6. MME → eNB: S1AP Handover Command.
7. eNB → UE: RRC-команда HO в целевую RAT; UE переходит.
8. UE в 2G/3G: доступ к CS; MSC завершает transfer.
9. MSC → MME: Sv Complete Notification; MME освобождает ресурсы LTE.

### 1.3.5 Ключевые сущности

- **STN-SR** (Session Transfer Number for SRVCC) — номер/URI, по которому MSC инициирует transfer в IMS; хранится в HSS, передаётся MME (S6a) и в MSC (Sv).
- **C-MSISDN** — корреляционный MSISDN для связи PS- и CS-сессии.
- **ATCF/ATGW** (eSRVCC) — якорь в IMS, уменьшает время прерывания (цель < 300 мс; eSRVCC ~150–200 мс).
- **vSRVCC** — SRVCC для видео (в UTRAN), использует те же Sv-процедуры с признаком video.

### 1.3.6 Отличия

| | SRVCC | eSRVCC | vSRVCC |
|---|---|---|---|
| Что переносится | голос | голос | голос + видео |
| Целевая RAT | UTRAN/GERAN | UTRAN/GERAN | UTRAN (HSPA) |
| Якорь в IMS | SCC AS | ATCF + ATGW + SCC AS | SCC AS |
| Время прерывания | до ~300 мс | ~150–200 мс | — |

## 1.4 Handover

### 1.4.1 Типы

- По интерфейсу: X2-based (eNB↔eNB), S1-based (через MME), Xn-based (gNB↔gNB), N2-based (через AMF).
- По частоте: intra-frequency, inter-frequency.
- По RAT: intra-LTE, inter-RAT (E-UTRAN ↔ UTRAN/GERAN), inter-system (EPS ↔ 5GS).
- Служебные: PS HO, CSFB HO, SRVCC, redirection.

### 1.4.2 Интерфейсы LTE

| Интерфейс | Между | Протокол | Назначение |
|---|---|---|---|
| X2 | eNB ↔ eNB | X2AP (SCTP) | HO, SON, MLB, ANR |
| S1-MME | eNB ↔ MME | S1AP | HO через MME, NAS |
| S10 | MME ↔ MME | GTPv2-C | HO между MME (pool) |
| S3 | MME ↔ SGSN | GTPv2-C | inter-RAT PS HO |
| S16 | SGSN ↔ SGSN | GTPv2-C | HO между SGSN |
| S4 | SGSN ↔ SGW | GTPv2-C | PS-доступ 2G/3G в EPC |
| S11 | MME ↔ SGW | GTPv2-C | управление bearer'ами |
| S5/S8 | SGW ↔ PGW | GTPv2-C | управление PDN-сессиями |
| S1-U | eNB ↔ SGW | GTP-U | пользовательские данные |

### 1.4.3 X2-based HO (шаги)

1. UE отправляет Measurement Report; eNB1 решает HO.
2. eNB1 → eNB2: X2AP Handover Request (E-RAB-контекст).
3. eNB2: admission control; X2AP Handover Request Acknowledge.
4. eNB1 → UE: RRCConnectionReconfiguration (mobilityControlInfo).
5. UE синхронизируется с eNB2, Random Access.
6. eNB2 → MME: S1AP Path Switch Request.
7. MME → SGW: GTPv2-C Modify Bearer Request (новый S1-U TEID/адрес).
8. SGW → MME: Modify Bearer Response; MME → eNB2: Path Switch Request Acknowledge.
9. eNB1: X2AP UE Context Release.

### 1.4.4 S1-based HO (шаги)

1. eNB1 → MME: S1AP Handover Required.
2. MME → eNB2: S1AP Handover Request.
3. eNB2 → MME: Handover Request Acknowledge (target-to-source контейнер).
4. MME → eNB1: Handover Command.
5. UE → eNB2: HO; eNB2 → MME: Handover Notify.
6. MME → SGW: Modify Bearer Request; MME → eNB1: UE Context Release Command.

### 1.4.5 Inter-RAT PS HO (E-UTRAN → UTRAN/GERAN)

1. eNB → MME: Handover Required (цель — RNC/BSS).
2. MME выбирает SGSN, → S3: Forward Relocation Request.
3. SGSN создаёт bearer-контексты, → MME: Forward Relocation Response (target-to-source контейнер).
4. MME → eNB: Handover Command; UE уходит в 2G/3G.
5. SGSN → MME: Forward Relocation Complete Notification; MME → SGSN: Ack.
6. Данные: indirect data forwarding через SGW (GTPv2-C Create Indirect Data Forwarding Tunnel Request/Response по S11/S4).

### 1.4.6 5G Handover

| Интерфейс | Между | Протокол |
|---|---|---|
| Xn | gNB ↔ gNB | XnAP |
| N2 | gNB ↔ AMF | NGAP |
| N3 | gNB ↔ UPF | GTP-U |
| N9 | UPF ↔ UPF | GTP-U |
| N14 | AMF ↔ AMF | SBI (HTTP/2) |
| N26 | AMF ↔ MME | GTPv2-C (interworking EPS↔5GS) |

- **Xn-based HO**: gNB1 → gNB2 XnAP Handover Request; Path Switch на AMF (NGAP Path Switch Request); AMF → SMF (N11) → UPF (N4 Modify).
- **N2-based HO**: AMF участвует в HO (Handover Required/Command/Notify).
- **N26**: передача контекста UE и сессий между 5GS и EPS (Forward Relocation Request/Response по GTPv2-C); при отсутствии N26 — без непрерывности сессий.

### 1.4.7 Сводная таблица интерфейсов

| Процедура | Ключевые интерфейсы |
|---|---|
| CSFB | SGs (SGsAP), S1-MME (S1AP), Uu, A/Iu-CS |
| SRVCC/eSRVCC | Sv (Diameter), S1-MME, Iu/A, Mw/I2/Gm (SIP), Rx/Gx |
| LTE HO | X2AP или S1AP, S11/S5/S8 (GTPv2-C), S1-U |
| Inter-RAT PS HO | S3/S4/S16 (GTPv2-C), S1AP |
| 5G HO | XnAP или NGAP, N2/N3/N9, N14, N26 (interworking) |

## 1.5 Сравнение вариантов голоса

| | CSFB | VoLTE | SRVCC/eSRVCC | VoNR |
|---|---|---|---|---|
| Домен | CS (2G/3G) | PS (IMS) | PS → CS | PS (IMS, 5GC) |
| Когда | до/вместо VoLTE | в LTE | при потере LTE во время вызова | в 5G SA |
| Время установления | высокое (fallback) | низкое | — | низкое |
| HD Voice | зависит от RAT | AMR-WB/EVS | да (до перехода) | EVS |
| Основной интерфейс | SGs | Gm/Rx/Gx | Sv | N1/N5/N7 |

## 1.6 Вопросы для самопроверки

1. Почему для CSFB используется SGsAP, а не Diameter?
2. Какие SGsAP-сообщения обслуживают MO и MT вызовы?
3. Что происходит при combined attach и зачем он нужен?
4. Чем PS Handover отличается от redirection при CSFB?
5. Какие интерфейсы задействованы в SRVCC и какой протокол на Sv?
6. Зачем нужны STN-SR и C-MSISDN?
7. Что добавляет eSRVCC по сравнению с SRVCC?
8. Назовите разницу между X2-based и S1-based HO.
9. Какие интерфейсы используются для inter-RAT PS HO?
10. Как выполняется interworking EPS↔5GS и что такое N26?
11. В каком случае VoLTE-вызов переводится в CS и как это отражается на IMS?
12. Какие узлы участвуют в eSRVCC на стороне IMS?

## 1.7 Спецификации

- TS 23.272 — CSFB в EPS
- TS 29.118 — SGsAP
- TS 23.216 — SRVCC
- TS 29.280 — Sv (Diameter)
- TS 23.237 — IMS Service Continuity (ATCF/ATGW)
- TS 23.401 — GPRS/EPS архитектура
- TS 36.413 — S1AP; TS 36.423 — X2AP
- TS 38.413 — NGAP; TS 38.423 — XnAP
- TS 29.274 — GTPv2-C
- TS 23.502 — процедуры 5GS
