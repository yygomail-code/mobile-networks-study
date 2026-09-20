# 21. PS Core — подробный конспект

> Кому: инженерам, изучающим ядро пакетной коммутации (2G/3G GPRS, 4G EPC, 5G 5GC).
> Цель: разобрать узлы, интерфейсы, процедуры, выбор узлов, роуминг, протоколы и диагностику
> PS Core — от PDP-контекстов до PDU-сессий.
>
> Связанные файлы: `04_IMS_uzly_i_PS_Core.md` (IMS и связь с ядром), `10_Posobie_Interfejsy_dlya_novichka.md`
> (интерфейсы), `15_QoS_detalny_konspekt.md` (QoS), `13_Diameter_detalny_konspekt.md` (Diameter),
> `18_5G_SA_NSA_DSS_detalny_konspekt.md` (5G), `01_CSFB_eSRVCC_Handover.md` (переходы).

## 21.1 Что такое PS Core и зачем

**PS Core (Packet Switched Core)** — ядро пакетной коммутации: часть сети, которая даёт абоненту
IP-доступ к данным. В отличие от CS-домена (голос по коммутации каналов), PS-домен передаёт
пакеты и не держит выделенный канал на всё время разговора.

Функции PS Core:

- **мобильность** — где абонент, куда доставлять данные (регистрация, paging, handover);
- **сессии** — PDP-контексты (2G/3G), bearer'ы (4G), PDU-сессии (5G);
- **IP-адресация** — выдача адреса, привязка к APN/DNN;
- **туннели** — передача данных между радио и внешними сетями (GTP);
- **политики и QoS** — какие классы, какие лимиты (см. файл 15);
- **тарификация** — учёт трафика (online/offline);
- **безопасность** — аутентификация, шифрование сигнализации, защита периметра;
- **роуминг** — работа в гостевых сетях.

Две плоскости:

- **C-plane (управление)**: сигнализация — NAS, S1AP/NGAP, GTPv2-C, Diameter, PFCP;
- **U-plane (данные)**: сам трафик абонента — GTP-U (4G/3G), PFCP-программируемые туннели (5G).

Эволюция:

```
2G/3G: SGSN + GGSN          → PDP-контексты, GTP
4G:    MME + SGW + PGW      → EPS bearer'ы, GTPv2
5G:    AMF + SMF + UPF      → PDU-сессии, QoS flows, SBA
```

## 21.2 2G/3G PS Core (GPRS/UMTS)

Узлы:

| Узел | Роль |
|---|---|
| SGSN | мобильность, сессии, аутентификация, маршрутизация к GGSN (аналог MME+части SGW) |
| GGSN | шлюз в IP-сети, якорь IP-адреса, политики (аналог PGW) |
| HLR/HSS | подписка, аутентификация |
| DNS | выбор GGSN по APN |
| CG/BG | тарификация и биллинг (offline) |
| MSC/VLR | CS-домен (взаимодействие через Gs) |

Интерфейсы:

| Интерфейс | Между кем | Протокол |
|---|---|---|
| Gb | BSS ↔ SGSN (2G) | BSSGP |
| Iu-PS | RNC ↔ SGSN (3G) | RANAP |
| Gn / Gp | SGSN ↔ GGSN (внутри / между сетями) | GTP-C, GTP-U |
| Gr | SGSN ↔ HLR | MAP (SS7) |
| Gs | SGSN ↔ MSC/VLR | BSSAP+ (combined attach, CSFB) |
| Gi | GGSN ↔ внешние сети | IP |
| Ga | SGSN/GGSN ↔ CG | GTP' |
| Gx/Gy | GGSN ↔ PCRF/OCS | Diameter |

Ключевые понятия:

- **PDP-контекст** — «сессия данных» с APN, IP-адресом и QoS-профилем;
- **APN** — точка доступа (имя сети, например `internet`);
- **GTP** — протокол туннелирования: GTP-C (управление) и GTP-U (данные), **TEID** — идентификатор туннеля;
- процедуры: **Attach / GPRS Attach**, **PDP Context Activation**, **RAU** (Routing Area Update), **Paging**.

## 21.3 4G EPC (Evolved Packet Core)

### 21.3.1 Узлы и функции

