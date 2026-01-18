# БАГ #48: Price axis не учитывает ширину crosshair label

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#834](https://github.com/tradingview/lightweight-charts/issues/834)  
> **Версии:** v3.x - v5.x  
> **Статус:** 🔴 Open (частично исправлено, но проблемы остаются)  
> **Последнее обновление:** Сентябрь 2021

## 📋 Описание проблемы

### Суть проблемы

Price axis (ценовая шкала) **не учитывает ширину метки crosshair** при расчёте своей оптимальной ширины. Это приводит к тому, что метка crosshair с ценой **обрезается или выходит за границы** price scale, особенно при использовании длинных чисел или кастомного форматирования.

### Детали

1. **Симптомы:**
   - Метка crosshair обрезается справа
   - Цена частично скрыта
   - При движении crosshair label "дёргается" или меняет размер

2. **Когда возникает:**
   - Использование чисел с большим количеством знаков после запятой
   - Кастомный `priceFormatter` с длинным выводом
   - Добавление суффиксов/префиксов к цене (валюта, единицы измерения)
   - Разные локализации (разделители тысяч)

3. **Техническая причина:**
   - Ширина price scale рассчитывается на основе статичных меток шкалы
   - Crosshair label может быть шире этих меток
   - Нет динамической подстройки под crosshair

### Сценарии возникновения

```typescript
import { createChart } from 'lightweight-charts';

// ❌ ПРОБЛЕМА: Длинный формат цены
const chart = createChart(container, {
  localization: {
    priceFormatter: (price: number) => {
      // Длинный формат: "$ 1,234,567.89012345 USD"
      return `$ ${price.toLocaleString('en-US', {
        minimumFractionDigits: 8,
        maximumFractionDigits: 8,
      })} USD`;
    },
  },
});

const series = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 8,
    minMove: 0.00000001,
  },
});

series.setData([
  { time: '2024-01-01', value: 0.00001234 },
  { time: '2024-01-02', value: 0.00001567 },
]);

// Crosshair label будет обрезана, т.к. price scale слишком узкая
```

### Визуализация проблемы

```
Ожидаемое поведение:        Реальное поведение:
┌──────────────────┬──────┐  ┌──────────────────┬────┐
│                  │      │  │                  │    │
│      График      │ 0.001│  │      График      │0.00│← обрезано!
│                  │      │  │                  │    │
│       ⊕─────────▶│0.0012│  │       ⊕─────────▶│0.00│← обрезано!
│                  │      │  │                  │    │
│                  │ 0.002│  │                  │0.00│
└──────────────────┴──────┘  └──────────────────┴────┘
```

### Реальные сценарии

1. **Криптовалюты с малыми ценами:**
   - SHIB: 0.00001234
   - PEPE: 0.000000789

2. **Forex с 5 знаками:**
   - EUR/USD: 1.12345

3. **Кастомное форматирование:**
   - Валютные символы: "€ 1,234.56"
   - Единицы измерения: "1,234 MW/h"

4. **Локализация:**
   - Немецкий формат: "1.234,56 €"
   - Индийский формат: "₹ 1,23,456.78"

## 🔍 Найденные решения

### Решение 1: Фиксированная минимальная ширина price scale (⭐ Простое)

**Оценка:** ⭐⭐⭐ (3/5) - Быстрый workaround

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  rightPriceScale: {
    // Устанавливаем минимальную ширину вручную
    minimumWidth: 100, // пикселей
  },
  // Или для левой шкалы
  leftPriceScale: {
    visible: true,
    minimumWidth: 100,
  },
});

