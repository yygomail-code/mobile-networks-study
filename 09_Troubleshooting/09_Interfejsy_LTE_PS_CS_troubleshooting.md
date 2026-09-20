# 9. Интерфейсы LTE, PS и CS — конспект для траблшутинга

> Цель: карта интерфейсов LTE/EPC, PS и CS 2G/3G, IMS; акцент — на интерфейсах, сообщениях
> и полях, которые чаще всего используются при анализе проблем (регистрация, сессии, голос, HO, тарификация).

## 9.1 Как пользоваться конспектом

- Раздел 9.7 — ранжированный список «что смотреть первым».
- Раздел 9.8 — поля и коды, которые ищут в сообщениях.
- Раздел 9.9 — корреляция идентификаторов (главный навык анализа).
- Разделы 9.10–9.11 — типовые кейсы и чек-лист по симптомам.
- Раздел 9.12 — инструменты и фильтры.

Методика: симптом → слой (Radio / Transport / Core / IMS) → интерфейс → сообщение → поле/cause → первое отклонение.

## 9.2 Карта интерфейсов

### LTE / EPC

```
                     IMS (Gm, Mw, ISC, Cx, Sh, Rx, Ro)
                                   |
UE --Uu-- eNB --S1-MME-- MME --S6a-- HSS
           |  \--S1-U-- SGW --S5/S8-- PGW --SGi-- PDN/Internet
           X2           |             |
           |            S4            Gx/Gy/Sd
         (eNB)          |             |
                      SGSN          PCRF/OCS/TDF
```

Дополнительно: S10 (MME↔MME), S3 (MME↔SGSN), S16 (SGSN↔SGSN), SGs (MME↔MSC),
Sv (MME↔MSC-S, SRVCC), S13 (MME↔EIR), Gxx (PCRF↔SGW), Rx (P-CSCF↔PCRF),
Sy (PCRF↔OCS), Sp (PCRF↔SPR), S6b/SWx (AAA), S8 (роуминг).

### 2G/3G PS

```
MS --Um-- BSS --Gb-- SGSN --Gn/Gp-- GGSN --Gi-- PDN
UE --Uu-- RNS --Iu-PS-- SGSN
SGSN --Gr-- HLR/HSS;  SGSN --Gf-- EIR;  SGSN --Gd-- SMS;  SGSN --Ga-- CG;  SGSN --Gs-- MSC
```

### 2G/3G CS

```
MS --Um-- BSS --A-- MSC --E-- MSC
UE --Uu-- RNS --Iu-CS-- MSC
MSC --C-- HLR;  MSC --D-- VLR;  MSC --F-- EIR;  MSC --Gs-- SGSN
MSC-S --Mc-- MGW;  MSC-S --Nc-- MSC-S
```

### IMS

```
UE --Gm-- P-CSCF --Mw-- I-CSCF/S-CSCF --ISC-- AS (TAS/SCC AS)
CSCF --Cx-- HSS;  AS --Sh-- HSS;  P-CSCF --Rx-- PCRF;  AS/MRF --Ro/Rf-- OCS/CDF
MGCF --Mn-- IM-MGW;  BGCF/MGCF --Mg/Mj/Mi/Mk-- CSCF;  ATCF --I2-- SCC AS
```

## 9.3 LTE интерфейсы (детально)

