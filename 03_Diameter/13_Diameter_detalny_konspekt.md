# 13. Diameter: очень подробный конспект

> Кому: инженерам, изучающим сигнализацию мобильных сетей (2G/3G/4G/5G/IMS).
> Цель: разобрать Diameter «от заголовка до прикладных сценариев»: базовый протокол, AVP, команды,
> приложения и интерфейсы, агентов (DRA/DEA), безопасность, диагностику и типовые проблемы.
>
> Связанные файлы: `03_Diameter_CCR_CCA.md` (тарификация и политики), `04_IMS_uzly_i_PS_Core.md` (IMS),
> `09_Interfejsy_LTE_PS_CS_troubleshooting.md` (диагностика), `11_Schema_interfejsov.svg` (общая схема).

## 13.1 Что такое Diameter и почему он повсюду

**Diameter** — протокол AAA (Authentication, Authorization, Accounting): аутентификация, авторизация,
учёт. Он пришёл на смену RADIUS и частично SS7/MAP и стал единым «языком» для:

- данных абонента и аутентификации — HSS/UDM (S6a, S6d, SWx, Cx, Sh);
- политик и QoS — PCRF/PCF (Gx, Gxx, Rx, Sd, Np, Sy);
- тарификации — OCS/CHF (Gy, Ro, Sy);
- мобильности и interworking — MME/SGSN (S6a, S6d, S3/S4 через GTP, Sv);
- проверки устройств — EIR (S13);
- IMS — CSCF/AS (Cx, Sh, Rx, Ro);
- Wi-Fi calling — ePDG/AAA (SWm, SWx, S6b);
- IoT/SCEF (S6m, S6t, T6a/T6b), LCS (SLg, SLh), SMS (S6c, SGd).

Ключевые свойства:

- **peer-to-peer**: обе стороны равноправны, каждая может быть клиентом или сервером;
- **надёжный транспорт**: TCP или SCTP (не UDP);
- **расширяемость**: собственные AVP и приложения;
- **агенты**: relay/proxy/redirect/translation — можно строить маршрутизацию (DRA);
- **сессии и таймеры**: контроль состояния соединений и транзакций.

Название — шутка стандартизаторов: «вдвое лучше RADIUS» (RADIUS — «радиус», Diameter — «диаметр»).

Базовые документы: **RFC 6733** (Diameter Base Protocol, заменяет RFC 3588),
**RFC 8506** (Diameter Credit-Control, заменяет RFC 4006), **RFC 4072** (EAP), **RFC 4740** (SIP-приложение),
**RFC 7683** (перегрузка), **RFC 7944** (приоритеты, DRMP).

В 5G появляется сервисная архитектура SBA (HTTP/2 + JSON), но Diameter не исчезает: он остаётся
в EPS, IMS, 2G/3G и на стыках interworking (через функции-переводчики IWF). Подробнее — раздел 13.9.

## 13.2 Базовый протокол

### 13.2.1 Модель и роли

- **Diameter Client / Server** — инициатор запроса и сторона, которая отвечает (роли могут меняться на уровне приложения).
- **Peer** — сосед по соединению; **Realm** — «домен» маршрутизации (обычно домен оператора).
- **Diameter Identity** — FQDN узла (например, `mme01.epc.mnc001.mcc250.3gppnetwork.org`).
- Агенты (RFC 6733):
  - **Relay** — пересылает сообщения, проверяет Route-Record, не меняет прикладные AVP;
  - **Proxy** — может менять AVP, применять политики, скрывать топологию;
  - **Redirect** — отвечает клиенту Redirect-Host, перенаправляя к нужному серверу;
  - **Translation** — перевод между RADIUS и Diameter.

### 13.2.2 Транспорт

- Порт **3868/TCP** и **3868/SCTP** (IANA).
- Защищённый транспорт: **5868/TLS (TCP)** и **5868/DTLS (SCTP)** (IANA).
- SCTP предпочтителен: multi-homing, устойчивость к отказам, сохранение границ сообщений.
- В 3GPP защита узлов — **NDS/IP** (TS 33.210/33.310): IPsec, сертификаты, защищённые домены.

### 13.2.3 Установка соединения, состояния, таймеры

При установке соединения стороны обмениваются возможностями:

1. **CER/CEA** (Capabilities-Exchange-Request/Answer, команда 257): версия, Origin-Host/Realm,
   Host-IP-Address, Vendor-Id, Product-Name, список поддерживаемых приложений
   (Auth-Application-Id, Acct-Application-Id, Vendor-Specific-Application-Id).
2. **DWR/DWA** (Device-Watchdog-Request/Answer, команда 280): проверка живости peer'а
   (в RFC 6733 значение таймера Tw — 30 секунд).
3. **DPR/DPA** (Disconnect-Peer-Request/Answer, команда 282): корректное закрытие соединения
   с причиной (Disconnect-Cause: REBOOTING, BUSY, DO_NOT_WANT_TO_TALK_TO_YOU).

Состояния peer-соединения: Closed → I-Open → R-Open → Open → Closing.
Если CER/CEA не согласовали общее приложение, соединение не поднимется (или приложение будет недоступно).

### 13.2.4 Сообщение Diameter и AVP

Заголовок (20 байт + AVPs):

