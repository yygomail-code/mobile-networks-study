# Материалы для изучения: мобильные сети (Core, IMS, 5G, QoS, RAN)

Учебный комплект по мобильным сетям. Материалы разложены по папкам: **одна папка — один
материал (тема)**. Внутри каждой папки: конспекты (краткий и подробный), PDF, схемы (SVG/PNG),
рисунки и видеолекции. В папке `00_Voprosy_pervogo_zadaniya/` — сводные ответы на 7 вопросов
первого задания с навигацией по всем материалам; в папке `12_Perekhrestnye_voprosy/` —
подготовка к перекрёстным вопросам руководителей отделов (траблшутинг, оптимизация,
радиоизмерения).

## Структура

| Папка | Материал | Состав |
|---|---|---|
| `00_Voprosy_pervogo_zadaniya/` | Первое задание: ответы на 7 вопросов | сводный конспект (md + PDF), карта «вопрос → материалы» |
| `01_CSFB_eSRVCC_Handover/` | CSFB, eSRVCC, Handover | краткий (01) + подробный (27) конспекты, PDF, схемы 28/29, рисунки, видео |
| `02_5G_SA_NSA_DSS/` | 5G: SA, NSA, DSS | краткий (02) + подробный (18), PDF, схемы 19/20, рисунки, видео (2 модуля) |
| `03_Diameter/` | Diameter (CCR/CCA, S6a, Gx, Gy) | краткий (03) + подробный (13), PDF, схема 14, рисунки, видео (2 модуля) |
| `04_PS_Core/` | PS Core: 2G/3G → EPC → 5GC | краткий (04) + подробный (21), PDF, схемы 22/23, рисунки, видео |
| `05_IMS/` | IMS: VoLTE/VoNR, услуги, роуминг | подробный (30), PDF, схемы 31/32, рисунки, видео |
| `06_QoS/` | QoS: 4G/5G, PCC, практика | краткий (06) + подробный (15), PDF, схемы 16/17, рисунки, видео (2 модуля) |
| `07_Filtraciya/` | Фильтрация трафика, DPI, политики | краткий (05) + подробный (33), PDF, схемы 34/35, рисунки, видео |
| `08_Optimizaciya_RAN/` | Оптимизация RAN со стороны Core | краткий (07) + подробный (24), PDF, схемы 25/26, рисунки, видео (2 модуля) |
| `09_Troubleshooting/` | Траблшутинг интерфейсов LTE/PS/CS | краткий (09) + подробный (36), PDF, схемы 37/38, рисунки, видео |
| `10_Interfejsy_dlya_novichka/` | Пособие для начинающих + схема интерфейсов | пособие (10) md+pdf, схема 11, промт 12, рисунки, видео (2 модуля) |
| `11_Shpargalka/` | Шпаргалка и мини-квиз | 08 |
| `12_Perekhrestnye_voprosy/` | Перекрёстные вопросы руководителей (Траблшутинг, Оптимизация, Радиоизмерения) | конспект (75 Q&A, кейсы, шпаргалка), PDF, схемы 40/41, рисунки, видео (2 модуля) |
| корень | Индекс и демо | `README.md`, `demo_videolekciya.mp4` |

## Состав по папкам

### 00_Voprosy_pervogo_zadaniya
- `00_Otvety_na_voprosy.md` / `.pdf` — ответы на 7 вопросов первого задания (CSFB/eSRVCC/Handover,
  5G SA/NSA/DSS, Diameter CCA/CCR, IMS и PS Core, фильтрация, QoS, оптимизация RAN) +
  карта «вопрос → папка/схемы/видео»
- `00_cover.svg` / `.png` — карта ответов (обложка)

### 01_CSFB_eSRVCC_Handover
- `01_CSFB_eSRVCC_Handover.md` — краткий конспект
- `27_CSFB_eSRVCC_Handover_detalny_konspekt.md` / `.pdf` — подробный конспект (4 рисунка)
- `28_Schema_CSFB_SRVCC.svg` / `.png` — CSFB и eSRVCC: архитектура и процедуры
- `29_Schema_Handover_map.svg` / `.png` — карта переходов (X2/S1, Xn/N2, N26, S3/S4, SGs/Sv)
- `figures/` — рисунки для PDF/видео (CSFB MO, SRVCC)
- `video/27_CSFB_modul1.mp4`

