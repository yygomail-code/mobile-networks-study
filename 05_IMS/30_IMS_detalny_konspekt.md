# 30. IMS — подробный конспект

> Кому: инженерам, изучающим IP-мультимедийную подсистему (VoLTE/VoNR, услуги, роуминг);
> для подготовки к защите. Архитектура, идентификаторы, интерфейсы, процедуры, диагностика.
>
> Связанные файлы: `04_IMS_uzly_i_PS_Core.md` (краткий), `21_PS_Core_detalny_konspekt.md` (PS Core),
> `27_CSFB_eSRVCC_Handover_detalny_konspekt.md` (голосовые сценарии), `15_QoS_detalny_konspekt.md` (QoS).

## 30.1 Что такое IMS и зачем

**IMS (IP Multimedia Subsystem)** — архитектура доставки мультимедийных услуг поверх IP:
голос (VoLTE/VoNR), видео-вызовы, SMS over IMS, supplementary-услуги, конференции.

Основы:

- сигнализация — **SIP** (RFC 3261) поверх IP; описание медиа — **SDP** (RFC 4566);
- медиа — **RTP/RTCP**, защита — **SRTP**;
- IMS — «сеть услуг»: PS-домен даёт транспорт (bearer/QoS), IMS — сессии и услуги;
- в 4G: VoLTE; в 5G: VoNR (та же IMS, другое ядро); без VoLTE — CSFB/SRVCC (файл 27).

Принципы:

1. Полностью IP-сеть (нет CS для голоса).
2. Разделение сигнализации и медиа (SIP vs RTP).
3. QoS по запросу: PCRF/PCF + Rx/N5 → выделенный bearer/поток (QCI 1 / 5QI 1).
4. Услуги — в AS (серверах приложений), а не в ядре.

## 30.2 Архитектура IMS

| Узел | Роль |
|---|---|
| P-CSCF | первый контакт UE; SIP-прокси, регистрация, политика, взаимодействие с PCRF/PCF (Rx/N5) |
| I-CSCF | вход в домашнюю сеть; выбор S-CSCF (через HSS), маршрутизация |
| S-CSCF | «мозг» сессии: регистрация, маршрутизация услуг, AS-триггеры |
| HSS | профиль абонента IMS, данные регистрации (Cx), услуги (Sh) |
| SLF | выбор HSS, если их несколько |
| AS | серверы услуг: MMTel (голос/видео/доп. услуги), SCC AS (непрерывность, SRVCC), IP-SM-GW (SMS), TAS |
| MRF/MRFP | медиа-ресурсы: конференции, объявления, тональные сигналы |
| BGCF | выбор шлюза к PSTN |
| MGCF/MGW | стык с CS/PSTN (SIP↔ISUP/BICC, RTP↔PCM) |
| IBCF/TrGW | граница между IMS-сетями (роуминг, интерконнект), безопасность |
| DNS/ENUM | поиск узлов (NAPTR/SRV/A) и преобразование E.164↔SIP URI |
| SEG | шлюз безопасности (IPsec/TLS) |

## 30.3 Идентификаторы

| Идентификатор | Смысл |
|---|---|
| IMPI | приватный ID (аутентификация; обычно IMSI@ims.mncXXX.mccYYY.3gppnetwork.org) |
| IMPU | публичный ID (SIP URI / Tel URI); может быть несколько |
| PSI | публичный ID услуги (например, сервисные номера) |
| GRUU | глобально маршрутизируемый уникальный URI устройства |
| E.164 | телефонный номер |
| SIP URI | sip:user@domain; Tel URI — tel:+7... |

## 30.4 Интерфейсы IMS

