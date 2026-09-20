# Шпаргалка для повторения

> Быстрый конспект по всем темам. Использовать после прочтения основных файлов и перед интервью/обсуждением.

## 1. CSFB / eSRVCC / Handover

| Процедура | Ключевые интерфейсы | Протоколы |
|---|---|---|
| CSFB | SGs (MME↔MSC), S1-MME, Uu, A/Iu-CS | SGsAP, S1AP, RRC/NAS, BSSAP/RANAP |
| SRVCC/eSRVCC | Sv (MME↔MSC-S), S1-MME, Iu/A, Mw/I2/Gm, Rx/Gx | Diameter (Sv), S1AP, SIP, Diameter |
| LTE HO | X2 или S1-MME; S11/S5/S8 | X2AP/S1AP, GTPv2-C |
| Inter-RAT PS HO | S3/S4/S16 | GTPv2-C |
| 5G HO | Xn/N2; N3/N9; N26 | XnAP/NGAP, GTP-U, GTPv2-C |

Ключевое:

- SGsAP — не Diameter; Sv — Diameter.
- Combined attach → SGs-ассоциация.
- STN-SR и C-MSISDN — для SRVCC.
- ATCF/ATGW — eSRVCC-ускорение (прерывание ~150–200 мс).
- N26 — interworking EPS↔5GS.

## 2. 5G: SA / NSA / DSS

- NSA (Option 3x): LTE-якорь + NR; EPC; X2; S1-U через SgNB; нет slicing/URLLC.
- SA (Option 2): NR + 5GC (AMF/SMF/UPF/PCF/UDM...); N1/N2/N3/N4/N6; slicing, URLLC, VoNR.
- DSS: один спектр для LTE и NR; SSB в MBSFN LTE; rate matching вокруг CRS; единый вендор; минус — падение пиковой ёмкости.

## 3. Diameter CCR/CCA

- CCR/CCA: Gy (онлайн-тарификация), Gx (PCC-политики), Ro (IMS).
- CC-Request-Type: INITIAL / UPDATE / TERMINATION / EVENT.
- MSCC: Rating-Group + RSU/USU/GSU; FUI (TERMINATE/REDIRECT/RESTRICT_ACCESS); Validity-Time.
- Gx: Flow-Information, QoS-Information, Charging-Rule-Install, APN-AMBR, ARP, QCI.
- Не путать: S6a (ULR/AIR), Rx (AAR/AAA), Rf (ACR/ACA), Sy (SLR/SLA).

## 4. IMS и PS Core

- Узлы: P-CSCF, I-CSCF, S-CSCF, HSS, SLF, AS (TAS/SCC AS), BGCF, MGCF, IM-MGW, MRFC/MRFP, E-CSCF/LRF, IBCF/TrGW, ATCF/ATGW.
- Ключевые интерфейсы: Gm, Mw, ISC, Cx, Sh, Rx, Rf/Ro, S6a, I2.
- VoLTE: PDN ims → P-CSCF (PCO) → QCI 5 → INVITE → Rx AAR → PCRF → Gx → dedicated bearer QCI 1.
- 5G: DNN ims, N5/N7/N4, 5QI 5/1, VoNR/EPS fallback.
- T-ADS: TAS/SCC AS ↔ HSS (Sh), данные от MME (S6a).

## 5. Фильтрация трафика

- L4: TFT (UE), SDF/PCC (PGW/UPF).
- L7: DPI/TDF (Sd → PCRF).
- Безопасность: GTP firewall (FS.11/IR.33), Gi/SGi firewall (NDS/IP).
- DNS/URL, parental control.
- FUP, zero-rating, throttling.
- Barring: ACB, EAB, SSAC.
- Роуминг: IPX, SEPP (N32), SoR.
- LI: X1/X2/X3, СОРМ.
- 5G: URSP, ULCL/BP, N6.

## 6. QoS

- 4G: QCI 1 (голос), 5 (сигнализация), 9 (default); GBR/non-GBR; ARP (1–15); APN-AMBR, UE-AMBR.
- 5G: QoS flows, QFI, 5QI, SDAP; reflective QoS; QNC; delay-critical GBR 82–85.
- VoLTE: QCI 5 + QCI 1 (2 — видео).
- Настройка: HSS/UDM (подписка), PCRF/PCF (динамика), Rx/N5 (AF).
- Турбо, FUP, VIP (ARP), emergency (QCI 1, pre-emption).

## 7. Оптимизация RAN со стороны Core

- SON: ANR (MME-релей), MRO, MLB, energy saving.
- Мобильность: MME pool/AMF set, overload control, paging, TAI list.
- Idle: eDRX/PSM (IoT).
- Покрытие: MDT (signaling/management based), trace, RLF.
- Перегрузка: RCAF (Np) → PCRF; UPCON.
- Steering: ATSSS, ULCL/MEC, AF influence.
- Слайсинг: S-NSSAI, NSSF; RAN slice-aware scheduling.
- Sharing: MOCN/MORAN.
- DC: E-RAB Modification / Path Switch → Core обновляет путь (GTPv2-C/N4).

## Мини-квиз (проверь себя)

1. Назовите 4 интерфейса SRVCC и их протоколы.
2. Чем Option 3x отличается от Option 3a?
3. Что такое FUI и какие у него действия?
4. Как устанавливается dedicated bearer для VoLTE?
5. Чем SDF-фильтр отличается от TFT?
6. Что такое ARP и как он используется при перегрузке?
7. Как MME участвует в ANR?
8. Что такое N26 и зачем он нужен?
9. Какой интерфейс у RCAF и что он передаёт?
10. Что описывают правила URSP?
