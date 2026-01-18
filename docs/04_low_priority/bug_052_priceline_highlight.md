# БАГ #52: Не подсвечивается цена на price line

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#273](https://github.com/tradingview/lightweight-charts/issues/273)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2019

## 📋 Описание проблемы

### Суть проблемы

При **наведении курсора на price line** (горизонтальную линию уровня цены) **отсутствует визуальная обратная связь** — линия не подсвечивается, не меняет толщину или цвет. Пользователи ожидают, что при hover над интерактивными элементами будет визуальный feedback.

### Детали

1. **Симптомы:**
   - Price line визуально не реагирует на hover
   - Нет изменения курсора при наведении
   - Нет визуального индикатора, что элемент интерактивен

2. **Ожидаемое поведение:**
   - Изменение толщины или цвета линии при hover
   - Изменение курсора (pointer)
   - Подсветка метки на оси

3. **Почему это важно:**
   - UX: пользователь не понимает, что линия кликабельна
   - Интерактивность: нужна обратная связь
   - Accessibility: визуальные подсказки

### Сценарии использования

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

// Создаём интерактивный price line
const priceLine = series.createPriceLine({
  price: 100,
  title: 'Support Level',
  color: '#2962FF',
  lineWidth: 2,
});

// При наведении:
// ❌ Нет визуального изменения
// ❌ Курсор остаётся default
// ❌ Пользователь не понимает, что можно кликнуть
```

### Визуализация проблемы

```
Ожидаемое при hover:              Реальное:
┌────────────────────────┐        ┌────────────────────────┐
│                        │        │                        │
│ ═══════════════════ ▸  │ hover  │ ─────────────────────  │ ← без изменений
│ (толще + pointer)      │        │                        │
│                        │        │                        │
└────────────────────────┘        └────────────────────────┘
```

## 🔍 Найденные решения

### Решение 1: Отслеживание hover через subscribeCrosshairMove (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐ (4/5) - Рабочее решение

```typescript
import { createChart, IChartApi, ISeriesApi, IPriceLine, LineStyle } from 'lightweight-charts';

interface HoverablePriceLine {
  priceLine: IPriceLine;
  price: number;
  normalWidth: number;
  hoverWidth: number;
  normalColor: string;
  hoverColor: string;
}

class PriceLineHoverManager {
  private _chart: IChartApi;
  private _series: ISeriesApi<any>;
  private _container: HTMLElement;
  private _lines: HoverablePriceLine[] = [];
  private _hoveredLine: HoverablePriceLine | null = null;
  private _hoverThreshold = 5; // пикселей
  
  constructor(chart: IChartApi, series: ISeriesApi<any>, container: HTMLElement) {
    this._chart = chart;
    this._series = series;
    this._container = container;
    
    this._setupHoverTracking();
  }
  
  addPriceLine(
    price: number,
    title: string,
    color: string,
    options?: {
      hoverColor?: string;
      normalWidth?: number;
      hoverWidth?: number;
    }
  ): IPriceLine {
    const normalWidth = options?.normalWidth ?? 1;
    const hoverWidth = options?.hoverWidth ?? 3;
    const hoverColor = options?.hoverColor ?? this._lightenColor(color, 30);
    
    const priceLine = this._series.createPriceLine({
      price,
      title,
      color,
      lineWidth: normalWidth,
      lineStyle: LineStyle.Solid,
      axisLabelVisible: true,
    });
    
    this._lines.push({
      priceLine,
      price,
      normalWidth,
      hoverWidth,
      normalColor: color,
      hoverColor,
    });
    
    return priceLine;
  }
  
  private _setupHoverTracking(): void {
    this._chart.subscribeCrosshairMove((param) => {
      if (!param.point) {
        this._unhoverAll();
        return;
      }
      
      const { y } = param.point;
      let foundHover = false;
      
      for (const line of this._lines) {
        const lineY = this._series.priceToCoordinate(line.price);
        
        if (lineY !== null && Math.abs(y - lineY) <= this._hoverThreshold) {
          if (this._hoveredLine !== line) {
            this._unhoverAll();
            this._setHovered(line);
          }
          foundHover = true;
          break;
        }
      }
      
      if (!foundHover && this._hoveredLine) {
        this._unhoverAll();
      }
    });
    
    // Также отслеживаем mousemove для курсора
    this._container.addEventListener('mousemove', (e) => {
      const rect = this._container.getBoundingClientRect();
      const y = e.clientY - rect.top;
      
      let isOverLine = false;
      
      for (const line of this._lines) {
        const lineY = this._series.priceToCoordinate(line.price);
        if (lineY !== null && Math.abs(y - lineY) <= this._hoverThreshold) {
          isOverLine = true;
          break;
        }
      }
      
      this._container.style.cursor = isOverLine ? 'pointer' : 'default';
    });
  }
  
  private _setHovered(line: HoverablePriceLine): void {
    this._hoveredLine = line;
    
    line.priceLine.applyOptions({
      lineWidth: line.hoverWidth,
      color: line.hoverColor,
    });
  }
  
