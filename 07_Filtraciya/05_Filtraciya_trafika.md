# 5. Типы фильтрации трафика в мобильных сетях

> Цель: систематизировать виды фильтрации трафика — от TFT в UE до DPI, GTP/Gi firewall,
> барринга и правил 5G (URSP/ULCL), понимать, где какой фильтр применяется и зачем.

## 5.1 Классификация

1. По уровню: L2/L3, L4 (5-tuple), L7 (DPI), DNS/URL.
2. По точке: UE (TFT), RAN (barring), Core (PGW/UPF/TDF), граница (Gi/SGi firewall), IMS (SBC), роуминг (IPX/SEPP).
3. По цели: QoS/классификация, тарификация, безопасность, родительский контроль, регуляторика, anti-fraud.

## 5.2 L2–L4: ACL, 5-tuple, TFT

- 5-tuple: src/dst IP, src/dst port, protocol.
- SDF (Service Data Flow) — поток, описываемый фильтром.
- **TFT** (Traffic Flow Template) — набор UL-фильтров в UE: какой трафик в какой bearer. Устанавливается через NAS при активации dedicated bearer.
- В PGW/UPF — DL-фильтры SDF (из PCC-правил).
- Пример Flow-Description: `permit out 17 from 10.0.0.0/8 1000-2000 to any 5060` (RTP).

## 5.3 PCC/SDF-фильтры

- PCRF/PCF формирует PCC-правило: SDF-фильтр + QCI/5QI + GBR/MBR + ARP + charging.
- Gx/N7 передаёт правило в PCEF/SMF.
- Приоритет (precedence) определяет порядок применения.
- SDF-фильтр может включать: IP-адрес/маску, порт/диапазон, протокол, direction (UL/DL), DSCP/TOS, Flow Label (IPv6).
- Классификация трафика → bearer/QoS flow → тарификация (rating group).

## 5.4 DPI и TDF

- DPI (Deep Packet Inspection) — анализ L7: HTTP Host, SNI, сигнатуры, поведение.
- TDF (Traffic Detection Function) — узел с DPI; интерфейс Sd к PCRF (TSR/TSA).
- Применение: тарифы по приложениям, zero-rating, блокировка, parental control, anti-fraud, приоритизация.
- В 5G: DPI в UPF; PCF через N7/N5; AF influence (traffic influence) через NEF.

## 5.5 GTP-фильтрация и Gi/SGi firewall

- GTP-C (TS 29.274) и GTP-U (TS 29.281) — туннелирование.
- **GTP firewall**: валидация GTP-C (IMSI, TEID, message type whitelist, sequence), защита от spoofing/DoS, контроль роуминга.
- GSMA FS.11 (GTP-C security), IR.33 (GTP-U).
- **Gi/SGi firewall** (NDS/IP, TS 33.210/33.310): stateful inspection, NAT, DDoS-защита, anti-spoofing (IMSI↔IP), фильтрация портов, URL/DNS.
- Корреляция GTP-C/GTP-U (один и тот же абонент/туннель) — защита от инъекций.

## 5.6 DNS/URL-фильтрация и родительский контроль

- DNS-фильтрация: блокировка доменов на уровне ответов DNS (быстро, но обходится).
- URL-фильтрация: категории сайтов, чёрные/белые списки; обычно в DPI.
- Родительский контроль: профили в PCRF/OCS, расписания, категории.
- Защита от malware/phishing: блокировка известных доменов/IP.

## 5.7 FUP, zero-rating, throttling

- **FUP** (Fair Usage Policy): при превышении порога — снижение AMBR (через Gx/PCRF) или блокировка.
- **Zero-rating**: трафик не тарифицируется (Gy rating-group с нулевой ценой) + классификация DPI.
- Whitelist/blacklist услуг.
- **Throttling** (policing/shaping): token bucket в PGW/UPF; понижение приоритета/скорости.

## 5.8 Барринг и управление доступом

- ACB (Access Class Barring), EAB (Extended ACB для IoT), SSAC (Service Specific Access Control — блокирует VoLTE/CSFB).
- RRC Connection Reject с wait time; cell barring, forbidden PLMN, CSG.
- MPS (Multimedia Priority Service), emergency calls bypass barring.
- Применяется при перегрузке RAN или Core (сигнал от MME Overload Start).

## 5.9 Роуминг

- IPX/GRX firewall: фильтрация GTP/Diameter/SIP на границе.
- Steering of Roaming (SoR): управление выбором сети роуминга.
- SEPP (5G, N32): защита межсетевого SBI-обмена, фильтрация и валидация.
- Политики LBO (local breakout) vs HR (home routed).
- Роуминговый firewall на S6a/S8/Gy: защита от атак «изнутри» роуминга.

## 5.10 Lawful Interception (LI)

- ETSI/3GPP: HI1 (администрирование), HI2 (сигнализация), HI3 (контент); X1/X2/X3.
- В РФ — СОРМ; реализуется в узлах Core (MME, PGW/UPF, IMS, OCS).
- Не является «фильтрацией» для абонента, но это отдельный канал съёма данных.

## 5.11 5G: URSP, ULCL/BP, N6

- **URSP** (UE Route Selection Policy, TS 24.526): правила в UE — какой трафик в какую PDU-сессию/слайс/DNN.
- **ULCL** (Uplink Classifier) / **BP** (Branching Point): разгрузка трафика локально (MEC), фильтрация на UPF по правилам SMF.
- Traffic influence: AF через NEF влияет на маршрутизацию.
- N6 firewall для доступа к внешним сетям.
- Slice-based isolation как форма фильтрации (S-NSSAI).

## 5.12 Сводная таблица

| Тип фильтрации | Где | Что фильтруется | Зачем |
|---|---|---|---|
| TFT | UE | UL 5-tuple → bearer | правильная привязка трафика |
| SDF/PCC | PGW/UPF | DL/UL 5-tuple | QoS, тарификация |
| DPI/TDF | Core | L7-приложения | тарифы, блокировки |
| GTP firewall | Граница Core | GTP-C/U | безопасность, anti-fraud |
| Gi/SGi firewall | Граница | IP/порты/DDoS | защита сети |
| DNS/URL | Core/DNS | домены, категории | parental, защита |
| ACB/EAB/SSAC | RAN/Core | доступ UE | перегрузка |
| IPX/SEPP | Роуминг | межсетевой обмен | безопасность |
| LI | Core | данные абонента | законные требования |
| URSP/ULCL | UE/UPF | маршрутизация | слайсы, MEC |

## 5.13 Вопросы для самопроверки

1. Чем SDF-фильтр отличается от TFT?
2. Где хранится TFT и как он устанавливается?
3. Что делает TDF и через какой интерфейс общается с PCRF?
4. Зачем нужен GTP firewall и что он проверяет?
5. Что такое zero-rating и как он реализуется технически?
6. Чем ACB отличается от SSAC?
7. Как SEPP защищает 5G-роуминг?
8. Что описывают правила URSP?
9. Для чего нужен ULCL?
10. Какие интерфейсы LI используются?

## 5.14 Спецификации

- TS 23.060/23.401 — TFT, bearer
- TS 23.203/29.212 — PCC
- TS 29.274/29.281 — GTP
- GSMA FS.11, IR.33
- TS 33.210/33.310 — NDS/IP
- TS 33.107/33.108 — LI; ETSI TS 101 671
- TS 23.501/23.502, TS 24.526 — URSP, ULCL
- TS 23.251 — Network sharing