| Интерфейс | Между кем | Протокол | Назначение |
|---|---|---|---|
| Gm | UE ↔ P-CSCF | SIP | сигнализация, регистрация |
| Mw | CSCF ↔ CSCF | SIP | маршрутизация |
| ISC | S-CSCF ↔ AS | SIP | триггеры услуг |
| Cx | CSCF ↔ HSS | Diameter | профиль, регистрация |
| Dx | CSCF ↔ SLF | Diameter | выбор HSS |
| Sh | AS ↔ HSS | Diameter | данные услуг |
| Rx / N5 | P-CSCF ↔ PCRF/PCF | Diameter / SBI | запрос QoS под медиа |
| Mi/Mj/Mg | CSCF ↔ BGCF/MGCF | SIP | выход в PSTN |
| Mr | CSCF ↔ MRF | SIP | медиа-ресурсы |
| Ut | UE ↔ AS | XCAP/HTTP | настройки услуг |
| Ici/Izi | IBCF ↔ IBCF/TrGW | SIP / IP | межсетевые стыки |
| ISC/Sh для SCC | AS ↔ HSS/ATCF | Diameter/SIP | непрерывность (SRVCC) |

## 30.5 Регистрация IMS

1. **P-CSCF discovery**: UE получает адрес P-CSCF (PCO в attach/PDU-сессии, DHCP или статически).
2. UE → P-CSCF → I-CSCF: REGISTER (IMPI/IMPU).
3. I-CSCF → HSS (Cx: UAR/UAA): выбор S-CSCF.
4. S-CSCF: 401 Unauthorized (AKA-челлендж) → UE отвечает REGISTER с ответом → 200 OK.
5. S-CSCF загружает профиль (Cx: SAR/SAA), делает **third-party REGISTER** в AS (MMTel/SCC).
6. Регистрация обновляется (re-REGISTER) по таймеру (обычно ~30 мин с рандомизацией).

Ошибки: 403/404 (нет IMPU/профиля), 401 повторно (неверные креды), таймауты (P-CSCF/S-CSCF, DNS).

## 30.6 Сессия VoLTE (установление вызова)

1. UE: SIP INVITE (SDP offer: кодек, полосы) → P-CSCF → S-CSCF.
2. S-CSCF: триггеры AS (MMTel); маршрутизация к вызываемому.
3. P-CSCF: Rx/N5 AAR → PCRF/PCF → Gx/N7 → PGW/UPF: выделенный bearer QCI 1 / 5QI 1 (GBR).
4. 100 Trying, 183 Session Progress (SDP answer), PRACK/UPDATE (preconditions), 180 Ringing.
5. 200 OK → ACK; медиа RTP/SRTP идёт по bearer'у QCI 1.
6. Завершение: BYE → освобождение bearer'а (Rx STR / Gx RAR).
7. Сигнализация IMS — QCI 5 (default bearer APN ims).

Кодеки: AMR-WB 12,65–23,85 кбит/с, EVS 5,9–24,4 (VoLTE/VoNR), видео H.264/HEVC.
Метрики: setup time (< 2 с), MOS (≥ 3,5–4,0), RTP loss (< 1%), jitter (< 30–50 мс).

## 30.7 Услуги и специальные сценарии

- **MMTel AS**: базовые услуги (CDIV, CW, HOLD, CONF, OIP/OIR), video-вызовы.
- **SCC AS**: непрерывность (SRVCC/eSRVCC), якорь медиа (ATCF/ATGW).
- **SMS over IMS**: через IP-SM-GW; в 5G — SMSF; альтернатива — SMS over NAS.
- **Экстренные вызовы**: E-CSCF → PSAP (TS 23.167); определение местоположения (LCS).
- **VoNR (5G)**: та же IMS, 5GC; голос — QoS flow 5QI 1; при недоступности — EPS fallback (N26).
- **Роуминг**: S8HR (медиа и сигнализация — через домашнюю сеть), LBO/IBCF; SEPP (5G).

## 30.8 Безопасность

- Аутентификация IMS-AKA (как в 3GPP); IPsec для Gm (UE↔P-CSCF), TLS на стыках;
- SEG — защита периметра IMS; IBCF/TrGW — фильтрация и топология;
- SRTP для медиа; проверка сертификатов, антифрод (проверка IMPU/E.164).

## 30.9 Диагностика IMS

Коды SIP (примеры):

