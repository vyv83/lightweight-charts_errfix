# БАГ #26: Автоскейл и отрицательные значения

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#360](https://github.com/tradingview/lightweight-charts/issues/360)  
> **Связанные Issues:** [#392](https://github.com/tradingview/lightweight-charts/issues/392), [#1393](https://github.com/tradingview/lightweight-charts/issues/1393)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы

При автоматическом масштабировании ценовой оси (Price Axis) график может отображать отрицательные значения, даже когда данные содержат только положительные значения. Это происходит из-за того, что алгоритм автоскейла добавляет padding (отступы) для лучшей визуализации, и нижняя граница может уходить в отрицательную зону.

### Почему это проблема

1. **Финансовые данные**: Цены активов (акции, криптовалюты) не могут быть отрицательными
2. **Визуальная путаница**: Пользователи видят лишнее пустое пространство ниже нуля
3. **Профессиональный вид**: Трейдинг-платформы не должны показывать бессмысленные отрицательные цены
4. **Алгоритм подписей**: Метки на оси Y могут включать отрицательные значения

### Сценарии возникновения

```javascript
// Пример данных, где все значения положительные
const data = [
  { time: '2024-01-01', value: 5.2 },
  { time: '2024-01-02', value: 4.8 },
  { time: '2024-01-03', value: 5.5 },
  { time: '2024-01-04', value: 3.2 },  // Низкое значение близко к 0
  { time: '2024-01-05', value: 6.1 },
];

// При автоскейле ось Y может показать диапазон от -1 до 7
// вместо ожидаемого диапазона от 0 до 7
```

### Частота проблемы

- 100% при данных с малыми положительными значениями близкими к 0
- Особенно критично для low-cap криптовалют (SHIB, PEPE и т.д.)
- Проявляется при высокой волатильности около нуля

---

## 🔍 Найденные решения

### Решение 1: autoscaleInfoProvider с минимальным значением 0

**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

**Описание:** Использование `autoscaleInfoProvider` для задания минимального значения 0 на ценовой оси.

**Преимущества:**
- Официально поддерживаемое API
- Работает на уровне отдельной серии
- Полный контроль над диапазоном масштабирования
- Не влияет на другие серии

**Недостатки:**
- Требует установки на каждую серию отдельно
- Может скрыть реальные отрицательные данные (если они есть)

```javascript
const series = chart.addLineSeries({
  autoscaleInfoProvider: (original) => {
    const res = original();
    if (res !== null && res.priceRange) {
      // Ограничиваем минимум нулём
      res.priceRange.minValue = Math.max(0, res.priceRange.minValue);
    }
    return res;
  },
});
```

---

### Решение 2: Фиксированный диапазон через priceScale options

**Оценка: 6/10**

**Описание:** Установка фиксированных границ ценовой шкалы через настройки.

**Преимущества:**
- Простая реализация
- Работает для всех серий на одной шкале

**Недостатки:**
- Отключает автоскейл полностью
- Требует знания диапазона данных заранее
- Не адаптируется к изменению данных

```javascript
// Отключаем автоскейл и задаём фиксированный диапазон
chart.priceScale('right').applyOptions({
  autoScale: false,
  scaleMargins: {
    top: 0.1,
    bottom: 0,  // Минимальный отступ снизу
  },
});

// Вручную устанавливаем visible range
chart.priceScale('right').applyOptions({
  entireTextOnly: false,
});
```

---

### Решение 3: Валидация данных перед установкой

**Оценка: 7/10**

**Описание:** Предварительная фильтрация/модификация данных для исключения отрицательных значений.

**Преимущества:**
- Данные гарантированно корректны
- Работает независимо от настроек графика
- Универсальный подход

**Недостатки:**
- Изменяет исходные данные
- Не решает проблему padding при автоскейле
- Требует дополнительной обработки

```javascript
// Функция валидации данных
function sanitizeData(data) {
  return data.map(item => ({
    ...item,
    value: Math.max(0, item.value),
    // Для OHLC данных
    open: item.open !== undefined ? Math.max(0, item.open) : undefined,
    high: item.high !== undefined ? Math.max(0, item.high) : undefined,
    low: item.low !== undefined ? Math.max(0, item.low) : undefined,
    close: item.close !== undefined ? Math.max(0, item.close) : undefined,
  }));
}

series.setData(sanitizeData(rawData));
```

---

### Решение 4: scaleMargins с нулевым нижним отступом

**Оценка: 5/10**

**Описание:** Минимизация нижнего margin для уменьшения отрицательной области.

**Преимущества:**
- Простая настройка
- Уменьшает видимую отрицательную область

**Недостатки:**
- Не гарантирует отсутствие отрицательных значений
- Данные могут быть слишком близко к краю
- Проблема решается не полностью

```javascript
series.applyOptions({
  priceScaleId: 'right',
  scaleMargins: {
    top: 0.1,    // 10% отступ сверху
    bottom: 0,   // 0% отступ снизу
  },
});
```

---

### Решение 5: Комбинированный подход (autoscaleInfoProvider + scaleMargins)

**Оценка: 10/10** ⭐⭐ ЛУЧШЕЕ РЕШЕНИЕ

**Описание:** Сочетание autoscaleInfoProvider для ограничения минимума и scaleMargins для оптимального визуального представления.

**Преимущества:**
- Полный контроль над границами
- Оптимальное визуальное представление
- Работает с любыми данными
- Масштабируемое решение

**Недостатки:**
- Более сложная реализация
- Требует понимания обоих механизмов

---

## ✅ Рекомендуемое решение

### Полный код реализации

```javascript
import { createChart } from 'lightweight-charts';

/**
 * Создаёт autoscaleInfoProvider с ограничением минимального значения
 * @param {number} minFloor - Минимальное значение (по умолчанию 0)
 * @param {number} padding - Дополнительный отступ сверху в процентах (по умолчанию 0.1)
 * @returns {Function} Provider функция для autoscaleInfoProvider
 */
function createNonNegativeScaleProvider(minFloor = 0, padding = 0.1) {
  return (original) => {
    const result = original();
    
    if (result !== null && result.priceRange) {
      const { minValue, maxValue } = result.priceRange;
      
      // Ограничиваем минимум
      const clampedMin = Math.max(minFloor, minValue);
      
      // Добавляем padding сверху для лучшей визуализации
      const range = maxValue - clampedMin;
      const paddedMax = maxValue + (range * padding);
      
      result.priceRange = {
        minValue: clampedMin,
        maxValue: paddedMax,
      };
    }
    
    return result;
  };
}

// Создание графика
const chart = createChart(document.getElementById('chart'), {
  width: 800,
  height: 400,
  layout: {
    background: { type: 'solid', color: '#1E222D' },
    textColor: '#DDD',
  },
  grid: {
    vertLines: { color: '#2B2B43' },
    horzLines: { color: '#2B2B43' },
  },
  rightPriceScale: {
    borderVisible: true,
    borderColor: '#2B2B43',
    // Явно включаем автоскейл
    autoScale: true,
  },
});

// Создание серии с ограничением отрицательных значений
const lineSeries = chart.addLineSeries({
  color: '#2962FF',
  lineWidth: 2,
  // Ключевая настройка - autoscaleInfoProvider
  autoscaleInfoProvider: createNonNegativeScaleProvider(0, 0.1),
  // Дополнительные настройки для оптимального отображения
  priceScaleId: 'right',
});

// Пример данных
const data = [
  { time: '2024-01-01', value: 5.2 },
  { time: '2024-01-02', value: 4.8 },
  { time: '2024-01-03', value: 2.1 },
  { time: '2024-01-04', value: 0.5 },  // Близко к нулю, но не ниже!
  { time: '2024-01-05', value: 3.7 },
  { time: '2024-01-06', value: 6.3 },
];

lineSeries.setData(data);

// Автоматическая подгонка видимого диапазона
chart.timeScale().fitContent();
```

### Пример для нескольких серий

```javascript
// Универсальный хелпер для создания серий без отрицательных значений
function addNonNegativeSeries(chart, type, options = {}) {
  const seriesOptions = {
    ...options,
    autoscaleInfoProvider: createNonNegativeScaleProvider(0, 0.1),
  };
  
  switch (type) {
    case 'line':
      return chart.addLineSeries(seriesOptions);
    case 'area':
      return chart.addAreaSeries(seriesOptions);
    case 'candlestick':
      return chart.addCandlestickSeries(seriesOptions);
    case 'bar':
      return chart.addBarSeries(seriesOptions);
    case 'histogram':
      return chart.addHistogramSeries(seriesOptions);
    default:
      return chart.addLineSeries(seriesOptions);
  }
}

// Использование
const priceSeries = addNonNegativeSeries(chart, 'candlestick', {
  upColor: '#26a69a',
  downColor: '#ef5350',
  wickUpColor: '#26a69a',
  wickDownColor: '#ef5350',
});

const volumeSeries = addNonNegativeSeries(chart, 'histogram', {
  color: '#26a69a',
  priceFormat: { type: 'volume' },
  priceScaleId: 'volume',
});
```

### React компонент

```tsx
import React, { useEffect, useRef } from 'react';
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

interface NonNegativeChartProps {
  data: { time: string; value: number }[];
  width?: number;
  height?: number;
  minFloor?: number;
}

export const NonNegativeChart: React.FC<NonNegativeChartProps> = ({
  data,
  width = 600,
  height = 400,
  minFloor = 0,
}) => {
  const chartContainerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Line'> | null>(null);

  useEffect(() => {
    if (!chartContainerRef.current) return;

    // Создание графика
    chartRef.current = createChart(chartContainerRef.current, {
      width,
      height,
      layout: {
        background: { type: 'solid', color: '#1E222D' },
        textColor: '#DDD',
      },
    });

    // Создание серии с autoscaleInfoProvider
    seriesRef.current = chartRef.current.addLineSeries({
      color: '#2962FF',
      autoscaleInfoProvider: (original) => {
        const result = original();
        if (result !== null && result.priceRange) {
          result.priceRange.minValue = Math.max(
            minFloor,
            result.priceRange.minValue
          );
        }
        return result;
      },
    });

    return () => {
      chartRef.current?.remove();
    };
  }, [width, height, minFloor]);

  // Обновление данных
  useEffect(() => {
    if (seriesRef.current && data) {
      seriesRef.current.setData(data);
      chartRef.current?.timeScale().fitContent();
    }
  }, [data]);

  return <div ref={chartContainerRef} />;
};
```

---

## 📊 Сравнительная таблица решений

| Критерий | autoscaleInfoProvider | Фикс. диапазон | Валидация данных | scaleMargins | Комбинированный |
|----------|:--------------------:|:--------------:|:----------------:|:------------:|:---------------:|
| **Эффективность** | ✅ Полная | ⚠️ Частичная | ⚠️ Частичная | ❌ Минимальная | ✅ Полная |
| **Простота** | ✅ Простой | ✅ Простой | ⚠️ Средняя | ✅ Простой | ⚠️ Средняя |
| **Гибкость** | ✅ Высокая | ❌ Низкая | ⚠️ Средняя | ❌ Низкая | ✅ Высокая |
| **Поддержка API** | ✅ Официальная | ✅ Официальная | - | ✅ Официальная | ✅ Официальная |
| **Производительность** | ✅ Отличная | ✅ Отличная | ⚠️ Средняя | ✅ Отличная | ✅ Отличная |
| **Универсальность** | ✅ Да | ❌ Нет | ⚠️ Частично | ❌ Нет | ✅ Да |
| **Оценка** | **9/10** | **6/10** | **7/10** | **5/10** | **10/10** |

---

## 🎯 Дополнительные рекомендации

### 1. Динамическое переключение ограничений

```javascript
// Для случаев, когда нужно показывать/скрывать отрицательные значения
function updateScaleConstraints(series, allowNegative) {
  series.applyOptions({
    autoscaleInfoProvider: allowNegative
      ? undefined  // Стандартное поведение
      : createNonNegativeScaleProvider(0),
  });
}
```

### 2. Разные минимумы для разных типов данных

```javascript
// Для процентных значений (0-100%)
const percentageSeries = chart.addLineSeries({
  autoscaleInfoProvider: createNonNegativeScaleProvider(0, 0.05),
});

// Для объёмов (всегда >= 0)
const volumeSeries = chart.addHistogramSeries({
  autoscaleInfoProvider: createNonNegativeScaleProvider(0, 0.2),
});

// Для индикаторов типа RSI (0-100)
const rsiSeries = chart.addLineSeries({
  autoscaleInfoProvider: (original) => {
    return {
      priceRange: { minValue: 0, maxValue: 100 },
    };
  },
});
```

### 3. Обработка ошибок

```javascript
function safeAutoscaleProvider(minFloor = 0) {
  return (original) => {
    try {
      const result = original();
      if (result?.priceRange) {
        result.priceRange.minValue = Math.max(minFloor, result.priceRange.minValue);
      }
      return result;
    } catch (error) {
      console.warn('Autoscale provider error:', error);
      return null;  // Fallback к стандартному поведению
    }
  };
}
```

---

## 🔗 Источники

1. **GitHub Issue #360** - [Negative values on Price Axis](https://github.com/tradingview/lightweight-charts/issues/360)
2. **GitHub Issue #392** - [Add ability to override series' autoscale range](https://github.com/tradingview/lightweight-charts/issues/392)
3. **GitHub Issue #1393** - [Option to limit price scale range](https://github.com/tradingview/lightweight-charts/issues/1393)
4. **Lightweight Charts Documentation** - [Series Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/SeriesOptionsCommon#autoscaleinfoprovider)
5. **Stack Overflow** - [Exclude Series from Default/Auto Visible Range](https://stackoverflow.com/questions/75287789)
6. **GitHub Discussion** - [Price scale customization](https://github.com/tradingview/lightweight-charts/discussions)

---

## 📝 Примечания

- `autoscaleInfoProvider` доступен начиная с версии 3.8
- В версии 5.x API полностью совместим с предыдущими версиями
- Решение работает для всех типов серий (Line, Area, Candlestick, Bar, Histogram)
- При использовании нескольких серий на одной price scale, autoscaleInfoProvider нужно настроить для каждой серии

---

*Документ создан: 18 января 2026*  
*Последнее обновление: 18 января 2026*