```
 0                   1                   2                   3
 +-------+-------------------------------+-------------------------------+
 | Версия|        Длина сообщения (24 бита)                              |
 +-------+---------------+-------+---------------------------------------+
 | R P E T |           Код команды (24 бита)                             |
 +-------------------------------+---------------------------------------+
 |                        Application-Id (32 бита)                       |
 +-----------------------------------------------------------------------+
 |                       Hop-by-Hop Identifier (32)                      |
 +-----------------------------------------------------------------------+
 |                      End-to-End Identifier (32)                       |
 +-----------------------------------------------------------------------+
 |                            AVPs ...                                   |
```

Флаги заголовка:

- **R** (Request) — это запрос (иначе ответ);
- **P** (Proxiable) — сообщение можно пересылать через агента;
- **E** (Error) — ответ содержит ошибку;
- **T** (Potentially re-transmitted) — возможна повторная передача (дубликат).

AVP (Attribute-Value-Pair):

```
 +-----------------------------------------------------------------------+
 |                          AVP Code (32)                                |
 +-------+-------------------------------+-------------------------------+
 | V M P |       Длина AVP (24 бита)     |  Vendor-Id (32, если V=1)     |
 +-------+-------------------------------+-------------------------------+
 |                          Данные (выровнены до 4 байт)                 |
 +-----------------------------------------------------------------------+
```

Флаги AVP:

- **V** (Vendor-specific) — есть Vendor-Id (0 = IETF, **10415 = 3GPP**);
- **M** (Mandatory) — незнание AVP = ошибка;
- **P** (Protected) — шифрование end-to-end (используется редко).

Типы данных AVP: OctetString, Integer32/64, Unsigned32/64, Float32/64, Address, Time,
UTF8String, DiameterIdentity, DiameterURI, Enumerated, IPFilterRule, QoSFilterRule, **Grouped** (вложенные AVP).

### 13.2.5 Команды базового протокола (IANA)

| Код | Команда | Назначение |
|---|---|---|
| 257 | CER / CEA | обмен возможностями при соединении |
| 258 | RAR / RAA | Re-Auth (переавторизация сессии) |
| 271 | ACR / ACA | учёт (accounting) |
| 272 | CCR / CCA | Credit-Control (Gy/Ro/Gx) |
| 274 | ASR / ASA | Abort-Session |
| 275 | STR / STA | Session-Termination |
| 280 | DWR / DWA | watchdog |
| 282 | DPR / DPA | разрыв соединения |
| 265 | AAR / AAA | AA-Request (Rx, NASREQ) |
| 268 | DER / DEA | EAP (SWm) |

3GPP-команды (примеры, IANA):

| Код | Команда | Интерфейс |
|---|---|---|
| 283–288 | UAR/UAA, SAR/SAA, LIR/LIA, MAR/MAA, RTR/RTA, PPR/PPA | Cx/Dx (RFC 4740, 3GPP-профиль) |
| 306–309 | UDR/UDA, PUR/PUA, SNR/SNA, PNR/PNA | Sh (TS 29.328) |
| 316–324 | ULR/ULA, CLR/CLA, AIR/AIA, IDR/IDA, DSR/DSA, PUR/PUA, RSR/RSA, NOR/NOA, ECR/ECA | S6a/S6d, S13 (TS 29.272) |
| 8388620–8388621 | PLR/PLA, LRR/LRA | SLg (TS 29.172) |
| 8388622 | RIR/RIA | SLh (TS 29.173) |
| 8388635–8388636 | SLR/SLA, SNR/SNA | Sy (TS 29.219) |
| 8388637 | TSR/TSA | Sd (TS 29.212) |

### 13.2.6 Result-Code (AVP 268): коды и категории

Диапазоны: 1xxx — информационные, 2xxx — успех, 3xxx — протокольные ошибки,
4xxx — временные (transient), 5xxx — постоянные (permanent). Базовые значения (RFC 6733, IANA):

| Код | Имя | Смысл |
|---|---|---|
| 2001 | DIAMETER_SUCCESS | успех |
| 2002 | DIAMETER_LIMITED_SUCCESS | успех с ограничениями |
| 3001 | DIAMETER_COMMAND_UNSUPPORTED | команда не поддерживается |
| 3002 | DIAMETER_UNABLE_TO_DELIVER | доставка невозможна (маршрутизация) |
| 3003 | DIAMETER_REALM_NOT_SERVED | realm не обслуживается |
| 3004 | DIAMETER_TOO_BUSY | узел перегружен |
| 3005 | DIAMETER_LOOP_DETECTED | обнаружен цикл маршрутизации |
| 3006 | DIAMETER_REDIRECT_INDICATION | требуется редирект |
| 3007 | DIAMETER_APPLICATION_UNSUPPORTED | приложение не поддерживается |
| 3008 | DIAMETER_INVALID_HDR_BITS | неверные биты заголовка |
| 3009 | DIAMETER_INVALID_AVP_BITS | неверные биты AVP |
| 3010 | DIAMETER_UNKNOWN_PEER | неизвестный peer |
| 4001 | DIAMETER_AUTHENTICATION_REJECTED | аутентификация отклонена |
| 4002 | DIAMETER_OUT_OF_SPACE | нет места |
| 4003 | ELECTION_LOST | проиграна «выборная» процедура |
| 4010 | DIAMETER_END_USER_SERVICE_DENIED | услуга запрещена абоненту |
| 4011 | DIAMETER_CREDIT_CONTROL_NOT_APPLICABLE | кредит-контроль неприменим |
| 4012 | DIAMETER_CREDIT_LIMIT_REACHED | лимит исчерпан |
| 4013 | DIAMETER_USER_NAME_REQUIRED | требуется User-Name |
| 5001 | DIAMETER_AVP_UNSUPPORTED | AVP не поддерживается |
| 5002 | DIAMETER_UNKNOWN_SESSION_ID | неизвестный Session-Id |
| 5003 | DIAMETER_AUTHORIZATION_REJECTED | авторизация отклонена |
| 5004 | DIAMETER_INVALID_AVP_VALUE | неверное значение AVP |
| 5005 | DIAMETER_MISSING_AVP | отсутствует обязательный AVP |
| 5006 | DIAMETER_RESOURCES_EXCEEDED | превышены ресурсы |
| 5007 | DIAMETER_CONTRADICTING_AVPS | противоречивые AVP |
| 5008 | DIAMETER_AVP_NOT_ALLOWED | AVP не разрешён |
| 5009 | DIAMETER_AVP_OCCURS_TOO_MANY_TIMES | AVP встречается слишком часто |
| 5010 | DIAMETER_NO_COMMON_APPLICATION | нет общего приложения |
| 5011 | DIAMETER_UNSUPPORTED_VERSION | версия не поддерживается |
| 5012 | DIAMETER_UNABLE_TO_COMPLY | невозможно выполнить |
| 5013 | DIAMETER_INVALID_BIT_IN_HEADER | неверный бит в заголовке |
| 5014 | DIAMETER_INVALID_AVP_LENGTH | неверная длина AVP |
| 5015 | DIAMETER_INVALID_MESSAGE_LENGTH | неверная длина сообщения |
| 5016 | DIAMETER_INVALID_AVP_BIT_COMBO | неверная комбинация битов AVP |
| 5017 | DIAMETER_NO_COMMON_SECURITY | нет общей политики безопасности |