const series = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 8,
    minMove: 0.00000001,
  },
});
```

**Плюсы:**
- Простая реализация
- Гарантированная ширина

**Минусы:**
- Статическое значение, не адаптируется
- Может быть слишком широкой или узкой

---

### Решение 2: Динамический расчёт ширины (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Адаптивное решение

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

interface PriceScaleWidthManager {
  updateWidth(): void;
  getOptimalWidth(): number;
}

/**
 * Создаёт менеджер для динамического расчёта ширины price scale
 */
function createPriceScaleWidthManager(
  chart: IChartApi,
  series: ISeriesApi<'Line' | 'Candlestick' | 'Area'>,
  options: {
    priceFormatter: (price: number) => string;
    paddingPx?: number;
    minWidthPx?: number;
    maxWidthPx?: number;
  }
): PriceScaleWidthManager {
  const { 
    priceFormatter, 
    paddingPx = 20, 
    minWidthPx = 50,
    maxWidthPx = 200,
  } = options;
  
  // Создаём скрытый canvas для измерения текста
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d')!;
  ctx.font = '12px -apple-system, BlinkMacSystemFont, sans-serif';
  
  function measureTextWidth(text: string): number {
    return ctx.measureText(text).width;
  }
  
  function getOptimalWidth(): number {
    const data = series.data();
    if (data.length === 0) return minWidthPx;
    
    // Собираем все возможные значения для отображения
    const values: number[] = [];
    
    for (const point of data) {
      if ('value' in point && point.value !== undefined) {
        values.push(point.value);
      }
      if ('close' in point) {
        values.push(point.open, point.high, point.low, point.close);
      }
    }
    
    if (values.length === 0) return minWidthPx;
    
    // Находим min/max для оценки диапазона
    const min = Math.min(...values);
    const max = Math.max(...values);
    
    // Генерируем тестовые значения
    const testValues = [
      min,
      max,
      (min + max) / 2,
      min * 0.9,
      max * 1.1,
    ];
    
    // Измеряем максимальную ширину форматированного текста
    let maxTextWidth = 0;
    
    for (const value of testValues) {
      const formatted = priceFormatter(value);
      const width = measureTextWidth(formatted);
      maxTextWidth = Math.max(maxTextWidth, width);
    }
    
    // Добавляем padding
    const optimalWidth = maxTextWidth + paddingPx;
    
    // Ограничиваем диапазон
    return Math.min(maxWidthPx, Math.max(minWidthPx, optimalWidth));
  }
  
  function updateWidth(): void {
    const width = getOptimalWidth();
    
    const priceScaleId = series.options().priceScaleId || 'right';
    
    if (priceScaleId === 'right') {
      chart.applyOptions({
        rightPriceScale: {
          minimumWidth: width,
        },
      });
    } else if (priceScaleId === 'left') {
      chart.applyOptions({
        leftPriceScale: {
          minimumWidth: width,
        },
      });
    }
  }
  
  return {
    updateWidth,
    getOptimalWidth,
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const priceFormatter = (price: number): string => {
  return `$ ${price.toLocaleString('en-US', {
    minimumFractionDigits: 8,
    maximumFractionDigits: 8,
  })}`;
};

const chart = createChart(container, {
  localization: {
    priceFormatter,
  },
});

const series = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 8,
    minMove: 0.00000001,
  },
});

const widthManager = createPriceScaleWidthManager(chart, series, {
  priceFormatter,
  paddingPx: 25,
});

// Обновляем после загрузки данных
series.setData(data);
widthManager.updateWidth();

// Обновляем при изменении данных
function onDataUpdate(newData: LineData[]): void {
  series.setData(newData);
  widthManager.updateWidth();
}
```

**Плюсы:**
- Автоматическая адаптация
- Учитывает реальный формат
- Настраиваемые границы

**Минусы:**
- Дополнительный код
- Небольшой overhead на измерение

---

### Решение 3: CSS overflow fix

**Оценка:** ⭐⭐⭐ (3/5) - Визуальный workaround

```typescript
// Стили для предотвращения обрезки
const styles = `
  .tv-lightweight-charts {
    overflow: visible !important;
  }
  
  /* Для crosshair label */
  .tv-lightweight-charts canvas {
    overflow: visible;
  }
  
  /* Контейнер графика */
  #chart-container {
    padding-right: 50px; /* Дополнительное пространство для label */
    position: relative;
  }
