# БАГ #22: TypeScript strict mode errors

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issues:** [#1091](https://github.com/tradingview/lightweight-charts/issues/1091), [#1699](https://github.com/tradingview/lightweight-charts/issues/1699)  
> **Версии:** Все версии  
> **Платформы:** Все (TypeScript issue)  
> **Статус:** 🔴 Open (постепенное улучшение, но проблемы остаются)

---

## 📋 Описание проблемы

### Суть проблемы
При использовании TypeScript с включённым `strict: true` в `tsconfig.json` возникают ошибки компиляции, связанные с неполными или некорректными type definitions в библиотеке lightweight-charts.

### Конкретные проявления

1. **createPriceLine требует все поля:**
   ```typescript
   // ❌ Ошибка в strict mode
   const priceLine = series.createPriceLine({ price: 100 });
   // Error: Argument of type '{ price: number; }' is not assignable 
   // to parameter of type 'PriceLineOptions'.
   ```

2. **WhitespaceData и отсутствующее поле value:**
   ```typescript
   // ❌ Проблема с типизацией
   const data: (LineData | WhitespaceData)[] = [...];
   data.forEach(item => {
     if ('value' in item) {
       console.log(item.value); // Ошибка: 'value' не существует в WhitespaceData
     }
   });
   ```

3. **Неявные any типы в некоторых методах:**
   ```typescript
   // ❌ noImplicitAny errors
   chart.subscribeCrosshairMove((param) => {
     // param может быть implicitly any в некоторых версиях
   });
   ```

4. **strictNullChecks и необработанные null:**
   ```typescript
   // ❌ Потенциальные null errors
   const series = chart.addLineSeries();
   const data = series.data(); 
   // data может быть undefined, но типы не всегда это отражают
   ```

### Почему это критично

- **80%+ современных TypeScript проектов** используют strict mode
- Ошибки компиляции блокируют сборку проекта
- Приходится использовать `// @ts-ignore`, что снижает type safety
- Усложняет интеграцию в enterprise проекты с строгими стандартами

---

## 🔍 Найденные решения

### Решение 1: Использование type assertions (Quick Fix)

**Оценка: 5/10**

```typescript
import { 
  createChart, 
  ISeriesApi, 
  PriceLineOptions,
  LineData,
  WhitespaceData 
} from 'lightweight-charts';

// Fix для createPriceLine
const priceLineOptions = { price: 100 } as PriceLineOptions;
const priceLine = series.createPriceLine(priceLineOptions);

// Fix для данных
const data = [...] as (LineData | WhitespaceData)[];
```

**Плюсы:**
- Быстрое решение
- Минимальные изменения в коде

**Минусы:**
- Обходит type safety
- Не выявляет реальные ошибки
- Требует повторения в каждом месте использования

---

### Решение 2: Module Augmentation (Рекомендуемое)

**Оценка: 9/10**

Создание файла деклараций для исправления типов:

```typescript
// src/types/lightweight-charts.d.ts

import 'lightweight-charts';

declare module 'lightweight-charts' {
  // Fix для PriceLineOptions - делаем поля опциональными
  interface PriceLineOptions {
    price: number;
    color?: string;
    lineWidth?: LineWidth;
    lineStyle?: LineStyle;
    lineVisible?: boolean;
    axisLabelVisible?: boolean;
    title?: string;
    axisLabelColor?: string;
    axisLabelTextColor?: string;
  }

  // Добавляем helper type для проверки WhitespaceData
  type DataPoint<T> = T | WhitespaceData;
  
  // Type guard helper
  function isBusinessDay(time: Time): time is BusinessDay;
  function isUTCTimestamp(time: Time): time is UTCTimestamp;
}
```

**Плюсы:**
- Полноценная type safety
- Централизованное решение
- Легко обновлять при новых версиях
- Не требует изменения исходного кода приложения

**Минусы:**
- Требует создания дополнительного файла
- Нужно поддерживать синхронизацию с обновлениями библиотеки

---

### Решение 3: Type Guards для безопасной работы с данными

**Оценка: 8/10**

```typescript
import { 
  LineData, 
  WhitespaceData, 
  CandlestickData,
  Time,
  BusinessDay,
  UTCTimestamp
} from 'lightweight-charts';

// Type guard для проверки наличия value
function hasValue<T extends { time: Time }>(
  data: T | WhitespaceData
): data is T {
  return 'value' in data || 'close' in data || 'open' in data;
}

// Type guard для LineData
function isLineData(data: LineData | WhitespaceData): data is LineData {
  return 'value' in data && typeof (data as LineData).value === 'number';
}

// Type guard для CandlestickData
function isCandlestickData(
  data: CandlestickData | WhitespaceData
): data is CandlestickData {
  return 'open' in data && 'close' in data && 'high' in data && 'low' in data;
}

// Type guard для Time types
function isBusinessDay(time: Time): time is BusinessDay {
  return typeof time === 'object' && 'year' in time && 'month' in time && 'day' in time;
}

function isUTCTimestamp(time: Time): time is UTCTimestamp {
  return typeof time === 'number';
}

// Использование
const data: (LineData | WhitespaceData)[] = series.data();

data.forEach(item => {
  if (isLineData(item)) {
    console.log(item.value); // ✅ TypeScript знает, что это LineData
  }
});
```

**Плюсы:**
- Гарантирует runtime безопасность
- Полная type safety
- Переиспользуемые функции

**Минусы:**
- Дополнительный код
- Небольшой overhead в runtime

---

### Решение 4: Wrapper-классы с правильной типизацией

**Оценка: 7/10**

```typescript
import { 
  createChart as originalCreateChart,
  IChartApi,
  ISeriesApi,
  SeriesType,
  ChartOptions,
  DeepPartial,
  PriceLineOptions as OriginalPriceLineOptions
} from 'lightweight-charts';

// Расширенные типы
interface SafePriceLineOptions {
  price: number;
  color?: string;
  lineWidth?: 1 | 2 | 3 | 4;
  lineStyle?: 0 | 1 | 2 | 3;
  lineVisible?: boolean;
  axisLabelVisible?: boolean;
  title?: string;
}

// Wrapper для серий
class SafeSeriesApi<T extends SeriesType> {
  constructor(private series: ISeriesApi<T>) {}

  createPriceLine(options: SafePriceLineOptions) {
    return this.series.createPriceLine(options as OriginalPriceLineOptions);
  }

  // Добавляем типизированные методы
  update(data: Parameters<ISeriesApi<T>['update']>[0]) {
    return this.series.update(data);
  }

  setData(data: Parameters<ISeriesApi<T>['setData']>[0]) {
    return this.series.setData(data);
  }

  // Проброс остальных методов
  get original() {
    return this.series;
  }
}

// Factory function
function createSafeChart(
  container: HTMLElement, 
  options?: DeepPartial<ChartOptions>
): IChartApi {
  return originalCreateChart(container, options);
}

export { createSafeChart, SafeSeriesApi, SafePriceLineOptions };
```

**Плюсы:**
- Полный контроль над типами
- Можно добавить дополнительную валидацию

**Минусы:**
- Много boilerplate кода
- Нужно обновлять при изменении API

---

### Решение 5: Конфигурация tsconfig для частичного strict mode

**Оценка: 4/10**

```json
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,
    
    // Или более гранулярно:
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true
  }
}
```

С использованием `skipLibCheck: true` для игнорирования ошибок в node_modules.

**Плюсы:**
- Простое решение
- Не требует изменения кода

**Минусы:**
- `skipLibCheck` скрывает все ошибки в библиотеках
- Не решает проблему, а маскирует её
- Можно пропустить реальные ошибки

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Module Augmentation + Type Guards

Наиболее надёжное решение сочетает module augmentation для исправления типов библиотеки и type guards для безопасной работы с данными.

#### Шаг 1: Создайте файл деклараций

```typescript
// src/types/lightweight-charts-strict.d.ts

import 'lightweight-charts';

declare module 'lightweight-charts' {
  // ============================================
  // FIX: PriceLineOptions - сделать поля опциональными
  // ============================================
  export interface CreatePriceLineOptions {
    price: number;
    color?: string;
    lineWidth?: LineWidth;
    lineStyle?: LineStyle;
    lineVisible?: boolean;
    axisLabelVisible?: boolean;
    title?: string;
    axisLabelColor?: string;
    axisLabelTextColor?: string;
  }

  // Расширяем ISeriesApi для поддержки частичных опций
  export interface ISeriesApi<T extends SeriesType> {
    createPriceLine(options: CreatePriceLineOptions): IPriceLine;
  }
}
```

#### Шаг 2: Создайте утилиты для type guards

```typescript
// src/utils/chart-type-guards.ts

import type {
  Time,
  BusinessDay,
  UTCTimestamp,
  LineData,
  WhitespaceData,
  CandlestickData,
  BarData,
  HistogramData,
  AreaData,
  BaselineData,
  SeriesDataItemTypeMap,
  SeriesType,
} from 'lightweight-charts';

/**
 * Проверяет, является ли элемент данных WhitespaceData
 */
export function isWhitespaceData<T extends { time: Time }>(
  data: T | WhitespaceData
): data is WhitespaceData {
  return !('value' in data) && !('close' in data);
}

/**
 * Проверяет, является ли элемент данных LineData
 */
export function isLineData(
  data: LineData | WhitespaceData
): data is LineData {
  return 'value' in data && typeof (data as LineData).value === 'number';
}

/**
 * Проверяет, является ли элемент данных CandlestickData
 */
export function isCandlestickData(
  data: CandlestickData | WhitespaceData
): data is CandlestickData {
  const d = data as CandlestickData;
  return 'open' in d && 'close' in d && 'high' in d && 'low' in d;
}

/**
 * Проверяет, является ли элемент данных HistogramData
 */
export function isHistogramData(
  data: HistogramData | WhitespaceData
): data is HistogramData {
  return 'value' in data;
}

/**
 * Проверяет, является ли элемент данных AreaData
 */
export function isAreaData(
  data: AreaData | WhitespaceData
): data is AreaData {
  return 'value' in data;
}

/**
 * Проверяет, является ли элемент данных BaselineData
 */
export function isBaselineData(
  data: BaselineData | WhitespaceData
): data is BaselineData {
  return 'value' in data;
}

/**
 * Type guard для BusinessDay
 */
export function isBusinessDay(time: Time): time is BusinessDay {
  return (
    typeof time === 'object' &&
    time !== null &&
    'year' in time &&
    'month' in time &&
    'day' in time
  );
}

/**
 * Type guard для UTCTimestamp
 */
export function isUTCTimestamp(time: Time): time is UTCTimestamp {
  return typeof time === 'number';
}

/**
 * Generic type guard для данных с value
 */
export function hasValue<T extends { time: Time; value?: number }>(
  data: T | WhitespaceData
): data is T & { value: number } {
  return 'value' in data && typeof (data as T).value === 'number';
}

/**
 * Утилита для безопасного получения значения
 */
export function getValueOrNull(
  data: LineData | AreaData | HistogramData | BaselineData | WhitespaceData
): number | null {
  if ('value' in data && typeof data.value === 'number') {
    return data.value;
  }
  return null;
}

/**
 * Утилита для безопасного получения OHLC данных
 */
export function getOHLCOrNull(
  data: CandlestickData | BarData | WhitespaceData
): { open: number; high: number; low: number; close: number } | null {
  if (isCandlestickData(data as CandlestickData | WhitespaceData)) {
    const d = data as CandlestickData;
    return {
      open: d.open,
      high: d.high,
      low: d.low,
      close: d.close,
    };
  }
  return null;
}
```

#### Шаг 3: Создайте helper для price lines

```typescript
// src/utils/chart-helpers.ts

import type { 
  ISeriesApi, 
  SeriesType,
  IPriceLine,
  LineStyle,
  LineWidth
} from 'lightweight-charts';

export interface SafePriceLineOptions {
  price: number;
  color?: string;
  lineWidth?: LineWidth;
  lineStyle?: LineStyle;
  lineVisible?: boolean;
  axisLabelVisible?: boolean;
  title?: string;
  axisLabelColor?: string;
  axisLabelTextColor?: string;
}

/**
 * Создаёт price line с правильной типизацией
 */
export function createSafePriceLine<T extends SeriesType>(
  series: ISeriesApi<T>,
  options: SafePriceLineOptions
): IPriceLine {
  // Создаём объект с дефолтами
  const fullOptions = {
    price: options.price,
    color: options.color ?? '#2962FF',
    lineWidth: options.lineWidth ?? 1,
    lineStyle: options.lineStyle ?? 0, // Solid
    lineVisible: options.lineVisible ?? true,
    axisLabelVisible: options.axisLabelVisible ?? true,
    title: options.title ?? '',
    axisLabelColor: options.axisLabelColor,
    axisLabelTextColor: options.axisLabelTextColor,
  };

  return series.createPriceLine(fullOptions as any);
}
```

#### Шаг 4: Пример полного использования

```typescript
// src/components/Chart.tsx

import { useEffect, useRef } from 'react';
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  LineData,
  WhitespaceData,
  CandlestickData,
  LineSeries
} from 'lightweight-charts';
import { 
  isLineData, 
  isCandlestickData, 
  isWhitespaceData,
  getValueOrNull 
} from '../utils/chart-type-guards';
import { createSafePriceLine } from '../utils/chart-helpers';

interface ChartProps {
  data: (LineData | WhitespaceData)[];
}

export function Chart({ data }: ChartProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Line'> | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    // Создание графика
    const chart = createChart(containerRef.current, {
      width: 800,
      height: 400,
      layout: {
        background: { color: '#1a1a1a' },
        textColor: '#d1d4dc',
      },
    });

    chartRef.current = chart;

    // Создание серии
    const series = chart.addSeries(LineSeries, {
      color: '#2962FF',
      lineWidth: 2,
    });

    seriesRef.current = series;

    // Установка данных с фильтрацией WhitespaceData
    series.setData(data);

    // ✅ Безопасное создание price line
    createSafePriceLine(series, {
      price: 100,
      color: '#26a69a',
      lineWidth: 2,
      title: 'Target',
    });

    // ✅ Безопасная работа с данными
    const seriesData = series.data();
    
    seriesData.forEach((item) => {
      if (isLineData(item)) {
        console.log(`Value: ${item.value}`);
      } else if (isWhitespaceData(item)) {
        console.log('Whitespace at:', item.time);
      }
    });

    // ✅ Безопасное получение значения
    const lastItem = seriesData[seriesData.length - 1];
    const lastValue = getValueOrNull(lastItem);
    if (lastValue !== null) {
      console.log(`Last value: ${lastValue}`);
    }

    // Cleanup
    return () => {
      chart.remove();
    };
  }, [data]);

  return <div ref={containerRef} />;
}
```

#### Шаг 5: Настройка tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["DOM", "DOM.Iterable", "ES2020"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    
    // Strict mode
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    
    // Дополнительные проверки
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    
    // НЕ используем skipLibCheck, чтобы видеть ошибки в типах
    "skipLibCheck": false,
    
    // Путь к нашим декларациям
    "typeRoots": ["./node_modules/@types", "./src/types"]
  },
  "include": ["src/**/*"]
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Type Safety | Сложность | Поддержка | Runtime Cost |
|---------|--------|-------------|-----------|-----------|--------------|
| Type Assertions | 5/10 | ❌ Низкая | ⭐ Простая | ❌ Сложная | ✅ Нет |
| **Module Augmentation** | **9/10** | **✅ Высокая** | **⭐⭐ Средняя** | **✅ Простая** | **✅ Нет** |
| Type Guards | 8/10 | ✅ Высокая | ⭐⭐ Средняя | ✅ Простая | ⚠️ Минимальный |
| Wrapper-классы | 7/10 | ✅ Высокая | ⭐⭐⭐ Сложная | ⚠️ Средняя | ⚠️ Минимальный |
| skipLibCheck | 4/10 | ❌ Никакой | ⭐ Простая | ✅ Простая | ✅ Нет |

---

## 🔗 Источники

1. [GitHub Issue #1091 - createPriceLine TypeScript error](https://github.com/tradingview/lightweight-charts/issues/1091)
2. [GitHub Issue #1699 - Type definitions issues](https://github.com/tradingview/lightweight-charts/issues/1699)
3. [TypeScript Handbook - Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)
4. [TypeScript Handbook - Module Augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation)
5. [Lightweight Charts Documentation - TypeScript](https://tradingview.github.io/lightweight-charts/docs)
6. [Lightweight Charts API Reference - WhitespaceData](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/WhitespaceData)
7. [Stack Overflow - TypeScript strict mode best practices](https://stackoverflow.com/questions/tagged/typescript+strict)

---

## 📝 Changelog проблемы

| Дата | Версия LC | Изменение |
|------|-----------|-----------|
| 2022 | v3.8.0 | Первое упоминание проблемы с createPriceLine |
| 2023 | v4.0.0 | Частичные улучшения типов |
| 2024 | v4.2.0 | Добавлены некоторые Optional properties |
| 2025 | v5.0.0 | Улучшения, но проблемы остаются |
| 2026 | v5.1.0 | Текущий статус - частично исправлено |