**Важно:** в 3GPP-приложениях числовые значения кодов переопределяются профилем приложения.
Пример S6a/S6d (TS 29.272): 5001 = DIAMETER_ERROR_USER_UNKNOWN, 5004 = DIAMETER_ERROR_ROAMING_NOT_ALLOWED,
5420 = DIAMETER_ERROR_UNKNOWN_EPS_SUBSCRIPTION, 5421 = DIAMETER_ERROR_RAT_NOT_ALLOWED,
5422 = DIAMETER_ERROR_EQUIPMENT_UNKNOWN, 5423 = DIAMETER_ERROR_UNKNOWN_SERVING_NODE.
Поэтому «5001» в базовом протоколе и «5001» в S6a — разные вещи: всегда смотрите приложение
и то, в каком AVP пришёл код — Result-Code (268) или Experimental-Result (297) / Experimental-Result-Code (298).

### 13.2.7 Сессии, повторные передачи, маршрутизация

- **Session-Id** (AVP 263) — уникальный идентификатор сессии; формат: `<DiameterIdentity>;<high32>;<low32>[;optional]`.
- **Hop-by-Hop Id** меняется на каждом участке, **End-to-End Id** сохраняется от источника до адресата.
- **T-флаг** и повторные передачи: при таймауте клиент повторяет запрос с тем же Session-Id/CC-Request-Number;
  сервер обязан распознать дубликат и не выполнять операцию дважды.
- **Маршрутизация**: по Destination-Realm (обычно) и Destination-Host (если известен конкретный узел);
  Route-Record защищает от петель; Proxy-Info служит для обратной связи через прокси.
- **Редиректы**: агент может ответить Redirect-Host + Redirect-Host-Usage (261) + Redirect-Max-Cache-Time (262).
- **Failover**: Session-Binding (270) и Session-Server-Failover (271): REFUSE_SERVICE / TRY_AGAIN / ALLOW_SERVICE /
  TRY_AGAIN_ALLOW_SERVICE — поведение при отказе сервера сессии.

## 13.3 Приложения Diameter в мобильной сети (Application-Id)

Значения подтверждены по реестру IANA (AAA Parameters). Диапазон 0–16777215 — стандартные приложения,
16777216+ — вендорные/3GPP.

| App-Id | Приложение | Интерфейс(ы) | Команды | Спецификация |
|---|---|---|---|---|
| 0 | Diameter common | база | CER/CEA, DWR/DWA, DPR/DPA... | RFC 6733 |
| 3 | Accounting (Rf) | Rf | ACR/ACA | RFC 6733 |
| 4 | Credit-Control (Gy, Ro) | Gy, Ro | CCR/CCA | RFC 8506 |
| 16777216 | 3GPP Cx | Cx, Dx | UAR, SAR, LIR, MAR, RTR, PPR | TS 29.228/29.229 |
| 16777217 | 3GPP Sh | Sh | UDR, PUR, SNR, PNR | TS 29.328/29.329 |
| 16777236 | 3GPP Rx | Rx | AAR/AAA, RAR/RAA | TS 29.214 |
| 16777238 | 3GPP Gx | Gx | CCR/CCA, RAR/RAA | TS 29.212 |
| 16777251 | 3GPP S6a | S6a, S6d | ULR, CLR, AIR, IDR, DSR, PUR, RSR, NOR | TS 29.272 |
| 16777252 | 3GPP S13 | S13, S13' | ECR/ECA | TS 29.272 |
| 16777255 | SLg | SLg | PLR/PLA, LRR/LRA | TS 29.172 |
| 16777264 | 3GPP SWm | SWm | EAP-процедуры | TS 29.273 |
| 16777265 | 3GPP SWx | SWx | SAR, MAR, RTR, PPR | TS 29.273 |
| 16777266 | 3GPP Gxx | Gxx | CCR/CCA | TS 29.212 |
| 16777267 | 3GPP S9 | S9 | (политики фиксированного доступа) | TS 29.215 |
| 16777272 | 3GPP S6b | S6b | RAR/RAA, ASR/ASA | TS 29.273 |
| 16777291 | 3GPP SLh | SLh | RIR/RIA | TS 29.173 |
| 16777302 | 3GPP Sy | Sy | SLR/SLA, SNR/SNA | TS 29.219 |
| 16777303 | 3GPP Sd | Sd | TSR/TSA | TS 29.212 |
| 16777310 | 3GPP S6m | S6m | (SCEF ↔ HSS) | TS 29.336 |
| 16777312 | 3GPP S6c | S6c | SRR/SRA | TS 29.338 |
| 16777313 | 3GPP SGd | SGd | (SMS через PS) | TS 29.338 |
| 16777342 | 3GPP Np | Np | (RCAF ↔ PCRF) | TS 29.217 |
| 16777345 | 3GPP S6t | S6t | (SCEF ↔ HSS) | TS 29.336 |
| 16777346 | 3GPP T6a/T6b | T6a/T6b | (SCEF ↔ MME/SGSN) | TS 29.128 |
| — | 3GPP Sv | Sv | SRVCC PS to CS... | TS 29.280 |

