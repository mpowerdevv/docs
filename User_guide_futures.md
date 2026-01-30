# Binance Perp Futures Analytics (Scanners + Bots, multi-user)

Одностраничное приложение (**React/Vite**) + backend (**Node.js/Express**) для анализа **Binance USDT Perpetual Futures** и запуска **сканеров/ботов**.

Основные возможности:
- Список USDT-perp инструментов, поиск как в Binance
- График свечей (TradingView-like UI на `lightweight-charts`) + переключение интервалов
- Индикаторы на графике: EMA/SMA, MACD, RSI, ADX, зоны
- **Scanner highlighting**: кликаете найденную пару — бары, удовлетворяющие условию, подсвечиваются на графике
- **Runtime-конфигурируемые Scanner’ы** через JSON DSL (группы AND/OR, правила индикаторов, cross/экстремумы)
- **Bots** (paper/live) со StopLoss / TakeProfit / Trailing Profit/Loss
- **Multi-user auth** (MongoDB): админ создаёт пользователей, выдаёт роли, отключает, сбрасывает пароли
- Пользовательские настройки и ключи (Binance/OpenAI) хранятся в MongoDB (опционально шифруются)

> ⚠️ Важно про данные рынка: публичные market streams/klines берутся **только из LIVE** (server-wide).
> DEMO/LIVE влияет только на **signed/private endpoints** (аккаунт/ордера/позиции) на уровне пользователя/бота.

---


# Использование (кейсы)

## Термины
- **Scanner (Сканер)** — “фильтр рынка”: пробегает по множеству символов, применяет DSL-условие и отдаёт совпадения. **Не торгует**.
- **Bot (Бот)** — “исполнитель”: по сигналу открывает/закрывает позицию(и) в режимах Backtest/Paper/Live.

---

## Кейс 1. Создание Scanner

### Как сканер работает
- Берёт **universe**: top-N USDT-perp по `quoteVolume` (ограничение `universeLimit`).
- Вычисляет индикаторы по таймфреймам/окнам, требуемым DSL.
- Фильтрует символы по `condition`.
- Результат — таблица `rows` + метаданные, доступно по WS/API.

### Шаги (через UI)
1) **Scanners → New** (или Clone)
2) Настройте:
   - **Name**
   - **Universe limit** (сколько символов проверять)
   - **Columns** (какие поля показывать в таблице)
3) Соберите **Condition (DSL)** (группы AND/OR + правила)
4) **Save**
5) **Start**

### Параметры расписания сканера
- `runOnBarClose = true` (дефолт)
  - Сканер запускается **на закрытии бара** минимального таймфрейма, который используется в условии (`+250мс` задержка).
- `runOnBarClose = false`
  - Сканер запускается раз в `scheduleMs` (деф. `30000`).

---

## Кейс 2. Создание Bot → Backtest

Backtest запускается когда `backtestEnabled=true`. Логика “прогоняет” историю свечей и симулирует сделки.

### Шаги (через UI)
1) **Bots → New Bot**
2) Настройте:
   - `pairs`, `direction`, `leverage`, `sizing`
   - `openCondition`
   - выходы: `stopLoss`, `takeProfit`, `trailingLoss`, `trailingProfit`
3) Включите `backtestEnabled=true`
4) Задайте диапазон `startTime/endTime` (если не задать — **сегодня 00:00 → сейчас**)
5) **Start Backtest** → смотрите отчёт/сделки

---

## Кейс 3. Bot → Paper Run (симуляция в реальном времени)

- `tradeMode = PAPER`
- На биржу ордера **не отправляются**
- Баланс и позиции ведутся виртуально (старт: `paperStartBalanceUSDT`, деф. `10000`)

### Шаги
1) `tradeMode=PAPER`, `backtestEnabled=false`
2) Выставьте `paperStartBalanceUSDT`
3) **Start** → бот начнёт открывать/закрывать “бумажные” позиции

---

## Кейс 4. Bot → Live Run (реальная торговля)

⚠️ Реальные ордера. Начинайте с DEMO ключей.

### Что нужно заранее
1) В **User settings** добавить Binance ключи:
   - отдельно для **DEMO** и **LIVE**
2) Убедиться, что бот настроен корректно (пары, плечо, сайзинг, выходы)

### Шаги
1) Установить `tradeMode=LIVE`
2) Установить `accountMode=DEMO` (рекомендовано сначала) или `LIVE`
3) **Start**

Backend автоматически поднимает User Data Stream (WS) для подписок аккаунта.

---

# Scanner DSL (структура и параметры)

## Типы узлов

### Group

