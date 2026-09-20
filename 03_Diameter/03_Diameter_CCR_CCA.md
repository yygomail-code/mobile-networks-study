# 3. Diameter CCA/CCR: какие данные передаются и зачем

> Цель: разобрать пару сообщений Credit-Control-Request/Answer (CCR/CCA): где применяется,
> какие AVP передаются, какие сценарии закрывает (тарификация, политики, QoS).

## 3.1 Diameter кратко

- Протокол AAA, peer-to-peer, команды типа Request/Answer.
- Сообщение = Command Code + Application-Id + AVPs (Attribute-Value-Pairs).
- Сессия идентифицируется Session-Id; корреляция запросов/ответов — CC-Request-Number.
- **CCR** (Credit-Control-Request) / **CCA** (Credit-Control-Answer) — пара команд приложения Credit-Control (RFC 4006) и производных 3GPP-приложений.

## 3.2 Где применяются CCR/CCA

| Интерфейс | Между | Приложение | Назначение |
|---|---|---|---|
| Gy | PCEF (PGW/UPF) ↔ OCS | Credit-Control (RFC 4006) | онлайн-тарификация (prepaid, FUP) |
| Gx | PCEF ↔ PCRF/PCF | 3GPP Gx (TS 29.212) | PCC-политики, QoS |
| Ro | IMS AS/MRFC ↔ OCS | Credit-Control | онлайн-тарификация IMS-услуг |
| Sy | PCRF ↔ OCS | 3GPP Sy (TS 29.219) | spending limits (НЕ CCR/CCA: SLR/SLA) |
| Rf | CTF ↔ CDF | Accounting (RFC 6733) | офлайн-тарификация (ACR/ACA) |
| S6a | MME ↔ HSS | 3GPP S6a | мобильность/аутентификация (ULR/AIR) |
| Rx | AF (P-CSCF) ↔ PCRF | 3GPP Rx | авторизация медиа (AAR/AAA) |
| Sd | TDF ↔ PCRF | 3GPP Sd | DPI-детект приложений (TSR/TSA) |

Ключевое: CCR/CCA — это credit-control (Gy/Ro) и policy (Gx). На S6a, Rx, Rf, Sy — **другие команды**.

## 3.3 Структура CCR

Обязательные/типовые AVP:

- Session-Id, Origin-Host, Origin-Realm, Destination-Realm, Destination-Host
- Auth-Application-Id (4 для Gy, 16777238 для Gx), Service-Context-Id
- **CC-Request-Type**: INITIAL_REQUEST(1) / UPDATE_REQUEST(2) / TERMINATION_REQUEST(3) / EVENT_REQUEST(4)
- CC-Request-Number (порядковый номер в сессии)
- Subscription-Id (IMSI/MSISDN)
- **Multiple-Services-Credit-Control (MSCC)** — по одному на Rating-Group:
  - Rating-Group;
  - Requested-Service-Unit (RSU): CC-Time / CC-Total-Octets / CC-Input-Octets / CC-Output-Octets / CC-Service-Specific-Units;
  - Used-Service-Unit (USU): сколько израсходовано с прошлого отчёта;
  - Service-Identifier.
- Для Gx дополнительно: Bearer-Identifier, Bearer-Operation, Flow-Information (5-tuple), QoS-Information, Default-EPS-Bearer-QoS, APN-Aggregate-Max-Bitrate-DL/UL, 3GPP-User-Location-Info (TAI/ECGI), RAT-Type, IP-CAN-Type, User-Equipment-Info (IMEI), Event-Trigger.

## 3.4 Структура CCA

- Result-Code (2001 = SUCCESS и др.)
- CC-Request-Type, CC-Request-Number (эхо запроса)
- MSCC с:
  - Granted-Service-Unit (GSU): выданная квота (время/объём);
  - Validity-Time — срок действия квоты;
  - **Final-Unit-Indication (FUI)**: что делать при исчерпании — TERMINATE / REDIRECT / RESTRICT_ACCESS;
  - Quota-Holding-Time — когда закрывать сессию при простое;
  - Tariff-Time-Change / Tariff-Change-Usage — смена тарифа;
  - Time-Quota-Threshold / Volume-Quota-Threshold — порог досрочного отчёта.
- Для Gx: Charging-Rule-Install / Charging-Rule-Remove (PCC-правила), QoS-Information, Revalidation-Time, Event-Trigger.

## 3.5 Жизненный цикл Gy-сессии

| Этап | Сообщение | Что передаётся | Зачем |
|---|---|---|---|
| Начало | CCR INITIAL | IMSI, APN, RSU (запрос квоты) | авторизация услуги, выдача квоты |
| Периодически/по порогу | CCR UPDATE | USU (израсходовано), RSU (новая квота) | контроль расхода, продление |
| Смена тарифа | CCR UPDATE | USU, Tariff-Time-Change | корректная тарификация |
| Конец | CCR TERMINATION | USU (финальный) | закрытие сессии, финальный расчёт |
| Разовое событие | CCR EVENT | параметры события | тарификация разовых услуг |