Примечания:

- **S6d** использует тот же Application-Id, что и S6a (16777251).
- Для **Sv** в реестре IANA AAA отдельной записи нет: значение Application-Id задаётся TS 29.280 —
  уточняйте по актуальной версии спецификации и по трассировке.
- Исторические (не используются в новых сетях): Wx (16777219), Gq (16777222), Gx по TS 29.210 (16777224),
  Rx по TS 29.211 (16777229), Zh/Zn (GBA).
- 3GPP Vendor-Id = **10415**.

## 13.4 Агенты и топологии: DRA, DEA, SLF, IPX, IWF

```
                         ┌────────────────────────────┐
                         │        IMS (Cx, Sh, Rx)    │
                         │  P/I/S-CSCF, TAS/SCC AS    │
                         └─────────────┬──────────────┘
                                       │
 [MME]──S6a──┐                 ┌───────┴────────┐
 [SGSN]─S6d──┼─────DRA/DEA─────│  HSS / SLF     │
 [PGW]──Gx───┤   (Proxy/Relay) │  EIR, UDM      │
 [P-CSCF]─Rx─┘                 └───────┬────────┘
                                       │
 [PGW]──Gy──┐                 ┌───────┴────────┐
 [TDF]──Sd──┼─────DRA/DEA─────│ PCRF / PCF     │
 [RCAF]─Np──┘                 │ OCS / CHF      │
                              └────────────────┘
        в роуминге:  DEA ── IPX/GRX ── DEA
```

- **DRA** (Diameter Routing Agent) — центральный маршрутизатор Diameter: выбор сервера по realm/host,
  балансировка, sticky-сессии, скрытие топологии, защита, rate limiting, протокольная нормализация.
- **DEA** (Diameter Edge Agent) — DRA на границе сети; в роуминге соединяется через **IPX/GRX**
  с DEA партнёра. Требования — GSMA (в т.ч. FS.19 по безопасности сигнализации Diameter).
- **SLF** (Subscriber Location Function) — если HSS несколько: CSCF по интерфейсу **Dx** спрашивает SLF,
  какой HSS обслуживает абонента.
- **IWF** (Interworking Function) — перевод Diameter ↔ HTTP/2 (5G SBI), Diameter ↔ MAP.
- **SEPP** (5G) — защита межсетевого обмена N32 (HTTP/2); аналог DEA для 5G.
- Типовые топологии: прямые соединения (малые сети), hub через DRA (крупные), роуминг через IPX+DEA,
  мульти-вендорные сети (нормализация на DRA).

## 13.5 Ключевые интерфейсы: команды, AVP, потоки, проблемы

### 13.5.1 S6a/S6d — «паспортный стол» абонента

**Между:** MME/SGSN ↔ HSS (S6d — SGSN ↔ HSS). **Приложение:** 16777251.

| Команда | Код | Назначение |
|---|---|---|
| AIR / AIA | 318 | аутентификация: запрос/ответ векторов |
| ULR / ULA | 316 | регистрация: обновление местоположения, загрузка подписки |
| CLR / CLA | 317 | отмена местоположения |
| IDR / IDA | 319 | вставка/изменение данных подписки |
| DSR / DSA | 320 | удаление данных подписки |
| PUR / PUA | 321 | purge (абонент недоступен/отсоединён) |
| RSR / RSA | 322 | сброс (после рестарта HSS) |
| NOR / NOA | 323 | уведомления (например, для MT-SMS) |

Ключевые AVP (имена):

- **AIR**: User-Name (IMSI), Requested-EUTRAN-Authentication-Info (Number-Of-Requested-Vectors,
  Immediate-Response-Preferred, Re-synchronization-Info), Visited-PLMN-Id.
- **AIA**: Authentication-Info → E-UTRAN-Vector (RAND, XRES, AUTN, KASME), Result-Code.
- **ULR**: ULR-Flags (Single-Registration-Indication, S6a/S6d-Indicator), Visited-PLMN-Id, RAT-Type,
  Terminal-Information (IMEI), UE-SRVCC-Capability.
- **ULA**: Subscription-Data (MSISDN, STN-SR, APN-Configuration-Profile, AMBR, Subscriber-Status,
  RAT-ограничения, Regional-Subscription-Zone-Code), ULA-Flags.
