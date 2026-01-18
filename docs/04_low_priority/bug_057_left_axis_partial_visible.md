# БАГ #57: Левая ось видна только частично

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#513](https://github.com/tradingview/lightweight-charts/issues/513)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2020

## 📋 Описание проблемы

### Суть проблемы

При использовании **левой ценовой оси** (`leftPriceScale: { visible: true }`) в некоторых случаях **ось может быть частично обрезана или выходить за границы контейнера**. Метки и граница оси могут быть не полностью видны.

### Детали

1. **Симптомы:**
   - Левая часть ценовой оси обрезана
   - Метки цен частично видны
   - Граница оси выходит за пределы контейнера

2. **Когда возникает:**
   - Контейнер с `overflow: hidden`
   - Недостаточная ширина для метки
   - Большие числа на оси (требуют больше места)

3. **Ожидаемое поведение:**
   - Левая ось полностью видна
   - Автоматическое резервирование места

### Визуализация проблемы

```
Ожидаемое:                      Реальное:
┌───────┬─────────────────┐     ┬─────────────────┐
│ 1200  │                 │     200  │                 │
│       │                 │          │                 │
│ 1150  │   Chart Area    │     150  │   Chart Area    │
│       │                 │          │                 │
│ 1100  │                 │     100  │                 │
└───────┴─────────────────┘     ┴─────────────────┘
  ↑ Полностью видна               ↑ Обрезана
```

### Сценарий воспроизведения

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;
container.style.cssText = `
  width: 400px;
  overflow: hidden; /* ❌ Может обрезать левую ось */
`;

const chart = createChart(container, {
  leftPriceScale: {
    visible: true,
    borderVisible: true,
  },
  rightPriceScale: {
    visible: false,
  },
});

const series = chart.addLineSeries({
  priceScaleId: 'left',
});

series.setData([
  { time: '2024-01-01', value: 12345.67 }, // Длинное число
  { time: '2024-01-02', value: 12456.78 },
]);

// Метки "12345.67" могут не поместиться и обрезаться
```

## 🔍 Найденные решения

### Решение 1: CSS overflow visible (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Простое и эффективное

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;

// Важно: overflow должен быть visible
container.style.cssText = `
  width: 400px;
  height: 300px;
  overflow: visible; /* ✅ Позволяет оси быть видимой */
`;

// Или добавляем padding слева
container.style.cssText = `
  width: 400px;
  height: 300px;
  padding-left: 60px; /* Резервируем место для оси */
  box-sizing: border-box;
`;

const chart = createChart(container, {
  leftPriceScale: {
    visible: true,
  },
});
```

**Плюсы:**
- Простое решение
- Не требует дополнительного кода

**Минусы:**
- Может влиять на layout страницы

---

### Решение 2: Wrapper с padding

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Контролируемое решение

```typescript
import { createChart } from 'lightweight-charts';

// HTML структура
const wrapper = document.createElement('div');
wrapper.style.cssText = `
  width: 100%;
  padding-left: 60px; /* Место для левой оси */
  box-sizing: border-box;
`;

const chartContainer = document.createElement('div');
chartContainer.style.cssText = `
  width: 100%;
  height: 300px;
`;

wrapper.appendChild(chartContainer);
document.getElementById('app')!.appendChild(wrapper);

const chart = createChart(chartContainer, {
  autoSize: true,
  leftPriceScale: {
    visible: true,
  },
  rightPriceScale: {
    visible: false,
  },
});

const series = chart.addLineSeries({ priceScaleId: 'left' });
series.setData(data);
```

**Плюсы:**
- Явное резервирование места
- Предсказуемый layout

**Минусы:**
- Дополнительный wrapper элемент

---

### Решение 3: Динамический расчёт ширины оси

**Оценка:** ⭐⭐⭐⭐ (4/5) - Адаптивное решение

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

class LeftAxisManager {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _series: ISeriesApi<any> | null = null;
  private _padding = 10;
  