| Узел | Роль | Ключевые интерфейсы |
|---|---|---|
| MME | NAS, мобильность, управление bearer'ами, выбор SGW/PGW, аутентификация | S1-MME, S11, S6a, S10, SGs, Sv |
| SGW | транспорт данных, якорь при handover внутри LTE и inter-RAT | S1-U, S11, S5/S8, S4, Gxc |
| PGW | якорь IP, APN, enforcement QoS (PCEF), тарификация | S5/S8, SGi, Gx, Gy, S6b |
| HSS | подписка, аутентификация, профиль APN | S6a, S6d |
| PCRF | политики (PCC), правила QoS | Gx, Gxx, Rx, S9 |
| eNB | радио, S1-подключение | S1, X2 |
| ePDG | Wi-Fi доступ (VoWiFi, данные) | SWu, SWm, SWx |

Термины: **EPS bearer** (default/dedicated), **E-RAB** (радио+S1), **APN-AMBR**, **TAI/TAL** (списки зон),
**MME pool** и **SGW/PGW pool** (резервирование и балансировка через S1-Flex и DNS).

### 21.3.2 Интерфейсы EPC

| Интерфейс | Между кем | Протокол | Назначение |
|---|---|---|---|
| S1-MME | eNB ↔ MME | S1AP (SCTP) | управление, NAS-транспорт |
| S1-U | eNB ↔ SGW | GTP-U | данные |
| S11 | MME ↔ SGW | GTPv2-C | сессии и bearer'ы |
| S5/S8 | SGW ↔ PGW | GTPv2-C/U | внутри сети / роуминг |
| S6a | MME ↔ HSS | Diameter | подписка, аутентификация |
| Gx | PGW ↔ PCRF | Diameter | правила PCC |
| Gy | PGW ↔ OCS | Diameter | онлайн-тарификация |
| SGi | PGW ↔ внешние сети | IP | выход в интернет/сервисы |
| S10 | MME ↔ MME | GTPv2-C | переезды между MME |
| S3/S4 | MME/SGW ↔ SGSN | GTPv2-C | interworking с 2G/3G |
| SGs | MME ↔ MSC | SCTP | CSFB и SMS через MME |
| Sv | MME ↔ MSC | Diameter | SRVCC (перевод голоса в 2G/3G) |
| S9 | PCRF ↔ PCRF | Diameter | политики в роуминге |
| Gxc | SGW ↔ PCRF | Diameter | QoS при PMIP |
| S6b | PGW ↔ AAA | Diameter | non-3GPP доступ |
| SWu/SWm/SWx | ePDG ↔ UE/AAA/HSS | IPsec/Diameter | Wi-Fi доступ |

### 21.3.3 Процедуры

**Attach (регистрация):**

1. UE → eNB → MME: Attach Request (IMSI/GUTI, возможности).
2. MME ↔ HSS (S6a): аутентификация (AIR/AIA), Update Location (ULR/ULA), подписка.
3. MME → SGW (S11) → PGW (S5/S8): Create Session; PGW ↔ PCRF (Gx): правила и AMBR.
4. MME → eNB: Initial Context Setup (E-RAB QoS); UE получает Attach Accept и default bearer.
5. Данные: UE ↔ eNB ↔ SGW ↔ PGW (GTP-U).

**Другие процедуры:**

- **Service Request** — переход из idle в connected, восстановление bearer'ов;
- **TAU (Tracking Area Update)** — при перемещении; periodic TAU по таймеру (T3412);
- **Handover**: X2-based (без смены MME/SGW) и S1-based; inter-RAT — через S3/S4;
- **Paging** — поиск абонента по TA list;
- **Bearer activation/modification/deactivation** — управление QoS (файл 15);
- **Detach** — отключение (UE, сеть, по таймеру).

### 21.3.4 Выбор узлов (DNS)

- **MME** выбирает eNB: S1-Flex — по GUMMEI, TAI и весам (pool).
- **SGW/PGW** выбирает MME по DNS:
  - имя: `APN.epc.mncXXX.mccYYY.3gppnetwork.org`;
  - записи **S-NAPTR** → адреса SGW/PGW (A/AAAA);
  - учитываются топология (TAI), приоритеты, веса, доступность;
- **PGW**: статический (из подписки) или динамический; в роуминге — **APN-OI-Replacement**
  (переопределение APN на гостевой/домашний);
- **ePDG**: выбор по DNS (ePDG-FQDN), приоритеты по оператору.

## 21.4 5G Core (5GC)

Кратко (подробно — файл 18):

| Функция | Роль | Аналог в EPC |
|---|---|---|
| AMF | доступ и мобильность, NAS | MME |
| SMF | управление сессиями | SGW-C + PGW-C |
| UPF | пользовательская плоскость | SGW-U + PGW-U |
| UDM | подписка | HSS |
| PCF | политики | PCRF |
| AUSF | аутентификация | — |
| NRF | реестр сервисов | — |
| NSSF | выбор слайса | — |
| CHF | тарификация | OCS/OFCS |

