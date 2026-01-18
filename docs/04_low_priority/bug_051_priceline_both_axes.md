# БАГ #51: Метки price line на обеих осях одновременно

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#795](https://github.com/tradingview/lightweight-charts/issues/795)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2020

## 📋 Описание проблемы

### Суть проблемы

При использовании **двух ценовых осей** (левой и правой) **метки price line отображаются только на одной оси**, к которой привязана серия. Пользователи хотят видеть **метку уровня цены на обеих осях** одновременно для лучшей ориентации.

### Детали

1. **Симптомы:**
   - Price line label виден только на одной оси
   - При использовании двух осей нет симметрии
   - Метка привязана только к priceScaleId серии

2. **Ожидаемое поведение:**
   - Опция для отображения метки на обеих осях
   - Или автоматическое отображение на всех видимых осях

3. **Текущее поведение:**
   - Метка привязана к priceScaleId серии
   - Нет API для отображения на другой оси

### Визуализация проблемы

```
Ожидаемое:                        Реальное:
┌───────┬──────────────────┬───────┐  ┌───────┬──────────────────┬───────┐
│ 1.20  │                  │ 1.20  │  │       │                  │ 1.20  │
│       │                  │       │  │       │                  │       │
│ 1.15 ─│── Price Line ────│─ 1.15 │  │       │── Price Line ────│─ 1.15 │
│       │                  │       │  │       │                  │       │
│ 1.10  │                  │ 1.10  │  │       │                  │ 1.10  │
└───────┴──────────────────┴───────┘  └───────┴──────────────────┴───────┘
  LEFT         CHART          RIGHT      LEFT         CHART          RIGHT
```

### Сценарии использования

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  leftPriceScale: { visible: true },
  rightPriceScale: { visible: true },
});

const series = chart.addLineSeries({
  priceScaleId: 'right', // Привязка к правой оси
});

series.setData(data);

// Price line с меткой
const priceLine = series.createPriceLine({
  price: 1.15,
  title: 'Support',
  axisLabelVisible: true, // Метка только на правой оси!
});

// ❌ Нет способа показать метку на левой оси
```

## 🔍 Найденные решения

### Решение 1: Дополнительная серия на другой оси (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐ (4/5) - Работающий workaround

```typescript
import { createChart, IChartApi, ISeriesApi, LineStyle } from 'lightweight-charts';

interface DualAxisPriceLineOptions {
  price: number;
  title?: string;
  color?: string;
  lineWidth?: number;
  lineStyle?: LineStyle;
}

/**
 * Создаёт price line с метками на обеих осях
 */
function createDualAxisPriceLine(
  chart: IChartApi,
  mainSeries: ISeriesApi<any>,
  options: DualAxisPriceLineOptions
): { main: ReturnType<typeof mainSeries.createPriceLine>; mirror: ISeriesApi<'Line'> } {
  const { price, title = '', color = 'red', lineWidth = 1, lineStyle = LineStyle.Solid } = options;
  
  // Получаем ID основной оси
  const mainScaleId = mainSeries.options().priceScaleId || 'right';
  const mirrorScaleId = mainScaleId === 'right' ? 'left' : 'right';
  
  // Создаём price line на основной серии
  const mainPriceLine = mainSeries.createPriceLine({
    price,
    title,
    color,
    lineWidth,
    lineStyle,
    axisLabelVisible: true,
  });
  
  // Создаём невидимую серию на зеркальной оси для метки
  const data = mainSeries.data();
  const mirrorSeries = chart.addLineSeries({
    priceScaleId: mirrorScaleId,
    visible: false, // Линия не видна
    lastValueVisible: false,
    priceLineVisible: false,
    crosshairMarkerVisible: false,
  });
  
  // Минимальные данные для привязки к временной шкале
  if (data.length >= 2) {
    mirrorSeries.setData([
      { time: data[0].time, value: price },
      { time: data[data.length - 1].time, value: price },
    ]);
  }
  
  // Создаём price line на зеркальной серии (только для метки)
  mirrorSeries.createPriceLine({
    price,
    title,
    color,
    lineWidth: 0, // Линия не рисуется (уже есть на main)
    lineStyle: LineStyle.Solid,
    axisLabelVisible: true,
  });
  
  return { main: mainPriceLine, mirror: mirrorSeries };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container, {
  leftPriceScale: { visible: true },
  rightPriceScale: { visible: true },
});