  constructor(container: HTMLElement) {
    this._container = container;
    
    // Создаём wrapper
    const wrapper = document.createElement('div');
    wrapper.style.cssText = `
      display: flex;
      width: 100%;
      height: 100%;
    `;
    
    // Spacer для левой оси
    const leftSpacer = document.createElement('div');
    leftSpacer.id = 'left-axis-spacer';
    leftSpacer.style.cssText = `
      width: 60px; /* Начальная ширина */
      flex-shrink: 0;
    `;
    
    // Chart container
    const chartContainer = document.createElement('div');
    chartContainer.style.cssText = `
      flex: 1;
      min-width: 0;
    `;
    
    wrapper.appendChild(leftSpacer);
    wrapper.appendChild(chartContainer);
    container.appendChild(wrapper);
    
    this._chart = createChart(chartContainer, {
      autoSize: true,
      leftPriceScale: {
        visible: true,
      },
      rightPriceScale: {
        visible: false,
      },
    });
  }
  
  addSeries(data: { time: Time; value: number }[]): ISeriesApi<'Line'> {
    this._series = this._chart.addLineSeries({
      priceScaleId: 'left',
    });
    
    this._series.setData(data);
    this._adjustAxisWidth(data);
    
    return this._series;
  }
  
  private _adjustAxisWidth(data: { value: number }[]): void {
    // Находим максимальное значение для расчёта ширины
    const maxValue = Math.max(...data.map(d => d.value));
    const minValue = Math.min(...data.map(d => d.value));
    
    // Определяем сколько символов нужно
    const maxDigits = Math.max(
      maxValue.toFixed(2).length,
      minValue.toFixed(2).length
    );
    
    // Примерно 8px на символ + padding
    const neededWidth = maxDigits * 8 + this._padding * 2;
    
    const spacer = this._container.querySelector('#left-axis-spacer') as HTMLElement;
    if (spacer) {
      spacer.style.width = `${neededWidth}px`;
    }
  }
  