### 02_5G_SA_NSA_DSS
- `02_5G_SA_NSA_DSS.md` — краткий конспект
- `18_5G_SA_NSA_DSS_detalny_konspekt.md` / `.pdf` — подробный конспект (6 рисунков)
- `19_Schema_5G_SA_NSA.svg` / `.png` — NSA (EN-DC) и SA: архитектуры и интерфейсы
- `20_Schema_DSS.svg` / `.png` — DSS: LTE и NR в одной несущей
- `figures/` — рисунки (опции 3GPP, EN-DC, регистрация SA, EPS fallback)
- `video/18_5G_modul1.mp4`, `video/18_5G_modul2.mp4`

### 03_Diameter
- `03_Diameter_CCR_CCA.md` — краткий конспект
- `13_Diameter_detalny_konspekt.md` / `.pdf` — подробный конспект (4 рисунка)
- `14_Schema_Diameter.svg` / `.png` — Diameter-ландшафт: узлы, интерфейсы, DRA
- `figures/` — рисунки (сообщение/AVP, S6a-flow, Gy)
- `video/13_Diameter_modul1.mp4`, `video/13_Diameter_modul2.mp4`

### 04_PS_Core
- `04_IMS_uzly_i_PS_Core.md` — краткий конспект (IMS и PS Core)
- `21_PS_Core_detalny_konspekt.md` / `.pdf` — подробный конспект (6 рисунков)
- `22_Schema_PS_Core.svg` / `.png` — эволюция 2G/3G → EPC → 5GC
- `23_Schema_PS_Core_roaming.svg` / `.png` — роуминг: home-routed и local breakout
- `figures/` — рисунки (GTP-U, attach, выбор узлов, CUPS)
- `video/21_PS_Core_modul1.mp4`

### 05_IMS
- `30_IMS_detalny_konspekt.md` / `.pdf` — подробный конспект (4 рисунка)
- `31_Schema_IMS.svg` / `.png` — архитектура и интерфейсы IMS
- `32_Schema_VoLTE_e2e.svg` / `.png` — VoLTE: сквозной путь установления
- `figures/` — рисунки (регистрация IMS, установление вызова)
- `video/30_IMS_modul1.mp4`

### 06_QoS
- `06_QoS.md` — краткий конспект
- `15_QoS_detalny_konspekt.md` / `.pdf` — подробный конспект (5 рисунков)
- `16_Schema_QoS_4G.svg` / `.png` — QoS в 4G: bearer'ы и параметры
- `17_Schema_QoS_5G.svg` / `.png` — QoS в 5G: PDU-сессия и QoS flows
- `figures/` — рисунки (bearer/TFT, классы 5QI, VoLTE)
- `video/15_QoS_modul1.mp4`, `video/15_QoS_modul2.mp4`

### 07_Filtraciya
- `05_Filtraciya_trafika.md` — краткий конспект
- `33_Filtraciya_detalny_konspekt.md` / `.pdf` — подробный конспект (4 рисунка)
- `34_Schema_Filtraciya.svg` / `.png` — уровни и механизмы фильтрации
- `35_Schema_UPF_pipeline.svg` / `.png` — конвейер UPF: PDR → FAR/QER/URR
- `figures/` — рисунки (правила/приоритеты, DPI)
- `video/33_Filter_modul1.mp4`

### 08_Optimizaciya_RAN
- `07_Optimizaciya_RAN_so_storony_Core.md` — краткий конспект
- `24_RAN_optimizaciya_detalny_konspekt.md` / `.pdf` — подробный конспект для защиты (6 рисунков)
- `25_Schema_RAN_levers.svg` / `.png` — карта рычагов Core → RAN
- `26_Schema_RAN_loop.svg` / `.png` — петля оптимизации: KPI, PDCA, инструменты
- `figures/` — рисунки (KPI-дерево, RFSP, paging, handover)
- `video/24_RAN_modul1.mp4`, `video/24_RAN_modul2.mp4`

