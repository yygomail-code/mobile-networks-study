# 36. Траблшутинг интерфейсов LTE/PS/CS — подробный конспект

> Кому: инженерам эксплуатации и Core, разбирающим отказы по интерфейсам;
> для подготовки к защите. Методика, интерфейсы, коды ошибок, фильтры, кейсы, вопросы.
>
> Связанные файлы: `09_Interfejsy_LTE_PS_CS_troubleshooting.md` (краткий), `21_PS_Core_detalny_konspekt.md`,
> `24_RAN_optimizaciya_detalny_konspekt.md`, `27_CSFB_eSRVCC_Handover_detalny_konspekt.md`,
> `30_IMS_detalny_konspekt.md`, `33_Filtraciya_detalny_konspekt.md`.

## 36.1 Методика: как искать причину

Принципы:

1. **Один симптом — один первый слой.** Не менять несколько гипотез одновременно.
2. **Сначала данные, потом действия.** Собрать: счётчики, трейсы, конфиг, логи — и только потом менять.
3. **Идти по пути вызова:** UE → радио → транспорт → Core → сервис. На каждом стыке смотреть
   «что вошло / что вышло».
4. **Найти первое расхождение** (где ожидаемое поведение впервые нарушено) — это и есть точка отказа.
5. **Классифицировать**: сеть/конфигурация/нагрузка/софт/железо/внешняя система.
6. **Проверить гипотезу минимальным тестом** и зафиксировать результат.

Инструменты:

- трассировки по IMSI/GUTI на MME/AMF, SGW/UPF, eNB/gNB, MSC;
- Wireshark/tshark (см. 36.7), счётчики (TS 32.425/28.552), KPI (TS 32.450);
- конфигурация: CM-данные узлов, DNS, маршрутизация, лицензии.

## 36.2 Карта интерфейсов и протоколов

| Интерфейс | Между кем | Протокол | Транспорт/порт |
|---|---|---|---|
| Uu | UE ↔ eNB/gNB | RRC / NAS | радио |
| S1-MME | eNB ↔ MME | S1AP | SCTP 36412 |
| S1-U | eNB ↔ SGW | GTP-U | UDP 2152 |
| X2 | eNB ↔ eNB | X2AP | SCTP 36422 |
| S11 | MME ↔ SGW | GTPv2-C | UDP 2123 |
| S5/S8 | SGW ↔ PGW | GTPv2-C/U | UDP 2123/2152 |
| S6a | MME ↔ HSS | Diameter | SCTP 3868 |
| Gx | PGW ↔ PCRF | Diameter | SCTP 3868 |
| Gy | PGW ↔ OCS | Diameter | SCTP 3868 |
| SGi | PGW ↔ внешние сети | IP | — |
| SGs | MME ↔ MSC | SGsAP | SCTP 29118 |
| Sv | MME ↔ MSC | Diameter | SCTP 3868 |
| S3/S4 | MME/SGW ↔ SGSN | GTPv2-C | UDP 2123 |
| Gb | BSS ↔ SGSN | BSSGP | Frame Relay/IP |
| Iu-PS | RNC ↔ SGSN | RANAP | SCTP 25471 |
| N1/N2 | UE ↔ AMF / gNB ↔ AMF | NAS / NGAP | SCTP 38412 |
| N3 | gNB ↔ UPF | GTP-U | UDP 2152 |
| N4 | SMF ↔ UPF | PFCP | UDP 8805 |
| N26 | AMF ↔ MME | GTPv2-C | UDP 2123 |
| Xn | gNB ↔ gNB | XnAP | SCTP 38422 |

## 36.3 LTE/EPC: по интерфейсам

**Uu (RRC/NAS):** проверить RRC-состояние, причины отказов (reject cause), NAS-сообщения
(Attach/TAU/Service Request), сообщения о релизе. Первое, что смотреть при «не подключается».

