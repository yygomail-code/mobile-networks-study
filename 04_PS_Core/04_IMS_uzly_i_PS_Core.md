# 4. Узлы IMS и их связь с PS Core

> Цель: знать состав IMS, назначение узлов, интерфейсы и то, как IMS опирается на EPC/5GC
> (VoLTE/VoNR, PCC, T-ADS, SMS, interworking с CS).

## 4.1 Уровни IMS

- Транспортный уровень: IP-сеть, SBC/TrGW, DNS/ENUM.
- Уровень управления сессиями: CSCF (P/I/S), HSS, SLF.
- Уровень приложений: AS (TAS/MMTel, SCC AS, MRF).
- Шлюзы к CS/PSTN: BGCF, MGCF, IM-MGW.
- Граница с другими сетями: IBCF, TrGW, SEG.

## 4.2 Узлы

### P-CSCF (Proxy CSCF)

- Первая точка входа для UE (SIP outbound proxy), интерфейс Gm.
- Регистрация, маршрутизация SIP, сжатие, безопасность (IPsec/TLS).
- Взаимодействие с PCRF/PCF через Rx/N5 для авторизации bearer'ов.
- Может быть совмещён с ATCF (eSRVCC).

### I-CSCF (Interrogating CSCF)

- Точка входа из других сетей/для регистрации.
- Запрос к HSS (Cx: UAR/UAA) для выбора S-CSCF.
- Маршрутизация в S-CSCF.

### S-CSCF (Serving CSCF)

- Регистратор и центр управления сессией.
- Аутентификация (Cx: MAR/MAA), загрузка профиля (SAR/SAA).
- Триггеры к AS через ISC (Initial Filter Criteria).
- Маршрутизация вызовов, взаимодействие с BGCF/MGCF.

### HSS (Home Subscriber Server)

- Единая база абонента: CS/PS/IMS-профили.
- Интерфейсы: Cx (CSCF), Sh (AS), S6a (MME), S6d (SGSN), SLh (LCS), SWx (AAA).
- Хранит STN-SR, C-MSISDN, iFC, данные для T-ADS.

### SLF (Subscriber Location Function)

- Определяет, какой HSS обслуживает абонента (если HSS несколько).
- Интерфейс Dx (CSCF ↔ SLF).

### AS (Application Server)

- **TAS/MMTel**: телефония, доп. услуги (CFU, CW, конференции, T-ADS).
- **SCC AS**: централизация услуг и continuity (SRVCC, Service Continuity).
- Другие: voicemail, MRF-контроллер и т.д.

### MRF (MRFC + MRFP)

- MRFC — управление (SIP, Mr).
- MRFP — медиа-ресурсы: конференции, объявления, тоны (H.248, Mp).

### BGCF / MGCF / IM-MGW

- BGCF — выбор точки выхода в PSTN.
- MGCF — преобразование SIP ↔ ISUP/BICC; управляет IM-MGW по Mn.
- IM-MGW — медиашлюз TDM/IP.

### E-CSCF / LRF

- Экстренные вызовы (112/911): E-CSCF обрабатывает, LRF определяет местоположение.

### IBCF / TrGW

- Граница с другими IP-сетями (IPX): Ici (IBCF-IBCF), Izi (IBCF-TrGW).
- SEG — IPsec-шлюз для защиты межсетевого обмена.

### ATCF / ATGW (eSRVCC)

- ATCF — управление access transfer, интерфейс I2 к SCC AS.
- ATGW — анкоринг медиа; управляется ATCF.

## 4.3 Интерфейсы IMS (сводная таблица)

| Интерфейс | Между | Протокол | Назначение |
|---|---|---|---|
| Gm | UE ↔ P-CSCF | SIP | регистрация, вызовы |
| Mw | CSCF ↔ CSCF (и MSC-S ↔ ATCF) | SIP | маршрутизация |
| Mg | MGCF ↔ I-CSCF | SIP | приём вызовов из PSTN |
| Mi | S-CSCF ↔ BGCF | SIP | маршрутизация к BGCF |
| Mj | BGCF ↔ MGCF | SIP | выбор MGCF |
| Mk | BGCF ↔ BGCF | SIP | междоменный breakout |
| Mn | MGCF ↔ IM-MGW | H.248 | управление медиашлюзом |
| Mp | MRFC ↔ MRFP | H.248 | управление медиа-ресурсами |
| Mr | S-CSCF ↔ MRFC | SIP | запрос ресурсов |
| ISC | S-CSCF ↔ AS | SIP | сервисные триггеры |
| Cx | CSCF ↔ HSS | Diameter | регистрация, профиль |
| Dx | CSCF ↔ SLF | Diameter | поиск HSS |
| Sh | AS ↔ HSS | Diameter | данные абонента для AS |
| Ut | UE ↔ AS | XCAP/HTTP | настройка услуг |
| Rx | P-CSCF ↔ PCRF | Diameter | авторизация медиа/QoS |
| Rf/Ro | AS/MRF ↔ CDF/OCS | Diameter | тарификация |
| Ici/Izi | IBCF ↔ IBCF / TrGW | SIP/H.248 | межсетевое взаимодействие |
| I2 | ATCF ↔ SCC AS | SIP | eSRVCC |
| S6a | MME ↔ HSS | Diameter | мобильность + T-ADS |
| S6d | SGSN ↔ HSS | Diameter | мобильность 2G/3G |