`;

// Добавляем стили
const styleSheet = document.createElement('style');
styleSheet.textContent = styles;
document.head.appendChild(styleSheet);

// Или через wrapper
const chartWrapper = document.createElement('div');
chartWrapper.style.cssText = `
  position: relative;
  padding-right: 50px;
  box-sizing: border-box;
`;
container.appendChild(chartWrapper);

const chart = createChart(chartWrapper);
```

**Плюсы:**
- Не требует изменения логики
- Быстрая реализация

**Минусы:**
- Не решает проблему полностью
- Может нарушить layout
- Зависит от внутренней структуры DOM

---

### Решение 4: Адаптивный priceFormatter

**Оценка:** ⭐⭐⭐⭐ (4/5) - Умный форматтер

```typescript
import { createChart, PriceFormatterFn } from 'lightweight-charts';

interface AdaptiveFormatterOptions {
  maxLength?: number;
  currency?: string;
  locale?: string;
}

/**
 * Создаёт адаптивный форматтер, который сокращает вывод при необходимости
 */
function createAdaptivePriceFormatter(
  options: AdaptiveFormatterOptions = {}
): PriceFormatterFn {
  const { 
    maxLength = 12, 
    currency = '$',
    locale = 'en-US',
  } = options;
  
  return (price: number): string => {
    // Базовое форматирование
    let formatted: string;
    const absPrice = Math.abs(price);
    
    if (absPrice >= 1e9) {
      // Миллиарды
      formatted = `${(price / 1e9).toFixed(2)}B`;
    } else if (absPrice >= 1e6) {
      // Миллионы
      formatted = `${(price / 1e6).toFixed(2)}M`;
    } else if (absPrice >= 1e3) {
      // Тысячи
      formatted = `${(price / 1e3).toFixed(2)}K`;
    } else if (absPrice >= 1) {
      // Целые числа
      formatted = price.toFixed(2);
    } else if (absPrice >= 0.01) {
      // Центы
      formatted = price.toFixed(4);
    } else if (absPrice >= 0.0001) {
      // Мелкие значения
      formatted = price.toFixed(6);
    } else {
      // Очень мелкие (crypto)
      formatted = price.toExponential(2);
    }
    
    // Добавляем валюту, если помещается
    const withCurrency = `${currency}${formatted}`;
    
    if (withCurrency.length <= maxLength) {
      return withCurrency;
    }
    
    // Укорачиваем если не помещается
    return formatted.slice(0, maxLength);
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const formatter = createAdaptivePriceFormatter({
  maxLength: 10,
  currency: '$ ',
});

const chart = createChart(container, {
  localization: {
    priceFormatter: formatter,
  },
  rightPriceScale: {
    minimumWidth: 80,
  },
});

// Тесты форматтера
console.log(formatter(1234567890));  // "$ 1.23B"
console.log(formatter(1234567));     // "$ 1.23M"
console.log(formatter(12345));       // "$ 12.35K"
console.log(formatter(123.456));     // "$ 123.46"
console.log(formatter(0.00001234));  // "$ 0.000012"
console.log(formatter(0.000000001)); // "$ 1.00e-9"
```

**Плюсы:**
- Умное сокращение
- Всегда помещается
- Читаемый вывод

**Минусы:**
- Потеря точности при сокращении
- Не подходит для точных значений

---

### Решение 5: Полный контроль через overlay

**Оценка:** ⭐⭐⭐⭐ (4/5) - Кастомный crosshair label