const mainSeries = chart.addCandlestickSeries({
  priceScaleId: 'right',
});

mainSeries.setData(candleData);

// Создаём price line с метками на обеих осях
const dualPriceLine = createDualAxisPriceLine(chart, mainSeries, {
  price: 100,
  title: 'Support',
  color: '#2962FF',
});
```

**Плюсы:**
- Работает с текущим API
- Метки на обеих осях
- Полный контроль над стилем

**Минусы:**
- Дополнительная серия в памяти
- Нужно синхронизировать при обновлении

---

### Решение 2: Custom primitive с двойными метками

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Полный контроль

```typescript
import { 
  createChart, 
  IChartApi,
  ISeriesPrimitive,
  SeriesAttachedParameter,
  PriceToCoordinateConverter,
  Time,
} from 'lightweight-charts';

interface DualLabelPriceLineOptions {
  price: number;
  title?: string;
  color?: string;
  lineWidth?: number;
  leftLabelBgColor?: string;
  rightLabelBgColor?: string;
  textColor?: string;
}

class DualLabelPriceLine implements ISeriesPrimitive<Time> {
  private _options: Required<DualLabelPriceLineOptions>;
  private _chart: IChartApi | null = null;
  private _series: SeriesAttachedParameter<Time> | null = null;
  
  constructor(options: DualLabelPriceLineOptions) {
    this._options = {
      price: options.price,
      title: options.title ?? '',
      color: options.color ?? '#FF0000',
      lineWidth: options.lineWidth ?? 1,
      leftLabelBgColor: options.leftLabelBgColor ?? '#2962FF',
      rightLabelBgColor: options.rightLabelBgColor ?? '#2962FF',
      textColor: options.textColor ?? '#FFFFFF',
    };
  }
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._series = param;
    this._chart = param.chart;
  }
  
  detached(): void {
    this._series = null;
    this._chart = null;
  }
  
  updateAllViews(): void {
    this._series?.requestUpdate();
  }
  
  priceAxisViews() {
    const views = [];
    const { price, title, leftLabelBgColor, rightLabelBgColor, textColor } = this._options;
    const formattedPrice = price.toFixed(2);
    const text = title ? `${title}: ${formattedPrice}` : formattedPrice;
    
    // Метка для левой оси
    views.push({
      coordinate: () => this._series?.priceToCoordinate(price) ?? 0,
      text: () => text,
      textColor: () => textColor,
      backColor: () => leftLabelBgColor,
      visible: () => true,
    });
    
    return views;
  }
  
  paneViews() {
    return [{
      renderer: () => ({
        draw: (target: any) => {
          const ctx = target.context;
          const { price, color, lineWidth } = this._options;
          
          const y = this._series?.priceToCoordinate(price);
          if (y === null || y === undefined) return;
          
          const width = target.mediaSize.width;
          
          ctx.save();
          ctx.strokeStyle = color;
          ctx.lineWidth = lineWidth;
          ctx.setLineDash([5, 5]);
          
          ctx.beginPath();
          ctx.moveTo(0, y);
          ctx.lineTo(width, y);
          ctx.stroke();
          
          ctx.restore();
        },
      }),
    }];
  }
  
  setPrice(price: number): void {
    this._options.price = price;
    this.updateAllViews();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container, {
  leftPriceScale: { visible: true },
  rightPriceScale: { visible: true },
});

const series = chart.addLineSeries();
series.setData(data);

const dualPrimitive = new DualLabelPriceLine({
  price: 100,
  title: 'Support',
  color: '#2962FF',
});

