# 27. CSFB, eSRVCC и Handover — подробный конспект

> Кому: инженерам Core/RAN, изучающим голосовые сценарии LTE и переходы между поколениями;
> для подготовки к защите. Точные процедуры, интерфейсы, коды ошибок, KPI и вопросы-ответы.
>
> Связанные файлы: `01_CSFB_eSRVCC_Handover.md` (краткий), `21_PS_Core_detalny_konspekt.md` (PS Core),
> `30_IMS_detalny_konspekt.md` (IMS), `15_QoS_detalny_konspekt.md` (QoS), `24_RAN_optimizaciya_detalny_konspekt.md` (RAN).

## 27.1 Рамка: голос в LTE и переходы

Пока LTE строили как «данные-only» сеть, голос обслуживали три механизма:

| Механизм | Что это | Когда используется |
|---|---|---|
| CSFB | перевод в 2G/3G для вызова | LTE без VoLTE; легаси-абоненты |
| SRVCC | перевод **активного** голоса в 2G/3G | уход из LTE во время разговора |
| VoLTE | голос в LTE (IMS) | целевой механизм |

Ключевые термины: **CSFB (CS Fallback)**, **eSRVCC (enhanced SRVCC)**, **PS HO**, **redirect**,
**SGs**, **Sv**, **TAC↔LAC mapping**, **fast return**.

## 27.2 CSFB: архитектура

- **SGs** — интерфейс MME ↔ MSC (SCTP, протокол SGsAP): «мост» между PS и CS.
- **Combined attach / combined TAU**: UE регистрируется и в PS (MME), и в CS (MSC через SGs);
  MSC получает данные UE, MME хранит связь (VLR-ассоциация).
- **TAC ↔ LAC mapping**: MME должен сопоставлять Tracking Area (LTE) с Location Area (2G/3G);
  ошибки mapping — классическая причина отказов CSFB.
- **MSC pool**: SGs-ассоциации MME↔MSC в пуле (балансировка, резервирование).
- **Возврат**: после вызова UE возвращается в LTE (fast return / PS HO / redirect).

Схема сигнализации (упрощённо):

```
UE ── eNB ── MME ──SGs── MSC/VLR ── BSS/RNC (2G/3G)
                │
                └── (UE переводится в 2G/3G для вызова)
```

## 27.3 CSFB: процедуры

### 27.3.1 Исходящий вызов (MO)

1. UE (LTE) инициирует вызов → NAS Extended Service Request (CSFB) → MME.
2. MME → MSC: SGsAP Service Request; MSC отвечает (или CSFB indication).
3. MME инициирует перевод: **PS HO** (в 2G/3G с поддержкой) или **redirect** (RRC Release
   с redirect info); UE переходит в 2G/3G.
4. В 2G/3G UE выполняет обычную CS-процедуру (CM Service Request, Setup) — вызов идёт в CS.
5. После завершения — возврат в LTE (fast return: UE сразу ищет LTE).

### 27.3.2 Входящий вызов (MT)

1. Вызов приходит в MSC → MSC → SGsAP Paging Request → MME.
2. MME пейджит UE в LTE (S1AP Paging, CS-домен) → UE отвечает (Extended Service Request).
3. Далее — как в MO: перевод в 2G/3G, вызов, возврат.

### 27.3.3 SMS и emergency

- SMS через SGs (SMS over SGs): передаётся в NAS-сообщениях (UL/DL NAS Transport).
- Emergency CSFB: приоритетный сценарий, особые требования к задержке; в 5G — EPS fallback
  или emergency VoNR.

## 27.4 CSFB: оптимизация и проблемы

Составляющие задержки CSFB (ориентиры):

| Этап | Типичное время |
|---|---|
| Paging + Extended Service Request | 0,5–1 с |
| Перевод (PS HO или redirect) | 1–3 с |
| CS-установление в 2G/3G | 1–3 с |
| Итого до начала вызова | 2–8 с (цель ≤ 6 с) |

Типовые проблемы:

1. **TAC-LAC mismatch** → MSC не знает зону → отказ paging/SGs.
2. **SGs-ассоциация потеряна** → MT-вызовы не доходят (проверять SGsAP status).
3. **Redirect без соседей** → UE не находит 2G/3G (проверять RRC Release redirect info).
4. **Медленный возврат** → UE «застревает» в 2G/3G (fast return настройки).
5. **Combined attach не проходит** → UE только PS; проверять SGs + MSC.

KPI CSFB: CSFB SR, CSFB setup time, return-to-LTE time, SGs paging SR, drop rate.

## 27.5 eSRVCC: перевод активного голоса

### 27.5.1 Архитектура

- **Sv** — интерфейс MME ↔ MSC (Diameter, TS 29.280): подготовка SRVCC-перевода.
- **ATCF/ATGW** (Access Transfer Control/User Function) — в IMS: якорь медиа при переводе
  (SCC AS), чтобы голос не прерывался; **SCC AS** — услуга непрерывности (TS 23.237).
- **MSC** — принимает голос в 2G/3G; **MGW** — медиа-шлюз.

### 27.5.2 Процедура (E-UTRAN → UTRAN/GERAN)

1. UE (VoLTE-вызов, QCI 1) уходит из покрытия LTE; eNB решает перевести в 2G/3G.
2. eNB → MME: Handover Required (SRVCC).
3. MME → MSC: Sv PS-to-CS Request; MSC готовит CS-ресурсы (через RNC/BSS).
4. MME → eNB: Handover Command; UE переходит в 2G/3G.
5. Голос продолжается через CS; PS-сессии (если есть) — по политике (transfer/дроп).
6. **vSRVCC** — вариант для видео-звонков (перевод видео в CS/через ATCF).