| Интерфейс | Между | Протокол/стек | Что важно при траблшутинге |
|---|---|---|---|
| Uu | UE ↔ eNB | RRC (SRB0/1/2), NAS (EMM/ESM), DRB | RRC Setup/Reject (waitTime), RRC Release cause, RLF, Reestablishment, измерения (A2/A3/B1/B2), качество радио |
| X2 | eNB ↔ eNB | X2AP (SCTP), GTP-U (X2-U) | Handover Request/Response, cause, SN Status Transfer, UE Context Release; ANR/MLB (Resource Status, Configuration Update) |
| S1-MME | eNB ↔ MME | S1AP (SCTP) | InitialUEMessage, InitialContextSetup, E-RABSetup/Modify/Release, Paging, Handover, UEContextRelease; cause; ID UE |
| S1-U | eNB ↔ SGW | GTP-U (UDP) | TEID, Echo Request/Response, sequence, потери/джиттер; путь данных |
| S11 | MME ↔ SGW | GTPv2-C (UDP) | Create/Modify/Delete Session, Create/Update/Delete Bearer, DDN, Release Access Bearers; cause; TEID; IMSI |
| S10 | MME ↔ MME | GTPv2-C | Context Transfer, Forward Relocation (inter-MME HO) |
| S5/S8 | SGW ↔ PGW | GTPv2-C/U (или PMIPv6) | Create Session, cause (APN, ресурсы), QoS; GTP-U путь; S8 — роуминг |
| SGi | PGW ↔ PDN | IP | DNS, DHCP, NAT, firewall, MTU, ping/traceroute |
| S6a | MME ↔ HSS | Diameter (SCTP) | AIR/AIA, ULR/ULA, CLR, IDR, DSR, PUR, RSR, NOR; Result-Code; Subscription-Data |
| S13 | MME ↔ EIR | Diameter | EC-Request/Answer; статус IMEI (white/grey/black) |
| SGs | MME ↔ MSC | SGsAP (SCTP) | LU, Service Request, Paging, Unitdata; SGs-ассоциация; CSFB/SMS |
| Gx | PGW ↔ PCRF | Diameter CCR/CCA | PCC-правила, QoS, AMBR, Event-Trigger |
| Gy | PGW ↔ OCS | Diameter CCR/CCA | квоты (GSU/USU/RSU), FUI, 4012 (credit limit reached) |
| Gxx | SGW ↔ PCRF | Diameter | QoS для PMIP (S5/S8) |
| Rx | P-CSCF ↔ PCRF | Diameter AAR/AAA | медиа-компоненты, QoS; VoLTE |
| Sd | TDF ↔ PCRF | Diameter | DPI-детект приложений |
| Sy | PCRF ↔ OCS | Diameter | spending limits (SLR/SLA) |
| S6b/SWx | PGW/AAA ↔ HSS | Diameter | non-3GPP доступ (ePDG/Wi-Fi calling) |
| S3/S4/S16 | MME↔SGSN, SGSN↔SGW, SGSN↔SGSN | GTPv2-C | inter-RAT PS HO |
| N26 | AMF ↔ MME | GTPv2-C | interworking EPS↔5GS (контекст) |
| N2/N3 | gNB↔AMF / gNB↔UPF | NGAP / GTP-U | 5G-эквиваленты S1-MME / S1-U |

## 9.4 PS 2G/3G интерфейсы

| Интерфейс | Между | Протокол | Что важно при траблшутинге |
|---|---|---|---|
| Gb | BSS ↔ SGSN | BSSGP/NS | attach, PDP-контекст, paging; BSSGP cause, P-TMSI, flow control |
| Iu-PS | RNS ↔ SGSN | RANAP + GTP-U | RAB Assignment, PDP, direct tunnel; RANAP cause |
| Gn | SGSN ↔ GGSN (intra-PLMN) | GTPv1-C/U | Create/Update/Delete PDP Context; cause (APN, ресурсы) |
| Gp | SGSN ↔ GGSN (inter-PLMN) | GTPv1-C/U | роуминг PS |
| Gi | GGSN ↔ PDN | IP | DNS, NAT, firewall |
| Gr | SGSN ↔ HLR/HSS | MAP (или S6d) | подписка, location |
| Gf | SGSN ↔ EIR | MAP | IMEI check |
| Gd | SGSN ↔ SMS | MAP | SMS через PS |
| Ga | SGSN/GGSN ↔ CG | GTP' | CDR |
| Gs | SGSN ↔ MSC | BSSAP+ | combined attach, CS-пейджинг через PS |
| S4/S16 | SGSN ↔ SGW / SGSN | GTPv2-C | interworking с EPC |
| S6d | SGSN ↔ HSS | Diameter | подписка в EPC |