series.attachPrimitive(dualPrimitive);
```

**Плюсы:**
- Полный контроль над рендерингом
- Единый объект для обеих меток
- Нет лишних серий

**Минусы:**
- Сложная реализация
- Требует знания Primitives API

---

### Решение 3: HTML overlay для меток

**Оценка:** ⭐⭐⭐ (3/5) - Простой workaround

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

interface OverlayLabel {
  element: HTMLElement;
  price: number;
  side: 'left' | 'right';
}

class DualAxisLabelManager {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _labels: OverlayLabel[] = [];
  private _series: ISeriesApi<any>;
  
  constructor(chart: IChartApi, container: HTMLElement, series: ISeriesApi<any>) {
    this._chart = chart;
    this._container = container;
    this._series = series;
    
    // Подписываемся на изменения для обновления позиций
    this._chart.subscribeCrosshairMove(() => this._updatePositions());
    this._chart.timeScale().subscribeVisibleLogicalRangeChange(() => this._updatePositions());
  }
  
  addPriceLine(price: number, title: string, color: string): void {
    // Создаём стандартный price line на серии
    this._series.createPriceLine({
      price,
      title,
      color,
      axisLabelVisible: true,
    });
    
    // Создаём HTML метку для противоположной оси
    const seriesScaleId = this._series.options().priceScaleId || 'right';
    const oppositeSide = seriesScaleId === 'right' ? 'left' : 'right';
    
    const label = this._createLabel(price, title, color, oppositeSide);
    this._labels.push(label);
    this._updatePositions();
  }
  
  private _createLabel(
    price: number, 
    title: string, 
    color: string, 
    side: 'left' | 'right'
  ): OverlayLabel {
    const element = document.createElement('div');
    element.style.cssText = `
      position: absolute;
      padding: 2px 6px;
      background: ${color};
      color: white;
      font-size: 11px;
      font-family: sans-serif;
      border-radius: 2px;
      white-space: nowrap;
      pointer-events: none;
      z-index: 10;
    `;
    element.textContent = `${title}: ${price.toFixed(2)}`;
    
    this._container.appendChild(element);
    
    return { element, price, side };
  }
  
  private _updatePositions(): void {
    for (const label of this._labels) {
      const y = this._series.priceToCoordinate(label.price);
      
      if (y === null) {
        label.element.style.display = 'none';
        continue;
      }
      
      label.element.style.display = 'block';
      label.element.style.top = `${y - 10}px`;
      
      if (label.side === 'left') {
        label.element.style.left = '0px';
        label.element.style.right = 'auto';
      } else {
        label.element.style.right = '0px';
        label.element.style.left = 'auto';
      }
    }
  }
  
  destroy(): void {
    for (const label of this._labels) {
      label.element.remove();
    }
    this._labels = [];
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.position = 'relative';

const chart = createChart(container, {
  leftPriceScale: { visible: true },
  rightPriceScale: { visible: true },
});

const series = chart.addLineSeries({ priceScaleId: 'right' });
series.setData(data);

const labelManager = new DualAxisLabelManager(chart, container, series);
labelManager.addPriceLine(100, 'Support', '#2962FF');
```

**Плюсы:**
- Простая реализация
- Полный контроль над стилем метки
- Не зависит от внутреннего API

**Минусы:**
- HTML элементы поверх canvas
- Нужна синхронизация позиций
- Может быть менее производительно

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 1** (дополнительная серия):

```typescript
// Минимальный пример
function addDualAxisPriceLine(
  chart: IChartApi,
  mainSeries: ISeriesApi<any>,
  price: number,
  title: string
): void {
  const mainScaleId = mainSeries.options().priceScaleId || 'right';
  const mirrorScaleId = mainScaleId === 'right' ? 'left' : 'right';
  
  // Price line на основной серии
  mainSeries.createPriceLine({ price, title, axisLabelVisible: true });
  
  // Невидимая серия на другой оси
  const data = mainSeries.data();
  const mirrorSeries = chart.addLineSeries({
    priceScaleId: mirrorScaleId,
    visible: false,
  });
  
  mirrorSeries.setData([
    { time: data[0].time, value: price },
    { time: data[data.length - 1].time, value: price },
  ]);
  
  mirrorSeries.createPriceLine({ price, title, lineWidth: 0, axisLabelVisible: true });
}
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Производительность | Гибкость | Рекомендация |
|---------|----------|-------------------|----------|--------------|
| **#1 Доп. серия** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Универсальное |
| **#2 Custom primitive** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Сложные кейсы |
| **#3 HTML overlay** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | Быстрый прототип |

## 🔗 Источники

- [GitHub Issue #795](https://github.com/tradingview/lightweight-charts/issues/795) - Price line labels on both axes
- [Price Lines Documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceLineOptions)
- [Custom Primitives](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/price-axis-view)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
