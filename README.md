# adam-indicators

Кратко: репозиторий содержит индикатор MOMENTUM 2 и стратегию для бэктеста на его сигналах.

Файлы:
- `first.pinescript` — индикатор MOMENTUM 2 (сигналы и визуал).
- `strategy.pinescript` — стратегия для бэктеста на тех же сигналах.
- `adam.pinescript` — стратегия, использующая сигналы индикатора через скрытые серии.

## first.pinescript — индикатор MOMENTUM 2

Кратко: это Pine Script v6 индикатор для TradingView, который ищет **импульсные свечи** относительно «нормы» размера тела свечи и, при совпадении фильтров тренда и моментума, рисует **long/short сигналы** в отдельной панели (overlay=false). Также рисует линию моментума, фон, метки цены входа и отладочные метки причин блокировки.

### Что делает
- Считает «норму» размера тела свечи в процентах и определяет импульс.
- Строит линию моментума на базе RSI и EMA.
- Применяет фильтры тренда (EMA200, ADX, ATR‑режим) и фильтр моментума.
- В зависимости от режима определяет точку входа (касание цели по цене или подтверждение по закрытию).
- Визуализирует сигналы, цену входа и статус импульса в таблице.

### Что не делает
- Не является стратегией и не исполняет сделки.
- Не рассчитывает выходы, стоп‑лоссы, тейк‑профиты, риск‑менеджмент и размер позиции.
- Не учитывает комиссии/проскальзывание.
- Не гарантирует совпадение исторических и реальных сигналов в режимах с внутрисвечной логикой (см. ниже).

## Математика (TeX)

### Моментум (Fast Wave)

$$
RSI_t = RSI(src,\, fastLen)
$$
$$
FastWave_t = EMA(RSI_t - 50,\, 3)
$$

Дополнительно:
- рост: $FastWave_t > FastWave_{t-1}$
- падение: $FastWave_t < FastWave_{t-1}$
- допустимый диапазон: $|FastWave_t| \le maxFastWave$

### Импульсная свеча

Процент тела текущей свечи:
$$
BodyPct_t = \frac{|Close_t - Open_t|}{Open_t} \cdot 100
$$

Предварительная «норма» (на закрытых свечах):
$$
PreNormal_t = SMA\left( \frac{|Close_{t-1} - Open_{t-1}|}{Open_{t-1}} \cdot 100,\, impulseLookback \right)
$$

Признак импульса:
$$
ImpulseRaw_t = BodyPct_t > PreNormal_t \cdot impulseMult
$$

Норма с исключением импульсных свечей (если включено `excludeImpulse`):
$$
FilteredSize_t = \begin{cases}
\text{NA}, & \text{если } ImpulseRaw_t \text{ и excludeImpulse}\\
BodyPct_t, & \text{иначе}
\end{cases}
$$

$$
NormalSize_t = SMA(FilteredSize_{t-1},\, impulseLookback)
$$

Если `NormalSize` недоступен (NA), используется `PreNormal`:
$$
SafeNormal_t = \begin{cases}
NormalSize_t, & \text{если } NormalSize_t \ne NA\\
PreNormal_t, & \text{иначе}
\end{cases}
$$

Цель импульса и уровни касания:
$$
ImpulseTargetPct_t = SafeNormal_t \cdot impulseMult
$$
$$
PriceUp_t = Open_t \cdot \left(1 + \frac{ImpulseTargetPct_t}{100}\right)
$$
$$
PriceDown_t = Open_t \cdot \left(1 - \frac{ImpulseTargetPct_t}{100}\right)
$$

### Фильтры тренда

$$
EMA_t = EMA(Close_t,\, emaLen)
$$
$$
ADX_t = DMI(adxLen,\, adxSmooth).ADX
$$
$$
ATR_t = ATR(atrLen),\quad ATRMA_t = EMA(ATR_t,\, atrMaLen)
$$

Разрешение входа:
$$
AllowLong = (\neg useEMA200 \lor Close_t > EMA_t) \land (\neg useADX \lor ADX_t > adxMin) \land (\neg useATRRegime \lor ATR_t > ATRMA_t)
$$
$$
AllowShort = (\neg useEMA200 \lor Close_t < EMA_t) \land (\neg useADX \lor ADX_t > adxMin) \land (\neg useATRRegime \lor ATR_t > ATRMA_t)
$$

### Подтверждение моментума (режим MAX)

Если `impulseMode = EASY`, моментум не ограничивает сигналы. Если `impulseMode = MAX`, то:

Long:
$$
(FastWave_t > 0 \;\text{или}\; FastWave_t > -fwZone) \land FastWave_t > FastWave_{t-1} \land |FastWave_t| \le maxFastWave
$$

Short:
$$
(FastWave_t < 0 \;\text{или}\; FastWave_t < fwZone) \land FastWave_t < FastWave_{t-1} \land |FastWave_t| \le maxFastWave
$$

## Логика сигналов

### Режимы определения импульса (`impulseDetectMode`)

- **По цене**: сигнал появляется после касания `PriceUp/PriceDown` (см. `touchMode`). Фильтры фиксируются на открытии свечи.
- **LIVE**: касание `PriceUp/PriceDown` (см. `touchMode`), но фильтры проверяются динамически в момент касания.
- **По закрытию**: импульс определяется по закрытию (тело свечи), сигнал формируется только после закрытия.

### Касание цели (`touchMode`)

