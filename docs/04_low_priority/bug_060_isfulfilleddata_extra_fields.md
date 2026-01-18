# БАГ #60: isFulfilledData проверяет лишние поля

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#1526](https://github.com/tradingview/lightweight-charts/issues/1526)  
> **Версии:** v4.x+, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2023

## 📋 Описание проблемы

### Суть проблемы

Внутренняя функция **`isFulfilledData`** (используется для проверки валидности данных) **проверяет наличие дополнительных полей**, которые пользователь мог добавить к объектам данных. Это может приводить к неожиданным результатам, когда данные с кастомными полями отклоняются или обрабатываются некорректно.

### Детали

1. **Симптомы:**
   - Данные с дополнительными полями могут быть отклонены
   - Неожиданное поведение при использовании расширенных объектов
   - TypeScript типы не позволяют дополнительные поля

2. **Ожидаемое поведение:**
   - Проверка только обязательных полей (time, value/OHLC)
   - Игнорирование дополнительных кастомных полей
   - Возможность хранить metadata в объектах данных

3. **Текущее поведение:**
   - Строгая проверка структуры данных
   - Дополнительные поля могут влиять на валидацию

### Сценарий проблемы

```typescript
import { createChart, LineData, Time } from 'lightweight-charts';

// Расширенный тип с кастомными полями
interface ExtendedLineData extends LineData {
  time: Time;
  value: number;
  metadata?: {
    source: string;
    confidence: number;
  };
  customId?: string;
}

const chart = createChart(container);
const series = chart.addLineSeries();

// Данные с дополнительными полями
const dataWithMetadata: ExtendedLineData[] = [
  { 
    time: '2024-01-01', 
    value: 100,
    metadata: { source: 'API', confidence: 0.95 },
    customId: 'point-1',
  },
  { 
    time: '2024-01-02', 
    value: 105,
    metadata: { source: 'API', confidence: 0.87 },
    customId: 'point-2',
  },
];

// TypeScript может выдать ошибку
// series.setData(dataWithMetadata); // ❌ Type error

// Или данные могут быть неправильно обработаны внутренней валидацией
```

## 🔍 Найденные решения

### Решение 1: Маппинг данных перед setData (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Чистое решение

```typescript
import { createChart, LineData, Time, ISeriesApi } from 'lightweight-charts';

// Ваш расширенный тип
interface ExtendedLineData {
  time: Time;
  value: number;
  metadata?: Record<string, any>;
  customId?: string;
}

/**
 * Извлекает только необходимые поля для библиотеки
 */
function toLineData(data: ExtendedLineData[]): LineData[] {
  return data.map(({ time, value }) => ({ time, value }));
}

/**
 * Для candlestick данных
 */
interface ExtendedCandlestickData {
  time: Time;
  open: number;
  high: number;
  low: number;
  close: number;
  metadata?: Record<string, any>;
  volume?: number;
}

function toCandlestickData(data: ExtendedCandlestickData[]): CandlestickData[] {
  return data.map(({ time, open, high, low, close }) => ({
    time, open, high, low, close,
  }));
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();

// Ваши данные с метаданными
const extendedData: ExtendedLineData[] = [
  { time: '2024-01-01', value: 100, metadata: { source: 'API' }, customId: '1' },
  { time: '2024-01-02', value: 105, metadata: { source: 'API' }, customId: '2' },
];

// Передаём только нужные поля
series.setData(toLineData(extendedData));

// Храните оригинальные данные отдельно для доступа к метаданным
const dataStore = new Map(extendedData.map(d => [d.time.toString(), d]));

// Получение метаданных по времени
chart.subscribeCrosshairMove((param) => {
  if (param.time) {
    const originalData = dataStore.get(param.time.toString());
    if (originalData?.metadata) {
      console.log('Metadata:', originalData.metadata);
    }
  }
});
```

**Плюсы:**
- Чистое разделение данных
- Типобезопасность
- Метаданные доступны отдельно

**Минусы:**
- Дублирование данных
- Нужна синхронизация

---

### Решение 2: Type assertion с фильтрацией

**Оценка:** ⭐⭐⭐⭐ (4/5) - Быстрое решение

```typescript
import { createChart, LineData, Time } from 'lightweight-charts';

interface ExtendedLineData {
  time: Time;
  value: number;
  [key: string]: any; // Разрешаем дополнительные поля
}

/**
 * Фильтрует объект, оставляя только нужные поля
 */
function pickFields<T extends object, K extends keyof T>(
  obj: T,
  keys: K[]
): Pick<T, K> {
  const result = {} as Pick<T, K>;
  for (const key of keys) {
    if (key in obj) {
      result[key] = obj[key];
    }
  }
  return result;
}

/**
 * Безопасно конвертирует расширенные данные
 */
function sanitizeLineData(data: ExtendedLineData[]): LineData[] {
  return data.map(item => pickFields(item, ['time', 'value'])) as LineData[];
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();

const extendedData: ExtendedLineData[] = [
  { time: '2024-01-01', value: 100, extra: 'ignored', customField: 123 },
  { time: '2024-01-02', value: 105, extra: 'ignored', customField: 456 },
];

// Type assertion после фильтрации
series.setData(sanitizeLineData(extendedData));
```