- **IDR/DSR**: Subscription-Data, IDR-Flags; **PUR**: PUR-Flags; **CLR**: Cancellation-Type.

**Поток Attach:** Attach Request → MME AIR→AIA → MME ULR→ULA → Attach Accept.
**Если не работает:** абонент не регистрируется (Attach Reject), нет услуг; при рестарте HSS без RSR —
рассинхронизация. **Типовые коды:** 5001 USER_UNKNOWN, 5004 ROAMING_NOT_ALLOWED,
5420 UNKNOWN_EPS_SUBSCRIPTION, 5421 RAT_NOT_ALLOWED, 5422 EQUIPMENT_UNKNOWN, 5423 UNKNOWN_SERVING_NODE.
**Анализ:** `diameter.cmd.code==316/318`, `diameter.Result-Code`, Subscription-Data (какие APN/AMBR пришли),
таймеры и повторы.

### 13.5.2 S13/S13' — проверка устройства (EIR)

**Между:** MME ↔ EIR (S13), SGSN ↔ EIR (S13'). **Команды:** ECR/ECA (324).
**AVP:** Terminal-Information (IMEI), Equipment-Status (WHITELISTED / GREYLISTED / BLACKLISTED).
**Если не работает:** устройства не проверяются (риск фрода) либо массово блокируются.
**Анализ:** результат Equipment-Status в ECA; сверка IMEI.

### 13.5.3 Gx/Gxx — политика и QoS

**Между:** PGW ↔ PCRF (Gx), SGW ↔ PCRF (Gxx, PMIP). **Команды:** CCR/CCA (272), RAR/RAA (258).
**CC-Request-Type:** INITIAL(1) / UPDATE(2) / TERMINATION(3) / EVENT(4).

Ключевые AVP (имена): Bearer-Identifier, Bearer-Operation, Network-Request-Support,
Packet-Filter-Information / Flow-Information (Flow-Description, Flow-Direction), QoS-Information,
Default-EPS-Bearer-QoS, APN-Aggregate-Max-Bitrate-DL/UL, Guaranteed-Bitrate-DL/UL,
Allocation-Retention-Priority (Priority-Level, Pre-emption-Capability, Pre-emption-Vulnerability),
Charging-Rule-Install / Charging-Rule-Remove / Charging-Rule-Report, Event-Trigger, Revalidation-Time,
Session-Release-Cause, IP-CAN-Type, RAT-Type, 3GPP-User-Location-Info, User-Equipment-Info.

**Примеры:** установка default bearer при attach; создание dedicated bearer для VoLTE (QCI 1);
изменение правил по событию; разрыв по Session-Release-Cause.
**Если не работает:** политики не применяются (или услуга блокируется — зависит от fail-open/fail-closed).
**Анализ:** какие правила установлены (Charging-Rule-Install), какие Event-Trigger,
какой Result-Code, нет ли расхождения QoS с S1AP/S11.

### 13.5.4 Gy / Ro — онлайн-тарификация (Credit-Control)

**Между:** PCEF (PGW) ↔ OCS (Gy), IMS AS/MRFC ↔ OCS (Ro). **Приложение:** 4 (RFC 8506).
**Команды:** CCR/CCA (272).

Ключевые AVP (коды подтверждены IANA):

| AVP | Код | Смысл |
|---|---|---|
| Subscription-Id | 443 | идентификатор абонента (IMSI/MSISDN/SIP URI/NAI) |
| Subscription-Id-Data | 444 | значение идентификатора |
| Subscription-Id-Type | 450 | 0=E164, 1=IMSI, 2=SIP URI, 3=NAI |
| CC-Request-Type | 416 | 1=INITIAL, 2=UPDATE, 3=TERMINATION, 4=EVENT |
| CC-Request-Number | 415 | номер запроса в сессии |
| Multiple-Services-Credit-Control | 456 | контейнер квот по услугам |
| Rating-Group | 432 | тарифная группа |
| Requested-Service-Unit | 437 | запрос квоты |
| Used-Service-Unit | 446 | израсходовано |
| Granted-Service-Unit | 431 | выдано |
| Final-Unit-Indication | 430 | поведение при исчерпании |
| Final-Unit-Action | 449 | 0=TERMINATE, 1=REDIRECT, 2=RESTRICT_ACCESS |
| Validity-Time | 448 | срок действия квоты |
| Tariff-Time-Change | 451 | время смены тарифа |
| Service-Context-Id | 461 | контекст услуги |
| User-Equipment-Info | 458 | тип/значение (IMEI/IMEISV) |

**Поток (prepaid):** INITIAL (запрос квоты) → CCA (GSU) → периодические UPDATE (USU + новая квота)
→ TERMINATION (финальный отчёт). **Коды:** 4010/4011/4012 (см. выше).
**Если не работает:** по политике — либо услуга без тарификации, либо блокировка.
**Анализ:** CC-Request-Type/Number, GSU/USU, FUI, Result-Code; проверка failover (CC-Session-Failover).

### 13.5.5 Rx — авторизация медиа (VoLTE/VoNR)

**Между:** P-CSCF (AF) ↔ PCRF/PCF. **Приложение:** 16777236. **Команды:** AAR/AAA (265), RAR/RAA (258).

Ключевые AVP (имена): Media-Component-Description (Media-Component-Number, Media-Type,
Max-Requested-Bandwidth-UL/DL, Flow-Description, Flow-Number, Flow-Usage), AF-Charging-Identifier,
Specific-Action, Service-Info-Status, Priority-Sharing-Indicator, SIP-Forking-Indication.

**Поток VoLTE:** INVITE → P-CSCF AAR → PCRF создаёт PCC-правило → Gx → PGW → S11 Create Bearer →
S1AP E-RABSetup → bearer QCI 1. **Если не работает:** нет dedicated bearer — голос без гарантий
(«кваканье», обрывы). **Анализ:** медиа-компоненты и полосы в AAR, ответ PCRF, связка Rx→Gx→S11→S1AP.

### 13.5.6 Cx/Dx — регистрация и аутентификация IMS

**Между:** I/S-CSCF ↔ HSS (Cx), CSCF ↔ SLF (Dx). **Приложение:** 16777216.
**Команды:** UAR/UAA (283), SAR/SAA (284), LIR/LIA (285), MAR/MAA (286), RTR/RTA (287), PPR/PPA (288).

Ключевые AVP (имена): Public-Identity, Visited-Network-Identifier, User-Authorization-Type,
Server-Assignment-Type, User-Data (XML: профиль услуг, iFC), User-Data-Already-Available,
SIP-Number-Auth-Items, SIP-Auth-Data-Item (SIP-Authentication-Scheme, SIP-Authenticate, SIP-Authorization,
Confidentiality-Key, Integrity-Key), Charging-Information, Deregistration-Reason.

**Поток регистрации:** REGISTER → I-CSCF UAR→UAA → S-CSCF MAR→MAA (challenge) → REGISTER (ответ) →
SAR→SAA (профиль). **Если не работает:** IMS-регистрация невозможна — нет VoLTE.
**Типовые коды:** 5001 USER_UNKNOWN, 5002 IDENTITIES_DONT_MATCH, 5003 IDENTITY_NOT_REGISTERED,
5004 ROAMING_NOT_ALLOWED, 5005 IDENTITY_ALREADY_REGISTERED, 5007 IN_ASSIGNMENT_TYPE, 5008 TOO_MUCH_DATA.

### 13.5.7 Sh — данные для серверов услуг и T-ADS

**Между:** AS (TAS/SCC AS) ↔ HSS. **Приложение:** 16777217.
**Команды:** UDR/UDA (306), PUR/PUA (307), SNR/SNA (308), PNR/PNA (309).

Ключевые AVP: Data-Reference (какие данные: Repository-Data, IMS-Public-User-Identity, MSISDN и др.),
Service-Indication, Requested-Domain, Subs-Req-Type (Subscribe/Unsubscribe), User-Data.

**Применение:** T-ADS (куда доставлять вызов — IMS или CS), данные для услуг (переадресации и т.д.).
**Если не работает:** входящий вызов может уйти не в тот домен или услуги не применяются.
**Анализ:** Data-Reference и User-Data в ответе, ошибки подписки.

### 13.5.8 Sy / Sd / Np — лимиты, DPI, перегрузка

- **Sy** (PCRF ↔ OCS, 16777302): SLR/SLA, SNR/SNA — лимиты расходов (spending limits), уведомления.
- **Sd** (TDF ↔ PCRF, 16777303): TSR/TSA — DPI-детект приложений, тарифы по приложениям.
- **Np** (RCAF ↔ PCRF, 16777342): информация о перегрузке пользовательской плоскости.
**Если не работает:** не применяются лимиты/тарифы по приложениям/политики перегрузки.

### 13.5.9 Sv — SRVCC (перевод вызова в CS)

**Между:** MME ↔ MSC Server (enhanced for SRVCC). **Команды:** SRVCC PS to CS Request/Response,
Complete Notification/Acknowledge, Cancel Notification/Response. **AVP:** STN-SR, C-MSISDN,
контейнеры source/target-to-source. **Application-Id:** по TS 29.280 (в IANA AAA записи нет).
**Если не работает:** VoLTE-вызов обрывается при уходе из LTE. **Анализ:** связка S1AP (Handover Required
с SRVCC) → Sv → Mw/I2 в IMS.

### 13.5.10 SWm / SWx / S6b — Wi-Fi calling (non-3GPP доступ)

- **SWm** (ePDG ↔ AAA, 16777264): EAP-аутентификация UE.
- **SWx** (AAA ↔ HSS, 16777265): SAR/SAA, MAR/MAA, RTR/RTA, PPR/PPA — подписка и аутентификация.
- **S6b** (PGW ↔ AAA, 16777272): RAR/RAA, ASR/ASA — авторизация сессии и QoS.
**Если не работает:** нет Wi-Fi calling, не устанавливаются сессии через ePDG.

### 13.5.11 Прочие интерфейсы (кратко)

| Интерфейс | App-Id | Между | Назначение |
|---|---|---|---|
| S6c | 16777312 | SMSC ↔ HSS | маршрутизация SMS (SRR/SRA) |
| SGd | 16777313 | SMSC ↔ SGSN/MME | доставка SMS через PS |
| S6m | 16777310 | SCEF ↔ HSS | IoT-подписка |
| S6t | 16777345 | SCEF ↔ HSS | IoT-услуги, мониторинг |
| T6a/T6b | 16777346 | SCEF ↔ MME/SGSN | доставка данных IoT |
| Tsp | 16777309 | SCEF/MTC-IWF ↔? | триггеры устройств |
| SLg | 16777255 | GMLC ↔ MME | определение местоположения |
| SLh | 16777291 | GMLC ↔ HSS | маршрутизация LCS |
| S9 | 16777267 | PCRF ↔ BPCF | политики фиксированного доступа |
| S15 | 16777318 | PCRF ↔? | политики (TS 29.212) |
| PC6/PC7 | 16777340 | ProSe | связь устройств напрямую |

## 13.6 Диагностика Diameter: как искать проблемы

### Что смотреть в трассировке

Полезные поля/фильтры Wireshark:

- `diameter` — весь Diameter;
- `diameter.cmd.code == 318` — только AIR; `== 316` — ULR; `== 272` — CCR;
- `diameter.Result-Code` — коды результата;
- `diameter.Experimental-Result-Code` — коды 3GPP (если код в Experimental-Result);
- `diameter.Session-Id` — привязка сессии;
- `diameter.CC-Request-Type` — тип запроса Gy/Gx;
- `diameter.Subscription-Id-Data` — IMSI/MSISDN;
- `diameter.Origin-Host` / `diameter.Destination-Host` — от кого/кому.

Примеры tshark:

```
tshark -r trace.pcap -Y "diameter.cmd.code==318"
tshark -r trace.pcap -Y "diameter.Result-Code"
tshark -r trace.pcap -Y "diameter.CC-Request-Type==1"
```

### Пошаговый алгоритм

1. **Соединение**: есть ли CER/CEA, согласованы ли приложения (app-id), нет ли DPR/DPA.
2. **Watchdog**: идут ли DWR/DWA; пропуски → peer недоступен/сеть.
3. **Транзакция**: найдите запрос (R) и ответ (E/Result-Code) по Hop-by-Hop Id.
4. **Код**: успех 2001 или ошибка; определите категорию (3xxx/4xxx/5xxx) и приложение.
5. **Прикладная логика**: даже при 2001 услуга может не работать (нет нужных данных в ответе —
   например, нет APN в Subscription-Data, нет правил в Charging-Rule-Install).
6. **Корреляция**: Session-Id, IMSI, время (NTP), соседние интерфейсы (S1AP/NAS, GTP, SIP).

### Типовые проблемы и признаки

| Признак | Вероятная причина | Где смотреть |
|---|---|---|
| Нет CER/CEA | транспорт, TLS, нет общего приложения (5010), несовместимые версии | соединение, app-id |
| Пропали DWR/DWA | peer down, сеть, перегрузка | watchdog, логи DRA |
| 3002 UNABLE_TO_DELIVER / 3003 REALM_NOT_SERVED | маршрутизация realm/host, DRA-конфиг | DRA, Destination-Realm |
| 3005 LOOP_DETECTED | петля через агентов | Route-Record |
| 3004 TOO_BUSY / OC-OLR | перегрузка узла | OC-Supported-Features, логи |
| 5010 NO_COMMON_APPLICATION | несовпадение app-id на концах | CER/CEA |
| 5012 UNABLE_TO_COMPLY | отказ по внутренним причинам | логи узла |
| 5001 (S6a/Cx) | абонент неизвестен | HSS/UDM |
| 5420 UNKNOWN_EPS_SUBSCRIPTION | нет подписки EPS | HSS |
| 5421 RAT_NOT_ALLOWED | запрет технологии (RAT) | подписка |
| 5422 EQUIPMENT_UNKNOWN | устройство не в списках | EIR |
| 5423 UNKNOWN_SERVING_NODE | узел не найден | HSS |
| 4012 CREDIT_LIMIT_REACHED | исчерпан лимит/баланс | OCS, Gy |
| Повторы с T-флагом | таймаут, потеря ответа | Hop-by-Hop, ретрансмиссии |
| «Всё успешно, но не работает» | нет нужных данных/правил | тело ответа (AVP) |

### Кейсы

1. **Attach reject**: S6a ULR→ULA с 5420 → в NAS придёт EMM cause (например, 15). Проверяйте подписку.
2. **Нет интернета**: Gx не установил правила или Gy вернул 4012. Смотрите Charging-Rule-Install и GSU/USU.
3. **VoLTE не работает**: Cx 5001/5003 (нет IMS-профиля) или Rx/Gx не создали bearer QCI 1.
4. **Роуминг**: 5004 ROAMING_NOT_ALLOWED в S6a; проверяйте DEA/IPX и соглашения.
5. **SRVCC**: нет/неверный STN-SR, ошибка Sv; проверяйте цепочку S1AP→Sv→IMS.

## 13.7 Безопасность Diameter

- Транспорт: TLS/DTLS (5868), IPsec; в 3GPP — NDS/IP (TS 33.210/33.310).
- На границе: DEA с фильтрацией peer'ов, whitelisting, rate limiting, топология-скрытие.
- Роуминг: IPX/GRX, рекомендации GSMA (в т.ч. FS.19 по безопасности Diameter-сигнализации).
- Угрозы: подмена узла (spoofing), DoS/перегрузка, фаззинг AVPs, нелегитимные запросы подписки,
  мошенничество через сигнализацию.
- Практика: минимальные привилегии, контроль аномалий (всплески транзакций), логирование,
  регулярные проверки конфигурации DRA/DEA; в 5G — SEPP (N32) как аналог защиты на границе.

## 13.8 Производительность, надёжность, перегрузка

- **Таймеры**: watchdog (Tw ≈ 30 c), таймауты транзакций, повторы — по RFC 6733 и профилю.
- **Перегрузка**: RFC 7683 (OC-Supported-Features 621, OC-OLR 623, OC-Sequence-Number,
  OC-Reduction-Percentage), приоритеты DRMP (301, RFC 7944), Load AVP (650, RFC 8583).
- **Отказоустойчивость**: несколько peer'ов, Session-Binding/Session-Server-Failover,
  резервирование DRA, кластеры HSS/PCRF/OCS.
- **Планирование ёмкости**: attach storm (массовые AIR/ULR), пиковые CCR от Gy/Gx,
  задержки на DRA, буферизация при перегрузке, приоритезация сигнализации.
- **State**: сессии в OCS/PCRF, sticky-привязка на DRA; при сбое — согласованная очистка сессий.

## 13.9 Diameter в 5G: что меняется

- 5GC использует **SBA**: HTTP/2 + JSON, сервисные интерфейсы N8/N10/N11/N40 и др.;
  функции: UDM, PCF, CHF, NRF.
- Diameter остаётся:
  - в EPS (S6a, Gx, Gy, Rx...);
  - в IMS (Cx, Sh, Rx, Ro);
  - на стыках interworking (IWF: Diameter ↔ HTTP/2);
  - для 2G/3G (S6d, Gr через MAP-шлюзы).
- Роуминг 5G — SEPP/N32; для interworking с EPS — N26 (GTPv2-C) и функции согласования.
- Тарификация: Gy/Ro → N40/CHF (конвергентная тарификация).
- Практический вывод: специалисту по Core нужно знать и Diameter, и SBI; миграция растянута на годы.

## 13.10 Сравнения

| Критерий | RADIUS | Diameter | HTTP/2 SBI (5G) |
|---|---|---|---|
| Транспорт | UDP (в основном) | TCP/SCTP | TCP/TLS |
| Надёжность | слабая | высокая (retransmit, failover) | высокая |
| Агенты | ограниченно | relay/proxy/redirect | SCP/NRF |
| Расширяемость | средняя | высокая (AVP, приложения) | высокая (JSON) |
| Типичное применение | broadband AAA | мобильные Core/IMS | 5GC |

| | Diameter | GTP | MAP (SS7) |
|---|---|---|---|
| Назначение | AAA/политики/тарификация | туннели и сессии | мобильность 2G/3G |
| Модель | запрос-ответ, peer-to-peer | процедуры/туннели | операции TCAP |
| Примеры | S6a, Gx, Gy, Rx | S11, S5/S8, S1-U | C/D, Gr |

## 13.11 Мини-словарь

- **AVP** — атрибут-значение Diameter.
- **CER/CEA** — обмен возможностями; **DWR/DWA** — watchdog; **DPR/DPA** — разрыв.
- **DRA/DEA** — маршрутизатор Diameter внутри сети / на границе.
- **Realm / Host** — домен и конкретный узел маршрутизации.
- **Hop-by-Hop / End-to-End Id** — идентификаторы транзакции на участке и сквозной.
- **Session-Id** — идентификатор сессии (тарификации, политики и т.д.).
- **CC-Request-Type** — тип запроса кредит-контроля.
- **MSCC / RSU / USU / GSU** — контейнер квот / запрошено / использовано / выдано.
- **FUI** — действие при исчерпании квоты.
- **Experimental-Result** — «вендорный» код результата.
- **OC-OLR** — индикатор перегрузки Diameter.
- **NDS/IP** — домен защищённой IP-сигнализации 3GPP.

## 13.12 Вопросы для самопроверки

1. Почему Diameter работает поверх TCP/SCTP, а не UDP?
2. Что проверяется в CER/CEA и что будет при отсутствии общего приложения?
3. Чем Hop-by-Hop Id отличается от End-to-End Id?
4. Что означает флаг T в заголовке?
5. Чем Relay отличается от Proxy?
6. Назовите app-id для S6a, Gx, Gy, Rx, Cx, Sh.
7. Какие команды у S6a и какие коды означают «абонент неизвестен»?
8. Как устроен жизненный цикл Gy-сессии?
9. Что означает код 4012 и что происходит дальше?
10. Как связаны Rx, Gx и S11/S1AP при установке VoLTE?
11. Зачем нужен DRA и что он умеет?
12. Какие интерфейсы Diameter заменяются в 5G и чем?
13. Что смотреть в трассировке, если CER/CEA прошли, а услуга не работает?
14. Как обнаружить петлю маршрутизации Diameter?
15. Чем 3GPP-коды отличаются от базовых и где они передаются?

## 13.13 Источники и спецификации

- RFC 6733 — Diameter Base Protocol
- RFC 8506 — Diameter Credit-Control (заменяет RFC 4006)
- RFC 4072 — Diameter EAP; RFC 4740 — Diameter SIP
- RFC 7683 — Diameter Overload; RFC 7944 — DRMP; RFC 8583 — Load
- IANA AAA Parameters (application-id, команды, AVP, Result-Code): https://www.iana.org/assignments/aaa-parameters/
- TS 29.272 — S6a/S6d/S13; TS 29.212 — Gx/Gxx/Sd; TS 29.214 — Rx; TS 29.219 — Sy
- TS 29.228/29.229 — Cx/Dx; TS 29.328/29.329 — Sh
- TS 29.273 — SWm/SWx/S6b; TS 29.280 — Sv
- TS 29.172 — SLg; TS 29.173 — SLh; TS 29.336 — S6m/S6t; TS 29.338 — S6c/SGd; TS 29.128 — T6a/T6b
- TS 32.299 — тарификационные AVP; TS 33.210/33.310 — NDS/IP
- GSMA FS.19 — безопасность Diameter-сигнализации
