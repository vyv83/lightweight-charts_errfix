# БАГ #50: Отображается только текущая цена на price axis

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1452](https://github.com/tradingview/lightweight-charts/issues/1452)  
> **Версии:** v3.8+, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** Ноябрь 2023

## 📋 Описание проблемы

### Суть проблемы

На правой ценовой оси (price scale) **отображается только метка текущей (последней) цены**, в то время как **дополнительные уровни цен не показываются**. Ожидается, что ось будет отображать несколько уровней цен для лучшей ориентации.

### Детали

1. **Симптомы:**
   - Только одна метка на price axis (последняя цена)
   - Нет промежуточных уровней (tick marks)
   - Ось выглядит "пустой"

2. **Когда возникает:**
   - При определённых настройках `priceFormat`
   - Малый диапазон цен (все значения близки друг к другу)
   - Неправильно настроенный `minMove`
   - Слишком высокая `precision`

3. **Техническая причина:**
   - Tick marks рассчитываются на основе `priceFormat` серии
   - При неправильной конфигурации алгоритм не может найти подходящие уровни
   - `minMove` влияет на шаг между метками

### Сценарии возникновения

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  height: 390,
  rightPriceScale: {
    visible: true,
    ticksVisible: true, // Включено!
  },
});

const series = chart.addCandlestickSeries({
  priceFormat: {
    type: 'price',
    precision: 0, // ❌ ПРОБЛЕМА: precision = 0
    minMove: 0,   // ❌ ПРОБЛЕМА: minMove = 0 (или слишком маленький)
  },
});

series.setData([
  { time: '2024-01-01', open: 75, high: 82.8, low: 70, close: 80 },
  { time: '2024-01-02', open: 77, high: 88, low: 73, close: 86 },
  // ...
]);

// Результат: только метка "86" (последняя close) видна на оси
```

### Визуализация проблемы

```
Ожидаемое поведение:        Реальное поведение:
┌──────────────────┬──────┐  ┌──────────────────┬──────┐
│                  │ 90   │  │                  │      │
│                  │      │  │                  │      │
│   Candlestick    │ 85   │  │   Candlestick    │      │
│      Chart       │      │  │      Chart       │  86  │ ← только одна!
│                  │ 80   │  │                  │      │
│                  │      │  │                  │      │
│                  │ 75   │  │                  │      │
│                  │ 70   │  │                  │      │
└──────────────────┴──────┘  └──────────────────┴──────┘
```

### Реальные сценарии

1. **Forex с неправильным minMove:**
   - EUR/USD отображается как целое число
   - Потеря дробной части

2. **Криптовалюты:**
   - Bitcoin с precision: 0
   - Altcoins без правильной настройки

3. **Копирование конфигурации:**
   - Шаблонный код с неподходящими настройками
   - Отсутствие кастомизации под актив

## 🔍 Найденные решения

### Решение 1: Правильная настройка priceFormat (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Основное решение

```typescript
import { createChart, PriceFormatBuiltIn } from 'lightweight-charts';

const chart = createChart(container, {
  height: 400,
  rightPriceScale: {
    visible: true,
    ticksVisible: true,
  },
});

// ✅ ПРАВИЛЬНО: Корректный priceFormat для разных типов активов

// Для акций (2 знака после запятой)
const stockSeries = chart.addCandlestickSeries({
  priceFormat: {
    type: 'price',
    precision: 2,
    minMove: 0.01, // Минимальный шаг цены
  },
});

// Для криптовалют (8 знаков)
const cryptoSeries = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 8,
    minMove: 0.00000001, // Satoshi
  },
});

// Для Forex (5 знаков - pipettes)
const forexSeries = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 5,
    minMove: 0.00001,
  },
});

// Для процентов
const percentSeries = chart.addLineSeries({
  priceFormat: {
    type: 'percent',
  },
});

// Для объёмов
const volumeSeries = chart.addHistogramSeries({
  priceFormat: {
    type: 'volume',
  },
});
```

**Плюсы:**
- Решает проблему в корне
- Правильное отображение для разных типов данных
- Стандартный API

**Минусы:**
- Нужно знать правильные параметры для каждого типа актива

---

### Решение 2: Автоматическое определение precision

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Универсальное решение

```typescript
import { createChart, ISeriesApi, PriceFormatBuiltIn } from 'lightweight-charts';

interface PriceFormatConfig {
  precision: number;
  minMove: number;
}

/**
 * Автоматически определяет оптимальный priceFormat на основе данных
 */