Пример: предоплаченный абонент. OCS выдаёт 10 МБ (GSU). При 80 % расхода (порог) PGW отправляет UPDATE с USU и запрашивает новую квоту. При нуле — CCA с FUI=REDIRECT (редирект на страницу «пополните баланс») или TERMINATE.

## 3.6 Жизненный цикл Gx

| Этап | Сообщение | Что передаётся | Зачем |
|---|---|---|---|
| Установка IP-CAN сессии | CCR INITIAL | IMSI, IP, APN, RAT, location | получить PCC-правила |
| Изменение (bearer, location, RAT) | CCR UPDATE | Bearer-Operation, Event-Trigger | актуализация политик |
| Запрос услуги от AF | (Rx AAR → PCRF) → CCA/CCR | Flow-Information, QoS | установка dedicated bearer |
| Конец сессии | CCR TERMINATION | причина | освобождение ресурсов |

Пример VoLTE: P-CSCF (AF) отправляет AAR на PCRF; PCRF формирует PCC-правило и через Gx CCA (Charging-Rule-Install) передаёт PCEF (PGW) QCI=1, GBR, flow-фильтр; PGW инициирует создание dedicated bearer.

## 3.7 Что и зачем — сводная таблица

| Данные | Где | Зачем |
|---|---|---|
| Subscription-Id (IMSI/MSISDN) | CCR/CCA (Gy, Gx) | идентификация абонента для тарификации/политики |
| User-Equipment-Info (IMEI) | CCR (Gy/Gx) | контроль устройств, тарифы по типу устройства |
| Rating-Group | MSCC | привязка к тарифному плану/услуге |
| RSU/USU/GSU | MSCC | запрос/отчёт/выдача квоты |
| CC-Time/Octets/Events | RSU/USU/GSU | измерение услуги |
| Final-Unit-Indication | CCA | поведение при исчерпании (redirect/terminate/restrict) |
| Validity-Time | CCA | срок жизни квоты |
| Quota-Holding-Time | CCA | закрытие «молчащей» сессии |
| Tariff-Time-Change | CCA/CCR | смена тарифа по времени |
| QoS-Information / APN-AMBR / QCI / ARP | Gx | установка QoS |
| Flow-Information | Gx | SDF-фильтр (5-tuple) для bearer'а |
| Charging-Rule-Install/Remove | Gx CCA | установка/снятие PCC-правил |
| Event-Trigger | Gx | условия отправки CCR UPDATE |
| 3GPP-User-Location-Info, RAT-Type | Gx/CCR | политика по месту/технологии |
| Result-Code | CCA | успех/ошибка обработки |

## 3.8 Ошибки и особые коды

- 2001 — DIAMETER_SUCCESS.
- 4010 — DIAMETER_END_USER_SERVICE_DENIED (услуга запрещена абоненту).
- 4011 — DIAMETER_CREDIT_CONTROL_NOT_APPLICABLE (кредит-контроль не применим).
- 4012 — DIAMETER_CREDIT_LIMIT_REACHED (лимит исчерпан).
- 4xxx — transient failures (можно повторить), 5xxx — permanent failures.
- Failover: при недоступности OCS — поведение по настройке (allow/deny), резервирование, retry/backoff.

## 3.9 Практические кейсы

1. Prepaid data: Gy INITIAL/UPDATE/TERMINATION; FUI redirect.
2. FUP: порог объёма → PCRF/OCS снижает APN-AMBR (Gx) или блокирует.
3. Zero-rating: отдельный Rating-Group с нулевой ценой (Gy); классификация трафика — DPI/TDF.
4. «Турбо-кнопка»: OCS/PCRF временно повышает квоту/QoS (Gx).
5. Роуминг: тарификация через OCS домашней сети (Gy), политика — PCRF.
6. 5G: конвергентная тарификация через CHF (сервисный интерфейс Nchf; SMF↔CHF — N40); Diameter Gy/Gx остаётся для EPC/interworking.

## 3.10 Вопросы для самопроверки

1. Чем CCR/CCA отличаются от ACR/ACA и ULR/ULA?
2. Какие CC-Request-Type существуют и когда используются?
3. Что такое MSCC и Rating-Group?
4. Что содержат GSU, RSU, USU?
5. Что произойдёт при FUI=REDIRECT?
6. Какие данные Gx используются для установки QoS?
7. Зачем нужен Event-Trigger?
8. Что означают Result-Code 4012 и 4011?
9. Как тарифицируется VoLTE (Ro/Rx/Gx)?
10. Как меняется тарификация в 5G (CHF)?

## 3.11 Спецификации

- RFC 4006 — Diameter Credit-Control Application
- RFC 6733 — Diameter Base Protocol
- TS 29.212 — Gx; TS 29.213 — PCC signalling flows
- TS 29.214 — Rx; TS 29.219 — Sy; TS 29.274 — GTPv2-C
- TS 32.299 — Charging AVPs; TS 32.240 — Charging architecture
- TS 32.290/32.291 — 5G charging (CHF)
- TS 29.244 — PFCP
