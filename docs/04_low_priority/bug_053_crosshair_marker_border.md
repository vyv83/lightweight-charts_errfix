# БАГ #53: Отсутствие границы у crosshair marker

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#677](https://github.com/tradingview/lightweight-charts/issues/677)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2021

## 📋 Описание проблемы

### Суть проблемы

Круглый маркер crosshair (точка, показывающая текущую позицию на серии) **не имеет опции для добавления границы (border/stroke)**. При использовании светлых цветов маркера на светлом фоне или тёмных на тёмном — маркер теряется визуально.

### Детали

1. **Симптомы:**
   - Маркер crosshair сливается с фоном
   - Нет контрастной границы
   - Плохая видимость при определённых цветовых схемах

2. **Ожидаемое поведение:**
   - Опция `crosshairMarkerBorderColor` и `crosshairMarkerBorderWidth`
   - Возможность добавить контрастную обводку

3. **Доступные опции:**
   - `crosshairMarkerRadius` — размер маркера ✅
   - `crosshairMarkerBackgroundColor` — цвет заливки ✅
   - `crosshairMarkerBorderColor` — ❌ НЕТ
   - `crosshairMarkerBorderWidth` — ❌ НЕТ

### Визуализация проблемы

```
Ожидаемое:                    Реальное:
┌─────────────────────┐       ┌─────────────────────┐
│        ●            │       │        ●            │
│       /│\           │       │       /│\           │
│      / │ \  ⬤       │       │      / │ \  ●       │ ← без границы
│     /  │  \ (border)│       │     /  │  \ (no border)
│    ────┼────        │       │    ────┼────        │
└─────────────────────┘       └─────────────────────┘
```

### Проблемные сценарии

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  layout: {
    background: { color: '#FFFFFF' }, // Белый фон
  },
});

const series = chart.addLineSeries({
  color: '#FFFFFF', // Белая линия
  crosshairMarkerBackgroundColor: '#FFFFFF', // Белый маркер
  crosshairMarkerRadius: 6,
  // ❌ Нет опции для border!
  // crosshairMarkerBorderColor: '#000000', // Не существует
  // crosshairMarkerBorderWidth: 2, // Не существует
});

// Результат: белый маркер на белом фоне — не видно!
```

## 🔍 Найденные решения

### Решение 1: Контрастный цвет маркера (⭐ Простейшее)

**Оценка:** ⭐⭐⭐ (3/5) - Базовый workaround

```typescript
import { createChart } from 'lightweight-charts';

/**
 * Определяет контрастный цвет для маркера
 */
function getContrastColor(backgroundColor: string): string {
  // Конвертируем hex в RGB
  const hex = backgroundColor.replace('#', '');
  const r = parseInt(hex.substr(0, 2), 16);
  const g = parseInt(hex.substr(2, 2), 16);
  const b = parseInt(hex.substr(4, 2), 16);
  
  // Вычисляем яркость
  const brightness = (r * 299 + g * 587 + b * 114) / 1000;
  
  // Возвращаем контрастный цвет
  return brightness > 128 ? '#000000' : '#FFFFFF';
}

/**
 * Создаёт серию с контрастным маркером
 */