function detectPriceFormat(prices: number[]): PriceFormatConfig {
  if (prices.length === 0) {
    return { precision: 2, minMove: 0.01 };
  }
  
  // Анализируем значения
  const min = Math.min(...prices);
  const max = Math.max(...prices);
  const avgPrice = prices.reduce((a, b) => a + b, 0) / prices.length;
  
  // Определяем precision на основе величины цены
  let precision: number;
  let minMove: number;
  
  if (avgPrice >= 10000) {
    // Очень большие числа (индексы, BTC)
    precision = 0;
    minMove = 1;
  } else if (avgPrice >= 100) {
    // Обычные акции
    precision = 2;
    minMove = 0.01;
  } else if (avgPrice >= 1) {
    // Дешёвые акции, некоторые forex пары
    precision = 4;
    minMove = 0.0001;
  } else if (avgPrice >= 0.01) {
    // Forex, дешёвые активы
    precision = 5;
    minMove = 0.00001;
  } else if (avgPrice >= 0.0001) {
    // Мелкие криптовалюты
    precision = 6;
    minMove = 0.000001;
  } else {
    // Очень мелкие (SHIB, etc.)
    precision = 8;
    minMove = 0.00000001;
  }
  
  // Дополнительно проверяем фактические значения
  for (const price of prices.slice(0, 100)) {
    const decimalStr = price.toString().split('.')[1] || '';
    const actualPrecision = decimalStr.replace(/0+$/, '').length;
    
    if (actualPrecision > precision) {
      precision = Math.min(actualPrecision, 8);
      minMove = Math.pow(10, -precision);
    }
  }
  
  return { precision, minMove };
}

/**
 * Применяет автоматический priceFormat к серии
 */
