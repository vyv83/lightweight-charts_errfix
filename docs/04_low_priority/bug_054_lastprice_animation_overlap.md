# БАГ #54: LastPriceAnimation перекрывает price line labels

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#1711](https://github.com/tradingview/lightweight-charts/issues/1711)  
> **Версии:** v4.x+, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** Июнь 2024

## 📋 Описание проблемы

### Суть проблемы

При включённой **анимации последней цены** (`lastPriceAnimation`) **круглый анимированный индикатор может перекрывать метки price lines** на ценовой оси. Это создаёт визуальный артефакт, когда пульсирующий круг закрывает важную информацию.

### Детали

1. **Симптомы:**
   - Анимированный круг последней цены перекрывает метки price lines
   - Метки частично или полностью скрыты
   - Z-index конфликт между элементами

2. **Когда возникает:**
   - `lastPriceAnimation` включена (не `Disabled`)
   - Price line находится близко к текущей цене
   - Метка price line на той же оси

3. **Ожидаемое поведение:**
   - Метки price lines должны быть видны поверх анимации
   - Или анимация не должна затрагивать область меток

### Визуализация проблемы

```
Ожидаемое:                      Реальное:
┌────────────────────┬──────┐   ┌────────────────────┬──────┐
│                    │ 105  │   │                    │ 105  │
│                    │      │   │                    │      │
│ ───────────────────│ SL ◉ │   │ ───────────────────│●●●●●●│ ← анимация
│                    │      │   │                    │(SL?) │   перекрывает
│              ~~~◉~~│ 100  │   │              ~~~◉~~│ 100  │
│           (pulse)  │      │   │           (pulse)  │      │
└────────────────────┴──────┘   └────────────────────┴──────┘
```

### Сценарий воспроизведения

```typescript
import { createChart, LastPriceAnimationMode } from 'lightweight-charts';

const chart = createChart(container, {
  rightPriceScale: {
    visible: true,
  },
});

const series = chart.addCandlestickSeries({
  lastPriceAnimation: LastPriceAnimationMode.Continuous, // Включена анимация
});

series.setData([
  { time: '2024-01-01', open: 100, high: 105, low: 98, close: 102 },
  { time: '2024-01-02', open: 102, high: 106, low: 100, close: 103 }, // Последняя цена 103
]);

// Price line близко к текущей цене
series.createPriceLine({
  price: 104, // Близко к close: 103
  title: 'Stop Loss',
  axisLabelVisible: true,
  color: '#EF5350',
});

// ❌ Анимация последней цены (103) может перекрывать метку "Stop Loss" (104)
```

## 🔍 Найденные решения

### Решение 1: Отключение анимации (⭐ Простейшее)

**Оценка:** ⭐⭐⭐ (3/5) - Устраняет проблему, но теряется фича

```typescript
import { createChart, LastPriceAnimationMode } from 'lightweight-charts';

const chart = createChart(container);

const series = chart.addCandlestickSeries({
  lastPriceAnimation: LastPriceAnimationMode.Disabled, // Отключаем
});

// Теперь price line метки не перекрываются
```

**Плюсы:**
- Мгновенно решает проблему
- Нет дополнительного кода

**Минусы:**
- Теряется визуальный эффект анимации

---

### Решение 2: Custom price line с CSS z-index (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐ (4/5) - Обходит проблему с сохранением анимации

```typescript
import { createChart, IChartApi, ISeriesApi, LastPriceAnimationMode } from 'lightweight-charts';

interface HighZIndexPriceLineOptions {
  price: number;
  title: string;
  color?: string;
}

class HighZIndexPriceLine {
  private _chart: IChartApi;
  private _series: ISeriesApi<any>;
  private _container: HTMLElement;
  private _labelElement: HTMLElement;
  private _price: number;
  
  constructor(
    chart: IChartApi,
    series: ISeriesApi<any>,
    container: HTMLElement,
    options: HighZIndexPriceLineOptions
  ) {
    this._chart = chart;
    this._series = series;
    this._container = container;
    this._price = options.price;
    
    // Создаём стандартный price line (линия будет видна)
    this._series.createPriceLine({
      price: options.price,
      title: '', // Без стандартной метки
      color: options.color ?? '#EF5350',
      axisLabelVisible: false, // Отключаем стандартную метку
    });
    
    // Создаём HTML метку с высоким z-index
    this._labelElement = document.createElement('div');
    this._labelElement.style.cssText = `
      position: absolute;
      right: 0;
      padding: 2px 8px;
      background: ${options.color ?? '#EF5350'};
      color: white;
      font-size: 11px;
      font-family: -apple-system, BlinkMacSystemFont, sans-serif;
      border-radius: 2px;
      z-index: 100; /* Выше анимации */
      pointer-events: none;
      transform: translateY(-50%);
    `;
    this._labelElement.textContent = `${options.title}: ${options.price.toFixed(2)}`;
    
    this._container.appendChild(this._labelElement);
    
    // Обновляем позицию
    this._updatePosition();
    
    // Подписываемся на изменения
    this._chart.timeScale().subscribeVisibleLogicalRangeChange(() => this._updatePosition());
    this._chart.subscribeCrosshairMove(() => this._updatePosition());
  }
  
  private _updatePosition(): void {
    const y = this._series.priceToCoordinate(this._price);
    
    if (y === null) {
      this._labelElement.style.display = 'none';
    } else {
      this._labelElement.style.display = 'block';
      this._labelElement.style.top = `${y}px`;
    }
  }
  
  updatePrice(price: number): void {
    this._price = price;
    this._updatePosition();
  }
  
  remove(): void {
    this._labelElement.remove();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.position = 'relative';

const chart = createChart(container);

const series = chart.addCandlestickSeries({
  lastPriceAnimation: LastPriceAnimationMode.Continuous, // Анимация включена
});

series.setData(candleData);

// Создаём price line с высоким z-index
const stopLoss = new HighZIndexPriceLine(chart, series, container, {
  price: 104,
  title: 'SL',
  color: '#EF5350',
});

// Метка теперь всегда поверх анимации
```