  private _unhoverAll(): void {
    if (this._hoveredLine) {
      this._hoveredLine.priceLine.applyOptions({
        lineWidth: this._hoveredLine.normalWidth,
        color: this._hoveredLine.normalColor,
      });
      this._hoveredLine = null;
    }
    this._container.style.cursor = 'default';
  }
  
  private _lightenColor(hex: string, percent: number): string {
    const num = parseInt(hex.replace('#', ''), 16);
    const amt = Math.round(2.55 * percent);
    const R = Math.min(255, (num >> 16) + amt);
    const G = Math.min(255, ((num >> 8) & 0x00FF) + amt);
    const B = Math.min(255, (num & 0x0000FF) + amt);
    return `#${(0x1000000 + R * 0x10000 + G * 0x100 + B).toString(16).slice(1)}`;
  }
  
  removePriceLine(priceLine: IPriceLine): void {
    const index = this._lines.findIndex(l => l.priceLine === priceLine);
    if (index !== -1) {
      this._series.removePriceLine(priceLine);
      this._lines.splice(index, 1);
    }
  }
  
  destroy(): void {
    this._lines = [];
    this._hoveredLine = null;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const hoverManager = new PriceLineHoverManager(chart, series, container);

// Добавляем price line с hover эффектом
const supportLine = hoverManager.addPriceLine(95, 'Support', '#26A69A', {
  hoverWidth: 3,
});

const resistanceLine = hoverManager.addPriceLine(105, 'Resistance', '#EF5350', {
  hoverWidth: 3,
});
```

**Плюсы:**
- Визуальный feedback при hover
- Изменение курсора
- Гибкая настройка цветов и толщины

**Минусы:**
- Дополнительная логика отслеживания
- Небольшой overhead на crosshairMove

---

### Решение 2: Custom primitive с hover state

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Полный контроль

```typescript
import { 
  createChart, 
  ISeriesPrimitive,
  SeriesAttachedParameter,
  Time,
  MouseEventParams,
} from 'lightweight-charts';

interface HoverablePriceLineOptions {
  price: number;
  title?: string;
  color?: string;
  hoverColor?: string;
  lineWidth?: number;
  hoverLineWidth?: number;
}

class HoverablePriceLinePrimitive implements ISeriesPrimitive<Time> {
  private _options: Required<HoverablePriceLineOptions>;
  private _series: SeriesAttachedParameter<Time> | null = null;
  private _isHovered = false;
  private _hoverThreshold = 5;
  
  constructor(options: HoverablePriceLineOptions) {
    this._options = {
      price: options.price,
      title: options.title ?? '',
      color: options.color ?? '#2962FF',
      hoverColor: options.hoverColor ?? '#5B8DEF',
      lineWidth: options.lineWidth ?? 1,
      hoverLineWidth: options.hoverLineWidth ?? 3,
    };
  }
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._series = param;
    
    // Подписываемся на движение мыши
    param.chart.subscribeCrosshairMove((event) => {
      this._handleMouseMove(event);
    });
  }
  
  detached(): void {
    this._series = null;
  }
  
  private _handleMouseMove(event: MouseEventParams): void {
    if (!this._series || !event.point) {
      if (this._isHovered) {
        this._isHovered = false;
        this._series?.requestUpdate();
      }
      return;
    }
    
    const lineY = this._series.priceToCoordinate(this._options.price);
    if (lineY === null) return;
    
    const wasHovered = this._isHovered;
    this._isHovered = Math.abs(event.point.y - lineY) <= this._hoverThreshold;
    
    if (wasHovered !== this._isHovered) {
      this._series.requestUpdate();
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
          const { color, hoverColor, lineWidth, hoverLineWidth } = this._options;
          
          ctx.save();
          
          // Применяем hover стили
          ctx.strokeStyle = this._isHovered ? hoverColor : color;
          ctx.lineWidth = this._isHovered ? hoverLineWidth : lineWidth;
          
          // Рисуем линию
          ctx.beginPath();
          ctx.moveTo(0, y);
          ctx.lineTo(width, y);
          ctx.stroke();
          
          // Рисуем метку при hover
          if (this._isHovered && this._options.title) {
            const text = `${this._options.title}: ${this._options.price.toFixed(2)}`;
            ctx.font = '12px Arial';
            ctx.fillStyle = hoverColor;
            ctx.textAlign = 'left';
            ctx.fillText(text, 10, y - 5);
          }
          
          ctx.restore();
        },
      }),
    }];
  }
  
  priceAxisViews() {
    return [{
      coordinate: () => this._series?.priceToCoordinate(this._options.price) ?? 0,
      text: () => this._options.price.toFixed(2),
      textColor: () => '#FFFFFF',
      backColor: () => this._isHovered ? this._options.hoverColor : this._options.color,
      visible: () => true,
    }];
  }
  
  hitTest(x: number, y: number): { cursorStyle?: string } | null {
    if (!this._series) return null;
    
    const lineY = this._series.priceToCoordinate(this._options.price);
    if (lineY === null) return null;
    
    if (Math.abs(y - lineY) <= this._hoverThreshold) {
      return { cursorStyle: 'pointer' };
    }
    
    return null;
  }
  
  setPrice(price: number): void {
    this._options.price = price;
    this._series?.requestUpdate();
  }
  
  isHovered(): boolean {
    return this._isHovered;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const hoverableLine = new HoverablePriceLinePrimitive({
  price: 100,
  title: 'Support',
  color: '#26A69A',
  hoverColor: '#4DB6AC',
  lineWidth: 1,
  hoverLineWidth: 3,
});

series.attachPrimitive(hoverableLine);
```

**Плюсы:**
- Полный контроль над рендерингом
- hitTest для курсора
- Единый объект для всей логики

**Минусы:**
- Сложнее реализовать
- Требует знания Primitives API

---

### Решение 3: CSS transitions с HTML overlay

**Оценка:** ⭐⭐⭐ (3/5) - Быстрая реализация

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

class CSSHoverPriceLine {
  private _chart: IChartApi;
  private _series: ISeriesApi<any>;
  private _container: HTMLElement;
  private _lineElement: HTMLElement;
  private _price: number;
  private _isHovered = false;
  
  constructor(
    chart: IChartApi,
    series: ISeriesApi<any>,
    container: HTMLElement,
    price: number,
    title: string,
    color: string
  ) {
    this._chart = chart;
    this._series = series;
    this._container = container;
    this._price = price;
    
    // Создаём HTML элемент линии
    this._lineElement = document.createElement('div');
    this._lineElement.style.cssText = `
      position: absolute;
      left: 0;
      right: 60px;
      height: 2px;
      background: ${color};
      cursor: pointer;
      transition: height 0.15s ease, box-shadow 0.15s ease;
      z-index: 5;
    `;
    
    // Добавляем метку
    const label = document.createElement('span');
    label.style.cssText = `
      position: absolute;
      right: -60px;
      top: 50%;
      transform: translateY(-50%);
      background: ${color};
      color: white;
      padding: 2px 6px;
      font-size: 11px;
      border-radius: 2px;
      transition: transform 0.15s ease;
    `;
    label.textContent = `${title}: ${price.toFixed(2)}`;
    this._lineElement.appendChild(label);
    
    // Hover эффекты через CSS
    this._lineElement.addEventListener('mouseenter', () => {
      this._lineElement.style.height = '4px';
      this._lineElement.style.boxShadow = `0 0 8px ${color}`;
      label.style.transform = 'translateY(-50%) scale(1.1)';
      this._isHovered = true;
    });
    
    this._lineElement.addEventListener('mouseleave', () => {
      this._lineElement.style.height = '2px';
      this._lineElement.style.boxShadow = 'none';
      label.style.transform = 'translateY(-50%) scale(1)';
      this._isHovered = false;
    });
    
    this._container.appendChild(this._lineElement);
    
    // Обновляем позицию
    this._updatePosition();
    this._chart.timeScale().subscribeVisibleLogicalRangeChange(() => this._updatePosition());
    this._chart.subscribeCrosshairMove(() => this._updatePosition());
  }
  
  private _updatePosition(): void {
    const y = this._series.priceToCoordinate(this._price);
    
    if (y === null) {
      this._lineElement.style.display = 'none';
    } else {
      this._lineElement.style.display = 'block';
      this._lineElement.style.top = `${y - 1}px`;
    }
  }
  
  onClick(callback: () => void): void {
    this._lineElement.addEventListener('click', callback);
  }
  
  remove(): void {
    this._lineElement.remove();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.position = 'relative';

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const hoverLine = new CSSHoverPriceLine(
  chart,
  series,
  container,
  100,
  'Support',
  '#26A69A'
);

hoverLine.onClick(() => {
  console.log('Price line clicked!');
});
```

**Плюсы:**
- CSS transitions для плавности
- Простая реализация
- Легко стилизовать

**Минусы:**
- HTML overlay поверх canvas
- Нужна синхронизация позиции

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 1** с PriceLineHoverManager:

```typescript
// Минимальный рабочий пример
chart.subscribeCrosshairMove((param) => {
  if (!param.point) return;
  
  const lineY = series.priceToCoordinate(priceLineValue);
  const isHovered = lineY !== null && Math.abs(param.point.y - lineY) < 5;
  
  priceLine.applyOptions({
    lineWidth: isHovered ? 3 : 1,
    color: isHovered ? '#5B8DEF' : '#2962FF',
  });
  
  container.style.cursor = isHovered ? 'pointer' : 'default';
});
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Производительность | Плавность | Рекомендация |
|---------|----------|-------------------|-----------|--------------|
| **#1 CrosshairMove** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ Универсальное |
| **#2 Custom primitive** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Сложные кейсы |
| **#3 CSS overlay** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Быстрый прототип |

## 🔗 Источники

- [GitHub Issue #273](https://github.com/tradingview/lightweight-charts/issues/273) - Price line hover highlight
- [Price Lines API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IPriceLine)
- [Crosshair Move Event](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#subscribecrosshairmove)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