### 09_Troubleshooting
- `09_Interfejsy_LTE_PS_CS_troubleshooting.md` — краткий конспект
- `36_Troubleshooting_interfejsov_detalny_konspekt.md` / `.pdf` — подробный конспект (4 рисунка)
- `37_Schema_Troubleshooting_ladder.svg` / `.png` — лестница диагностики и дерево проверок
- `38_Schema_Stacks.svg` / `.png` — стеки протоколов по интерфейсам
- `figures/` — рисунки (лестница, кейс Attach)
- `video/36_Troubleshoot_modul1.mp4`

### 10_Interfejsy_dlya_novichka
- `10_Posobie_Interfejsy_dlya_novichka.md` / `.pdf` — пособие (3 рисунка)
- `11_Schema_interfejsov.svg` / `.png` — большая схема интерфейсов 2G/3G/4G/5G и IMS
- `12_Prompt_dlya_generatora_shemy.md` — промт для генерации схем (RU/EN) + Mermaid
- `figures/` — рисунки (C-plane/U-plane, лестница диагностики)
- `video/10_Interfejsy_modul1.mp4`, `video/10_Interfejsy_modul2.mp4`

### 11_Shpargalka
- `08_Shpargalka.md` — краткий конспект и мини-квиз для повторения

### 12_Perekhrestnye_voprosy
- `39_Perekhrestnye_voprosy.md` / `.pdf` — перекрёстные вопросы руководителей отделов
  Траблшутинга, Оптимизации и Радиоизмерений: структура ответа, 75 вопросов с ответами
  (по отделам + сквозные), 10 мини-кейсов для устного разбора, шпаргалка цифр
- `40_Schema_Vzaimodejstvie.svg` / `.png` — взаимодействие отделов и потоки данных
- `41_Schema_Voprosy_karta.svg` / `.png` — карта вопросов трёх отделов
- `figures/` — рисунки (структура ответа, шпаргалка цифр)
- `video/39_Cross_modul1.mp4`, `video/39_Cross_modul2.mp4`

## Видеолекции (MP4, 1920×1080, слайды + синтезированный голос)

| Файл | Содержание | Длительность |
|---|---|---|
| `10_Interfejsy_dlya_novichka/video/10_Interfejsy_modul1.mp4` | Интерфейсы: введение, узлы, методика, LTE/EPC (10.1–10.4) | 15:32 |
| `10_Interfejsy_dlya_novichka/video/10_Interfejsy_modul2.mp4` | Интерфейсы: PS/CS 2G/3G, IMS, 5G, диагностика (10.5–10.13) | 16:32 |
| `03_Diameter/video/13_Diameter_modul1.mp4` | Diameter: основы, транспорт, AVP, приложения (13.1–13.4) | 11:56 |
| `03_Diameter/video/13_Diameter_modul2.mp4` | Diameter: интерфейсы, DRA, безопасность, диагностика (13.5–13.13) | 20:37 |
| `06_QoS/video/15_QoS_modul1.mp4` | QoS: метрики, модель 4G, QCI/ARP/AMBR/TFT, PCC (15.1–15.3) | 10:09 |
| `06_QoS/video/15_QoS_modul2.mp4` | QoS: потоки 5G, IMS, практика, диагностика (15.4–15.11) | 12:43 |
| `02_5G_SA_NSA_DSS/video/18_5G_modul1.mp4` | 5G: архитектура, опции, NSA/EN-DC, SA (18.1–18.5) | 10:19 |
| `02_5G_SA_NSA_DSS/video/18_5G_modul2.mp4` | 5G: DSS, голос, переходы, KPI, диагностика (18.6–18.15) | 11:35 |
| `04_PS_Core/video/21_PS_Core_modul1.mp4` | PS Core: 2G/3G → EPC → 5GC, процедуры, роуминг (21.1–21.13) | 17:05 |
| `08_Optimizaciya_RAN/video/24_RAN_modul1.mp4` | RAN-оптимизация: KPI, рычаги, мобильность, голос, транспорт (24.1–24.8) | 17:51 |
| `08_Optimizaciya_RAN/video/24_RAN_modul2.mp4` | RAN-оптимизация: кейсы, вопросы для защиты, чек-листы (24.9–24.12) | 12:10 |
| `01_CSFB_eSRVCC_Handover/video/27_CSFB_modul1.mp4` | CSFB/eSRVCC/Handover: процедуры, SGs/Sv, карта переходов (27.1–27.9) | 12:34 |
| `05_IMS/video/30_IMS_modul1.mp4` | IMS: архитектура, регистрация, VoLTE, диагностика (30.1–30.11) | 11:09 |
| `07_Filtraciya/video/33_Filter_modul1.mp4` | Фильтрация: уровни, DPI/TDF, UPF-конвейер (33.1–33.9) | 10:43 |
| `09_Troubleshooting/video/36_Troubleshoot_modul1.mp4` | Траблшутинг: методика, интерфейсы, кейсы, фильтры (36.1–36.9) | 11:54 |
| `12_Perekhrestnye_voprosy/video/39_Cross_modul1.mp4` | Перекрёстные вопросы: структура ответа, Траблшутинг, Оптимизация (39.1–39.3) | 14:11 |
| `12_Perekhrestnye_voprosy/video/39_Cross_modul2.mp4` | Перекрёстные вопросы: Радиоизмерения, сквозные, кейсы, шпаргалка (39.4–39.8) | 17:24 |
| `demo_videolekciya.mp4` | Демо-ролик формата (в корне проекта) | 1:05 |