## 9.5 CS интерфейсы

| Интерфейс | Между | Протокол | Что важно при траблшутинге |
|---|---|---|---|
| Um | MS ↔ BSS | RR/MM/CC/SS | CS-вызовы, LAU, paging; CC cause |
| A | BSS ↔ MSC | BSSAP (DTAP/BSSMAP), SS7/SIGTRAN | вызовы, HO, paging; DTAP/BSSMAP cause |
| Iu-CS | RNS ↔ MSC | RANAP + Iu UP | RAB Assignment, HO, paging; RANAP cause |
| E | MSC ↔ MSC | BICC/ISUP (SIP-I) | basic call, inter-MSC HO; ISUP cause |
| Gs | MSC ↔ SGSN | BSSAP+ | combined attach, CS-пейджинг |
| C | HLR ↔ GMSC | MAP | Send Routing Info |
| D | HLR ↔ VLR | MAP | Location Update, subscriber data |
| B | VLR ↔ MSC | внутренний | — |
| F | MSC ↔ EIR | MAP | IMEI check |
| Mc / Nc / Nb | MSC-S↔MGW / MSC-S↔MSC-S / MGW↔MGW | H.248 / BICC | разделение управления и медиа |
| SGs | MME ↔ MSC | SGsAP | CSFB/SMS (LTE) |
| Sv | MME ↔ MSC-S | Diameter | SRVCC |

## 9.6 IMS интерфейсы (для голоса)

| Интерфейс | Между | Протокол | Что важно при траблшутинге |
|---|---|---|---|
| Gm | UE ↔ P-CSCF | SIP | REGISTER, INVITE, ответы (403/404/408/480/486/500/503) |
| Cx | CSCF ↔ HSS | Diameter | UAR/SAR/MAR; выбор S-CSCF; Result-Code |
| Sh | AS ↔ HSS | Diameter | T-ADS-данные |
| ISC | S-CSCF ↔ AS | SIP | MMTel-триггеры |
| Rx | P-CSCF ↔ PCRF | Diameter | AAR/AAA, медиа-компоненты, QoS bearer'а |
| Ro/Rf | AS/MRF ↔ OCS/CDF | Diameter | онлайн/офлайн-тарификация |
| Mg/Mn/Mj/Mi/Mk | MGCF/MGW/BGCF | SIP/H.248 | выход в PSTN |
| I2 | ATCF ↔ SCC AS | SIP | eSRVCC |
| Ici/Izi | IBCF ↔ IBCF/TrGW | SIP/H.248 | межсетевые вызовы |
| S6a | MME ↔ HSS | Diameter | T-ADS, STN-SR (связь с IMS) |

## 9.7 Топ интерфейсов для анализа (ранжировано)