function createSeriesWithContrastMarker(
  chart: IChartApi,
  lineColor: string,
  backgroundColor: string
): ISeriesApi<'Line'> {
  const markerColor = getContrastColor(backgroundColor);
  
  return chart.addLineSeries({
    color: lineColor,
    crosshairMarkerBackgroundColor: markerColor,
    crosshairMarkerRadius: 6,
    crosshairMarkerVisible: true,
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const bgColor = '#FFFFFF';

const chart = createChart(container, {
  layout: {
    background: { color: bgColor },
  },
});

const series = createSeriesWithContrastMarker(chart, '#2962FF', bgColor);
// Маркер будет чёрным на белом фоне
```

**Плюсы:**
- Простая реализация
- Не требует дополнительного кода

**Минусы:**
- Не настоящая граница
- Ограниченная кастомизация

---

### Решение 2: Custom primitive с crosshair marker (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Полный контроль

```typescript
import { 
  createChart, 
  ISeriesPrimitive,
  SeriesAttachedParameter,
  Time,
  MouseEventParams,
  CrosshairMode,
} from 'lightweight-charts';

interface BorderedCrosshairMarkerOptions {
  radius?: number;
  backgroundColor?: string;
  borderColor?: string;
  borderWidth?: number;
}

class BorderedCrosshairMarker implements ISeriesPrimitive<Time> {
  private _options: Required<BorderedCrosshairMarkerOptions>;
  private _series: SeriesAttachedParameter<Time> | null = null;
  private _currentPoint: { x: number; y: number } | null = null;
  
  constructor(options: BorderedCrosshairMarkerOptions = {}) {
    this._options = {
      radius: options.radius ?? 6,
      backgroundColor: options.backgroundColor ?? '#2962FF',
      borderColor: options.borderColor ?? '#FFFFFF',
      borderWidth: options.borderWidth ?? 2,
    };
  }
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._series = param;
    
    param.chart.subscribeCrosshairMove((event) => {
      this._handleCrosshairMove(event);
    });
  }
  
  detached(): void {
    this._series = null;
    this._currentPoint = null;
  }
  
  private _handleCrosshairMove(event: MouseEventParams): void {
    if (!this._series) return;
    
    if (!event.point || !event.time) {
      this._currentPoint = null;
      this._series.requestUpdate();
      return;
    }
    
    // Получаем значение серии в текущей точке
    const seriesData = event.seriesData.get(this._series.series);
    if (!seriesData) {
      this._currentPoint = null;
      this._series.requestUpdate();
      return;
    }
    
    const value = 'value' in seriesData ? seriesData.value : 
                  'close' in seriesData ? seriesData.close : null;
    
    if (value === null) {
      this._currentPoint = null;
      this._series.requestUpdate();
      return;
    }
    
    const y = this._series.priceToCoordinate(value);
    const x = this._series.chart.timeScale().timeToCoordinate(event.time);
    
    if (x === null || y === null) {
      this._currentPoint = null;
    } else {
      this._currentPoint = { x, y };
    }
    
    this._series.requestUpdate();
  }
  
  paneViews() {
    return [{
      renderer: () => ({
        draw: (target: any) => {
          if (!this._currentPoint) return;
          
          const ctx = target.context;
          const { x, y } = this._currentPoint;
          const { radius, backgroundColor, borderColor, borderWidth } = this._options;
          
          ctx.save();
          
          // Рисуем круг с границей
          ctx.beginPath();
          ctx.arc(x, y, radius, 0, 2 * Math.PI);
          
          // Заливка
          ctx.fillStyle = backgroundColor;
          ctx.fill();
          
          // Граница
          ctx.strokeStyle = borderColor;
          ctx.lineWidth = borderWidth;
          ctx.stroke();
          
          ctx.restore();
        },
      }),
    }];
  }
  
  applyOptions(options: Partial<BorderedCrosshairMarkerOptions>): void {
    Object.assign(this._options, options);
    this._series?.requestUpdate();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container, {
  layout: {
    background: { color: '#FFFFFF' },
  },
  crosshair: {
    mode: CrosshairMode.Normal,
  },
});

const series = chart.addLineSeries({
  color: '#2962FF',
  crosshairMarkerVisible: false, // Отключаем стандартный маркер
});

series.setData(data);

// Добавляем кастомный маркер с границей
const borderedMarker = new BorderedCrosshairMarker({
  radius: 6,
  backgroundColor: '#2962FF',
  borderColor: '#FFFFFF',
  borderWidth: 2,
});

series.attachPrimitive(borderedMarker);
```

**Плюсы:**
- Полный контроль над внешним видом
- Настраиваемая граница
- Работает с любыми цветами

**Минусы:**
- Сложнее реализовать
- Нужно отключать стандартный маркер

---

### Решение 3: SVG overlay маркер

**Оценка:** ⭐⭐⭐⭐ (4/5) - Альтернативный подход

```typescript
import { createChart, IChartApi, ISeriesApi, MouseEventParams } from 'lightweight-charts';

interface SVGMarkerOptions {
  radius: number;
  fillColor: string;
  strokeColor: string;
  strokeWidth: number;
}

class SVGCrosshairMarker {
  private _chart: IChartApi;
  private _series: ISeriesApi<any>;
  private _container: HTMLElement;
  private _svg: SVGSVGElement;
  private _circle: SVGCircleElement;
  private _options: SVGMarkerOptions;
  
  constructor(
    chart: IChartApi,
    series: ISeriesApi<any>,
    container: HTMLElement,
    options: Partial<SVGMarkerOptions> = {}
  ) {
    this._chart = chart;
    this._series = series;
    this._container = container;
    
    this._options = {
      radius: options.radius ?? 6,
      fillColor: options.fillColor ?? '#2962FF',
      strokeColor: options.strokeColor ?? '#FFFFFF',
      strokeWidth: options.strokeWidth ?? 2,
    };
    
    // Отключаем стандартный маркер
    this._series.applyOptions({
      crosshairMarkerVisible: false,
    });
    
    // Создаём SVG overlay
    this._svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
    this._svg.style.cssText = `
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
      z-index: 10;
    `;
    
    this._circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
    this._circle.setAttribute('r', String(this._options.radius));
    this._circle.setAttribute('fill', this._options.fillColor);
    this._circle.setAttribute('stroke', this._options.strokeColor);
    this._circle.setAttribute('stroke-width', String(this._options.strokeWidth));
    this._circle.style.display = 'none';
    
    this._svg.appendChild(this._circle);
    this._container.appendChild(this._svg);
    
    // Подписываемся на движение crosshair
    this._chart.subscribeCrosshairMove((param) => this._update(param));
  }
  
  private _update(param: MouseEventParams): void {
    if (!param.point || !param.time) {
      this._circle.style.display = 'none';
      return;
    }
    
    const seriesData = param.seriesData.get(this._series);
    if (!seriesData) {
      this._circle.style.display = 'none';
      return;
    }
    
    const value = 'value' in seriesData ? seriesData.value : 
                  'close' in seriesData ? seriesData.close : null;
    
    if (value === null) {
      this._circle.style.display = 'none';
      return;
    }
    
    const y = this._series.priceToCoordinate(value);
    const x = this._chart.timeScale().timeToCoordinate(param.time);
    
    if (x === null || y === null) {
      this._circle.style.display = 'none';
      return;
    }
    
    this._circle.setAttribute('cx', String(x));
    this._circle.setAttribute('cy', String(y));
    this._circle.style.display = 'block';
  }
  
  setOptions(options: Partial<SVGMarkerOptions>): void {
    Object.assign(this._options, options);
    
    this._circle.setAttribute('r', String(this._options.radius));
    this._circle.setAttribute('fill', this._options.fillColor);
    this._circle.setAttribute('stroke', this._options.strokeColor);
    this._circle.setAttribute('stroke-width', String(this._options.strokeWidth));
  }
  
  destroy(): void {
    this._svg.remove();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.position = 'relative';

const chart = createChart(container);
const series = chart.addLineSeries();
series.setData(data);

const svgMarker = new SVGCrosshairMarker(chart, series, container, {
  radius: 8,
  fillColor: '#2962FF',
  strokeColor: '#FFFFFF',
  strokeWidth: 3,
});

// Изменение стиля
svgMarker.setOptions({
  strokeColor: '#000000',
  strokeWidth: 2,
});
```

**Плюсы:**
- SVG качество (векторная графика)
- Простая стилизация
- CSS анимации возможны

**Минусы:**
- Дополнительный DOM элемент
- Возможные проблемы с синхронизацией

---

### Решение 4: Двойной маркер (имитация границы)

**Оценка:** ⭐⭐⭐ (3/5) - Простой хак

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

/**
 * Создаёт эффект границы через две серии
 * Одна с большим радиусом (граница), другая с меньшим (заливка)
 */
function createBorderedMarkerEffect(
  chart: IChartApi,
  data: { time: Time; value: number }[],
  options: {
    lineColor: string;
    markerColor: string;
    borderColor: string;
    markerRadius: number;
    borderWidth: number;
  }
): { mainSeries: ISeriesApi<'Line'>; borderSeries: ISeriesApi<'Line'> } {
  const { lineColor, markerColor, borderColor, markerRadius, borderWidth } = options;
  
  // Серия для "границы" (больший радиус)
  const borderSeries = chart.addLineSeries({
    color: 'transparent', // Линия невидима
    crosshairMarkerBackgroundColor: borderColor,
    crosshairMarkerRadius: markerRadius + borderWidth,
    crosshairMarkerVisible: true,
    lastValueVisible: false,
    priceLineVisible: false,
  });
  
  // Основная серия (меньший радиус поверх)
  const mainSeries = chart.addLineSeries({
    color: lineColor,
    crosshairMarkerBackgroundColor: markerColor,
    crosshairMarkerRadius: markerRadius,
    crosshairMarkerVisible: true,
  });
  
  // Загружаем одинаковые данные
  borderSeries.setData(data);
  mainSeries.setData(data);
  
  return { mainSeries, borderSeries };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);

const { mainSeries, borderSeries } = createBorderedMarkerEffect(chart, data, {
  lineColor: '#2962FF',
  markerColor: '#2962FF',
  borderColor: '#FFFFFF',
  markerRadius: 5,
  borderWidth: 2,
});

// При обновлении данных нужно обновлять обе серии
function updateData(newData: { time: Time; value: number }[]) {
  borderSeries.setData(newData);
  mainSeries.setData(newData);
}
```

**Плюсы:**
- Использует стандартный API
- Простая концепция

**Минусы:**
- Две серии в памяти
- Нужно синхронизировать данные
- Не идеальный круг (перекрытие)

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 2** (Custom primitive):

```typescript
// Минимальный пример с границей
series.applyOptions({ crosshairMarkerVisible: false });

const marker = new BorderedCrosshairMarker({
  radius: 6,
  backgroundColor: '#2962FF',
  borderColor: '#FFFFFF',
  borderWidth: 2,
});

series.attachPrimitive(marker);
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Качество | Производительность | Рекомендация |
|---------|----------|----------|-------------------|--------------|
| **#1 Контрастный цвет** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | Быстрый фикс |
| **#2 Custom primitive** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Рекомендуемое |
| **#3 SVG overlay** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Альтернатива |
| **#4 Двойной маркер** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | Workaround |

## 🔗 Источники

- [GitHub Issue #677](https://github.com/tradingview/lightweight-charts/issues/677) - Crosshair marker border
- [Series Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/LineStyleOptions#crosshairmarkerbackgroundcolor)
- [Custom Primitives Guide](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