## 4.4 Связь IMS с PS Core (EPC)

### 4.4.1 Подключение и регистрация

1. UE выполняет Attach и устанавливает PDN-соединение с APN/DNN = ims.
2. P-CSCF обнаруживается:
   - через PCO (Protocol Configuration Options) в NAS-сообщениях — наиболее частый способ;
   - через DHCP;
   - статически (редко).
3. Устанавливается default bearer с QCI=5 (IMS signalling) — через PCRF/Gx, если PCC включён.
4. UE выполняет IMS-регистрацию (SIP REGISTER) через P-CSCF → I-CSCF → S-CSCF; S-CSCF аутентифицирует через HSS (Cx).

### 4.4.2 Голосовой вызов VoLTE

1. INVITE с SDP (UE → P-CSCF).
2. P-CSCF → PCRF: Rx AAR (media components: audio/video, полосы, flow descriptions).
3. PCRF → PGW: Gx CCA Charging-Rule-Install (QCI=1, GBR, ARP, flow-фильтр).
4. PGW → SGW → MME: GTPv2-C Create Bearer Request.
5. MME → eNB: S1AP E-RAB Setup; eNB → UE: RRC + NAS (Activate Dedicated EPS Bearer, TFT).
6. SIP 183/180/200 OK; медиа идёт по dedicated bearer (QCI=1).
7. При завершении — bearer освобождается (Delete Bearer).

### 4.4.3 T-ADS (Terminating Access Domain Selection)

- TAS/SCC AS решает, куда доставлять входящий вызов: IMS или CS.
- Запрашивает HSS (Sh: UDR/UDA): «IMS voice over PS supported», «last known MME», T-ADS info.
- HSS получает эти данные от MME по S6a (при attach/TAU).

### 4.4.4 SMS/USSD

- SMS over IMS (SMSoIP) — SIP MESSAGE через IMS.
- SMS over SGs (CSFB) — через MME/MSC.
- SMS over NAS (5G) — через AMF.
- USSD — через CS или IMS (USSI).

### 4.4.5 Interworking с CS

- MGCF/IM-MGW — для вызовов в/из PSTN/CS.
- CSFB/SRVCC — для UE без VoLTE/при уходе из LTE.

## 4.5 IMS в 5G

- PDU-сессия с DNN=ims; 5QI=5 для сигнализации.
- PCF ↔ P-CSCF через N5 (вместо Rx); SMF ↔ UPF через N4; PCF ↔ SMF через N7.
- VoNR: голос как QoS flow 5QI=1 в NR.
- EPS fallback: gNB при запросе голоса переводит UE в LTE (HO/redirect), вызов идёт как VoLTE.
- SMS over NAS.
- Emergency: PDU-сессия для emergency, E-CSCF/LRF, 5QI=1.

## 4.6 Соответствие EPC ↔ 5GC для IMS

| Функция | EPC | 5GC |
|---|---|---|
| Политики | PCRF | PCF |
| Пользовательская плоскость | PGW | UPF |
| Управление сессией | SGW/MME | SMF |
| Данные абонента | HSS | UDM |
| Интерфейс к IMS | Rx (и Gx) | N5 (и N7) |
| Тарификация | OCS (Gy/Ro) | CHF (Nchf) |
| Сигнальный bearer | QCI=5 | 5QI=5 |
| Голосовой bearer | QCI=1 | 5QI=1 |

## 4.7 Вопросы для самопроверки

1. Перечислите CSCF и их роли.
2. Какие интерфейсы HSS используются в IMS и с кем?
3. Как UE находит P-CSCF?
4. Как устанавливается dedicated bearer для VoLTE (цепочка Rx→Gx→S11→S1AP)?
5. Что такое T-ADS и кто его выполняет?
6. Зачем нужны ATCF/ATGW?
7. Как передаётся SMS в LTE/5G?
8. Чем N5 отличается от Rx?
9. Какие узлы участвуют в вызове на PSTN?
10. Как реализуется экстренный вызов в IMS?

## 4.8 Спецификации

- TS 23.228 — IMS архитектура
- TS 24.229 — IMS SIP/процедуры
- TS 29.228/29.229 — Cx/Dx
- TS 29.328/29.329 — Sh
- TS 29.214 — Rx; TS 29.213 — PCC
- TS 23.167 — Emergency IMS
- TS 23.237/23.292 — Service Continuity / ICS
- TS 23.501/23.502 — 5G
- GSMA IR.92 — VoLTE; IR.94 — VoLTE video
- RFC 3261 — SIP
