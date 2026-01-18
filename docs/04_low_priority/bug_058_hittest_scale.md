# БАГ #58: hitTest не работает на price/time scale

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#1941](https://github.com/tradingview/lightweight-charts/issues/1941)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open (Feature Request)  
> **Последнее обновление:** Сентябрь 2025

## 📋 Описание проблемы

### Суть проблемы

Метод **`hitTest`** для определения объекта под курсором **не работает для элементов на ценовой (price scale) и временной (time scale) осях**. Невозможно программно определить, что пользователь кликнул или навёл курсор на метку оси, price line label или другой элемент шкал.

### Детали

1. **Симптомы:**
   - `hitTest()` возвращает `null` при клике на оси
   - Нет возможности определить клик по метке price line на оси
   - Отсутствует интерактивность для элементов шкал

2. **Ожидаемое поведение:**
   - `hitTest()` должен возвращать информацию о элементах на осях
   - Возможность определить клик по метке цены/времени
   - Hit detection для price line labels

3. **Текущий статус:**
   - Классифицировано как Feature Request
   - Пока не реализовано

### Сценарий использования

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

// Добавляем price line
const priceLine = series.createPriceLine({
  price: 100,
  title: 'Support',
  axisLabelVisible: true,
});

// Пытаемся определить клик по price line label
container.addEventListener('click', (event) => {
  const rect = container.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const y = event.clientY - rect.top;
  
  // hitTest для основной области
  const hit = chart.hitTest(x, y);
  
  // ❌ hit будет null если кликнули на price scale область
  // ❌ Нет информации о price line label
  // ❌ Нет информации о метках времени
  
  console.log(hit); // null для кликов на осях
});
```

## 🔍 Найденные решения

### Решение 1: Ручное определение области клика (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐ (4/5) - Работающий workaround

```typescript
import { createChart, IChartApi, ISeriesApi, IPriceLine } from 'lightweight-charts';

interface ClickZone {
  type: 'chart' | 'priceScale' | 'timeScale';
  scaleId?: 'left' | 'right';
}

interface PriceLineHitInfo {
  priceLine: IPriceLine;
  price: number;
  title: string;
}

class ScaleHitDetector {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _series: ISeriesApi<any>;
  private _priceLines: Map<IPriceLine, { price: number; title: string }> = new Map();
  private _hitThreshold = 10; // пикселей
  
  constructor(chart: IChartApi, container: HTMLElement, series: ISeriesApi<any>) {
    this._chart = chart;
    this._container = container;
    this._series = series;
  }
  
  /**
   * Регистрирует price line для hit detection
   */
  registerPriceLine(priceLine: IPriceLine, price: number, title: string): void {
    this._priceLines.set(priceLine, { price, title });
  }
  
  /**
   * Определяет зону клика
   */
  getClickZone(clientX: number, clientY: number): ClickZone {
    const rect = this._container.getBoundingClientRect();
    const x = clientX - rect.left;
    const y = clientY - rect.top;
    
    const chartWidth = this._container.clientWidth;
    const chartHeight = this._container.clientHeight;
    
    // Примерные размеры осей (можно уточнить)
    const priceScaleWidth = 60;
    const timeScaleHeight = 30;
    
    // Проверяем time scale (нижняя область)
    if (y > chartHeight - timeScaleHeight) {
      return { type: 'timeScale' };
    }
    
    // Проверяем левую price scale
    if (x < priceScaleWidth) {
      return { type: 'priceScale', scaleId: 'left' };
    }
    
    // Проверяем правую price scale
    if (x > chartWidth - priceScaleWidth) {
      return { type: 'priceScale', scaleId: 'right' };
    }
    
    return { type: 'chart' };
  }
  
  /**
   * Определяет price line под курсором
   */
  getPriceLineAtPoint(clientX: number, clientY: number): PriceLineHitInfo | null {
    const rect = this._container.getBoundingClientRect();
    const y = clientY - rect.top;
    
    for (const [priceLine, info] of this._priceLines) {
      const lineY = this._series.priceToCoordinate(info.price);
      
      if (lineY !== null && Math.abs(y - lineY) <= this._hitThreshold) {
        return {
          priceLine,
          price: info.price,
          title: info.title,
        };
      }
    }
    
    return null;
  }
  