  get chart(): IChartApi {
    return this._chart;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.height = '300px';

const manager = new LeftAxisManager(container);
manager.addSeries([
  { time: '2024-01-01', value: 12345.67 },
  { time: '2024-01-02', value: 12456.78 },
]);
```

**Плюсы:**
- Адаптируется к данным
- Автоматический расчёт

**Минусы:**
- Сложнее реализация
- Нужно пересчитывать при изменении данных

---

### Решение 4: Использование priceFormat для контроля ширины

**Оценка:** ⭐⭐⭐⭐ (4/5) - Контроль через API

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;
container.style.cssText = `
  width: 100%;
  height: 300px;
  padding-left: 50px;
  box-sizing: border-box;
`;

const chart = createChart(container, {
  leftPriceScale: {
    visible: true,
    autoScale: true,
  },
  rightPriceScale: {
    visible: false,
  },
});

const series = chart.addLineSeries({
  priceScaleId: 'left',
  priceFormat: {
    type: 'price',
    precision: 0, // Меньше знаков = меньше ширина
    minMove: 1,
  },
});

// Или кастомный formatter для сокращения
const seriesWithCustomFormat = chart.addLineSeries({
  priceScaleId: 'left',
  priceFormat: {
    type: 'custom',
    formatter: (price: number) => {
      if (price >= 1000000) return `${(price / 1000000).toFixed(1)}M`;
      if (price >= 1000) return `${(price / 1000).toFixed(1)}K`;
      return price.toFixed(0);
    },
    minMove: 1,
  },
});

series.setData(data);
```

**Плюсы:**
- Контроль через API
- Уменьшает ширину меток

**Минусы:**
- Теряется точность отображения

---

### Решение 5: CSS Grid с фиксированной колонкой

**Оценка:** ⭐⭐⭐⭐ (4/5) - Современный layout

```html
<div id="chart-container" style="
  display: grid;
  grid-template-columns: 60px 1fr;
  width: 100%;
  height: 300px;
">
  <div id="left-axis-area"></div>
  <div id="chart-area"></div>
</div>
```

```typescript
import { createChart } from 'lightweight-charts';

const chartArea = document.getElementById('chart-area')!;

const chart = createChart(chartArea, {
  autoSize: true,
  leftPriceScale: {
    visible: true,
  },
  rightPriceScale: {
    visible: false,
  },
  layout: {
    // Левая ось будет "выезжать" в grid область
  },
});

// CSS для позиционирования оси
const style = document.createElement('style');
style.textContent = `
  #chart-area .tv-lightweight-charts {
    overflow: visible;
    margin-left: -60px; /* Сдвигаем влево в зарезервированную область */
  }
`;
document.head.appendChild(style);
```

**Плюсы:**
- Grid layout
- Точное резервирование

**Минусы:**
- Требует CSS хаки
- Сложная структура

---

### Решение 6: Проверка и автоисправление

**Оценка:** ⭐⭐⭐⭐ (4/5) - Диагностическое решение

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

interface AxisVisibilityDiagnostics {
  isLeftAxisVisible: boolean;
  containerOverflow: string;
  estimatedAxisWidth: number;
  availableSpace: number;
  issues: string[];
}

/**
 * Диагностирует проблемы с видимостью левой оси
 */
function diagnoseLeftAxisVisibility(container: HTMLElement): AxisVisibilityDiagnostics {
  const style = getComputedStyle(container);
  const issues: string[] = [];
  
  // Проверяем overflow
  if (style.overflow === 'hidden' || style.overflowX === 'hidden') {
    issues.push('Container has overflow:hidden which may clip left axis');
  }
  
  // Проверяем padding
  const paddingLeft = parseFloat(style.paddingLeft) || 0;
  if (paddingLeft < 50) {
    issues.push(`Insufficient left padding (${paddingLeft}px, recommend 50-70px)`);
  }
  
  // Проверяем родителей
  let parent = container.parentElement;
  while (parent) {
    const parentStyle = getComputedStyle(parent);
    if (parentStyle.overflow === 'hidden') {
      issues.push(`Parent element has overflow:hidden`);
    }
    parent = parent.parentElement;
  }
  
  return {
    isLeftAxisVisible: issues.length === 0,
    containerOverflow: style.overflow,
    estimatedAxisWidth: 60, // Приблизительно
    availableSpace: paddingLeft,
    issues,
  };
}

/**
 * Автоматически исправляет проблемы с левой осью
 */
function fixLeftAxisVisibility(container: HTMLElement): void {
  const diagnostics = diagnoseLeftAxisVisibility(container);
  
  if (diagnostics.issues.length > 0) {
    console.warn('Left axis visibility issues:', diagnostics.issues);
    
    // Применяем исправления
    container.style.overflow = 'visible';
    container.style.paddingLeft = '60px';
    container.style.boxSizing = 'border-box';
  }
}

/**
 * Создаёт chart с гарантированно видимой левой осью
 */
function createChartWithLeftAxis(container: HTMLElement): IChartApi {
  fixLeftAxisVisibility(container);
  
  return createChart(container, {
    autoSize: true,
    leftPriceScale: {
      visible: true,
    },
    rightPriceScale: {
      visible: false,
    },
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;

// Диагностика
const diagnostics = diagnoseLeftAxisVisibility(container);
console.log('Diagnostics:', diagnostics);

// Создание с автоисправлением
const chart = createChartWithLeftAxis(container);

const series = chart.addLineSeries({ priceScaleId: 'left' });
series.setData(data);
```

**Плюсы:**
- Автоматическая диагностика
- Логирование проблем
- Автоисправление

**Минусы:**
- Изменяет стили контейнера

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 2** (Wrapper с padding):

```typescript
// Минимальный рабочий пример
const wrapper = document.createElement('div');
wrapper.style.cssText = `
  width: 100%;
  padding-left: 60px;
  box-sizing: border-box;
`;

const chartContainer = document.createElement('div');
chartContainer.style.height = '300px';
wrapper.appendChild(chartContainer);
container.appendChild(wrapper);

const chart = createChart(chartContainer, {
  autoSize: true,
  leftPriceScale: { visible: true },
  rightPriceScale: { visible: false },
});
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Надёжность | Layout Impact | Рекомендация |
|---------|----------|------------|---------------|--------------|
| **#1 overflow visible** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Быстрый фикс |
| **#2 Wrapper + padding** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ Рекомендуемое |
| **#3 Динамическая ширина** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Адаптивный |
| **#4 priceFormat** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Сокращение меток |
| **#5 CSS Grid** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Современный CSS |
| **#6 Диагностика** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Debug |

## 🔗 Источники

- [GitHub Issue #513](https://github.com/tradingview/lightweight-charts/issues/513) - Left axis visibility
- [Price Scale Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceScaleOptions)
- [Chart Layout](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ChartOptions)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