**Плюсы:**
- Анимация сохраняется
- Метка всегда видна
- Полный контроль над стилем

**Минусы:**
- HTML overlay
- Нужна синхронизация позиции

---

### Решение 3: Custom primitive для price line label

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Чистое решение через API

```typescript
import { 
  createChart, 
  ISeriesPrimitive,
  SeriesAttachedParameter,
  Time,
  LastPriceAnimationMode,
} from 'lightweight-charts';

interface PriceLineLabelOptions {
  price: number;
  title: string;
  color?: string;
  textColor?: string;
}

class PriceLineLabelPrimitive implements ISeriesPrimitive<Time> {
  private _options: Required<PriceLineLabelOptions>;
  private _series: SeriesAttachedParameter<Time> | null = null;
  
  constructor(options: PriceLineLabelOptions) {
    this._options = {
      price: options.price,
      title: options.title,
      color: options.color ?? '#EF5350',
      textColor: options.textColor ?? '#FFFFFF',
    };
  }
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._series = param;
  }
  
  detached(): void {
    this._series = null;
  }
  
  // Метка на ценовой оси (рисуется после анимации)
  priceAxisViews() {
    const { price, title, color, textColor } = this._options;
    
    return [{
      coordinate: () => this._series?.priceToCoordinate(price) ?? 0,
      text: () => `${title}: ${price.toFixed(2)}`,
      textColor: () => textColor,
      backColor: () => color,
      visible: () => true,
    }];
  }
  
  // Горизонтальная линия
  paneViews() {
    return [{
      renderer: () => ({
        draw: (target: any) => {
          if (!this._series) return;
          
          const ctx = target.context;
          const y = this._series.priceToCoordinate(this._options.price);
          
          if (y === null) return;
          
          const width = target.mediaSize.width;
          
          ctx.save();
          ctx.strokeStyle = this._options.color;
          ctx.lineWidth = 1;
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
    this._series?.requestUpdate();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);

const series = chart.addCandlestickSeries({
  lastPriceAnimation: LastPriceAnimationMode.Continuous,
});

series.setData(candleData);

// Добавляем primitive вместо стандартного price line
const stopLossPrimitive = new PriceLineLabelPrimitive({
  price: 104,
  title: 'SL',
  color: '#EF5350',
});

series.attachPrimitive(stopLossPrimitive);
```

**Плюсы:**
- Использует нативный API
- Правильный порядок отрисовки
- Нет HTML overlays

**Минусы:**
- Требует знания Primitives API
- Больше кода

---

### Решение 4: Смещение метки price line

**Оценка:** ⭐⭐⭐ (3/5) - Простой визуальный хак

```typescript
import { createChart, IChartApi, ISeriesApi, LastPriceAnimationMode } from 'lightweight-charts';

/**
 * Создаёт price line со смещённой меткой (не перекрывается анимацией)
 */
function createOffsetPriceLine(
  series: ISeriesApi<any>,
  price: number,
  title: string,
  color: string,
  priceOffset: number = 0.5 // Небольшое смещение цены для метки
): void {
  // Линия на правильной цене (без метки)
  series.createPriceLine({
    price,
    color,
    lineWidth: 1,
    axisLabelVisible: false,
  });
  
  // Метка чуть выше/ниже (не перекрывается анимацией)
  series.createPriceLine({
    price: price + priceOffset, // Смещаем
    title,
    color,
    lineWidth: 0, // Без линии
    axisLabelVisible: true,
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);

const series = chart.addCandlestickSeries({
  lastPriceAnimation: LastPriceAnimationMode.Continuous,
});

series.setData(candleData);

// Создаём price line со смещённой меткой
createOffsetPriceLine(series, 104, 'Stop Loss', '#EF5350', 1);

// Метка будет на уровне 105, линия на 104
// Это уменьшает вероятность перекрытия анимацией
```