1. **S1-MME (S1AP)** — основной «журнал» доступа и мобильности: attach/service request, пейджинг, установка E-RAB, HO, release. Здесь видно большинство причин отказов UE в LTE. Смотреть: InitialUEMessage/InitialContextSetup, E-RABSetup, Paging, UEContextRelease, cause, ID UE.
2. **S11 (GTPv2-C)** — управление PDN-сессиями и bearer'ами (MME↔SGW): Create/Modify/Delete Session, Create Bearer, DDN. Здесь видно, почему нет данных или не создаётся bearer. Смотреть: cause, TEID, IMSI, EBI.
3. **S6a (Diameter)** — подписка и аутентификация: AIR/AIA, ULR/ULA. Причина attach reject: user unknown, subscription, RAT/roaming. Смотреть: Result-Code/Experimental-Result, Subscription-Data.
4. **Uu (RRC + NAS)** — радио и NAS: RRC Reject/Release, RLF, измерения; EMM/ESM cause. Требует расшифровки NAS, но даёт первопричину.
5. **S1-U (GTP-U)** — реальный путь данных: TEID, Echo, потери/джиттер. Проверка «данные вообще идут?».
6. **S5/S8 (GTPv2-C + GTP-U)** — сессия до PGW, роуминг: Create Session cause, APN, QoS.
7. **SGs (SGsAP)** — CSFB и SMS: LU, Service Request, Paging, Unitdata; состояние ассоциации.
8. **X2 (X2AP)** — HO и SON: Handover Request/Response, cause, SN Status; MLB/ANR.
9. **Gx/Gy (Diameter)** — политики и тарификация: PCC-правила, квоты; причины блокировок (4012 и др.).
10. **Rx (Diameter)** — VoLTE: AAR/AAA, медиа-компоненты, запуск dedicated bearer.
11. **SGi** — IP-слой: DNS, NAT, firewall, MTU; здесь часто «не наша проблема».
12. **Gm/Cx/ISC (SIP/Diameter)** — IMS-регистрация и вызовы: REGISTER/INVITE, MAR/SAR, триггеры.
13. **Gb/Iu-PS/Iu-CS (2G/3G)** — legacy PS/CS: PDP-контексты, CS-вызовы, HO; нужны при interworking и CSFB.

## 9.8 Поля и коды, которые ищут в сообщениях

### S1AP

- MME UE S1AP ID, eNB UE S1AP ID — корреляция одного UE.
- cause (категории: radioNetwork / transport / nas / protocol / misc).
- E-RAB ID, E-RAB Level QoS Parameters, Transport Layer Address, GTP-TEID.
- Paging: TAI List, UE Identity Index Value, CN Domain.
- Примеры cause: Radio Connection With UE Lost, Handover Failure In Target eNB Or Target Cell, TX2RELOCOverall Expiry, Transport Resource Unavailable, User Inactivity, CS Fallback triggered.

### NAS (EMM/ESM)

- **EMM cause (примеры)**: 2 IMSI unknown in HSS, 3 Illegal UE, 5 IMEI not accepted, 6 Illegal ME, 7 EPS services not allowed, 8 EPS/non-EPS not allowed, 11 PLMN not allowed, 12 Tracking area not allowed, 13 Roaming not allowed in this tracking area, 14 EPS services not allowed in this PLMN, 15 No suitable cells in tracking area, 16 MSC temporarily not reachable, 17 Network failure, 18 CS domain not available, 22 Congestion, 25 Not authorized for this CSG, 35 Requested service option not authorized, 39 CS service temporarily not available, 40 No EPS bearer context activated, 42 Severe network failure, 95–101/111 — protocol errors.
- **ESM cause (примеры)**: 8 Operator determined barring, 26 Insufficient resources, 27 Missing or unknown APN, 29 User access to APN denied, 30 Request rejected by SGW/PGW, 31 Request rejected unspecified, 33 Service option not subscribed, 36 Regular deactivation, 37 EPS QoS not accepted, 38 Network failure, 39 Reactivation requested, 49 Last PDN disconnection not allowed, 50/51 PDN type IPv4/IPv6 only allowed, 54 PDN connection does not exist, 55 Multiple PDN connections for a given APN not allowed, 65 Maximum number of EPS bearers reached, 66 Requested APN not supported in current RAT/PLMN.
- Также: EPS Bearer ID, PTI, APN, PDN address, TFT operation.

### GTPv2-C (S11 / S5/S8 / S10 / S3 / S16)

- Cause (примеры): 16 Request accepted, 64 Context Not Found, 65 Invalid Message Format, 68 Service Not Supported by recipient, 70 Mandatory IE Missing, 72 System Failure (полный список — TS 29.274).
- TEID (control/user), Sequence Number, IMSI, MSISDN, EPS Bearer ID (EBI), LBI, RAT Type, APN, PDN Type, ULI (TAI/ECGI), Charging Characteristics.
- Смотреть: ответы с cause != 16, ретрансмиссии (повтор sequence), таймауты.

