# БАГ #55: Метки срезаются на краях с fixEdge

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#835](https://github.com/tradingview/lightweight-charts/issues/835)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2021

## 📋 Описание проблемы

### Суть проблемы

При использовании опции `fixLeftEdge: true` или `fixRightEdge: true` на временной шкале, **метки времени на крайних позициях могут обрезаться или частично выходить за пределы видимой области**. Это косметический дефект, влияющий на читаемость.

### Детали

1. **Симптомы:**
   - Метки времени частично обрезаны на левом/правом краю
   - Текст выходит за границы контейнера
   - Неполная информация на краях графика

2. **Когда возникает:**
   - `fixLeftEdge: true` или `fixRightEdge: true`
   - Первая/последняя метка близко к краю
   - При определённых размерах контейнера

3. **Ожидаемое поведение:**
   - Метки не должны обрезаться
   - Либо сдвигаться внутрь
   - Либо не отображаться если не помещаются

### Визуализация проблемы

```
Ожидаемое:                        Реальное:
┌─────────────────────────────┐   ┌─────────────────────────────┐
│                             │   │                             │
│    Chart Content            │   │    Chart Content            │
│                             │   │                             │
├─────────────────────────────┤   ├─────────────────────────────┤
│ 01 Jan │ 02 Jan │ 03 Jan  │   │an │ 02 Jan │ 03 Jan │ 04 Ja│
└─────────────────────────────┘   └─────────────────────────────┘
  ↑ метка полная                    ↑ обрезана      обрезана ↑
```

### Сценарий воспроизведения

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  width: 400,
  height: 300,
  timeScale: {
    fixLeftEdge: true,  // Фиксируем левый край
    fixRightEdge: true, // Фиксируем правый край
    borderVisible: true,
  },
});

const series = chart.addLineSeries();
series.setData([
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 105 },
  { time: '2024-01-03', value: 102 },
]);

// При определённых размерах метки "01 Jan" и "03 Jan" могут обрезаться
```

## 🔍 Найденные решения

### Решение 1: Добавление padding через rightOffset/leftOffset

**Оценка:** ⭐⭐⭐⭐ (4/5) - Простое решение

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  width: 400,
  height: 300,
  timeScale: {
    fixLeftEdge: true,
    fixRightEdge: true,
    rightOffset: 5,  // Отступ справа (в барах)
    // leftOffset не существует, используем другой подход
  },
  rightPriceScale: {
    borderVisible: true,
  },
});

// Для левого края используем whitespace данные
const series = chart.addLineSeries();

// Добавляем whitespace в начало данных
const dataWithPadding = [
  { time: '2023-12-30' }, // Whitespace
  { time: '2023-12-31' }, // Whitespace
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 105 },
  { time: '2024-01-03', value: 102 },
];

series.setData(dataWithPadding);
```

**Плюсы:**
- Использует стандартный API
- Работает для правого края

**Минусы:**
- Для левого края нужен workaround с whitespace
- Может влиять на scroll поведение

---

### Решение 2: CSS overflow visible (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Эффективное решение

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;

// Добавляем wrapper для управления overflow
const wrapper = document.createElement('div');
wrapper.style.cssText = `
  position: relative;
  width: 100%;
  padding: 0 20px; /* Padding для меток */
  box-sizing: border-box;
`;

const chartContainer = document.createElement('div');
chartContainer.style.cssText = `
  width: 100%;
  overflow: visible; /* Разрешаем выход за границы */
`;

wrapper.appendChild(chartContainer);
container.appendChild(wrapper);

const chart = createChart(chartContainer, {
  width: chartContainer.clientWidth,
  height: 300,
  timeScale: {
    fixLeftEdge: true,
    fixRightEdge: true,
  },
});