Отличия от EPC:

- **сервисная архитектура (SBA)**: HTTP/2 + JSON, реестр NRF;
- **PDU-сессии и QoS flows** вместо bearer'ов;
- **stateless** сетевые функции, облачная реализация (CNF);
- **слайсинг** (S-NSSAI), MEC;
- interworking с EPC: **N26** (AMF↔MME), комбинированные узлы SMF+PGW-C, UPF+PGW-U.

## 21.5 CUPS и NFV: разделение плоскостей

- **CUPS (Control and User Plane Separation)**: SGW-C/SGW-U (Sxb), PGW-C/PGW-U (Sxc);
  в 5G это основа: SMF (C) ↔ UPF (U) по N4 (PFCP).
- Зачем: независимое масштабирование данных и управления, централизация C-plane, edge-размещение U-plane.
- **NFV/облака**: VNF/CNF, оркестрация (MANO/K8s), отказоустойчивость, обновления без простоя.
- Планирование ёмкости: C-plane — по сигнализации (attach/TAU/HO), U-plane — по трафику (Гбит/с, сессии).

## 21.6 Роуминг в PS Core

- **Home-routed (HR)**: SGW в гостевой сети, **PGW — в домашней**; данные идут через GRX/IPX;
  S6a — через DRA/IPX; Gy — в домашнюю OCS. Классика для 4G.
- **Local breakout (LBO)**: **PGW в гостевой сети** — короткий путь данных, но политики/тарификация сложнее.
- **S8HR** (S8 Home Routing) — голос в роуминге остаётся в домашней сети (VoLTE-роуминг).
- 5G: **HR** и **LBO** аналогично; между сетями — **SEPP** (N32), защита сигнализации.
- Выбор PGW в роуминге — через **APN-OI-Replacement** и политики оператора.
- Защита стыков: Diameter Edge Agent, SS7/Diameter firewall, GTP firewall, IPX-фильтрация.

## 21.7 PS Core для IoT и SMS

- **eMTC/NB-IoT**: optimizations — **Control Plane CIoT** (данные в NAS, SCEF/NIDD, non-IP),
  **User Plane CIoT** (RRC suspend/resume); экономия батареи — **PSM/eDRX**.
- **SCEF** — сервер возможностей (NIDD API), связка с PGW-C.
- **SMS**: 4G — SMS через MME (SGs) или IMS; 5G — SMS over NAS (SMSF) или IMS.
- **Экстренные службы**: PSAP, eCall; требования по приоритету (файл 15, ARP/MPS).

## 21.8 Безопасность

- аутентификация: AKA (2G/3G — SIM, 4G/5G — USIM; Milenage/TUAK);
- шифрование NAS (EPS/5GS) и RRC; защита целостности;
- SCTP/IPsec на транспортных стыках (S1, S6a и др. — по политике);
- **GTP firewall** — защита от атак через GTP-U/C (подмена TEID, flooding);
- **Diameter Edge Agent** — фильтрация и маршрутизация Diameter на границе;
- 5G: **SEPP** — защита межсетевого взаимодействия (роуминг, N32);
- сегментация: отдельные VLAN/VRF, ACL, лимиты на интерфейсах.

## 21.9 Эксплуатация и диагностика

### 21.9.1 KPI

- доступность: Attach SR, Service Request SR, Paging SR;
- удержание: session drop rate, S1 release, HO success (X2/S1, inter-RAT);
- производительность: DL/UL throughput, latency, загрузка U-plane;
- сигнализация: TAU/RAU SR, число attached/active абонентов;
- роуминг: SR на S6a/S8, отказы IPX.

### 21.9.2 Типовые проблемы

1. **Attach падает** — HSS/S6a (нет ответа, коды Diameter), аутентификация, перегрузка MME.
2. **Сессия не устанавливается** — DNS-выбор SGW/PGW, APN-OI-Replacement, пул IP, лимиты PGW.
3. **Paging не находит абонента** — TA list/MME pool, периодический TAU, несоответствие зон.
4. **Handover падает** — S10/S1, X2-транспорт, смена SGW, inter-RAT настройки.
5. **Низкая скорость** — перегрузка GTP-U, ёмкость SGW/PGW, QoS/AMBR (файл 15), транспорт.
6. **Роуминг не работает** — IPX/DRA, фильтры на границе, неверный APN-OI-Replacement.