**Плюсы:**
- Простая реализация
- Использует стандартный API

**Минусы:**
- Метка на неточной цене
- Может быть confusing для пользователей

---

### Решение 5: Условное отключение анимации

**Оценка:** ⭐⭐⭐⭐ (4/5) - Умное управление анимацией

```typescript
import { createChart, IChartApi, ISeriesApi, LastPriceAnimationMode } from 'lightweight-charts';

class SmartLastPriceAnimation {
  private _series: ISeriesApi<any>;
  private _priceLines: { price: number }[] = [];
  private _proximityThreshold = 2; // Процент от цены
  
  constructor(series: ISeriesApi<any>) {
    this._series = series;
  }
  
  /**
   * Добавляет price line и проверяет необходимость анимации
   */
  addPriceLine(price: number, title: string, color: string): void {
    this._series.createPriceLine({
      price,
      title,
      color,
      axisLabelVisible: true,
    });
    
    this._priceLines.push({ price });
    this._checkAnimation();
  }
  
  /**
   * Вызывается при обновлении данных серии
   */
  onDataUpdate(): void {
    this._checkAnimation();
  }
  
  private _checkAnimation(): void {
    const data = this._series.data();
    if (data.length === 0) return;
    
    const lastItem = data[data.length - 1];
    const lastPrice = 'close' in lastItem ? lastItem.close : 
                      'value' in lastItem ? lastItem.value : null;
    
    if (lastPrice === null) return;
    
    // Проверяем близость к price lines
    const hasNearbyPriceLine = this._priceLines.some(line => {
      const diff = Math.abs(line.price - lastPrice) / lastPrice * 100;
      return diff < this._proximityThreshold;
    });
    
    // Отключаем анимацию если price line близко
    this._series.applyOptions({
      lastPriceAnimation: hasNearbyPriceLine 
        ? LastPriceAnimationMode.Disabled 
        : LastPriceAnimationMode.Continuous,
    });
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container);

const series = chart.addCandlestickSeries({
  lastPriceAnimation: LastPriceAnimationMode.Continuous,
});

const smartAnimation = new SmartLastPriceAnimation(series);

series.setData(candleData);

// Добавляем price line
smartAnimation.addPriceLine(104, 'Stop Loss', '#EF5350');

// При обновлении данных
function onNewData(newCandle: CandlestickData) {
  series.update(newCandle);
  smartAnimation.onDataUpdate(); // Проверяем анимацию
}
```

**Плюсы:**
- Анимация работает когда не мешает
- Автоматическое управление
- Лучший UX

**Минусы:**
- Дополнительная логика
- Анимация может "прыгать" вкл/выкл

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 2** (HTML overlay с высоким z-index):

```typescript
// Минимальный пример
const label = document.createElement('div');
label.style.cssText = `
  position: absolute;
  right: 0;
  padding: 2px 8px;
  background: #EF5350;
  color: white;
  font-size: 11px;
  z-index: 100;
  transform: translateY(-50%);
`;
label.textContent = 'SL: 104.00';
container.appendChild(label);

// Обновляем позицию
chart.subscribeCrosshairMove(() => {
  const y = series.priceToCoordinate(104);
  label.style.top = y !== null ? `${y}px` : '-9999px';
});
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Сохраняет анимацию | Надёжность | Рекомендация |
|---------|----------|-------------------|------------|--------------|
| **#1 Отключить анимацию** | ⭐⭐⭐⭐⭐ | ❌ | ⭐⭐⭐⭐⭐ | Если не нужна |
| **#2 HTML z-index** | ⭐⭐⭐⭐ | ✅ | ⭐⭐⭐⭐ | ✅ Универсальное |
| **#3 Custom primitive** | ⭐⭐ | ✅ | ⭐⭐⭐⭐⭐ | Сложные кейсы |
| **#4 Смещение метки** | ⭐⭐⭐⭐⭐ | ✅ | ⭐⭐⭐ | Быстрый хак |
| **#5 Условная анимация** | ⭐⭐⭐ | Частично | ⭐⭐⭐⭐ | Умный UX |

## 🔗 Источники

- [GitHub Issue #1711](https://github.com/tradingview/lightweight-charts/issues/1711) - Animation overlaps labels
- [LastPriceAnimationMode](https://tradingview.github.io/lightweight-charts/docs/api/enums/LastPriceAnimationMode)
- [Price Lines API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IPriceLine)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