### GTP-U (S1-U / S5/S8 / X2-U)

- TEID, Sequence Number, Echo Request/Response, N-PDU.
- Ошибки: нет ответа Echo, TEID mismatch, потери/джиттер, MTU (GTP-инкапсуляция добавляет ~36 байт для IPv4).

### Diameter (S6a / Gx / Gy / Rx / Sd / Sy)

- Result-Code / Experimental-Result: 2001 success; 5001 user unknown; 5004 roaming not allowed; 5420 unknown EPS subscription; 5421 RAT not allowed; 5422 equipment unknown; 5423 unknown serving node; Gy: 4010 end user service denied, 4011 credit control not applicable, 4012 credit limit reached.
- Session-Id, Subscription-Id (IMSI/MSISDN), Origin-Host/Realm, CC-Request-Type, Rating-Group, GSU/USU.

### SGsAP

- Cause, IMSI/TMSI, LAI, тип услуги; состояние SGs-ассоциации MME↔MSC.

### X2AP

- Handover cause (Handover Desirable for Radio Reasons, Time Critical Handover, Resource Optimisation Handover), Target Cell ID, UE History, SN Status Transfer, Resource Status (MLB).

### RRC

- RRC Connection Reject + waitTime; RRC Release cause; RLF; Reestablishment cause (reconfigurationFailure / handoverFailure / otherFailure); события измерений A2/A3/B1/B2.

### SIP

- Ответы: 401/407 (challenge — норма), 403, 404, 408, 480, 486, 487, 500, 503, 504; Call-ID, CSeq, Reason.

### CS (DTAP/ISUP)

- CC cause (примеры): 16 Normal Clearing, 17 User Busy, 18 No User Responding, 19 No Answer, 21 Call Rejected, 27 Destination Out of Order, 28 Invalid Number Format, 29 Facility Rejected, 31 Normal Unspecified, 34 No Circuit/Channel Available, 38 Network Out of Order, 41 Temporary Failure, 42 Switching Equipment Congestion, 47 Resource Unavailable, 102 Recovery on Timer Expiry.

## 9.9 Корреляция идентификаторов (главное в анализе)

| Идентификатор | Где встречается | Что связывает |
|---|---|---|
| IMSI | NAS, S1AP, S6a, S11, Gx, Gy, SGs, Gn/Gb | один абонент во всех протоколах |
| GUTI / S-TMSI | NAS, S1AP, Paging | временный ID UE |
| MME UE S1AP ID | S1AP | один UE на S1-MME |
| eNB UE S1AP ID | S1AP | один UE на eNB |
| TEID | GTP-C/GTP-U (S1-U, S5/S8, X2-U) | туннель |
| EPS Bearer ID / EBI | NAS, S1AP, GTPv2-C | bearer |
| LBI (Linked EPS Bearer ID) | GTPv2-C | связь dedicated/default bearer |
| Charging-ID / Charging Characteristics | GTPv2-C, Gx/Gy | тарификационная сессия |
| Session-Id | Diameter (S6a/Gx/Gy/Rx) | Diameter-сессия |
| Call-ID | SIP | вызов |
| P-TMSI / TMSI / LAI | Gb/Iu/A, SGs | 2G/3G и CS |

Практика: собрать pcap на S1, S11, S5/S8, S6a; найти один IMSI; построить «лестницу» сообщений по времени; найти первое сообщение с ошибкой (cause != success) — это и есть точка отказа. Обязательна синхронизация времени (NTP) между узлами.

## 9.10 Типовые сценарии (кейсы)

### Кейс 1. UE не регистрируется (attach reject)