**S1-MME (S1AP):** ключевые сообщения — Initial Context Setup, E-RAB Setup, Paging, Handover;
**Cause** — главный источник: «radio resources not available» (RAN/admission),
«transport resources unavailable» (транспорт), «mme overload», «no user plane».
Проверить: SCTP-ассоциации, пулы MME, лицензии, overload-статус.

**S1-U (GTP-U):** потери/джиттер данных; проверить туннели (TEID), MTU, DSCP, потери на транспорте,
ретрансмиссии. Симптом: «сигнализация есть, данных нет».

**X2 (X2AP):** HO и ANR; ошибки подготовки HO (X2 не сконфигурирован, перегруз, несовпадение
параметров); проверить соседство, X2-транспорт.

**S11/S5/S8 (GTPv2-C):** Create/Modify/Delete Bearer, причины отказа (Cause IE):
«context not found», «no resources», «APN denied». Проверить: сессии на SGW/PGW, лицензии,
IP-пулы, DNS-выбор.

**S6a (Diameter):** AIR/AIA, ULR/ULA; ошибки: 5001 (USER_UNKNOWN), 5004 (ROAMING_NOT_ALLOWED),
3002 (unable to deliver), таймауты. Проверить: маршрут к HSS, DRA, профиль абонента.

**Gx/Gy:** правила PCC (Charging-Rule-Install), лимиты (кредиты); ошибки: 5012 (no credit),
3002 (delivery). Проверить PCRF/OCS, Rating-Group, сценарии FUP.

**SGs (SGsAP):** combined attach, paging CS, SMS; ошибки: ассоциация потеряна, TAC↔LAC mismatch.
Проверить MSC, mapping зон.

**Sv (Diameter):** SRVCC; ошибки: нет ресурсов CS, таймауты; проверить MSC/ATCF.

**S3/S4, Gb/Iu-PS:** interworking с 2G/3G: PS HO, контексты, причины отказа; проверить SGSN.

## 36.4 5G: по интерфейсам

- **N1/N2 (NAS/NGAP):** Registration, PDU Session Resource Setup/Modify; reject causes
  (например, NSSAI, «slice not available»); проверить AMF, NSSF, слайсы.
- **N3 (GTP-U):** данные; QFI-туннели; проверить UPF, потери/MTU.
- **N4 (PFCP):** сессии UPF (PDR/FAR/QER/URR); ошибки: rule not found, ресурсы; проверить SMF↔UPF.
- **N7/N8/N10/N11:** SBI (HTTP/2) — коды ответов (4xx/5xx), таймауты; проверить NRF, TLS.
- **N26:** interworking EPS↔5GS; отсутствие/ошибки — срывы EPS fallback и переходов.
- **Xn:** HO/NR-DC; проверить транспорт и параметры.

## 36.5 IMS (кратко, детали — файл 30)

Gm (SIP): коды 4xx/5xx/6xx; Cx (Diameter): регистрация/профиль; Rx: QoS для медиа;
RTP: loss/jitter. Первое при «нет голоса» — регистрация IMS и SIP-коды.

## 36.6 Кейсы-разборы

**Кейс 1. UE не подключается (Attach fail).**
Логика: RRC reject? → S1AP Initial Context Setup fail (cause?) → S6a AIR (ошибка?) →
S11 Create Session (cause?) → E-RAB (admission?). Находим первое расхождение.

**Кейс 2. Данные есть, скорости нет.**
GTP-U: потери/MTU? QoS: AMBR/QCI (файл 15/24)? Транспорт: перегруз? RAN: PRB/BLER?
Смотреть обе стороны (DL/UL) и профиль абонента.

**Кейс 3. MT-вызов CSFB не доходит.**
SGs-ассоциация → paging CS (SGsAP) → TAC↔LAC mapping → перевод. Чаще всего — mapping/ассоциация.

**Кейс 4. VoLTE: вызов есть, голоса нет (one-way).**
SIP/SDP (стороны медиа) → Rx/Gx (QCI 1) → RTP-путь/NAT/firewall → SRTP. Проверить SDP и медиа-поток.

**Кейс 5. Handover падает на границе пулов.**
S10/S1, X2: причины, пулы, веса, соседи; проверить пары узлов и время.