function applyAutoPriceFormat(
  series: ISeriesApi<'Line' | 'Candlestick' | 'Area'>,
  data: { value?: number; close?: number }[]
): void {
  const prices = data
    .map(d => d.value ?? d.close)
    .filter((v): v is number => v !== undefined);
  
  const { precision, minMove } = detectPriceFormat(prices);
  
  series.applyOptions({
    priceFormat: {
      type: 'price',
      precision,
      minMove,
    },
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();

const data = [
  { time: '2024-01-01', value: 0.00001234 },
  { time: '2024-01-02', value: 0.00001567 },
  { time: '2024-01-03', value: 0.00001890 },
];

// Автоматически определяем формат
applyAutoPriceFormat(series, data);

series.setData(data);
// Теперь все уровни цен видны на оси!
```

**Плюсы:**
- Не нужно вручную подбирать параметры
- Работает с любыми данными
- Переиспользуемый код

**Минусы:**
- Анализ данных имеет overhead
- Может не идеально подойти для специфических случаев

---

### Решение 3: Кастомный priceFormatter с видимыми уровнями

**Оценка:** ⭐⭐⭐⭐ (4/5) - Полный контроль над форматированием

```typescript
import { createChart, PriceFormatterFn } from 'lightweight-charts';

interface SmartPriceFormatterOptions {
  minDecimals?: number;
  maxDecimals?: number;
  showTrailingZeros?: boolean;
}

/**
 * Создаёт умный форматтер, который адаптируется к значениям
 */
function createSmartPriceFormatter(
  options: SmartPriceFormatterOptions = {}
): PriceFormatterFn {
  const { 
    minDecimals = 2, 
    maxDecimals = 8,
    showTrailingZeros = false,
  } = options;
  
  return (price: number): string => {
    if (!Number.isFinite(price)) return 'N/A';
    
    const absPrice = Math.abs(price);
    
    // Определяем нужное количество знаков
    let decimals: number;
    
    if (absPrice === 0) {
      decimals = minDecimals;
    } else if (absPrice >= 10000) {
      decimals = 0;
    } else if (absPrice >= 100) {
      decimals = 2;
    } else if (absPrice >= 1) {
      decimals = 4;
    } else {
      // Для малых чисел определяем по первой значащей цифре
      const log = Math.floor(Math.log10(absPrice));
      decimals = Math.min(maxDecimals, Math.max(minDecimals, -log + 2));
    }
    
    let formatted = price.toFixed(decimals);
    
    // Убираем trailing zeros если не нужны
    if (!showTrailingZeros && decimals > minDecimals) {
      formatted = formatted.replace(/\.?0+$/, '');
      // Но оставляем минимум minDecimals
      const currentDecimals = (formatted.split('.')[1] || '').length;
      if (currentDecimals < minDecimals) {
        formatted = price.toFixed(minDecimals);
      }
    }
    
    return formatted;
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const formatter = createSmartPriceFormatter({
  minDecimals: 2,
  maxDecimals: 8,
});

const chart = createChart(container, {
  localization: {
    priceFormatter: formatter,
  },
  rightPriceScale: {
    visible: true,
    ticksVisible: true,
  },
});

const series = chart.addLineSeries({
  priceFormat: {
    type: 'custom',
    formatter,
    minMove: 0.00000001, // Важно установить!
  },
});

series.setData(data);
```

**Плюсы:**
- Полный контроль над форматированием
- Адаптируется к разным диапазонам
- Читаемый вывод

**Минусы:**
- Нужно отдельно настраивать minMove
- Больше кода

---

### Решение 4: Принудительное обновление price scale

**Оценка:** ⭐⭐⭐ (3/5) - Workaround для редких случаев

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

/**
 * Принудительно обновляет price scale для корректного отображения tick marks
 */
function forceUpdatePriceScale(
  chart: IChartApi,
  series: ISeriesApi<any>
): void {
  const priceScaleId = series.options().priceScaleId || 'right';
  const priceScale = chart.priceScale(priceScaleId);
  
  // Получаем текущие опции
  const currentOptions = priceScale.options();
  
  // Сбрасываем и применяем заново
  priceScale.applyOptions({
    ...currentOptions,
    autoScale: false,
  });
  
  // Небольшая задержка для перерисовки
  requestAnimationFrame(() => {
    priceScale.applyOptions({
      ...currentOptions,
      autoScale: true,
    });
  });
}

/**
 * Устанавливает visible range для price scale
 */
function setPriceScaleRange(
  chart: IChartApi,
  series: ISeriesApi<any>,
  minPrice: number,
  maxPrice: number
): void {
  const priceScaleId = series.options().priceScaleId || 'right';
  
  // Добавляем невидимые маркеры для установки диапазона
  const data = series.data();
  if (data.length < 2) return;
  
  const firstTime = data[0].time;
  const lastTime = data[data.length - 1].time;
  
  // Создаём вспомогательную серию
  const helperSeries = chart.addLineSeries({
    priceScaleId,
    visible: false,
    lastValueVisible: false,
    priceLineVisible: false,
  });
  
  helperSeries.setData([
    { time: firstTime, value: minPrice },
    { time: lastTime, value: maxPrice },
  ]);
  
  // Удаляем после применения
  setTimeout(() => {
    chart.removeSeries(helperSeries);
  }, 100);
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 4,
    minMove: 0.0001,
  },
});

series.setData(data);

// Если tick marks не отображаются, пробуем force update
forceUpdatePriceScale(chart, series);
```

**Плюсы:**
- Может помочь в edge cases
- Не требует изменения priceFormat

**Минусы:**
- Workaround, не решение
- Может вызвать мигание
- Не всегда помогает

---

### Решение 5: Полная диагностика и исправление

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Комплексный подход

```typescript
import { createChart, IChartApi, ISeriesApi, SeriesOptionsCommon } from 'lightweight-charts';

interface PriceScaleDiagnostics {
  hasTicks: boolean;
  currentPrecision: number;
  currentMinMove: number;
  recommendedPrecision: number;
  recommendedMinMove: number;
  issues: string[];
}

/**
 * Диагностирует проблемы с price scale
 */
function diagnosePriceScale(
  series: ISeriesApi<any>,
  data: { value?: number; close?: number; high?: number; low?: number }[]
): PriceScaleDiagnostics {
  const issues: string[] = [];
  const options = series.options();
  const priceFormat = options.priceFormat as any;
  
  const currentPrecision = priceFormat?.precision ?? 2;
  const currentMinMove = priceFormat?.minMove ?? 0.01;
  
  // Извлекаем все цены
  const prices: number[] = [];
  for (const item of data) {
    if (item.value !== undefined) prices.push(item.value);
    if (item.close !== undefined) prices.push(item.close);
    if (item.high !== undefined) prices.push(item.high);
    if (item.low !== undefined) prices.push(item.low);
  }
  
  if (prices.length === 0) {
    issues.push('No price data found');
    return {
      hasTicks: false,
      currentPrecision,
      currentMinMove,
      recommendedPrecision: 2,
      recommendedMinMove: 0.01,
      issues,
    };
  }
  
  const min = Math.min(...prices);
  const max = Math.max(...prices);
  const range = max - min;
  const avgPrice = prices.reduce((a, b) => a + b, 0) / prices.length;
  
  // Проверяем precision
  if (currentPrecision === 0 && avgPrice < 100) {
    issues.push('precision: 0 not suitable for small values');
  }
  
  // Проверяем minMove
  if (currentMinMove === 0) {
    issues.push('minMove: 0 prevents tick calculation');
  }
  
  if (currentMinMove > range) {
    issues.push(`minMove (${currentMinMove}) > price range (${range})`);
  }
  
  // Проверяем соответствие precision и minMove
  const expectedMinMove = Math.pow(10, -currentPrecision);
  if (Math.abs(currentMinMove - expectedMinMove) > expectedMinMove * 0.1) {
    issues.push(`minMove (${currentMinMove}) doesn't match precision (${currentPrecision})`);
  }
  
  // Рассчитываем рекомендуемые значения
  let recommendedPrecision: number;
  if (avgPrice >= 1000) recommendedPrecision = 0;
  else if (avgPrice >= 10) recommendedPrecision = 2;
  else if (avgPrice >= 0.1) recommendedPrecision = 4;
  else if (avgPrice >= 0.001) recommendedPrecision = 6;
  else recommendedPrecision = 8;
  
  const recommendedMinMove = Math.pow(10, -recommendedPrecision);
  
  return {
    hasTicks: issues.length === 0,
    currentPrecision,
    currentMinMove,
    recommendedPrecision,
    recommendedMinMove,
    issues,
  };
}

/**
 * Автоматически исправляет проблемы с price scale
 */
function fixPriceScale(
  series: ISeriesApi<any>,
  data: { value?: number; close?: number }[]
): void {
  const diagnostics = diagnosePriceScale(series, data);
  
  if (diagnostics.issues.length > 0) {
    console.warn('Price scale issues:', diagnostics.issues);
    console.log('Applying recommended settings:', {
      precision: diagnostics.recommendedPrecision,
      minMove: diagnostics.recommendedMinMove,
    });
    
    series.applyOptions({
      priceFormat: {
        type: 'price',
        precision: diagnostics.recommendedPrecision,
        minMove: diagnostics.recommendedMinMove,
      },
    });
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();

const data = [
  { time: '2024-01-01', value: 0.00001234 },
  { time: '2024-01-02', value: 0.00001567 },
];

// Диагностика
const diagnostics = diagnosePriceScale(series, data);
console.log('Diagnostics:', diagnostics);

// Автоисправление
fixPriceScale(series, data);

series.setData(data);
// Теперь tick marks отображаются корректно!
```

**Плюсы:**
- Полная диагностика проблемы
- Автоматическое исправление
- Логирование для отладки

**Минусы:**
- Больше кода
- Overhead на анализ

## ✅ Рекомендуемое решение

Используйте **комбинацию решений 2 и 5** для надёжного результата:

```typescript
// Минимальный production-ready пример

function setupSeriesWithAutoFormat(
  chart: IChartApi,
  data: { time: Time; value: number }[]
): ISeriesApi<'Line'> {
  // 1. Определяем формат на основе данных
  const prices = data.map(d => d.value);
  const avgPrice = prices.reduce((a, b) => a + b, 0) / prices.length;
  
  let precision: number;
  if (avgPrice >= 1000) precision = 0;
  else if (avgPrice >= 10) precision = 2;
  else if (avgPrice >= 0.1) precision = 4;
  else if (avgPrice >= 0.001) precision = 6;
  else precision = 8;
  
  const minMove = Math.pow(10, -precision);
  
  // 2. Создаём серию с правильным форматом
  const series = chart.addLineSeries({
    priceFormat: {
      type: 'price',
      precision,
      minMove,
    },
  });
  
  // 3. Загружаем данные
  series.setData(data);
  
  return series;
}
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Универсальность | Надёжность | Рекомендация |
|---------|----------|-----------------|------------|--------------|
| **#1 Ручной priceFormat** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Известные данные |
| **#2 Автоопределение** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Универсальное |
| **#3 Smart formatter** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Кастомный вывод |
| **#4 Force update** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | Workaround |
| **#5 Диагностика** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ Debug/Production |

## 🔧 Чеклист для отладки

1. ✅ Проверьте `precision` — не должен быть 0 для дробных чисел
2. ✅ Проверьте `minMove` — должен соответствовать precision
3. ✅ `minMove` должен быть меньше диапазона цен
4. ✅ `ticksVisible: true` включён в price scale options
5. ✅ `visible: true` для price scale
6. ✅ Данные корректны (нет NaN, Infinity)

## 🔗 Источники

- [GitHub Issue #1452](https://github.com/tradingview/lightweight-charts/issues/1452) - Price axis shows only current price
- [Price Format Documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceFormatBuiltIn) - Официальная документация
- [Series Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/SeriesOptionsCommon) - Опции серий

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