// Стили для canvas элементов
const style = document.createElement('style');
style.textContent = `
  .tv-lightweight-charts {
    overflow: visible !important;
  }
  .tv-lightweight-charts canvas {
    overflow: visible !important;
  }
`;
document.head.appendChild(style);
```

**Плюсы:**
- Метки видны полностью
- Не влияет на данные
- Простая реализация

**Минусы:**
- Может повлиять на layout страницы
- Нужно следить за padding

---

### Решение 3: Custom time axis labels через HTML

**Оценка:** ⭐⭐⭐⭐ (4/5) - Полный контроль

```typescript
import { createChart, IChartApi, Time } from 'lightweight-charts';

class CustomTimeAxisLabels {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _labelsContainer: HTMLElement;
  private _labels: HTMLElement[] = [];
  
  constructor(chart: IChartApi, container: HTMLElement) {
    this._chart = chart;
    this._container = container;
    
    // Отключаем стандартные метки времени
    this._chart.timeScale().applyOptions({
      visible: true,
      ticksVisible: false,
    });
    
    // Создаём контейнер для кастомных меток
    this._labelsContainer = document.createElement('div');
    this._labelsContainer.style.cssText = `
      position: absolute;
      bottom: 0;
      left: 0;
      right: 0;
      height: 30px;
      display: flex;
      justify-content: space-between;
      padding: 0 10px; /* Гарантированный padding */
      box-sizing: border-box;
      pointer-events: none;
    `;
    
    this._container.appendChild(this._labelsContainer);
    
    // Подписываемся на изменения
    this._chart.timeScale().subscribeVisibleTimeRangeChange(() => this._updateLabels());
  }
  
  private _updateLabels(): void {
    const visibleRange = this._chart.timeScale().getVisibleRange();
    if (!visibleRange) return;
    
    // Очищаем старые метки
    this._labelsContainer.innerHTML = '';
    this._labels = [];
    
    // Получаем видимые точки времени
    const timeScale = this._chart.timeScale();
    const width = this._container.clientWidth;
    
    // Создаём метки с учётом padding
    const labelCount = Math.min(5, Math.floor(width / 80));
    const step = width / (labelCount + 1);
    
    for (let i = 1; i <= labelCount; i++) {
      const x = step * i;
      const time = timeScale.coordinateToTime(x);
      
      if (time !== null) {
        const label = this._createLabel(time, x);
        this._labelsContainer.appendChild(label);
        this._labels.push(label);
      }
    }
  }
  
  private _createLabel(time: Time, x: number): HTMLElement {
    const label = document.createElement('div');
    label.style.cssText = `
      position: absolute;
      left: ${x}px;
      transform: translateX(-50%);
      font-size: 11px;
      color: #888;
      white-space: nowrap;
    `;
    
    // Форматируем время
    const date = new Date(time as number * 1000);
    label.textContent = date.toLocaleDateString('en-US', { 
      month: 'short', 
      day: 'numeric' 
    });
    
    return label;
  }
  