**Плюсы:**
- Минимальный код
- Работает с любыми дополнительными полями

**Минусы:**
- Теряются метаданные
- Type assertion может скрыть ошибки

---

### Решение 3: Wrapper класс для данных с метаданными

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Полное решение

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi, 
  LineData, 
  CandlestickData,
  Time,
  MouseEventParams,
} from 'lightweight-charts';

/**
 * Типы для расширенных данных
 */
interface DataWithMetadata<T, M> {
  chartData: T;
  metadata: M;
}

/**
 * Менеджер данных с метаданными
 */
class MetadataAwareDataManager<T extends { time: Time }, M> {
  private _series: ISeriesApi<any>;
  private _dataMap: Map<string, M> = new Map();
  private _onMetadataHover?: (metadata: M | undefined, time: Time | null) => void;
  
  constructor(
    series: ISeriesApi<any>,
    chart: IChartApi,
    onMetadataHover?: (metadata: M | undefined, time: Time | null) => void
  ) {
    this._series = series;
    this._onMetadataHover = onMetadataHover;
    
    // Подписываемся на hover для получения метаданных
    chart.subscribeCrosshairMove((param) => {
      this._handleCrosshairMove(param);
    });
  }
  
  /**
   * Устанавливает данные с метаданными
   */
  setData(data: DataWithMetadata<T, M>[]): void {
    // Очищаем старые метаданные
    this._dataMap.clear();
    
    // Сохраняем метаданные
    for (const item of data) {
      const key = this._timeToKey(item.chartData.time);
      this._dataMap.set(key, item.metadata);
    }
    
    // Устанавливаем данные в серию (только chartData)
    this._series.setData(data.map(d => d.chartData) as any);
  }
  
  /**
   * Обновляет точку с метаданными
   */
  update(item: DataWithMetadata<T, M>): void {
    const key = this._timeToKey(item.chartData.time);
    this._dataMap.set(key, item.metadata);
    this._series.update(item.chartData as any);
  }
  
  /**
   * Получает метаданные по времени
   */
  getMetadata(time: Time): M | undefined {
    const key = this._timeToKey(time);
    return this._dataMap.get(key);
  }
  
  /**
   * Получает все метаданные
   */
  getAllMetadata(): Map<string, M> {
    return new Map(this._dataMap);
  }
  
  private _handleCrosshairMove(param: MouseEventParams): void {
    if (!this._onMetadataHover) return;
    
    if (param.time) {
      const metadata = this.getMetadata(param.time);
      this._onMetadataHover(metadata, param.time);
    } else {
      this._onMetadataHover(undefined, null);
    }
  }
  