  /**
   * Полный hitTest включая оси
   */
  hitTest(clientX: number, clientY: number): {
    zone: ClickZone;
    priceLine: PriceLineHitInfo | null;
    chartHit: any;
  } {
    const rect = this._container.getBoundingClientRect();
    const x = clientX - rect.left;
    const y = clientY - rect.top;
    
    const zone = this.getClickZone(clientX, clientY);
    const priceLine = this.getPriceLineAtPoint(clientX, clientY);
    const chartHit = this._chart.hitTest(x, y);
    
    return { zone, priceLine, chartHit };
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const hitDetector = new ScaleHitDetector(chart, container, series);

// Регистрируем price lines
const supportLine = series.createPriceLine({
  price: 95,
  title: 'Support',
  axisLabelVisible: true,
});
hitDetector.registerPriceLine(supportLine, 95, 'Support');

const resistanceLine = series.createPriceLine({
  price: 105,
  title: 'Resistance',
  axisLabelVisible: true,
});
hitDetector.registerPriceLine(resistanceLine, 105, 'Resistance');

// Обработка кликов
container.addEventListener('click', (event) => {
  const result = hitDetector.hitTest(event.clientX, event.clientY);
  
  console.log('Click zone:', result.zone);
  console.log('Price line:', result.priceLine);
  console.log('Chart hit:', result.chartHit);
  
  if (result.priceLine) {
    console.log(`Clicked on price line: ${result.priceLine.title} at ${result.priceLine.price}`);
  }
});
```

**Плюсы:**
- Работает с текущим API
- Обнаружение price lines
- Определение зон

**Минусы:**
- Приблизительные размеры осей
- Нужно вручную регистрировать price lines

---

### Решение 2: Overlay с кликабельными элементами

**Оценка:** ⭐⭐⭐⭐ (4/5) - HTML-based solution

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

interface ClickableLabel {
  element: HTMLElement;
  price: number;
  onClick: () => void;
}

class ClickablePriceScaleLabels {
  private _chart: IChartApi;
  private _series: ISeriesApi<any>;
  private _container: HTMLElement;
  private _labelsContainer: HTMLElement;
  private _labels: ClickableLabel[] = [];
  
  constructor(chart: IChartApi, series: ISeriesApi<any>, container: HTMLElement) {
    this._chart = chart;
    this._series = series;
    this._container = container;
    
    // Создаём overlay контейнер для меток
    this._labelsContainer = document.createElement('div');
    this._labelsContainer.style.cssText = `
      position: absolute;
      top: 0;
      right: 0;
      width: 60px;
      height: 100%;
      pointer-events: none;
    `;
    
    this._container.appendChild(this._labelsContainer);
    
    // Подписываемся на изменения для обновления позиций
    this._chart.subscribeCrosshairMove(() => this._updatePositions());
    this._chart.timeScale().subscribeVisibleLogicalRangeChange(() => this._updatePositions());
  }
  
  /**
   * Добавляет кликабельную метку на price scale
   */
  addClickableLabel(
    price: number,
    text: string,
    color: string,
    onClick: () => void
  ): void {
    const element = document.createElement('div');
    element.style.cssText = `
      position: absolute;
      right: 0;
      padding: 2px 6px;
      background: ${color};
      color: white;
      font-size: 11px;
      font-family: sans-serif;
      border-radius: 2px;
      cursor: pointer;
      pointer-events: auto;
      transform: translateY(-50%);
      transition: transform 0.1s ease;
    `;
    element.textContent = text;
    
    // Hover эффект
    element.addEventListener('mouseenter', () => {
      element.style.transform = 'translateY(-50%) scale(1.1)';
    });
    element.addEventListener('mouseleave', () => {
      element.style.transform = 'translateY(-50%)';
    });
    
    // Click handler
    element.addEventListener('click', (e) => {
      e.stopPropagation();
      onClick();
    });
    
    this._labelsContainer.appendChild(element);
    this._labels.push({ element, price, onClick });
    
    this._updatePositions();
  }
  
  private _updatePositions(): void {
    for (const label of this._labels) {
      const y = this._series.priceToCoordinate(label.price);
      
      if (y === null) {
        label.element.style.display = 'none';
      } else {
        label.element.style.display = 'block';
        label.element.style.top = `${y}px`;
      }
    }
  }
  
  removeLabel(price: number): void {
    const index = this._labels.findIndex(l => l.price === price);
    if (index !== -1) {
      this._labels[index].element.remove();
      this._labels.splice(index, 1);
    }
  }
  
  destroy(): void {
    this._labelsContainer.remove();
    this._labels = [];
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.position = 'relative';

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const clickableLabels = new ClickablePriceScaleLabels(chart, series, container);

// Добавляем кликабельные метки
clickableLabels.addClickableLabel(95, 'SL: 95', '#EF5350', () => {
  console.log('Stop Loss clicked!');
  alert('Stop Loss level clicked');
});

clickableLabels.addClickableLabel(105, 'TP: 105', '#26A69A', () => {
  console.log('Take Profit clicked!');
  alert('Take Profit level clicked');
});
```

**Плюсы:**
- Настоящие клики
- Hover эффекты
- Полный контроль

**Минусы:**
- HTML overlay
- Нужна синхронизация позиций

---

### Решение 3: Custom primitive с hitTest

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Нативное решение

```typescript
import { 
  createChart, 
  ISeriesPrimitive,
  SeriesAttachedParameter,
  Time,
  PrimitiveHoveredItem,
} from 'lightweight-charts';

interface InteractivePriceLinePrimitiveOptions {
  price: number;
  title: string;
  color?: string;
  onLabelClick?: (price: number) => void;
  onLabelHover?: (price: number, isHovered: boolean) => void;
}

class InteractivePriceLinePrimitive implements ISeriesPrimitive<Time> {
  private _options: Required<InteractivePriceLinePrimitiveOptions>;
  private _series: SeriesAttachedParameter<Time> | null = null;
  private _isHovered = false;
  private _labelBounds: { x: number; y: number; width: number; height: number } | null = null;
  
  constructor(options: InteractivePriceLinePrimitiveOptions) {
    this._options = {
      price: options.price,
      title: options.title,
      color: options.color ?? '#2962FF',
      onLabelClick: options.onLabelClick ?? (() => {}),
      onLabelHover: options.onLabelHover ?? (() => {}),
    };
  }
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._series = param;
  }
  
  detached(): void {
    this._series = null;
  }
  
  /**
   * Hit test для определения наведения на метку
   */
  hitTest(x: number, y: number): PrimitiveHoveredItem | null {
    if (!this._labelBounds) return null;
    
    const { x: lx, y: ly, width, height } = this._labelBounds;
    
    // Проверяем попадание в область метки
    if (x >= lx && x <= lx + width && y >= ly && y <= ly + height) {
      const wasHovered = this._isHovered;
      this._isHovered = true;
      
      if (!wasHovered) {
        this._options.onLabelHover(this._options.price, true);
        this._series?.requestUpdate();
      }
      
      return {
        cursorStyle: 'pointer',
        externalId: `priceline-${this._options.price}`,
        zOrder: 'top',
      };
    }
    
    if (this._isHovered) {
      this._isHovered = false;
      this._options.onLabelHover(this._options.price, false);
      this._series?.requestUpdate();
    }
    
    return null;
  }
  
  /**
   * Обработка клика
   */
  onClick?(x: number, y: number): void {
    if (this._labelBounds) {
      const { x: lx, y: ly, width, height } = this._labelBounds;
      
      if (x >= lx && x <= lx + width && y >= ly && y <= ly + height) {
        this._options.onLabelClick(this._options.price);
      }
    }
  }
  
  paneViews() {
    return [{
      renderer: () => ({
        draw: (target: any) => {
          if (!this._series) return;
          
          const ctx = target.context;
          const y = this._series.priceToCoordinate(this._options.price);
          
          if (y === null) return;
          
          const width = target.mediaSize.width;
          const { color, title } = this._options;
          
          ctx.save();
          
          // Рисуем линию
          ctx.strokeStyle = color;
          ctx.lineWidth = this._isHovered ? 2 : 1;
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
  
  priceAxisViews() {
    return [{
      coordinate: () => {
        const y = this._series?.priceToCoordinate(this._options.price) ?? 0;
        
        // Сохраняем bounds для hit test
        // Примерные размеры метки
        this._labelBounds = {
          x: 0, // Относительно price scale
          y: y - 10,
          width: 60,
          height: 20,
        };
        
        return y;
      },
      text: () => `${this._options.title}: ${this._options.price.toFixed(2)}`,
      textColor: () => '#FFFFFF',
      backColor: () => this._isHovered ? this._lightenColor(this._options.color) : this._options.color,
      visible: () => true,
    }];
  }
  
  private _lightenColor(hex: string): string {
    const num = parseInt(hex.replace('#', ''), 16);
    const r = Math.min(255, (num >> 16) + 30);
    const g = Math.min(255, ((num >> 8) & 0xFF) + 30);
    const b = Math.min(255, (num & 0xFF) + 30);
    return `#${(r << 16 | g << 8 | b).toString(16).padStart(6, '0')}`;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const interactivePriceLine = new InteractivePriceLinePrimitive({
  price: 100,
  title: 'Support',
  color: '#26A69A',
  onLabelClick: (price) => {
    console.log(`Price line clicked at ${price}`);
    alert(`You clicked the Support level at ${price}`);
  },
  onLabelHover: (price, isHovered) => {
    console.log(`Price line ${isHovered ? 'hovered' : 'unhovered'} at ${price}`);
  },
});