- **По последней цене** (по умолчанию): приоритетно проверка по `close`; если касание могло потеряться из-за агрегированных тиков/пересчёта, применяется fallback по `high/low`.
- **По high/low**: касание фиксируется по `high/low` (включая тени).

### Направление
- Long — если импульс вверх (`PriceUp`/зеленая свеча) и фильтры + моментум подтверждают.
- Short — если импульс вниз (`PriceDown`/красная свеча) и фильтры + моментум подтверждают.

### Цена входа
- В режимах **По цене** и **LIVE** цена входа равна уровню касания `PriceUp/PriceDown`.
- В режиме **По закрытию** цена входа равна целевому уровню `PriceUp/PriceDown`, при котором свеча стала импульсной.

## strategy.pinescript — стратегия для бэктеста

Кратко: стратегия использует тот же алгоритм сигналов, что и индикатор, и открывает позиции по сигналам. Дополнительно поддерживает SL/TP и закрытие по противоположному сигналу.

Что делает:
- Входит в `Long/Short` по сформированному сигналу.
- Может закрывать позицию по противоположному сигналу.
- Может ставить `Stop Loss` и `Take Profit` как процент от средней цены позиции.

Что не делает:
- Не моделирует комиссии/проскальзывание (задаются в свойствах стратегии на TV).
- Не гарантирует вход ровно по цене касания в режимах внутрисвечной логики; вход рыночный на баре сигнала.

## adam.pinescript — стратегия, использующая индикатор

Кратко: стратегия не повторяет код индикатора, а читает его сигналы через скрытые `plot`‑серии из `first.pinescript`.

Как подключить:
1. Добавьте `first.pinescript` на график.
2. Добавьте `adam.pinescript` на график.
3. В настройках `adam.pinescript` выберите источники:
   - `Long сигнал (из индикатора)` → `Long Signal (series)`
   - `Short сигнал (из индикатора)` → `Short Signal (series)`
4. Включите `Сигналы подключены`.

Логика:
- Вход только когда позиция закрыта.
- Выход по `SL/TP` (процент от средней цены позиции).
- По умолчанию: `TP=1.8%`, `SL=0.6%`.

## Параметры по умолчанию (индикатор)

| Параметр | Значение |
| --- | --- |
| `src` | `close` |
| `fastLen` | `14` |
| `maxFastWave` | `20.0` |
| `impulseLookback` | `20` |
| `impulseMult` | `1.3` |
| `excludeImpulse` | `true` |
| `impulseDetectMode` | `По закрытию` |
| `touchMode` | `По последней цене` |
| `impulseMode` | `EASY` |
| `useFWZone` | `false` |
| `fwZone` | `5.0` |
| `useEMA200` | `true` |
| `emaLen` | `200` |
| `useADX` | `true` |
| `adxLen` | `7` |
| `adxSmooth` | `14` |
| `adxMin` | `22.0` |
| `useATRRegime` | `true` |
| `atrLen` | `7` |
| `atrMaLen` | `50` |
| `showFastWave` | `true` |
| `showSignals` | `true` |
| `showEntryPrice` | `true` |
| `showDebug` | `false` |
| `maxEntryLabels` | `200` |

## Параметры по умолчанию (стратегия)

| Параметр | Значение |
| --- | --- |
| `enableLong` | `true` |
| `enableShort` | `true` |
| `closeOnOpposite` | `true` |
| `useSL` | `true` |
| `useTP` | `true` |
| `slPct` | `1.0` |
| `tpPct` | `2.0` |

## Параметры по умолчанию (adam.pinescript)

| Параметр | Значение |
| --- | --- |
| `useSL` | `true` |
| `useTP` | `true` |
| `slPct` | `0.6` |
| `tpPct` | `1.8` |
| `signalsConnected` | `false` |

## Примеры использования и типовые сценарии настройки

- Консервативный тренд‑фоллоуинг с подтверждением закрытия. Настройки: `impulseDetectMode=По закрытию`, `impulseMode=MAX`, `useFWZone=false`, `impulseMult=1.5`, `adxMin=25`, остальные по умолчанию.
- Более быстрые сигналы внутри свечи (выше риск ложных). Настройки: `impulseDetectMode=LIVE`, `impulseMode=EASY`, `useADX=false` или `adxMin=18`, `impulseMult=1.2`, остальные по умолчанию.
- Отсев «шумных» импульсов на низкой волатильности. Настройки: `useATRRegime=true`, `atrMaLen=100`, `impulseMult=1.4`, `excludeImpulse=true`, остальные по умолчанию.
- Бэктест с фиксированным RR. Настройки: `useSL=true`, `slPct=1.0`, `useTP=true`, `tpPct=2.0`, `closeOnOpposite=false` (чтобы закрытие было только по SL/TP).
- Бэктест через индикаторные сигналы. Настройки: используйте `adam.pinescript`, подключите источники `Long/Short Signal (series)` и оставьте `TP=1.8%`, `SL=0.6%`.

## Визуал
- Линия моментума `FastWave` и базовая линия 0.
- Треугольники long/short сигналов.
- Фон при наличии сигнала.
- Метки цены входа.
- Отладочные метки причин блокировки (`EMA/ADX/ATR/FMW`).
- Таблица с нормой свечи, величиной импульса и статусом (только для последней свечи).

## Замечания
- При смене таймфрейма внутренние состояния сбрасываются.
- В режимах **По цене** и **LIVE** используется внутрисвечная логика; на истории сигнал определяется по `high/low` свечи и может отличаться от реального появления в моменте.