KPI: SRVCC interruption time (цель < 300 мс для голоса), SRVCC SR, drop rate при переводе.

## 27.6 Handover: полная карта

### 27.6.1 Intra-LTE

- **X2-based**: подготовка (Handover Request/ACK по X2), исполнение (RRC Reconfiguration),
  завершение (Path Switch к MME/SGW, Release).
- **S1-based**: через MME (Handover Required → ... → Handover Command), используется при
  отсутствии X2 или смене узлов.
- Смена SGW (при смене пула) — Create/Modify Bearer; MME relocation — S10.

### 27.6.2 Inter-RAT и межсистемные

- **LTE↔2G/3G (PS)**: S3/S4 (MME↔SGSN, SGW↔SGSN), PS HO.
- **CSFB/SRVCC**: голосовые сценарии (см. выше).
- **LTE↔5G**: N26 (MME↔AMF) — бесшовность EPS↔5GS; **EPS fallback** — голос из SA в LTE.
- **NR↔LTE**: Xn/N2 handover в 5G; inter-RAT — по N26/измерениям.

### 27.6.3 Ошибки и классификация (MRO)

- **Too early**: HO начался слишком рано → возврат/падение.
- **Too late**: HO не успел → radio link failure.
- **Wrong cell**: HO в неверную соту → восстановление.
Причины Core-зоны: соседи (ANR), пулы, S10/N26, ёмкость сигнализации, таймеры.

KPI: HO SR (intra/inter/X2/S1/N2), HO failure rate по типам, ping-pong rate, interruption time.

## 27.7 Сквозные сценарии и диагностика

- **VoLTE vs CSFB (сравнение установления)**: VoLTE — SIP INVITE + QCI1 bearer (1–2 с);
  CSFB — перевод + CS-установление (2–8 с).
- Инструменты: трассировки MME/MSC/eNB; Wireshark: `s1ap`, `sgsap`, `diameter` (Sv, S6a),
  `gsm_map`, `ranap`, `bssap`, `gtpv2`.
- Типовые отказы и первые проверки:

| Симптом | Первые проверки |
|---|---|
| MT-вызов не доходит | SGs-ассоциация, paging, TAC-LAC |
| MO-вызов долгий | PS HO/redirect, соседи, fast return |
| Голос обрывается при уходе из LTE | SRVCC (Sv, ATCF), покрытие 2G/3G |
| HO SR низкий | X2/S10/N26, пулы, соседи |
| Возврат в LTE медленный | fast return, приоритеты LTE |

## 27.8 Вопросы для защиты

**1. Зачем нужен SGs, если есть PS-домен?**
SGs — мост MME↔MSC: без него LTE-абонент «невидим» для CS (нет paging, вызовов, SMS).
Combined attach регистрирует UE и в MSC.

**2. Чем CSFB отличается от SRVCC?**
CSFB переводит UE **до** установления вызова (голос начинается в 2G/3G); SRVCC переводит
**активный** вызов из LTE в 2G/3G (непрерывность через ATCF).

**3. Что будет при ошибке TAC-LAC mapping?**
MSC не сопоставит зоны: paging/CSFB по этому TAC откажет или пойдёт «не туда».
Симптом: MT-вызовы не доходят на части сот.

**4. Почему CSFB долгий и как ускорить?**
Сумма этапов (paging, перевод, CS-установление). Ускорение: PS HO вместо redirect,
корректные соседи, fast return, оптимизация SGs.

**5. Как UE возвращается в LTE после вызова?**
Fast return: UE по приоритетам (RFSP/redirect) сразу ищет LTE; при PS HO — возврат через
TAU/attach. Медленный возврат — частая жалоба.

**6. Что такое ATCF/ATGW?**
Функции IMS-якоря медиа при SRVCC: сохраняют голосовой поток при переводе (SCC AS),
обеспечивая непрерывность (TS 23.237).

**7. Как связаны CSFB и 5G?**
В 5G голос — VoNR или EPS fallback (перевод в LTE) — это «идейный наследник» CSFB,
но с ядром 5GC и N26.

**8. Какие KPI показывают качество CSFB?**
CSFB SR, setup time, SGs paging SR, return-to-LTE time, drop rate; по сотам и MSC.

**9. Чем X2 HO отличается от S1 HO?**
X2 — прямой обмен между eNB (быстрее, без Core в подготовке); S1 — через MME (нужен при
отсутствии X2/смене узлов); после обоих — Path Switch к Core.

**10. Классификация MRO «too late» — что это?**
HO не успел до radio link failure: UE теряет LTE раньше, чем переключится. Лечится
порогами/соседями (RAN) и корректностью Core-конфигурации (пулы, S10).

## 27.9 Спецификации

- CSFB: TS 23.272, TS 29.118 (SGsAP), TS 24.301 (NAS), TS 36.413 (S1AP)
- SRVCC: TS 23.216, TS 29.280 (Sv), TS 23.237 (SCC/ATCF), TS 23.292 (ICS)
- IMS: TS 23.228, TS 24.229; EPS fallback: TS 23.502, TS 23.501
- Handover: TS 23.401 (EPC), TS 36.300/36.413/36.423, TS 38.300/38.413/38.423
- Interworking: TS 23.502 (N26), TS 29.274 (GTPv2-C), TS 29.303 (DNS)
- KPI: TS 32.450; QoS: TS 23.203