series.attachPrimitive(interactivePriceLine);
```

**Плюсы:**
- Использует нативный hitTest
- Интегрировано с API
- Hover и click события

**Минусы:**
- Сложная реализация
- hitTest для priceAxisViews может не работать идеально

---

### Решение 4: Event delegation по координатам

**Оценка:** ⭐⭐⭐ (3/5) - Простой workaround

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

interface ScaleClickHandler {
  onPriceScaleClick?: (price: number, scaleId: 'left' | 'right') => void;
  onTimeScaleClick?: (time: number) => void;
}

function setupScaleClickHandlers(
  chart: IChartApi,
  series: ISeriesApi<any>,
  container: HTMLElement,
  handlers: ScaleClickHandler
): void {
  container.addEventListener('click', (event) => {
    const rect = container.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;
    
    const chartWidth = container.clientWidth;
    const chartHeight = container.clientHeight;
    
    const priceScaleWidth = 60;
    const timeScaleHeight = 30;
    
    // Time scale click
    if (y > chartHeight - timeScaleHeight && handlers.onTimeScaleClick) {
      const time = chart.timeScale().coordinateToTime(x);
      if (time !== null) {
        handlers.onTimeScaleClick(time as number);
      }
      return;
    }
    
    // Left price scale click
    if (x < priceScaleWidth && handlers.onPriceScaleClick) {
      const price = series.coordinateToPrice(y);
      if (price !== null) {
        handlers.onPriceScaleClick(price, 'left');
      }
      return;
    }
    
    // Right price scale click
    if (x > chartWidth - priceScaleWidth && handlers.onPriceScaleClick) {
      const price = series.coordinateToPrice(y);
      if (price !== null) {
        handlers.onPriceScaleClick(price, 'right');
      }
      return;
    }
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

setupScaleClickHandlers(chart, series, container, {
  onPriceScaleClick: (price, scaleId) => {
    console.log(`Price scale (${scaleId}) clicked at price: ${price}`);
  },
  onTimeScaleClick: (time) => {
    console.log(`Time scale clicked at: ${new Date(time * 1000)}`);
  },
});
```

**Плюсы:**
- Простая реализация
- Обрабатывает клики по осям

**Минусы:**
- Не определяет конкретные элементы
- Приблизительные координаты

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 1** (ScaleHitDetector) для hit detection и **Решение 2** (ClickablePriceScaleLabels) для кликабельных меток:

```typescript
// Для hit detection
const hitDetector = new ScaleHitDetector(chart, container, series);
hitDetector.registerPriceLine(priceLine, price, title);

// Для кликабельных меток
const labels = new ClickablePriceScaleLabels(chart, series, container);
labels.addClickableLabel(100, 'Support', '#26A69A', () => {
  console.log('Support clicked!');
});
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Точность | Интерактивность | Рекомендация |
|---------|----------|----------|-----------------|--------------|
| **#1 ScaleHitDetector** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Hit detection |
| **#2 Clickable Labels** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ Клики |
| **#3 Custom primitive** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Продвинутый |
| **#4 Event delegation** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | Простой |

## 🔗 Источники

- [GitHub Issue #1941](https://github.com/tradingview/lightweight-charts/issues/1941) - hitTest for scales
- [hitTest API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#hittest)
- [Custom Primitives](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