  private _timeToKey(time: Time): string {
    if (typeof time === 'string') return time;
    if (typeof time === 'number') return time.toString();
    return `${time.year}-${time.month}-${time.day}`;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

// Определяем типы
interface MyLineData {
  time: Time;
  value: number;
}

interface MyMetadata {
  source: string;
  confidence: number;
  tags: string[];
}

// Создаём график
const chart = createChart(container);
const series = chart.addLineSeries();

// Создаём менеджер данных
const dataManager = new MetadataAwareDataManager<MyLineData, MyMetadata>(
  series,
  chart,
  (metadata, time) => {
    if (metadata) {
      console.log(`Hovered at ${time}: source=${metadata.source}, confidence=${metadata.confidence}`);
      // Обновляем tooltip или UI
      updateTooltip(metadata);
    } else {
      hideTooltip();
    }
  }
);

// Устанавливаем данные
dataManager.setData([
  {
    chartData: { time: '2024-01-01', value: 100 },
    metadata: { source: 'API', confidence: 0.95, tags: ['verified'] },
  },
  {
    chartData: { time: '2024-01-02', value: 105 },
    metadata: { source: 'Manual', confidence: 0.7, tags: ['unverified'] },
  },
]);

// Позже можно получить метаданные
const meta = dataManager.getMetadata('2024-01-01');
console.log(meta); // { source: 'API', confidence: 0.95, tags: ['verified'] }
```

**Плюсы:**
- Полное разделение данных и метаданных
- Типобезопасность
- Автоматический доступ к метаданным при hover

**Минусы:**
- Больше кода
- Дополнительная память для Map

---

### Решение 4: Использование WeakMap для метаданных

**Оценка:** ⭐⭐⭐⭐ (4/5) - Memory-efficient

```typescript
import { createChart, LineData, Time } from 'lightweight-charts';

// WeakMap для хранения метаданных без утечек памяти
const metadataStorage = new WeakMap<object, Record<string, any>>();

/**
 * Добавляет метаданные к объекту данных
 */
function attachMetadata<T extends object>(
  data: T,
  metadata: Record<string, any>
): T {
  metadataStorage.set(data, metadata);
  return data;
}

/**
 * Получает метаданные объекта
 */
function getMetadata<T extends object>(data: T): Record<string, any> | undefined {
  return metadataStorage.get(data);
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();

// Создаём данные с прикреплёнными метаданными
const dataPoints: LineData[] = [
  attachMetadata(
    { time: '2024-01-01' as Time, value: 100 },
    { source: 'API', customId: '1' }
  ),
  attachMetadata(
    { time: '2024-01-02' as Time, value: 105 },
    { source: 'API', customId: '2' }
  ),
];

series.setData(dataPoints);

// Получение метаданных
chart.subscribeCrosshairMove((param) => {
  if (param.seriesData.size > 0) {
    const seriesDataPoint = param.seriesData.get(series);
    if (seriesDataPoint) {
      // Находим оригинальный объект
      const originalPoint = dataPoints.find(
        p => p.time === seriesDataPoint.time
      );
      if (originalPoint) {
        const meta = getMetadata(originalPoint);
        console.log('Metadata:', meta);
      }
    }
  }
});
```

**Плюсы:**
- Нет утечек памяти (WeakMap)
- Минимальный overhead
- Не изменяет оригинальные объекты

**Минусы:**
- Нужно сохранять ссылки на объекты
- Сложнее с immutable данными

---

### Решение 5: Custom series с поддержкой метаданных

**Оценка:** ⭐⭐⭐ (3/5) - Для продвинутых случаев

```typescript
import { 
  createChart,
  customSeriesDefaultOptions,
  ICustomSeriesPaneRenderer,
  ICustomSeriesPaneView,
  CustomSeriesOptions,
  Time,
} from 'lightweight-charts';

// Тип данных с метаданными
interface MetadataLineData {
  time: Time;
  value: number;
  metadata?: Record<string, any>;
}

// Custom series который принимает метаданные
class MetadataLineSeries implements ICustomSeriesPaneView<Time, MetadataLineData> {
  private _data: MetadataLineData[] = [];
  
  update(data: MetadataLineData[], options: CustomSeriesOptions): void {
    this._data = data;
    // Метаданные доступны внутри серии
  }
  
  renderer(): ICustomSeriesPaneRenderer {
    return {
      draw: (target, priceConverter) => {
        // Рендеринг с доступом к метаданным
        const ctx = target.context;
        
        for (const point of this._data) {
          const y = priceConverter(point.value);
          // Можно использовать metadata для стилизации
          if (point.metadata?.highlight) {
            ctx.fillStyle = 'red';
          }
          // ... рисуем точку
        }
      },
    };
  }
  
  // Метод для получения метаданных
  getMetadataAtTime(time: Time): Record<string, any> | undefined {
    const point = this._data.find(d => d.time === time);
    return point?.metadata;
  }
}
```

**Плюсы:**
- Нативная поддержка метаданных
- Можно использовать в рендеринге

**Минусы:**
- Требует Custom Series
- Сложная реализация

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 3** (MetadataAwareDataManager):

```typescript
// Простой пример использования
const dataManager = new MetadataAwareDataManager(series, chart, (meta) => {
  if (meta) updateUI(meta);
});

dataManager.setData([
  { chartData: { time: '2024-01-01', value: 100 }, metadata: { id: '1' } },
  { chartData: { time: '2024-01-02', value: 105 }, metadata: { id: '2' } },
]);
```

Для простых случаев используйте **Решение 1** (маппинг данных):

```typescript
const toChartData = (data) => data.map(({ time, value }) => ({ time, value }));
series.setData(toChartData(extendedData));
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Типобезопасность | Memory | Рекомендация |
|---------|----------|------------------|--------|--------------|
| **#1 Маппинг** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ Простые случаи |
| **#2 Type assertion** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Быстрый фикс |
| **#3 DataManager** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ Полное решение |
| **#4 WeakMap** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Memory-efficient |
| **#5 Custom Series** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Продвинутый |

## 🔗 Источники

- [GitHub Issue #1526](https://github.com/tradingview/lightweight-charts/issues/1526) - isFulfilledData extra fields
- [Data Types Documentation](https://tradingview.github.io/lightweight-charts/docs/api#data-types)
- [Custom Series Guide](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