### 21.9.3 Инструменты

- трассировки по IMSI/GUTI на MME/SGW/PGW (и AMF/SMF/UPF), PCRF/PCF;
- Wireshark: `s1ap`, `gtpv2`, `gtp`, `diameter`, `pfcp`, `nas-eps`, `sctp`;
- счётчики и KPI по узлам, проверка DNS (dig/host по APN-FQDN);
- нагрузочные тесты сигнализации, контроль пулов и весов.

### 21.9.4 Мини-кейсы

1. **Attach с ошибкой аутентификации**: неверный Ki/OPc в HSS vs SIM — проверить S6a AIR/AIA.
2. **Create Session timeout**: PGW не отвечает (перегрузка/фильтр) — смотреть S11/S5.
3. **После TAU нет данных**: не обновился bearer/пул SGW — проверить S11 Modify Bearer.
4. **Скорость ниже тарифа**: APN-AMBR или перегрузка U-plane — сравнить Gx и загрузку.
5. **Роуминг: только голос**: данные блокируются IPX-фильтром или APN-OI-Replacement.

## 21.10 Сравнение поколений PS Core

| | 2G/3G | 4G EPC | 5G 5GC |
|---|---|---|---|
| Сессия | PDP-контекст | EPS bearer | PDU-сессия + QoS flows |
| Управление | SGSN | MME + SGW-C + PGW-C | AMF + SMF |
| Данные | SGSN + GGSN | SGW-U + PGW-U | UPF |
| Якорь IP | GGSN | PGW | UPF |
| Подписка | HLR | HSS | UDM |
| Политики | PCRF (Gx) | PCRF (Gx) | PCF (N7) |
| Туннель | GTP-U | GTP-U | GTP-U (N3) |
| Управление сессией | GTP-C | GTPv2-C | PFCP (N4) |
| Архитектура | монолит | функциональная | сервисная (SBA), CNF |
| Разделение C/U | нет | опция (CUPS) | основа (SMF/UPF) |

## 21.11 Мини-словарь

- **PS / CS** — пакетная / канальная коммутация.
- **SGSN / GGSN** — узлы GPRS/UMTS.
- **MME / SGW / PGW** — узлы EPC.
- **AMF / SMF / UPF** — узлы 5GC.
- **PDP-контекст / EPS bearer / PDU-сессия** — «сессии данных» поколений.
- **APN / DNN** — точка доступа (4G/5G).
- **TEID** — идентификатор туннеля GTP.
- **S1-Flex / pool** — резервирование и балансировка узлов.
- **S-NAPTR** — DNS-записи выбора шлюзов.
- **CUPS** — разделение управления и данных.
- **SCEF / NIDD** — IoT-доставка данных.
- **DRA / DEA** — маршрутизация и защита Diameter.
- **SEPP** — защита межсетевого взаимодействия в 5G.

## 21.12 Вопросы для самопроверки

1. Какие функции выполняет PS Core и чем отличается от CS-домена?
2. Чем SGSN/GGSN отличаются от MME/SGW/PGW?
3. Какие интерфейсы участвуют в Attach и что по ним передаётся?
4. Как MME выбирает SGW и PGW (DNS, S-NAPTR, топология)?
5. Что такое CUPS и зачем разделять C/U?
6. Чем home-routed роуминг отличается от local breakout?
7. Что такое S8HR и для чего он нужен?
8. Какие оптимизации PS Core применяются для IoT?
9. Как защищают стыки PS Core (GTP/Diameter firewall, SEPP)?
10. Какие KPI и инструменты использовать при проблемах с данными?

## 21.13 Спецификации

- TS 23.060 — GPRS (2G/3G PS); TS 29.060 — GTPv1; TS 29.061 — Gi
- TS 23.401 — EPS (EPC, процедуры); TS 23.402 — non-3GPP доступ
- TS 23.501, TS 23.502 — 5G System и процедуры; TS 23.503 — политики
- TS 29.274 — GTPv2-C; TS 29.281 — GTP-U; TS 29.244 — PFCP
- TS 29.212/29.213/29.214 — Gx/Rx; TS 29.272 — S6a; TS 29.273 — S6b/SWx
- TS 23.272 — CSFB; TS 23.216 — SRVCC; TS 23.236 — pools (S1-Flex)
- TS 29.303 — DNS-процедуры выбора (EPC); TS 23.682 — IoT (SCEF/NIDD)
- TS 33.401/33.501 — безопасность EPS/5GS
