# БАГ #47: Отсутствует поддержка миллисекунд в UTC ISO timestamps

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1884](https://github.com/tradingview/lightweight-charts/issues/1884)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Feature request  
> **Последнее обновление:** Май 2025

## 📋 Описание проблемы

### Суть проблемы

Библиотека **не поддерживает нативно timestamps с миллисекундной точностью** в формате UTC ISO 8601 (например, `"2025-05-06T20:50:35.018Z"`). Попытки конвертировать такие timestamps через `new Date().getTime()` или аналогичные методы приводят к **некорректному рендерингу** графика.

### Детали

1. **Текущие ограничения:**
   - Поддерживается только **секундная точность** для unix timestamps
   - Строковые даты в ISO формате работают, но **миллисекунды игнорируются**
   - `Date.getTime()` возвращает миллисекунды, но библиотека ожидает секунды

2. **Проблемы при использовании миллисекунд:**
   - Данные отображаются в неправильном порядке
   - Timestamps интерпретируются как далёкое будущее
   - Несколько точек данных "схлопываются" в одну при округлении до секунд

3. **Когда это критично:**
   - High-frequency trading (HFT) данные
   - Tick-by-tick визуализация
   - Real-time streaming с высокой частотой обновлений
   - Научные данные с миллисекундной точностью

### Сценарии возникновения

```typescript
// ❌ ПРОБЛЕМА 1: ISO string с миллисекундами
const data = [
  { time: '2025-05-06T20:50:35.018Z', value: 100 }, // .018 игнорируется
  { time: '2025-05-06T20:50:35.512Z', value: 101 }, // .512 игнорируется
  { time: '2025-05-06T20:50:35.999Z', value: 102 }, // .999 игнорируется
];
// Результат: все 3 точки имеют одинаковый time = '2025-05-06T20:50:35'!

// ❌ ПРОБЛЕМА 2: Unix timestamp в миллисекундах
const dataWithMs = [
  { time: 1714942235018, value: 100 }, // ms timestamp
  { time: 1714942235512, value: 101 },
  { time: 1714942235999, value: 102 },
];
// Результат: интерпретируется как год ~54000+!

// ❌ ПРОБЛЕМА 3: Потеря данных при конвертации
const tickData = [
  { time: Math.floor(Date.now() / 1000), value: 100 },
  { time: Math.floor(Date.now() / 1000), value: 101 }, // тот же timestamp!
];
// Вторая точка перезапишет первую
```

### Реальные сценарии

1. **High-Frequency Trading (HFT):**
   - Тысячи сделок в секунду
   - Каждый tick должен отображаться отдельно

2. **Криптобиржи:**
   - WebSocket feeds с миллисекундной точностью
   - Order book updates несколько раз в секунду

3. **IoT и научные данные:**
   - Сенсоры с высокой частотой опроса
   - Лабораторные измерения

4. **Market microstructure анализ:**
   - Изучение bid-ask spread dynamics
   - Анализ latency

## 🔍 Найденные решения

### Решение 1: Конвертация ms → seconds (⭐ Стандартный подход)

**Оценка:** ⭐⭐⭐ (3/5) - Простое, но с потерей точности

```typescript
import { Time, LineData } from 'lightweight-charts';

interface TickData {
  timestamp: number; // миллисекунды
  price: number;
}

/**
 * Конвертирует миллисекунды в секунды для lightweight-charts
 * ⚠️ ВАЖНО: Теряется миллисекундная точность!
 */
function convertMsToSeconds(timestampMs: number): Time {
  return Math.floor(timestampMs / 1000) as Time;
}

/**
 * Конвертирует ISO строку в unix timestamp (секунды)
 */
function isoToUnixSeconds(isoString: string): Time {
  return Math.floor(new Date(isoString).getTime() / 1000) as Time;
}

// Использование
const rawData: TickData[] = [
  { timestamp: 1714942235018, price: 100 },
  { timestamp: 1714942235512, price: 101 },
  { timestamp: 1714942236100, price: 102 },
];

const chartData: LineData[] = rawData.map(item => ({
  time: convertMsToSeconds(item.timestamp),
  value: item.price,
}));

series.setData(chartData);
```

**Плюсы:**
- Простая реализация
- Работает с библиотекой

**Минусы:**
- Потеря миллисекундной точности
- Данные в пределах одной секунды "схлопываются"

---

### Решение 2: Агрегация данных с сохранением деталей (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Сохранение информации через агрегацию

```typescript
import { Time, CandlestickData, LineData } from 'lightweight-charts';

interface TickData {
  timestamp: number; // ms
  price: number;
  volume?: number;
}

interface AggregatedTick {
  time: Time;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  tickCount: number;
  // Сохраняем оригинальные данные для tooltip
  originalTicks: TickData[];
}

/**
 * Агрегирует тики в секундные бары
 */
function aggregateTicksToSeconds(ticks: TickData[]): AggregatedTick[] {
  const aggregated = new Map<number, AggregatedTick>();
  
  for (const tick of ticks) {
    const secondTimestamp = Math.floor(tick.timestamp / 1000);
    
    if (!aggregated.has(secondTimestamp)) {
      aggregated.set(secondTimestamp, {
        time: secondTimestamp as Time,
        open: tick.price,
        high: tick.price,
        low: tick.price,
        close: tick.price,
        volume: tick.volume || 0,
        tickCount: 1,
        originalTicks: [tick],
      });
    } else {
      const bar = aggregated.get(secondTimestamp)!;
      bar.high = Math.max(bar.high, tick.price);
      bar.low = Math.min(bar.low, tick.price);
      bar.close = tick.price;
      bar.volume += tick.volume || 0;
      bar.tickCount++;
      bar.originalTicks.push(tick);
    }
  }
  
  return Array.from(aggregated.values())
    .sort((a, b) => (a.time as number) - (b.time as number));
}

/**
 * Конвертирует агрегированные данные в формат свечей
 */
function toCandlestickData(aggregated: AggregatedTick[]): CandlestickData[] {
  return aggregated.map(bar => ({
    time: bar.time,
    open: bar.open,
    high: bar.high,
    low: bar.low,
    close: bar.close,
  }));
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const tickStream: TickData[] = [
  { timestamp: 1714942235018, price: 100.5, volume: 10 },
  { timestamp: 1714942235125, price: 100.7, volume: 5 },
  { timestamp: 1714942235512, price: 100.3, volume: 15 },
  { timestamp: 1714942235999, price: 100.8, volume: 8 },
  { timestamp: 1714942236100, price: 101.0, volume: 20 },
];

const aggregated = aggregateTicksToSeconds(tickStream);
const candlestickData = toCandlestickData(aggregated);

const series = chart.addCandlestickSeries();
series.setData(candlestickData);

// Для tooltip можно использовать originalTicks
chart.subscribeCrosshairMove((param) => {
  if (param.time) {
    const bar = aggregated.find(b => b.time === param.time);
    if (bar) {
      console.log(`${bar.tickCount} ticks in this second`);
      console.log('Tick details:', bar.originalTicks);
    }
  }
});
```

**Плюсы:**
- Сохраняется вся информация через агрегацию
- OHLC показывает внутрисекундную динамику
- Оригинальные данные доступны для tooltip

**Минусы:**
- Больше кода
- Дополнительное потребление памяти

---

### Решение 3: Нормализованная временная ось (Custom Time Scale)

**Оценка:** ⭐⭐⭐⭐ (4/5) - Для точного отображения микроструктуры

```typescript
import { 
  createChart, 
  IChartApi,
  ISeriesApi,
  Time,
  LineData,
  TickMarkFormatter
} from 'lightweight-charts';

interface MillisecondData {
  timestampMs: number;
  value: number;
}

/**
 * Создаёт normalized time для tick-by-tick данных
 * Каждый тик получает уникальный индекс
 */
class MillisecondTimeNormalizer {
  private indexToTimestamp: Map<number, number> = new Map();
  private timestampToIndex: Map<number, number> = new Map();
  private currentIndex = 0;
  
  /**
   * Нормализует данные, присваивая уникальный индекс каждому тику
   */
  normalize(data: MillisecondData[]): LineData[] {
    return data.map(item => {
      const index = this.currentIndex++;
      this.indexToTimestamp.set(index, item.timestampMs);
      this.timestampToIndex.set(item.timestampMs, index);
      
      return {
        time: index as Time,
        value: item.value,
      };
    });
  }
  
  /**
   * Получает оригинальный timestamp по индексу
   */
  getTimestamp(index: number): number | undefined {
    return this.indexToTimestamp.get(index);
  }
  
  /**
   * Создаёт форматтер для time scale
   */
  createTickMarkFormatter(): TickMarkFormatter {
    return (time: Time): string => {
      const timestamp = this.indexToTimestamp.get(time as number);
      if (!timestamp) return '';
      
      const date = new Date(timestamp);
      const hours = date.getHours().toString().padStart(2, '0');
      const minutes = date.getMinutes().toString().padStart(2, '0');
      const seconds = date.getSeconds().toString().padStart(2, '0');
      const ms = date.getMilliseconds().toString().padStart(3, '0');
      
      return `${hours}:${minutes}:${seconds}.${ms}`;
    };
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const normalizer = new MillisecondTimeNormalizer();

const chart = createChart(container, {
  timeScale: {
    tickMarkFormatter: normalizer.createTickMarkFormatter(),
    timeVisible: true,
  },
  localization: {
    timeFormatter: (time: Time) => {
      const ts = normalizer.getTimestamp(time as number);
      if (!ts) return '';
      return new Date(ts).toISOString();
    },
  },
});

const rawData: MillisecondData[] = [
  { timestampMs: 1714942235018, value: 100 },
  { timestampMs: 1714942235125, value: 101 },
  { timestampMs: 1714942235234, value: 100.5 },
  { timestampMs: 1714942235345, value: 102 },
];

const series = chart.addLineSeries();
series.setData(normalizer.normalize(rawData));

// Tooltip с миллисекундной точностью
chart.subscribeCrosshairMove((param) => {
  if (param.time !== undefined) {
    const timestamp = normalizer.getTimestamp(param.time as number);
    if (timestamp) {
      const date = new Date(timestamp);
      console.log('Exact time:', date.toISOString()); // Показывает миллисекунды
    }
  }
});
```

**Плюсы:**
- Каждый тик отображается отдельно
- Сохраняется полная точность
- Миллисекунды видны в tooltip и time scale

**Минусы:**
- Time scale показывает индексы, а не реальное время (нужен форматтер)
- Нельзя использовать стандартный time-based scroll

---

### Решение 4: Microsecond-aware data wrapper

**Оценка:** ⭐⭐⭐⭐ (4/5) - Enterprise-grade решение

```typescript
import { Time, LineData, ISeriesApi } from 'lightweight-charts';

interface HighPrecisionData<T = number> {
  timestamp: string; // ISO 8601 с миллисекундами
  value: T;
}

interface StoredTickData<T> {
  chartTime: Time;
  originalTimestamp: string;
  value: T;
  subsecondIndex: number; // порядок внутри секунды
}

/**
 * Wrapper для работы с миллисекундными данными
 */
class HighPrecisionSeriesWrapper<T = number> {
  private storage: Map<number, StoredTickData<T>[]> = new Map();
  private series: ISeriesApi<'Line'>;
  
  constructor(series: ISeriesApi<'Line'>) {
    this.series = series;
  }
  
  /**
   * Добавляет данные с сохранением миллисекундной точности
   */
  setData(data: HighPrecisionData<T>[]): void {
    this.storage.clear();
    
    // Группируем по секундам
    for (const item of data) {
      const date = new Date(item.timestamp);
      const secondTimestamp = Math.floor(date.getTime() / 1000);
      
      if (!this.storage.has(secondTimestamp)) {
        this.storage.set(secondTimestamp, []);
      }
      
      const ticks = this.storage.get(secondTimestamp)!;
      ticks.push({
        chartTime: secondTimestamp as Time,
        originalTimestamp: item.timestamp,
        value: item.value,
        subsecondIndex: ticks.length,
      });
    }
    
    // Для графика используем последнее значение в каждой секунде
    const chartData: LineData[] = [];
    
    for (const [timestamp, ticks] of this.storage) {
      const lastTick = ticks[ticks.length - 1];
      chartData.push({
        time: timestamp as Time,
        value: lastTick.value as number,
      });
    }
    
    chartData.sort((a, b) => (a.time as number) - (b.time as number));
    this.series.setData(chartData);
  }
  
  /**
   * Обновляет данные в реальном времени
   */
  update(item: HighPrecisionData<T>): void {
    const date = new Date(item.timestamp);
    const secondTimestamp = Math.floor(date.getTime() / 1000);
    
    if (!this.storage.has(secondTimestamp)) {
      this.storage.set(secondTimestamp, []);
    }
    
    const ticks = this.storage.get(secondTimestamp)!;
    ticks.push({
      chartTime: secondTimestamp as Time,
      originalTimestamp: item.timestamp,
      value: item.value,
      subsecondIndex: ticks.length,
    });
    
    // Обновляем только последнее значение для секунды
    this.series.update({
      time: secondTimestamp as Time,
      value: item.value as number,
    });
  }
  
  /**
   * Получает все тики для конкретной секунды
   */
  getTicksForSecond(secondTimestamp: number): StoredTickData<T>[] {
    return this.storage.get(secondTimestamp) || [];
  }
  
  /**
   * Получает точное время для tooltip
   */
  getExactTimestamp(secondTimestamp: number, subsecondIndex?: number): string | null {
    const ticks = this.storage.get(secondTimestamp);
    if (!ticks || ticks.length === 0) return null;
    
    const index = subsecondIndex ?? ticks.length - 1;
    return ticks[index]?.originalTimestamp || null;
  }
  
  /**
   * Экспортирует данные с полной точностью
   */
  exportFullPrecisionData(): HighPrecisionData<T>[] {
    const result: HighPrecisionData<T>[] = [];
    
    for (const ticks of this.storage.values()) {
      for (const tick of ticks) {
        result.push({
          timestamp: tick.originalTimestamp,
          value: tick.value,
        });
      }
    }
    
    return result.sort((a, b) => 
      new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
    );
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const series = chart.addLineSeries();
const wrapper = new HighPrecisionSeriesWrapper(series);

// Загрузка данных с полной точностью
wrapper.setData([
  { timestamp: '2025-05-06T20:50:35.018Z', value: 100 },
  { timestamp: '2025-05-06T20:50:35.125Z', value: 101 },
  { timestamp: '2025-05-06T20:50:35.512Z', value: 100.5 },
  { timestamp: '2025-05-06T20:50:36.001Z', value: 102 },
]);

// Real-time updates
websocket.onmessage = (event) => {
  const tick = JSON.parse(event.data);
  wrapper.update({
    timestamp: tick.timestamp,
    value: tick.price,
  });
};

// Tooltip с полной точностью
chart.subscribeCrosshairMove((param) => {
  if (param.time) {
    const exactTime = wrapper.getExactTimestamp(param.time as number);
    const allTicks = wrapper.getTicksForSecond(param.time as number);
    
    console.log('Displayed time:', param.time);
    console.log('Exact timestamp:', exactTime);
    console.log(`Total ticks in second: ${allTicks.length}`);
  }
});
```

**Плюсы:**
- Сохраняется полная точность
- Поддержка real-time updates
- Экспорт с оригинальными timestamps

**Минусы:**
- На графике отображается только последнее значение секунды
- Дополнительное потребление памяти

---

### Решение 5: Масштабирование времени (субсекундная шкала)

**Оценка:** ⭐⭐⭐⭐ (4/5) - Для визуализации внутрисекундной динамики

```typescript
import { Time, LineData } from 'lightweight-charts';

/**
 * Масштабирует время для субсекундной визуализации
 * Добавляет "фракционные секунды" как отдельные точки
 */
function createSubsecondTimeline(
  data: { timestampMs: number; value: number }[],
  options: {
    baseTimestamp?: number;
    scaleFactor?: number;
  } = {}
): {
  chartData: LineData[];
  timeToOriginal: Map<number, number>;
  originalToTime: Map<number, number>;
} {
  const { baseTimestamp, scaleFactor = 1000 } = options;
  
  const timeToOriginal = new Map<number, number>();
  const originalToTime = new Map<number, number>();
  
  // Находим базовую точку отсчёта
  const minTimestamp = baseTimestamp ?? Math.min(...data.map(d => d.timestampMs));
  
  const chartData: LineData[] = data.map(item => {
    // Масштабируем: 1 секунда на графике = scaleFactor ms реальных данных
    const scaledTime = Math.floor((item.timestampMs - minTimestamp) / scaleFactor);
    
    timeToOriginal.set(scaledTime, item.timestampMs);
    originalToTime.set(item.timestampMs, scaledTime);
    
    return {
      time: scaledTime as Time,
      value: item.value,
    };
  });
  
  // Сортируем по времени
  chartData.sort((a, b) => (a.time as number) - (b.time as number));
  
  return { chartData, timeToOriginal, originalToTime };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

// Данные за 1 секунду с миллисекундной точностью
const tickData = [
  { timestampMs: 1714942235000, value: 100.0 },
  { timestampMs: 1714942235050, value: 100.1 },
  { timestampMs: 1714942235100, value: 100.2 },
  { timestampMs: 1714942235150, value: 100.1 },
  { timestampMs: 1714942235200, value: 100.3 },
  // ... каждые 50ms
];

const { chartData, timeToOriginal } = createSubsecondTimeline(tickData, {
  scaleFactor: 50, // 50ms = 1 "единица" на графике
});

const chart = createChart(container, {
  timeScale: {
    tickMarkFormatter: (time: Time) => {
      const originalMs = timeToOriginal.get(time as number);
      if (!originalMs) return '';
      
      const ms = originalMs % 1000;
      return `+${ms}ms`;
    },
  },
});

const series = chart.addLineSeries();
series.setData(chartData);
```

**Плюсы:**
- Визуализация внутрисекундной динамики
- Каждый тик виден отдельно
- Гибкий масштаб

**Минусы:**
- Нестандартная time scale
- Подходит только для коротких периодов

## ✅ Рекомендуемое решение

Для большинства случаев используйте **комбинацию решений 2 и 4**:

```typescript
// Минимальный production-ready пример

interface TickData {
  timestamp: string; // ISO 8601 с миллисекундами
  price: number;
}

class TickDataManager {
  private storage = new Map<number, TickData[]>();
  private series: ISeriesApi<'Line'>;
  
  constructor(series: ISeriesApi<'Line'>) {
    this.series = series;
  }
  
  addTick(tick: TickData): void {
    const secondTs = Math.floor(new Date(tick.timestamp).getTime() / 1000);
    
    if (!this.storage.has(secondTs)) {
      this.storage.set(secondTs, []);
    }
    this.storage.get(secondTs)!.push(tick);
    
    // Показываем последнюю цену секунды
    this.series.update({
      time: secondTs as Time,
      value: tick.price,
    });
  }
  
  getTicksAt(time: Time): TickData[] {
    return this.storage.get(time as number) || [];
  }
}

// Использование
const manager = new TickDataManager(series);

// WebSocket handler
ws.onmessage = (e) => {
  const tick = JSON.parse(e.data);
  manager.addTick({
    timestamp: tick.timestamp, // "2025-05-06T20:50:35.018Z"
    price: tick.price,
  });
};

// Tooltip
chart.subscribeCrosshairMove((param) => {
  if (param.time) {
    const ticks = manager.getTicksAt(param.time);
    showTooltip(`${ticks.length} ticks, last: ${ticks.at(-1)?.timestamp}`);
  }
});
```

## 📊 Сравнительная таблица решений

| Решение | Точность | Сложность | Real-time | Память | Рекомендация |
|---------|----------|-----------|-----------|--------|--------------|
| **#1 ms→s конвертация** | Низкая | Низкая | ✅ | Низкая | Простые случаи |
| **#2 Агрегация** | Средняя | Средняя | ✅ | Средняя | ✅ Рекомендуется |
| **#3 Normalized time** | Высокая | Высокая | ⚠️ | Средняя | Анализ microstructure |
| **#4 HP Wrapper** | Высокая | Средняя | ✅ | Высокая | ✅ Production |
| **#5 Scaled time** | Высокая | Средняя | ⚠️ | Низкая | Короткие периоды |

## 🔧 Дополнительные рекомендации

### Форматирование времени с миллисекундами

```typescript
const chart = createChart(container, {
  localization: {
    timeFormatter: (time: Time) => {
      // Ваша логика получения оригинального timestamp
      const ms = getOriginalTimestamp(time);
      if (!ms) return new Date((time as number) * 1000).toLocaleTimeString();
      
      return new Date(ms).toISOString().slice(11, 23); // HH:MM:SS.sss
    },
  },
  timeScale: {
    tickMarkFormatter: (time: Time) => {
      const date = new Date((time as number) * 1000);
      return `${date.getHours()}:${date.getMinutes().toString().padStart(2, '0')}`;
    },
  },
});
```

### Throttling для high-frequency updates

```typescript
function createThrottledUpdater(
  series: ISeriesApi<'Line'>,
  intervalMs: number = 100
) {
  let lastUpdate = 0;
  let pendingValue: number | null = null;
  let pendingTime: Time | null = null;
  
  return (time: Time, value: number) => {
    pendingTime = time;
    pendingValue = value;
    
    const now = Date.now();
    if (now - lastUpdate >= intervalMs) {
      series.update({ time: pendingTime, value: pendingValue });
      lastUpdate = now;
    }
  };
}
```

## 🔗 Источники

- [GitHub Issue #1884](https://github.com/tradingview/lightweight-charts/issues/1884) - Feature request for millisecond support
- [Time Documentation](https://tradingview.github.io/lightweight-charts/docs/api/types/Time) - Официальная документация Time
- [Business Days](https://tradingview.github.io/lightweight-charts/docs/time-zones) - Time zones guide

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