**Кейс 6. PDU-сессия 5G не устанавливается.**
N2 reject (NSSAI/AMF) → N4 (PFCP) → UPF/DNN; проверить слайс, DNN, ресурсы.

**Кейс 7. Роуминг: голос есть, данные нет.**
S6a/S8 через IPX/DRA, APN-OI-Replacement, фильтры на границе; проверить маршруты и политики.

**Кейс 8. После смены конфигурации деградация.**
Сравнить «до/после», найти изменённый параметр, откатить (по плану), подтвердить KPI.

## 36.7 Инструменты: фильтры Wireshark

| Интерфейс | Фильтр |
|---|---|
| S1AP | `s1ap` |
| NGAP | `ngap` |
| GTPv2-C | `gtpv2` |
| GTP-U | `gtp` |
| PFCP | `pfcp` |
| Diameter | `diameter` |
| SGsAP | `sgsap` |
| RANAP/BSSAP | `ranap` / `bssap` |
| NAS EPS/5GS | `nas-eps` / `nas-5gs` |
| SIP/RTP | `sip` / `rtp` |
| X2AP | `x2ap` |
| SCTP | `sctp` |

Полезно: `gtpv2.cause`, `s1ap.cause`, `ngap.cause`, `diameter.Result-Code`, `sip.Status-Code`.

## 36.8 Вопросы для защиты

**1. С чего начинать разбор «не работает»?**
С точного симптома и пути вызова; собрать данные, найти первое расхождение по стыкам, затем
менять один фактор. Не начинать с изменений.

**2. Где смотреть причину отказа E-RAB?**
В S1AP cause (radio/transport/overload) + E-RAB QoS (admission) + транспортные счётчики.

**3. Как отличить проблему Core от проблемы радио по S1AP?**
По cause: «radio resources not available» — RAN/admission; «transport resources unavailable» —
транспорт; «mme overload» — Core.

**4. Какие ошибки Diameter типовые на S6a?**
5001 (USER_UNKNOWN), 5004 (ROAMING_NOT_ALLOWED), 3002 (delivery), таймауты; смотреть маршрут
и профиль.

**5. Как проверить GTP-U туннель?**
По TEID/адресам, счётчики потерь, MTU; в Wireshark `gtp` + тест-трафик; сверить обе стороны.

**6. Что смотреть при срыве CSFB?**
SGs-ассоциация, SGsAP paging, TAC↔LAC, redirect/PS HO, покрытие 2G/3G (файл 27).

**7. Как понять, что дело в QoS, а не в радио?**
По S1AP/NGAP QoS-параметрам (QCI/5QI, GBR, ARP) и правилам Gx/N7; если правила верные,
а PRB/BLER высокие — радио; если правила неверные — Core/QoS.

**8. Что проверять при сбое PDU-сессии?**
N2 reject cause (NSSAI/слайс), N4/PFCP (правила UPF), DNN/ресурсы; проверить AMF/SMF/UPF.

**9. Как искать «односторонний звук»?**
SDP (адреса/порты), RTP-путь, NAT/firewall, SRTP; проверить Rx/Gx (QCI 1) и медиа-счётчики.

**10. Что делать после изменения конфигурации?**
Сравнить KPI до/после, проверить guard-метрики, при ухудшении — откат; зафиксировать результат.

## 36.9 Спецификации

- S1AP/X2AP: TS 36.413/36.423; NGAP/XnAP: TS 38.413/38.423
- GTP: TS 29.274 (C), TS 29.281 (U); PFCP: TS 29.244
- Diameter: TS 29.272 (S6a), TS 29.212 (Gx), TS 32.299 (Gy), TS 29.280 (Sv)
- SGs: TS 29.118; RANAP: TS 25.413; BSSGP: TS 48.018
- NAS: TS 24.301 (EPS), TS 24.501 (5GS); RRC: TS 36.331, TS 38.331
- KPI/PM: TS 32.425, TS 32.450, TS 28.552; Trace: TS 32.422