```json
{ "type":"group", "op":"AND|OR|AND_SAME_BAR|OR_SAME_BAR", "rules":[ ... ], "enabled":true, "not":false }
```

- `enabled=false` — узел игнорируется
- `not=true` — инверсия результата группы
- `AND_SAME_BAR / OR_SAME_BAR` — **всё поддерево** (включая ref) должно использовать **ровно один timeframe**

### Rule

Минимальный вид:

```json
{ "type":"rule", "indicator":"MACD", "timeframe":"1m", "op":"LT", "field":"MACD", "value":0 }
```

Общие поля:
- `enabled?: boolean` — если `false`, правило игнорируется
- `not?: boolean` — инверсия результата правила
- `indicator`: `MACD | RSI | SMA | EMA | ADX | VWAP | PRICE`
- `timeframe`: любой timeframe, который есть в runtime store (на практике UI даёт `1m|5m|15m|30m|1h`, но backend поддерживает больше)
- `params` — параметры индикатора (см. ниже)
- `field` — что сравниваем:
  - MACD: `MACD | signal | histogram`
  - PRICE: `open | high | low | close | volume`
- `op` (оператор):
  - `LT, GT, LTE, GTE, CROSS, MAX, MIN, ABS_DIFF_LT, ABS_DIFF_GT, PCT_DIFF_LT, PCT_DIFF_GT`
- `value?: number` — число для сравнения (если сравнение с константой)
- `ref?: { indicator,timeframe,params?,field?,barOffset? }` — сравнение с другой серией
- `pctOffset?: number` — процентный “сдвиг” RHS, например `-0.003` = разрешить 0.3% ниже
- `barOffset?: number` — сдвиг LHS (0 = текущий формирующийся бар, 1 = последний закрытый…)
- `lookbackBars?: number` — искать совпадение в последних N барах (обычно `>=1`, нормализуется до 1..2000)
- `windowBars?: number` — для `MAX/MIN`: окно экстремума (до 5000)
  - для MAX/MIN `lookbackBars` трактуется как “экстремум был в пределах последних N баров”; допускается `0` (только текущий бар)

## Параметры индикаторов (`params`)
- EMA / SMA: `{ "period": 20 }`
- RSI: `{ "period": 14 }`
- ADX: `{ "period": 14 }`
- MACD: `{ "fastPeriod": 12, "slowPeriod": 26, "signalPeriod": 9 }`

## Примеры

**MACD < 0 на 1m:**

```json
{
  "type":"rule",
  "indicator":"MACD",
  "timeframe":"1m",
  "params":{"fastPeriod":12,"slowPeriod":26,"signalPeriod":9},
  "field":"MACD",
  "op":"LT",
  "value":0,
  "lookbackBars":1
}
```

**MACD пересёк signal на 5m:**

```json
{
  "type":"rule",
  "indicator":"MACD",
  "timeframe":"5m",
  "params":{"fastPeriod":12,"slowPeriod":26,"signalPeriod":9},
  "op":"CROSS",
  "a":"MACD",
  "b":"signal",
  "direction":"ANY",
  "lookbackBars":10
}
```

---

# Bots: параметры и влияние (справочник)

Ниже перечислены ключевые параметры бота, их значения и то, **на что они реально влияют**.

## 1) Основные поля
- `direction`: `LONG|SHORT` (деф. `LONG`)
  - Определяет сторону позиции и какие BUY/SELL используются при открытии/закрытии.
- `tradeMode`: `PAPER|LIVE` (деф. `PAPER`)
  - PAPER: симуляция
  - LIVE: реальные ордера
- `accountMode`: `DEMO|LIVE` (деф. нормализуется в `LIVE`, но дефолтный бот создаётся с `DEMO`)
  - Используется **только** в `tradeMode=LIVE` для выбора signed endpoints.
- `pairs: string[]`
  - Какие символы бот будет торговать.
- `pairBars: Record<symbol, timeframe>` (деф. `"1m"` на пару)
  - Таймфрейм “стратегического бара” **для входов** и некоторых режимов оценок условий.
  - Разрешённые значения в engine: `1m|5m|15m|30m|1h`.

## 2) Плечо и сайзинг
- `leverage: 1..125` (деф. `10`)
- `sizing.mode: PERCENT|FIXED`
  - `PERCENT`: `sizing.percent` (деф. `10`) — процент от доступных USDT под маржу
  - `FIXED`: `sizing.fixedUSDT` (деф. `50`) — фикс USDT маржи
- `paperStartBalanceUSDT` (деф. `10000`) — стартовый баланс для PAPER/Backtest.