Формат: слайды 1080p из конспектов + офлайн-нейросетевой голос (Piper ru-RU, Дмитрий).
Слайды и озвучка генерируются автоматически из Markdown-конспектов.

## Как изучать

1. Открыть папку темы — там весь комплект: конспект, схемы, видео.
2. Читать тему целиком, не конспектируя при первом проходе.
3. После каждой темы ответить на «Вопросы для самопроверки» / «Вопросы для защиты» без подглядывания.
4. Перед обсуждением/интервью — прогнать `11_Shpargalka/08_Shpargalka.md` и мини-квиз;
   держать под рукой `10_Interfejsy_dlya_novichka/11_Schema_interfejsov.svg`.
5. Спорные/дрейфующие детали сверять с актуальными релизами 3GPP и вендорной документацией.

## План повторения (10 дней)

| День | Что повторить |
|---|---|
| 1 | `01_CSFB_eSRVCC_Handover` — конспекты, схемы 28/29, видео |
| 2 | `02_5G_SA_NSA_DSS` — конспекты, схемы 19/20, видео |
| 3 | `03_Diameter` — конспекты, схема 14, видео |
| 4 | `04_PS_Core` и `05_IMS` — конспекты, схемы, видео |
| 5 | `07_Filtraciya` — конспекты, схемы 34/35, видео |
| 6 | `06_QoS` — конспекты, схемы 16/17, видео |
| 7 | `08_Optimizaciya_RAN` — конспект, схемы 25/26, видео; `11_Shpargalka` |
| 8 | `09_Troubleshooting` — конспекты, схемы 37/38, видео |
| 9 | `10_Interfejsy_dlya_novichka` — пособие, схема 11, видео |
| 10 | Повторение: `03_Diameter` (подробный) + `08_Optimizaciya_RAN` (вопросы для защиты) |

## Ключевые спецификации (сводно)

- Архитектура EPS: TS 23.401; PCC: TS 23.203, TS 29.212, TS 29.213, TS 29.214
- CSFB: TS 23.272, TS 29.118; SRVCC: TS 23.216, TS 29.280, TS 23.237
- 5G: TS 23.501, TS 23.502; мультиконнективность: TS 37.340
- RAN-протоколы: TS 36.413/36.423 (S1AP/X2AP), TS 38.413/38.423 (NGAP/XnAP)
- Диаметр-тарификация: RFC 4006, RFC 6733, TS 32.299
- IMS: TS 23.228, TS 24.229, TS 29.228/29.229, TS 29.328/29.329
- MDT/Trace: TS 37.320, TS 32.422; RCAF: TS 29.217
- KPI/PM: TS 32.425, TS 32.450, TS 28.552; UPF/PFCP: TS 29.244

## Оговорка

Материал учебный, составлен по общедоступным 3GPP/GSMA-спецификациям и практике.
Значения параметров и процедуры могут отличаться в зависимости от релиза 3GPP,
вендора и конфигурации конкретной сети — перед применением на сети сверяйте с актуальной документацией.