```typescript
import { createChart, IChartApi, MouseEventParams } from 'lightweight-charts';

interface CustomCrosshairLabelOptions {
  formatter: (price: number) => string;
  backgroundColor?: string;
  textColor?: string;
  padding?: number;
  fontSize?: number;
}

/**
 * Создаёт кастомный crosshair label с полным контролем над размером
 */
function createCustomCrosshairLabel(
  chart: IChartApi,
  container: HTMLElement,
  series: ISeriesApi<any>,
  options: CustomCrosshairLabelOptions
): { destroy: () => void } {
  const {
    formatter,
    backgroundColor = '#4c525e',
    textColor = '#fff',
    padding = 5,
    fontSize = 12,
  } = options;
  
  // Создаём DOM элемент для label
  const label = document.createElement('div');
  label.style.cssText = `
    position: absolute;
    right: 0;
    background: ${backgroundColor};
    color: ${textColor};
    padding: ${padding}px ${padding * 2}px;
    font-size: ${fontSize}px;
    font-family: -apple-system, BlinkMacSystemFont, sans-serif;
    border-radius: 3px;
    pointer-events: none;
    z-index: 1000;
    white-space: nowrap;
    display: none;
    transform: translateY(-50%);
  `;
  container.style.position = 'relative';
  container.appendChild(label);
  
  // Отключаем встроенный crosshair label
  chart.applyOptions({
    crosshair: {
      horzLine: {
        labelVisible: false,
      },
    },
  });
  
  // Подписываемся на движение crosshair
  const handleCrosshairMove = (param: MouseEventParams): void => {
    if (!param.point || param.point.y === undefined) {
      label.style.display = 'none';
      return;
    }
    
    // Получаем цену в точке crosshair
    const price = series.coordinateToPrice(param.point.y);
    
    if (price === null) {
      label.style.display = 'none';
      return;
    }
    
    // Форматируем и отображаем
    label.textContent = formatter(price);
    label.style.display = 'block';
    label.style.top = `${param.point.y}px`;
  };
  
  chart.subscribeCrosshairMove(handleCrosshairMove);
  
  return {
    destroy: () => {
      chart.unsubscribeCrosshairMove(handleCrosshairMove);
      label.remove();
    },
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container, {
  rightPriceScale: {
    minimumWidth: 60, // Минимальная ширина для обычных меток
  },
});

const series = chart.addLineSeries();
series.setData(data);

// Создаём кастомный label
const customLabel = createCustomCrosshairLabel(chart, container, series, {
  formatter: (price) => `$ ${price.toFixed(8)} USD`,
  backgroundColor: '#2962ff',
  textColor: '#ffffff',
  padding: 8,
});

// При удалении графика
function cleanup(): void {
  customLabel.destroy();
  chart.remove();
}
```

**Плюсы:**
- Полный контроль над стилями и размером
- Нет ограничений на длину
- Кастомизируемый дизайн

**Минусы:**
- Больше кода
- Нужно синхронизировать с встроенным UI
- Дополнительный DOM элемент

## ✅ Рекомендуемое решение

Комбинация **решений 2 и 4** для оптимального результата:

```typescript
// Полный пример
const formatter = createAdaptivePriceFormatter({
  maxLength: 12,
  currency: '$ ',
});

const chart = createChart(container, {
  localization: {
    priceFormatter: formatter,
  },
});

const series = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: 8,
    minMove: 0.00000001,
  },
});

const widthManager = createPriceScaleWidthManager(chart, series, {
  priceFormatter: formatter,
  paddingPx: 20,
  minWidthPx: 60,
  maxWidthPx: 150,
});

series.setData(data);
widthManager.updateWidth();
```

## 📊 Сравнительная таблица решений

| Решение | Адаптивность | Сложность | Точность | Рекомендация |
|---------|--------------|-----------|----------|--------------|
| **#1 Фиксированная ширина** | ⭐ | Низкая | Полная | Простые случаи |
| **#2 Динамический расчёт** | ⭐⭐⭐⭐⭐ | Средняя | Полная | ✅ Рекомендуется |
| **#3 CSS fix** | ⭐⭐ | Низкая | Полная | Workaround |
| **#4 Адаптивный formatter** | ⭐⭐⭐⭐ | Низкая | Частичная | Общее применение |
| **#5 Custom overlay** | ⭐⭐⭐⭐⭐ | Высокая | Полная | Full control |

## 🔗 Источники

- [GitHub Issue #834](https://github.com/tradingview/lightweight-charts/issues/834) - Price axis doesn't respect crosshair label width
- [Price Scale Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceScaleOptions) - Документация опций
- [Crosshair Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/CrosshairOptions) - Настройки crosshair

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