## 3) Поведение при существующих позициях (важно для LIVE)
- `ignoreRepeatOpenSignal: boolean` (деф. `false`)
  - Если позиция уже есть в нужную сторону — повторный сигнал “open” не увеличит позицию.
- `oppositeSignalAction: REDUCE|CLOSE|CLOSE_AND_REVERSE` (деф. `REDUCE`)
  - Что делать, если на аккаунте уже есть **противоположная** нетто-позиция по символу:
    - `REDUCE` — уменьшить reduceOnly и **не открывать** новую
    - `CLOSE` — закрыть reduceOnly и **не открывать** новую
    - `CLOSE_AND_REVERSE` — закрыть и затем открыть в сторону бота

## 4) Таймаут позиции
- `positionExpiryMs` (деф. `0`)
  - `0` = без лимита. Если >0 — позиция будет принудительно закрыта по истечению времени (если сама не закрылась SL/TP/Trailing).

## 5) Частота быстрых проверок выхода
- `settings.realtimeExitCheckMs` (деф. `200`, ограничение 100..10000)
  - Как часто крутится “fast loop” для trailing/SL/TP в реальном времени.
  - Влияет на скорость реакции и нагрузку CPU.

---

## 6) StopLoss / TakeProfit (поля и дефолты)

Общая структура:

```json
{
  "enabled": true,
  "op": "OR",
  "roiPct": -2,
  "condition": null,
  "confirmMs": 2000,
  "oncePerBar": false
}
```

- `enabled` — включён/выключен
- `roiPct` — порог ROI в процентах (учитывает leverage)
  - дефолт SL: `-2`
  - дефолт TP: `+3`
- `condition` — дополнительное DSL-условие (если группа пустая — трактуется как `null`)
- `op` — как комбинировать `roiPct` и `condition` если оба заданы:
  - `OR` (дефолт) или `AND`
- `confirmMs` — анти-дребезг (в мс)
  - SL дефолт `2000`, TP дефолт `0`
- `oncePerBar` — если `true`, проверять только раз на бар (меньше шума, но медленнее реакция)

---

## 7) Trailing Profit / Trailing Loss

### trailingProfit (дефолты)

```json
{
  "enabled": false,
  "monitor": "REALTIME",
  "closedBarsAgo": 1,
  "activateRoiPct": 1,
  "pullbackRoiPct": 0.5,
  "confirmMs": 0,
  "checkEveryMs": 0
}
```

### trailingLoss (дефолты)

```json
{
  "enabled": false,
  "mode": "UNDERWATER_RECOVERY",
  "monitor": "REALTIME",
  "closedBarsAgo": 1,
  "activateRoiPct": -0.5,
  "pullbackRoiPct": 0.3,
  "resetIfRoiAbovePct": 0,
  "confirmMs": 0,
  "checkEveryMs": 0
}
```

Пояснения:
- `monitor`:
  - `REALTIME` — по текущей цене (включая формирующуюся свечу)
  - `CLOSED` — по close закрытого бара (`closedBarsAgo`)
- `activateRoiPct` — с какого ROI включать trailing
- `pullbackRoiPct` — насколько ROI должен откатиться от лучшего значения, чтобы закрыть
- `checkEveryMs` — троттлинг проверок (0 = каждый тик)
- `confirmMs` — подтверждение перед закрытием
- `trailingLoss.mode`:
  - `PULLBACK_FROM_PEAK` — классический trailing от пика
  - `UNDERWATER_RECOVERY` — трекинг восстановления “под водой” (если ROI <= activateRoiPct), закрытие при откате от “пика восстановления”
- `resetIfRoiAbovePct` — при восстановлении ROI до этого уровня underwater-состояние сбрасывается

---

# Backtest: настройки исполнения (реализм)

Настройки лежат в `bot.backtestSettings`. Дефолты “консервативные, но похожие на реальность”.

- `fillMode`: `NEXT_BAR_OPEN` (деф) | `RANDOM_IN_BAR`
- `pathModel`: `DIRECTIONAL` (деф) | `RANDOM`
  - Используется, если `RANDOM_IN_BAR` или требуется моделировать путь свечи.
- `takerFeeBps` (деф. `4` = 0.04%)
- `spreadBps` (деф. `1` = 0.01%) → используется как `halfSpreadRate`
- `slippageBps` (деф. `1` = 0.01%)
- `fundingBpsPer8h` (деф. `0`) — если 0, funding отключён
- `seed` (деф. `1337`) — детерминированная случайность
- `trailingClosedTimeframe`: `STRATEGY` (деф) | `BASE`
- `exitConditionEvalTf`: `STRATEGY` (деф) | `BASE`

---

