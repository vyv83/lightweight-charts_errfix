# БАГ #45: Зависание при экстремальных значениях

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1500](https://github.com/tradingview/lightweight-charts/issues/1500)  
> **Версии:** v4.1.2+, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** Январь 2024

## 📋 Описание проблемы

### Суть проблемы

При передаче **экстремально больших числовых значений** в данные серии (например, `459761321634093`), браузер **полностью зависает**. Несмотря на то, что такие значения меньше `Number.MAX_SAFE_INTEGER`, библиотека не может корректно обработать их.

### Детали

1. **Порог проблемы:**
   - Значения порядка 10^14 и выше вызывают freeze
   - `Number.MAX_SAFE_INTEGER` = 9,007,199,254,740,991 (~9×10^15)
   - Проблема начинается значительно раньше этого лимита

2. **Причина:**
   - Внутренние расчёты координат для canvas превышают допустимые пределы
   - Попытка отрисовать пиксель на координатах вне canvas приводит к бесконечному циклу
   - Price scale пытается отобразить слишком большой диапазон

3. **Симптомы:**
   - Полный freeze браузера
   - Вкладка не отвечает
   - Требуется force-kill процесса браузера

### Сценарии возникновения

```javascript
// Пример, вызывающий freeze
const chart = LightweightCharts.createChart(container);
const series = chart.addLineSeries();

// ЭТО ВЫЗОВЕТ ЗАВИСАНИЕ!
series.update({
  time: '2023-12-27',
  value: 459761321634093,  // ~4.6×10^14
});
```

### Реальные сценарии

1. **Криптовалюты с очень маленькими ценами:**
   - Shiba Inu, SafeMoon и подобные
   - При отображении в "satoshi" или минимальных единицах

2. **Market cap данные:**
   - Капитализация в триллионах
   - Total supply токенов

3. **Scientific data:**
   - Астрономические расстояния
   - Количество атомов
   - Финансовые агрегаты в малых валютах

4. **Ошибочные данные от API:**
   - Bitflip в числах
   - Некорректное преобразование типов
   - Ошибки в backend

## 🔍 Найденные решения

### Решение 1: Валидация и нормализация данных (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Превентивная защита

```typescript
import { 
  LineData, 
  CandlestickData, 
  BarData, 
  HistogramData,
  Time 
} from 'lightweight-charts';

// Безопасные пределы для lightweight-charts
const SAFE_VALUE_LIMITS = {
  MIN: -1e12,  // -1 триллион
  MAX: 1e12,   // 1 триллион
};

/**
 * Проверяет, является ли значение безопасным для отображения
 */
function isValueSafe(value: number): boolean {
  return (
    Number.isFinite(value) &&
    !Number.isNaN(value) &&
    value >= SAFE_VALUE_LIMITS.MIN &&
    value <= SAFE_VALUE_LIMITS.MAX
  );
}

/**
 * Валидирует и нормализует данные линейной серии
 */
function sanitizeLineData(
  data: LineData[],
  options?: {
    onInvalidValue?: (item: LineData, index: number) => void;
    fallbackValue?: number;
    skipInvalid?: boolean;
  }
): LineData[] {
  const { 
    onInvalidValue, 
    fallbackValue = 0, 
    skipInvalid = true 
  } = options || {};
  
  const result: LineData[] = [];
  
  for (let i = 0; i < data.length; i++) {
    const item = data[i];
    
    if (item.value === undefined || item.value === null) {
      // Whitespace data - пропускаем без изменений
      result.push(item);
      continue;
    }
    
    if (!isValueSafe(item.value)) {
      onInvalidValue?.(item, i);
      
      if (skipInvalid) {
        continue; // Пропускаем невалидную точку
      }
      
      // Заменяем на fallback
      result.push({
        ...item,
        value: fallbackValue,
      });
    } else {
      result.push(item);
    }
  }
  
  return result;
}

/**
 * Валидирует данные свечей
 */
function sanitizeCandlestickData(
  data: CandlestickData[],
  options?: {
    onInvalidValue?: (item: CandlestickData, field: string, index: number) => void;
    skipInvalid?: boolean;
  }
): CandlestickData[] {
  const { onInvalidValue, skipInvalid = true } = options || {};
  const result: CandlestickData[] = [];
  
  for (let i = 0; i < data.length; i++) {
    const item = data[i];
    const fields = ['open', 'high', 'low', 'close'] as const;
    let isValid = true;
    
    for (const field of fields) {
      if (!isValueSafe(item[field])) {
        onInvalidValue?.(item, field, i);
        isValid = false;
        break;
      }
    }
    
    if (isValid || !skipInvalid) {
      result.push(item);
    }
  }
  
  return result;
}

// Использование
const rawData: LineData[] = [
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 459761321634093 }, // ОПАСНО!
  { time: '2024-01-03', value: NaN }, // ОПАСНО!
  { time: '2024-01-04', value: Infinity }, // ОПАСНО!
  { time: '2024-01-05', value: 105 },
];

const safeData = sanitizeLineData(rawData, {
  onInvalidValue: (item, index) => {
    console.warn(`Invalid value at index ${index}:`, item.value);
  },
});

series.setData(safeData);
// Результат: только индексы 0 и 4 (безопасные значения)
```

**Плюсы:**
- Полная защита от freeze
- Настраиваемое поведение
- Логирование проблемных данных

**Минусы:**
- Нужно применять ко всем данным
- Overhead на валидацию

---

### Решение 2: Автоматическое масштабирование данных

**Оценка:** ⭐⭐⭐⭐ (4/5) - Нормализация к отображаемому диапазону

```typescript
import { LineData, ISeriesApi, SeriesType } from 'lightweight-charts';

interface ScaledSeriesConfig {
  /** Целевой диапазон для отображения */
  targetRange: { min: number; max: number };
  /** Сохранять оригинальные значения для tooltips */
  preserveOriginal?: boolean;
}

/**
 * Создаёт масштабированную серию с автоматической нормализацией
 */
function createScaledSeries<T extends LineData>(
  series: ISeriesApi<'Line'>,
  config: ScaledSeriesConfig
): {
  setData: (data: T[]) => void;
  update: (item: T) => void;
  getOriginalValue: (time: string) => number | undefined;
  getScaleFactor: () => { multiplier: number; offset: number };
} {
  const { targetRange, preserveOriginal = true } = config;
  const originalValues = new Map<string, number>();
  let scaleFactor = { multiplier: 1, offset: 0 };
  
  const calculateScale = (data: T[]): void => {
    const values = data
      .filter(d => d.value !== undefined && Number.isFinite(d.value))
      .map(d => d.value as number);
    
    if (values.length === 0) return;
    
    const dataMin = Math.min(...values);
    const dataMax = Math.max(...values);
    const dataRange = dataMax - dataMin || 1;
    
    const targetRangeSize = targetRange.max - targetRange.min;
    
    scaleFactor = {
      multiplier: targetRangeSize / dataRange,
      offset: targetRange.min - (dataMin * (targetRangeSize / dataRange)),
    };
  };
  
  const scaleValue = (value: number): number => {
    return value * scaleFactor.multiplier + scaleFactor.offset;
  };
  
  const scaleData = (data: T[]): T[] => {
    return data.map(item => {
      if (item.value === undefined || !Number.isFinite(item.value)) {
        return item;
      }
      
      const timeKey = String(item.time);
      if (preserveOriginal) {
        originalValues.set(timeKey, item.value);
      }
      
      return {
        ...item,
        value: scaleValue(item.value),
      };
    });
  };
  
  return {
    setData: (data: T[]) => {
      originalValues.clear();
      calculateScale(data);
      const scaledData = scaleData(data);
      series.setData(scaledData);
    },
    
    update: (item: T) => {
      if (item.value !== undefined && Number.isFinite(item.value)) {
        const timeKey = String(item.time);
        if (preserveOriginal) {
          originalValues.set(timeKey, item.value);
        }
        series.update({
          ...item,
          value: scaleValue(item.value),
        });
      }
    },
    
    getOriginalValue: (time: string) => originalValues.get(time),
    
    getScaleFactor: () => ({ ...scaleFactor }),
  };
}

// Использование
const series = chart.addLineSeries();
const scaledSeries = createScaledSeries(series, {
  targetRange: { min: 0, max: 1000 },
  preserveOriginal: true,
});

// Даже экстремальные значения будут нормализованы
scaledSeries.setData([
  { time: '2024-01-01', value: 459761321634093 },
  { time: '2024-01-02', value: 459761321634094 },
  { time: '2024-01-03', value: 459761321634095 },
]);

// Для tooltip можно получить оригинальное значение
const original = scaledSeries.getOriginalValue('2024-01-01');
```

**Плюсы:**
- Работает с любыми числами
- Сохраняет относительные пропорции
- Можно показывать оригинальные значения в tooltip

**Минусы:**
- Значения на price scale отличаются от реальных
- Нужна кастомизация форматтера

---

### Решение 3: Price formatter с научной нотацией

**Оценка:** ⭐⭐⭐⭐ (4/5) - Для отображения больших чисел

```typescript
import { createChart, PriceFormatterFn } from 'lightweight-charts';

/**
 * Форматтер для больших/малых чисел
 */
function createScientificFormatter(options?: {
  threshold?: number;
  precision?: number;
}): PriceFormatterFn {
  const { threshold = 1e9, precision = 2 } = options || {};
  
  return (price: number): string => {
    if (!Number.isFinite(price)) {
      return 'N/A';
    }
    
    const absPrice = Math.abs(price);
    
    if (absPrice >= threshold) {
      const exp = Math.floor(Math.log10(absPrice));
      const mantissa = price / Math.pow(10, exp);
      return `${mantissa.toFixed(precision)}e${exp}`;
    }
    
    if (absPrice >= 1e6) {
      return `${(price / 1e6).toFixed(precision)}M`;
    }
    
    if (absPrice >= 1e3) {
      return `${(price / 1e3).toFixed(precision)}K`;
    }
    
    return price.toFixed(precision);
  };
}

// Использование
const chart = createChart(container, {
  localization: {
    priceFormatter: createScientificFormatter({
      threshold: 1e12,
      precision: 3,
    }),
  },
});

// ⚠️ ВАЖНО: Это НЕ решает проблему freeze!
// Нужно комбинировать с валидацией данных
```

**Плюсы:**
- Красивое отображение больших чисел
- Не теряется информация

**Минусы:**
- НЕ предотвращает freeze
- Только для отображения

---

### Решение 4: Safe wrapper для серии

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Комплексная защита

```typescript
import { 
  ISeriesApi, 
  SeriesType, 
  LineData, 
  CandlestickData, 
  BarData,
  HistogramData,
  SeriesDataItemTypeMap
} from 'lightweight-charts';

type AnySeriesData = LineData | CandlestickData | BarData | HistogramData;

interface SafeSeriesOptions {
  /** Максимальное безопасное значение */
  maxValue?: number;
  /** Минимальное безопасное значение */
  minValue?: number;
  /** Callback при обнаружении опасного значения */
  onDangerousValue?: (value: number, time: any) => void;
  /** Стратегия обработки: skip - пропустить, clamp - обрезать, throw - выбросить ошибку */
  strategy?: 'skip' | 'clamp' | 'throw';
}

const DEFAULT_OPTIONS: Required<SafeSeriesOptions> = {
  maxValue: 1e12,
  minValue: -1e12,
  onDangerousValue: () => {},
  strategy: 'skip',
};

/**
 * Создаёт безопасную обёртку для серии
 */
function createSafeSeries<T extends SeriesType>(
  series: ISeriesApi<T>,
  options?: SafeSeriesOptions
): ISeriesApi<T> {
  const opts = { ...DEFAULT_OPTIONS, ...options };
  
  const isSafeValue = (value: number): boolean => {
    return (
      Number.isFinite(value) &&
      value >= opts.minValue &&
      value <= opts.maxValue
    );
  };
  
  const getNumericValues = (item: AnySeriesData): number[] => {
    if ('value' in item && item.value !== undefined) {
      return [item.value];
    }
    if ('open' in item) {
      return [item.open, item.high, item.low, item.close];
    }
    return [];
  };
  
  const validateItem = (item: AnySeriesData): boolean => {
    const values = getNumericValues(item);
    
    for (const value of values) {
      if (!isSafeValue(value)) {
        opts.onDangerousValue(value, item.time);
        
        if (opts.strategy === 'throw') {
          throw new Error(
            `Dangerous value detected: ${value} at time ${item.time}. ` +
            `Safe range: [${opts.minValue}, ${opts.maxValue}]`
          );
        }
        
        return false;
      }
    }
    
    return true;
  };
  
  const clampItem = <D extends AnySeriesData>(item: D): D => {
    if ('value' in item && item.value !== undefined) {
      return {
        ...item,
        value: Math.max(opts.minValue, Math.min(opts.maxValue, item.value)),
      };
    }
    if ('open' in item) {
      return {
        ...item,
        open: Math.max(opts.minValue, Math.min(opts.maxValue, (item as any).open)),
        high: Math.max(opts.minValue, Math.min(opts.maxValue, (item as any).high)),
        low: Math.max(opts.minValue, Math.min(opts.maxValue, (item as any).low)),
        close: Math.max(opts.minValue, Math.min(opts.maxValue, (item as any).close)),
      };
    }
    return item;
  };
  
  // Создаём proxy для перехвата вызовов
  return new Proxy(series, {
    get(target, prop, receiver) {
      if (prop === 'setData') {
        return (data: AnySeriesData[]) => {
          let safeData: AnySeriesData[];
          
          if (opts.strategy === 'clamp') {
            safeData = data.map(clampItem);
          } else {
            safeData = data.filter(validateItem);
          }
          
          return target.setData(safeData as any);
        };
      }
      
      if (prop === 'update') {
        return (item: AnySeriesData) => {
          if (opts.strategy === 'clamp') {
            return target.update(clampItem(item) as any);
          }
          
          if (validateItem(item)) {
            return target.update(item as any);
          }
          
          // Пропускаем невалидный update
          return;
        };
      }
      
      return Reflect.get(target, prop, receiver);
    },
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);

// Создаём обычную серию
const rawSeries = chart.addLineSeries();

// Оборачиваем в безопасную версию
const series = createSafeSeries(rawSeries, {
  maxValue: 1e12,
  minValue: -1e12,
  strategy: 'skip', // или 'clamp' или 'throw'
  onDangerousValue: (value, time) => {
    console.error(`⚠️ Dangerous value ${value} at ${time} - skipped`);
    // Можно отправить в мониторинг/Sentry
  },
});

// Теперь безопасно использовать
series.setData([
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 459761321634093 }, // Будет пропущен
  { time: '2024-01-03', value: 102 },
]);
// Результат: отображены только индексы 0 и 2
```

**Плюсы:**
- Прозрачная интеграция
- Полная защита
- Различные стратегии обработки
- Логирование проблем

**Минусы:**
- Proxy может немного влиять на производительность
- Нужно применять к каждой серии

---

### Решение 5: Defensive data preprocessing

**Оценка:** ⭐⭐⭐⭐ (4/5) - Централизованная обработка

```typescript
/**
 * Класс для безопасной предобработки данных
 */
class ChartDataProcessor {
  private static readonly SAFE_RANGE = {
    min: -1e12,
    max: 1e12,
  };
  
  /**
   * Проверяет данные на безопасность
   */
  static validate<T extends { time: any }>(
    data: T[],
    getValues: (item: T) => number[]
  ): { valid: T[]; invalid: T[]; errors: string[] } {
    const valid: T[] = [];
    const invalid: T[] = [];
    const errors: string[] = [];
    
    for (const item of data) {
      const values = getValues(item);
      let isValid = true;
      
      for (const value of values) {
        if (!Number.isFinite(value)) {
          errors.push(`Non-finite value ${value} at time ${item.time}`);
          isValid = false;
          break;
        }
        
        if (value < this.SAFE_RANGE.min || value > this.SAFE_RANGE.max) {
          errors.push(
            `Out of range value ${value} at time ${item.time}. ` +
            `Safe range: [${this.SAFE_RANGE.min}, ${this.SAFE_RANGE.max}]`
          );
          isValid = false;
          break;
        }
      }
      
      if (isValid) {
        valid.push(item);
      } else {
        invalid.push(item);
      }
    }
    
    return { valid, invalid, errors };
  }
  
  /**
   * Проверяет данные линейной серии
   */
  static validateLineData(data: LineData[]): ReturnType<typeof this.validate> {
    return this.validate(data, item => 
      item.value !== undefined ? [item.value] : []
    );
  }
  
  /**
   * Проверяет данные свечей
   */
  static validateCandlestickData(data: CandlestickData[]): ReturnType<typeof this.validate> {
    return this.validate(data, item => [item.open, item.high, item.low, item.close]);
  }
  
  /**
   * Масштабирует данные к безопасному диапазону
   */
  static normalizeToSafeRange<T extends LineData>(
    data: T[],
    targetRange: { min: number; max: number } = { min: 0, max: 1000 }
  ): { data: T[]; scale: { factor: number; offset: number } } {
    const values = data
      .filter(d => d.value !== undefined && Number.isFinite(d.value))
      .map(d => d.value as number);
    
    if (values.length === 0) {
      return { data, scale: { factor: 1, offset: 0 } };
    }
    
    const min = Math.min(...values);
    const max = Math.max(...values);
    const range = max - min || 1;
    
    const factor = (targetRange.max - targetRange.min) / range;
    const offset = targetRange.min - min * factor;
    
    const normalizedData = data.map(item => {
      if (item.value === undefined || !Number.isFinite(item.value)) {
        return item;
      }
      return {
        ...item,
        value: item.value * factor + offset,
      };
    });
    
    return {
      data: normalizedData as T[],
      scale: { factor, offset },
    };
  }
}

// Использование
const rawData: LineData[] = fetchDataFromAPI();

const { valid, invalid, errors } = ChartDataProcessor.validateLineData(rawData);

if (errors.length > 0) {
  console.warn('Data validation issues:', errors);
  // Отправить в мониторинг
}

series.setData(valid);
```

## ✅ Рекомендуемое решение

Комбинация **решений 1 и 4** для максимальной защиты:

```typescript
import { createChart, ISeriesApi, LineData, CandlestickData } from 'lightweight-charts';

// ==================== КОНСТАНТЫ ====================

const SAFE_VALUE_LIMITS = {
  MIN: -1e12,
  MAX: 1e12,
} as const;

// ==================== ХЕЛПЕРЫ ====================

function isValueSafe(value: number): boolean {
  return (
    Number.isFinite(value) &&
    value >= SAFE_VALUE_LIMITS.MIN &&
    value <= SAFE_VALUE_LIMITS.MAX
  );
}

function sanitizeData<T extends { value?: number }>(
  data: T[],
  onInvalid?: (item: T, index: number) => void
): T[] {
  return data.filter((item, index) => {
    if (item.value === undefined) return true; // whitespace
    
    const isSafe = isValueSafe(item.value);
    if (!isSafe && onInvalid) {
      onInvalid(item, index);
    }
    return isSafe;
  });
}

// ==================== СОЗДАНИЕ БЕЗОПАСНОГО ГРАФИКА ====================

function createSafeChart(container: HTMLElement) {
  const chart = createChart(container, {
    width: container.clientWidth,
    height: 400,
  });
  
  const addSafeLineSeries = () => {
    const series = chart.addLineSeries();
    
    return {
      setData: (data: LineData[]) => {
        const safeData = sanitizeData(data, (item, i) => {
          console.warn(`⚠️ Skipped unsafe value at index ${i}:`, item);
        });
        series.setData(safeData);
      },
      
      update: (item: LineData) => {
        if (item.value === undefined || isValueSafe(item.value)) {
          series.update(item);
        } else {
          console.warn('⚠️ Skipped unsafe update:', item);
        }
      },
      
      getSeries: () => series,
    };
  };
  
  return {
    chart,
    addSafeLineSeries,
    // Добавить другие методы для других типов серий
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const { chart, addSafeLineSeries } = createSafeChart(
  document.getElementById('chart')!
);

const series = addSafeLineSeries();

// Безопасно работает с любыми данными
series.setData([
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 459761321634093 }, // Будет пропущен
  { time: '2024-01-03', value: NaN }, // Будет пропущен
  { time: '2024-01-04', value: Infinity }, // Будет пропущен
  { time: '2024-01-05', value: 105 },
]);
// Результат: график с 2 точками (индексы 0 и 4)
```

## 📊 Сравнительная таблица решений

| Решение | Защита | Сложность | Производительность | Рекомендация |
|---------|--------|-----------|-------------------|--------------|
| **#1 Валидация данных** | ⭐⭐⭐⭐⭐ | Низкая | Высокая | ✅ Обязательно |
| **#2 Масштабирование** | ⭐⭐⭐⭐ | Средняя | Высокая | Для больших чисел |
| **#3 Scientific formatter** | ⭐⭐ | Низкая | Высокая | Только отображение |
| **#4 Safe wrapper** | ⭐⭐⭐⭐⭐ | Средняя | Средняя | ✅ Рекомендуется |
| **#5 Centralized processor** | ⭐⭐⭐⭐ | Средняя | Высокая | Для крупных проектов |

## 🔧 Дополнительные рекомендации

### Обработка ошибок на уровне API

```typescript
async function fetchChartData(symbol: string): Promise<LineData[]> {
  const response = await fetch(`/api/chart/${symbol}`);
  const rawData = await response.json();
  
  // Валидация сразу после получения
  const { valid, errors } = ChartDataProcessor.validateLineData(rawData);
  
  if (errors.length > 0) {
    // Отправляем ошибки в мониторинг
    reportToSentry('Invalid chart data', { symbol, errors });
  }
  
  return valid;
}
```

### TypeScript guard для безопасных значений

```typescript
type SafeNumber = number & { __brand: 'SafeNumber' };

function asSafeNumber(value: number): SafeNumber | null {
  if (isValueSafe(value)) {
    return value as SafeNumber;
  }
  return null;
}

interface SafeLineData {
  time: Time;
  value?: SafeNumber;
}
```

## 🔗 Источники

- [GitHub Issue #1500](https://github.com/tradingview/lightweight-charts/issues/1500) - Browser freezes when updating chart data
- [Issue #673](https://github.com/tradingview/lightweight-charts/issues/673) - Incorrect bar height when value is more than chart's height
- [StackOverflow: Large datasets](https://stackoverflow.com/questions/71746193/can-lightweight-charts-handle-large-datasets-like-1-2-millions-bar-candle-time) - Handling large datasets
- [MDN: Number.MAX_SAFE_INTEGER](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER) - JavaScript number limits
- [JSFiddle репродукция](https://jsfiddle.net/L3rvudpc/) - Демонстрация бага

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