  destroy(): void {
    this._labelsContainer.remove();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.position = 'relative';

const chart = createChart(container, {
  timeScale: {
    fixLeftEdge: true,
    fixRightEdge: true,
  },
});

const series = chart.addLineSeries();
series.setData(data);

const customLabels = new CustomTimeAxisLabels(chart, container);
```

**Плюсы:**
- Полный контроль над метками
- Гарантированный padding
- Кастомное форматирование

**Минусы:**
- Сложнее реализовать
- Нужно синхронизировать с zoom/scroll

---

### Решение 4: Динамический resize с учётом меток

**Оценка:** ⭐⭐⭐ (3/5) - Адаптивное решение

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

class AdaptiveChartSizer {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _labelPadding = 30; // Padding для меток
  
  constructor(container: HTMLElement) {
    this._container = container;
    
    // Создаём chart с уменьшенной шириной
    const availableWidth = container.clientWidth - this._labelPadding * 2;
    
    this._chart = createChart(container, {
      width: availableWidth,
      height: 300,
      timeScale: {
        fixLeftEdge: true,
        fixRightEdge: true,
      },
    });
    
    // Центрируем chart
    const chartElement = container.querySelector('.tv-lightweight-charts') as HTMLElement;
    if (chartElement) {
      chartElement.style.marginLeft = `${this._labelPadding}px`;
    }
    
    // ResizeObserver для адаптации
    const resizeObserver = new ResizeObserver(() => this._handleResize());
    resizeObserver.observe(container);
  }
  
  private _handleResize(): void {
    const availableWidth = this._container.clientWidth - this._labelPadding * 2;
    this._chart.resize(availableWidth, 300);
  }
  
  get chart(): IChartApi {
    return this._chart;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
const sizer = new AdaptiveChartSizer(container);

const series = sizer.chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Автоматическая адаптация
- Резервирует место для меток

**Минусы:**
- Уменьшает область графика
- Может не подойти для узких контейнеров

---

### Решение 5: Отключение fixEdge при малом размере

**Оценка:** ⭐⭐⭐⭐ (4/5) - Условное поведение

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

interface SmartEdgeFixOptions {
  minWidthForFixEdge: number;
  fixLeftEdge: boolean;
  fixRightEdge: boolean;
}

class SmartEdgeFix {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _options: SmartEdgeFixOptions;
  
  constructor(
    chart: IChartApi, 
    container: HTMLElement,
    options: Partial<SmartEdgeFixOptions> = {}
  ) {
    this._chart = chart;
    this._container = container;
    this._options = {
      minWidthForFixEdge: options.minWidthForFixEdge ?? 500,
      fixLeftEdge: options.fixLeftEdge ?? true,
      fixRightEdge: options.fixRightEdge ?? true,
    };
    
    this._updateEdgeFix();
    
    // Отслеживаем resize
    const resizeObserver = new ResizeObserver(() => this._updateEdgeFix());
    resizeObserver.observe(container);
  }
  
  private _updateEdgeFix(): void {
    const width = this._container.clientWidth;
    const shouldFix = width >= this._options.minWidthForFixEdge;
    
    this._chart.timeScale().applyOptions({
      fixLeftEdge: shouldFix && this._options.fixLeftEdge,
      fixRightEdge: shouldFix && this._options.fixRightEdge,
    });
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;

const chart = createChart(container, {
  timeScale: {
    fixLeftEdge: true,
    fixRightEdge: true,
  },
});

const series = chart.addLineSeries();
series.setData(data);

// Smart edge fix - отключает fix при малом размере
const smartEdge = new SmartEdgeFix(chart, container, {
  minWidthForFixEdge: 400,
});
```

**Плюсы:**
- Адаптивное поведение
- Работает на разных размерах экрана

**Минусы:**
- Разное поведение на разных размерах
- Может быть непривычно для пользователей

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 2** (CSS overflow):

```typescript
// Минимальный пример
const wrapper = document.createElement('div');
wrapper.style.cssText = 'padding: 0 20px; box-sizing: border-box;';

const chartContainer = document.createElement('div');
chartContainer.style.overflow = 'visible';

wrapper.appendChild(chartContainer);
container.appendChild(wrapper);

const chart = createChart(chartContainer, {
  timeScale: {
    fixLeftEdge: true,
    fixRightEdge: true,
  },
});
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Надёжность | Влияние на layout | Рекомендация |
|---------|----------|------------|-------------------|--------------|
| **#1 rightOffset** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Правый край |
| **#2 CSS overflow** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ Универсальное |
| **#3 Custom labels** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Полный контроль |
| **#4 Adaptive resize** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Резервирование места |
| **#5 Smart fix** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Адаптивный UX |

## 🔗 Источники

- [GitHub Issue #835](https://github.com/tradingview/lightweight-charts/issues/835) - Labels cut at edges
- [Time Scale Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions)
- [fixLeftEdge/fixRightEdge](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions#fixleftedge)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