- Интерфейсы: Uu → S1-MME → S6a → S11.
- Сообщения: RRC Setup, Attach Request, AIR/AIA, ULR/ULA, Attach Reject (EMM cause).
- Причины: 5001/5420 (подписка), EMM 11/12/13/14/15 (barring/roaming), EMM 22 (congestion), EMM 2/3/5/6 (identity/equipment), S1AP transport cause.

### Кейс 2. Attach OK, но нет интернета

- Интерфейсы: S11 → S5/S8 → S1-U → SGi → Gx/Gy.
- Сообщения: Create Session Response (cause), GTP-U Echo, DNS/NAT, PCC/квоты.
- Причины: ESM 27/29/30/31/33, GTP cause 64/68/72, Gy 4012, DNS/NAT/firewall, APN mismatch, отсутствие default bearer в RAN.

### Кейс 3. Нет VoLTE / вызов не устанавливается

- Интерфейсы: Gm → Cx → Rx → Gx → S11 → S1-MME.
- Сообщения: REGISTER, MAR/SAR, INVITE, AAR/AAA, CCR/CCA, Create Bearer, E-RABSetup.
- Причины: SIP 403/404/480/500, ошибки Rx, отсутствие PCC-правила, нет bearer'ов QCI 5/1, TFT.

### Кейс 4. CSFB не работает

- Интерфейсы: SGs → S1-MME → A/Iu-CS.
- Сообщения: LOCATION-UPDATE, Service Request, Paging, Extended Service Request, HO/redirect.
- Причины: нет SGs-ассоциации, SGs cause, MME не инициирует fallback, RAN не поддерживает HO, несовпадение LAI/TMSI.

### Кейс 5. Проблемы Handover

- Интерфейсы: X2 / S1-MME → S11.
- Сообщения: X2AP Handover Request/Response, S1AP Handover Required/Command/Notify, Path Switch, Modify Bearer.
- Причины: radioNetwork cause (HO failure in target, TX2RELOCoverall expiry), transport cause, отсутствие X2 (ANR/MLB), PCI confusion, слишком ранний/поздний HO (MRO).

### Кейс 6. SRVCC не срабатывает

- Интерфейсы: S1-MME → Sv → Mw/I2 (IMS).
- Сообщения: Handover Required (SRVCC), Sv PS to CS Request/Response, SIP INVITE (STN-SR).
- Причины: STN-SR отсутствует/неверный, ошибка Sv, MSC не поддерживает, ATCF недоступен, сессия не анкорена.

### Кейс 7. SMS не ходит

- Интерфейсы: SGs / Gm / NAS (5G).
- Сообщения: SGs Unitdata (SMS over SGs), SIP MESSAGE (SMSoIP), NAS (5G).
- Причины: нет SGs-ассоциации, ошибки IMS-регистрации, конфигурация SMS-центра.

### Кейс 8. Низкая скорость / потери

- Интерфейсы: S1-U / S5/S8 → Gx → RAN.
- Сообщения: GTP-U статистика (Echo, потери, джиттер), PCC-правила (AMBR/QCI).
- Причины: AMBR caps, перегрузка, packet loss на транспорте, неверный TEID/маршрут, деградация радио.

### Кейс 9. Роуминг

- Интерфейсы: S6a → S8 → IPX/GRX → Gy.
- Сообщения: ULR/ULA (5004), Create Session, CCR/CCA.
- Причины: запрет в HSS (5004), отсутствие соглашения, IPX-фильтрация, тарификационные ограничения.

### Кейс 10. 5G interworking (EPS↔5GS)

- Интерфейсы: N26 → N2/N3 → N1 → N11/N4.
- Сообщения: Forward Relocation Request/Response, NGAP Path Switch, NAS.
- Причины: нет N26 (разрыв сессий), несовместимость SMF+PGW-C, сбой context transfer.

## 9.11 Чек-лист по симптомам