| Код | Смысл | Типовая причина |
|---|---|---|
| 400 | Bad Request | некорректный SIP/SDP |
| 401/407 | Unauthorized | проблемы аутентификации |
| 403 | Forbidden | нет прав/услуги, блокировка |
| 404 | Not Found | нет IMPU/абонента |
| 408 | Timeout | недоступность узла/абонента |
| 480 | Temporarily Unavailable | абонент недоступен |
| 486 | Busy | занято |
| 500/503 | Server error/Unavailable | сбой/перегрузка AS/узла |
| 603 | Decline | отказ (например, блокировка) |

Типовые проблемы:

1. **Регистрация не проходит** — P-CSCF discovery, креды, профиль HSS, DNS.
2. **Вызов не устанавливается** — маршрутизация CSCF, AS-триггеры, кодек-несовместимость.
3. **Односторонний звук** — SDP/NAT, RTP-маршрут, firewall, SRTP-несоответствие.
4. **Плохой голос** — QoS (QCI 1 GBR), jitter/loss, транспорт (файл 15/24).
5. **Вызов срывается при переходе** — SRVCC/ATCF (файл 27), покрытие.
6. **SMS не ходят** — IP-SM-GW/SMSF, IMS-регистрация.

Инструменты: SIP-трейсы (P-CSCF/S-CSCF), Wireshark: `sip`, `rtp`, `diameter` (Cx/Rx/Sh),
`dns`; метрики: регистрация SR, call setup SR, ASR, drop rate, MOS.

## 30.10 Вопросы для защиты

**1. Зачем IMS, если есть SIP?**
IMS — архитектура (узлы, идентификаторы, политики, безопасность, QoS, услуги) вокруг SIP;
сам SIP — только протокол.

**2. Что делает P-CSCF и почему он первый?**
Первый контакт UE: регистрация, SIP-прокси, политика, Rx/N5 к PCRF/PCF для QoS медиа.

**3. Как выбирается S-CSCF?**
I-CSCF запрашивает HSS (Cx UAR/UAA); HSS возвращает возможности/имя S-CSCF (или список).

**4. Как голос получает QoS?**
P-CSCF → Rx/N5 → PCRF/PCF → Gx/N7 → PGW/UPF: выделенный bearer QCI 1 / 5QI 1 с GBR.

**5. Чем IMPI отличается от IMPU?**
IMPI — приватный (аутентификация, не маршрутизируется); IMPU — публичный (адрес для вызовов),
их может быть несколько у одного абонента.

**6. Что такое SCC AS и ATCF/ATGW?**
Функции непрерывности: SCC AS управляет переводом (SRVCC), ATCF/ATGW — якорь медиа,
чтобы голос не прерывался.

**7. Как работает роуминг VoLTE (S8HR)?**
Сигнализация и медиа идут через домашнюю сеть (S8 home routing): гостевая сеть — только
доступ; политика/услуги — домашние.

**8. Какие коды SIP укажут на проблемы аутентификации?**
401/407 (челлендж); повторяющиеся 401/403 — неверные креды/профиль/нет услуги.

**9. Почему односторонний звук — классика IMS?**
SDP-несоответствие/NAT/firewall: RTP идёт в одну сторону; проверять SDP, маршрут медиа,
SRTP, NAT-траверсал.

**10. Как VoNR отличается от VoLTE по IMS?**
IMS та же; отличается транспорт: 5GC (QoS flow 5QI 1, PCF/N5), при недоступности — EPS fallback.

## 30.11 Спецификации

- TS 23.228, TS 24.229 — архитектура и протоколы IMS
- TS 29.228/29.229 — Cx/Dx; TS 29.328/29.329 — Sh; TS 29.214 — Rx; TS 29.213 — PCC flows
- RFC 3261 (SIP), RFC 4566 (SDP), RFC 3550 (RTP), RFC 3711 (SRTP)
- TS 23.167 — экстренные вызовы; TS 23.237 — SCC/SRVCC; TS 23.292 — ICS
- TS 23.040/23.204 — SMS; TS 24.341 — SMS over IMS
- TS 33.203 — безопасность IMS; TS 23.501/23.502 — VoNR, EPS fallback
- TS 32.450 — KPI; TS 23.003 — идентификаторы