| Симптом | Первые интерфейсы | Ключевые сообщения | Типовые причины |
|---|---|---|---|
| Нет регистрации | Uu, S1-MME, S6a | Attach Reject, AIR/AIA, ULR/ULA | EMM cause, Diameter codes |
| Нет данных | S11, S5/S8, S1-U, SGi | Create Session, Echo, DNS/ping | GTP cause, TEID, APN, Gy |
| Нет VoLTE | Gm, Cx, Rx, Gx, S11, S1-MME | REGISTER, INVITE, AAR, CCR, Create Bearer, E-RAB | SIP/Rx/Gx коды, QCI 5/1 |
| Нет CSFB | SGs, S1-MME | LU, Service Request, Paging | SGs cause, ассоциация |
| HO падает | X2 / S1-MME | Handover Request/Required | cause, отсутствие X2 |
| SRVCC падает | Sv, S1-MME, Gm | PS to CS Req/Resp | STN-SR, cause |
| SMS | SGs, Gm, NAS | Unitdata, MESSAGE | ассоциация, SMS-центр |
| Медленно | S1-U, Gx, RAN | GTP-U статистика | AMBR, потери, радио |
| Роуминг | S6a, S8, IPX | ULR/ULA, Create Session | 5004, соглашения |

## 9.12 Инструменты и фильтры

- PCAP: Wireshark / tshark. Основные фильтры:
  - `s1ap` — сигнализация S1;
  - `gtpv2` — S11/S5/S8/S10 (control);
  - `gtp` — GTP-U (data);
  - `diameter` — S6a/Gx/Gy/Rx/Sd;
  - `sgsap` — CSFB;
  - `x2ap` — HO/SON;
  - `nas-eps` — NAS (при расшифровке);
  - `lte-rrc` — RRC LTE;
  - `sip` — IMS.
- Примеры tshark:
  - `tshark -r trace.pcap -Y "s1ap"`
  - `tshark -r trace.pcap -Y "gtpv2.cause"`
  - `tshark -r trace.pcap -Y "diameter.Result-Code"`
  - `tshark -r trace.pcap -Y "sgsap"`
- Call trace / IMSI trace на MME, SGW/PGW, HSS, PCRF, OCS.
- KPI: attach success rate, session setup success, HO success, paging success, E-RAB setup success.
- Trace/MDT: TS 32.422, TS 37.320; OSS-счётчики и отчёты.
- Обязательно: единое время (NTP) на узлах и в pcap — иначе корреляция невозможна.

## 9.13 Вопросы для самопроверки

1. Какие интерфейсы смотрят первыми при attach reject?
2. Где искать причину «attach успешен, интернета нет»?
3. Какие интерфейсы задействованы в установке VoLTE bearer?
4. Что такое TEID и на каких интерфейсах он встречается?
5. Какие EMM cause указывают на проблемы с подпиской/оборудованием?
6. Как отличить проблему S1-MME от проблемы S1-U?
7. Какие сообщения X2AP смотреть при падении HO?
8. Как коррелировать сессии между S6a и Gx?
9. Что смотреть при проблемах CSFB?
10. Как проверить проблему с Gy (нулевой баланс)?
11. Чем S11 отличается от S5/S8?
12. Какие поля NAS указывают на проблему с APN?

## 9.14 Спецификации

- TS 23.401, TS 23.060 — архитектура EPS/GPRS
- TS 36.413, TS 36.423 — S1AP, X2AP
- TS 29.274, TS 29.281 — GTPv2-C, GTP-U
- TS 29.272 — S6a/S6d; TS 29.212/29.214 — Gx/Rx; TS 32.299 — Gy AVPs
- TS 24.301 — NAS EPS (EMM/ESM causes)
- TS 29.118 — SGsAP; TS 29.280 — Sv
- TS 23.228, TS 24.229 — IMS
- TS 23.501, TS 23.502 — 5G и N26
- TS 32.422, TS 37.320 — trace/MDT
