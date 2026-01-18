# БАГ #30: Несоответствие позиции линии и метки при minMove

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1959](https://github.com/tradingview/lightweight-charts/issues/1959)  
> **Версии:** v5.0+  
> **Статус:** 🔴 Open  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы
При создании пользовательской price line с точным значением цены, которое не кратно `minMove`, возникает визуальное несоответствие:

- **Линия** отрисовывается на **корректной Y-координате** (точное значение)
- **Метка на оси цены** округляется до ближайшего значения `minMove`

Это создаёт путаницу для пользователей, так как они видят линию на одном уровне, а метка показывает другое значение.

### Пример проблемы
```javascript
// Серия с minMove = 0.0005
const series = chart.addSeries(CandlestickSeries, {
  priceFormat: {
    type: 'price',
    precision: 5,
    minMove: 0.0005  // Шаг цены 0.0005
  }
});

// Создаём линию на точном значении
const preciseValue = 1.15949;  // НЕ кратно 0.0005!
series.createPriceLine({
  price: preciseValue,         // 1.15949
  axisLabelVisible: true
});

// Результат:
// ✅ Линия нарисована на Y-координате для 1.15949
// ❌ Метка показывает 1.15950 (округлено до minMove)
```

### Почему это проблема?
1. **Неточность торговых уровней**: Stop-loss/take-profit линии показывают неверную цену
2. **Путаница пользователей**: Визуальное расположение не соответствует отображаемому значению
3. **Невозможность переопределить**: Нет API для отдельного форматирования меток price lines

### Воспроизведение
```html
<div id="c" style="width:800px;height:400px"></div>
<script type="module">
import { createChart, CandlestickSeries } from 'lightweight-charts';

const chart = createChart(document.getElementById('c'), {});
const series = chart.addSeries(CandlestickSeries, {
  priceScaleId: 'right',
  priceFormat: {
    type: 'price',
    precision: 5,
    minMove: 0.0005
  }
});

series.setData([
  { time: 1, open: 1.1590, high: 1.1600, low: 1.1580, close: 1.1592 },
  { time: 2, open: 1.1592, high: 1.1602, low: 1.1585, close: 1.1591 }
]);

// Проблемная линия
const preciseValue = 1.15949;
series.createPriceLine({
  price: preciseValue,
  axisLabelVisible: true,
  color: '#21ba45',
  lineWidth: 2
});
// Наблюдение: линия на 1.15949, метка показывает 1.15950
</script>
```

---

## 🔍 Найденные решения

### Решение 1: Использование Custom Formatter для серии
**Оценка: 4/10**

Переопределение `priceFormat.formatter` для всей серии.

```javascript
const series = chart.addSeries(CandlestickSeries, {
  priceFormat: {
    type: 'custom',
    precision: 5,
    minMove: 0.00001,  // Уменьшаем minMove для точности
    formatter: (price) => price.toFixed(5)
  }
});
```

**Плюсы:**
- Простая реализация
- Работает из коробки

**Минусы:**
- Меняет форматирование для ВСЕХ цен (свечи, кроссхэр, ось)
- minMove всё ещё влияет на внутреннюю логику
- Может сломать другие визуальные элементы
- Не решает проблему для торговых инструментов с реальным tick size

---

### Решение 2: Скрытие стандартной метки + Custom HTML Label
**Оценка: 7/10**

Отключаем встроенную метку и создаём собственную HTML-метку, позиционированную поверх price scale.

```javascript
// Создаём price line без встроенной метки
const priceLine = series.createPriceLine({
  price: 1.15949,
  axisLabelVisible: false,  // Скрываем стандартную метку
  color: '#21ba45',
  lineWidth: 2
});

// Создаём кастомную HTML-метку
function createCustomLabel(chart, series, price, options = {}) {
  const {
    color = '#21ba45',
    textColor = '#ffffff',
    formatter = (p) => p.toFixed(5)
  } = options;

  const container = chart.chartElement().parentElement;
  const label = document.createElement('div');
  
  label.style.cssText = `
    position: absolute;
    right: 0;
    background: ${color};
    color: ${textColor};
    padding: 2px 6px;
    font-size: 11px;
    font-family: -apple-system, BlinkMacSystemFont, sans-serif;
    z-index: 100;
    pointer-events: none;
    border-radius: 2px;
    white-space: nowrap;
  `;
  
  label.textContent = formatter(price);
  container.style.position = 'relative';
  container.appendChild(label);

  // Функция обновления позиции
  function updatePosition() {
    const priceScale = series.priceScale();
    const coordinate = series.priceToCoordinate(price);
    
    if (coordinate !== null) {
      const scaleWidth = priceScale.width();
      label.style.top = `${coordinate - label.offsetHeight / 2}px`;
      label.style.right = `0px`;
      label.style.width = `${scaleWidth}px`;
      label.style.textAlign = 'center';
      label.style.display = 'block';
    } else {
      label.style.display = 'none';
    }
  }

  // Обновляем при изменении visible range
  chart.timeScale().subscribeVisibleTimeRangeChange(updatePosition);
  
  // Инициализация
  requestAnimationFrame(updatePosition);

  return {
    label,
    update: updatePosition,
    remove: () => {
      label.remove();
      chart.timeScale().unsubscribeVisibleTimeRangeChange(updatePosition);
    }
  };
}

// Использование
const customLabel = createCustomLabel(chart, series, 1.15949, {
  color: '#21ba45',
  formatter: (p) => p.toFixed(5)
});
```

**Плюсы:**
- Полный контроль над форматированием
- Точное отображение значения
- Гибкая стилизация

**Минусы:**
- Дополнительная сложность кода
- Требуется синхронизация с прокруткой/зумом
- Возможны проблемы с производительностью при многих линиях
- Необходим cleanup при удалении

---

### Решение 3: Custom Series Primitive с собственным Label Renderer
**Оценка: 8/10**

Использование Plugin API для создания примитива с собственной отрисовкой метки.

```javascript
class PrecisePriceLinePrimitive {
  constructor(options) {
    this._options = {
      price: options.price,
      color: options.color || '#3179F5',
      lineWidth: options.lineWidth || 1,
      lineStyle: options.lineStyle || 0, // LightweightCharts.LineStyle.Solid
      labelBackgroundColor: options.labelBackgroundColor || options.color || '#3179F5',
      labelTextColor: options.labelTextColor || '#FFFFFF',
      precision: options.precision || 5,
      formatter: options.formatter || null
    };
    this._series = null;
  }

  attached(params) {
    this._series = params.series;
    this._chart = params.chart;
    this._requestUpdate = params.requestUpdate;
  }

  detached() {
    this._series = null;
    this._chart = null;
  }

  updateAllViews() {
    if (this._requestUpdate) {
      this._requestUpdate();
    }
  }

  priceAxisViews() {
    return [{
      coordinate: () => {
        if (!this._series) return null;
        return this._series.priceToCoordinate(this._options.price);
      },
      text: () => {
        const formatter = this._options.formatter;
        if (formatter) {
          return formatter(this._options.price);
        }
        return this._options.price.toFixed(this._options.precision);
      },
      textColor: () => this._options.labelTextColor,
      backColor: () => this._options.labelBackgroundColor,
      visible: () => true
    }];
  }

  paneViews() {
    return [{
      renderer: () => ({
        draw: (target) => {
          const ctx = target.context;
          if (!this._series) return;
          
          const coordinate = this._series.priceToCoordinate(this._options.price);
          if (coordinate === null) return;
          
          const width = target.bitmapSize.width;
          const pixelRatio = target.horizontalPixelRatio;
          
          ctx.save();
          ctx.strokeStyle = this._options.color;
          ctx.lineWidth = this._options.lineWidth * pixelRatio;
          
          if (this._options.lineStyle === 1) {
            ctx.setLineDash([6 * pixelRatio, 4 * pixelRatio]);
          } else if (this._options.lineStyle === 2) {
            ctx.setLineDash([2 * pixelRatio, 2 * pixelRatio]);
          }
          
          ctx.beginPath();
          ctx.moveTo(0, coordinate * target.verticalPixelRatio);
          ctx.lineTo(width, coordinate * target.verticalPixelRatio);
          ctx.stroke();
          ctx.restore();
        }
      })
    }];
  }
}

// Использование
const precisePriceLine = new PrecisePriceLinePrimitive({
  price: 1.15949,
  color: '#21ba45',
  lineWidth: 2,
  precision: 5,
  formatter: (price) => `SL: ${price.toFixed(5)}`  // Кастомный формат
});

series.attachPrimitive(precisePriceLine);
```

**Плюсы:**
- Полный контроль над отрисовкой
- Точное значение в метке
- Интеграция с Plugin API
- Правильная реакция на scroll/zoom

**Минусы:**
- Сложнее реализации
- Требует понимания Plugin API
- Необходима полная реализация всех views
- Дублирование функционала встроенных price lines

---

### Решение 4: Monkey-patching PriceAxisView
**Оценка: 3/10**

Перехват внутренней логики форматирования.

```javascript
// ⚠️ НЕ РЕКОМЕНДУЕТСЯ - может сломаться в новых версиях!
const originalFormatter = series.priceAxisFormatters?.labelFormatter;
if (originalFormatter) {
  series.priceAxisFormatters.labelFormatter = (price) => {
    // Проверяем, относится ли цена к нашим price lines
    const customLines = [1.15949, 1.16100];  // Список кастомных уровней
    const tolerance = 0.00001;
    
    for (const customPrice of customLines) {
      if (Math.abs(price - customPrice) < tolerance) {
        return customPrice.toFixed(5);
      }
    }
    return originalFormatter(price);
  };
}
```

**Плюсы:**
- Минимальные изменения в коде

**Минусы:**
- ❌ Использует internal API
- ❌ Может сломаться в любой версии
- ❌ Непредсказуемое поведение
- ❌ Сложно поддерживать

---

### Решение 5: Уменьшение minMove для серии
**Оценка: 5/10**

Установка минимально возможного minMove для избежания округления.

```javascript
const series = chart.addSeries(CandlestickSeries, {
  priceFormat: {
    type: 'price',
    precision: 6,           // Увеличиваем precision
    minMove: 0.000001       // Минимальный шаг
  }
});

// Теперь price lines не будут округляться (или округление будет минимальным)
series.createPriceLine({
  price: 1.15949,
  axisLabelVisible: true
});
// Метка покажет 1.159490
```

**Плюсы:**
- Простейшее решение
- Быстрая реализация

**Минусы:**
- Не работает для инструментов с реальным tick size
- Избыточная точность на оси цены
- Визуально загромождённая ось

---

## ✅ Рекомендуемое решение

### Решение 3: Custom Series Primitive

Для production-приложений рекомендуется использовать **Custom Series Primitive** (Решение 3), так как оно:

1. Использует официальный Plugin API
2. Обеспечивает полный контроль над форматированием
3. Правильно интегрируется с системой обновлений графика
4. Будет работать в будущих версиях библиотеки

### Полная реализация

```javascript
// precise-price-line.js
import { LineStyle } from 'lightweight-charts';

/**
 * Примитив для создания price line с точным отображением значения в метке
 */
export class PrecisePriceLine {
  constructor(options) {
    this._options = {
      price: options.price,
      color: options.color ?? '#3179F5',
      lineWidth: options.lineWidth ?? 1,
      lineStyle: options.lineStyle ?? LineStyle.Solid,
      labelVisible: options.labelVisible ?? true,
      labelBackgroundColor: options.labelBackgroundColor ?? (options.color ?? '#3179F5'),
      labelTextColor: options.labelTextColor ?? '#FFFFFF',
      labelText: options.labelText ?? null,  // Кастомный текст
      formatter: options.formatter ?? null,   // Кастомный formatter
      precision: options.precision ?? 5
    };
    
    this._series = null;
    this._chart = null;
    this._requestUpdate = null;
  }

  // Lifecycle
  attached(params) {
    this._series = params.series;
    this._chart = params.chart;
    this._requestUpdate = params.requestUpdate;
  }

  detached() {
    this._series = null;
    this._chart = null;
    this._requestUpdate = null;
  }

  // Public API
  updateOptions(options) {
    Object.assign(this._options, options);
    this._requestUpdate?.();
  }

  options() {
    return { ...this._options };
  }

  price() {
    return this._options.price;
  }

  // Views
  priceAxisViews() {
    if (!this._options.labelVisible) return [];

    const that = this;
    return [{
      coordinate() {
        if (!that._series) return null;
        return that._series.priceToCoordinate(that._options.price);
      },
      text() {
        if (that._options.labelText) {
          return that._options.labelText;
        }
        if (that._options.formatter) {
          return that._options.formatter(that._options.price);
        }
        return that._options.price.toFixed(that._options.precision);
      },
      textColor() {
        return that._options.labelTextColor;
      },
      backColor() {
        return that._options.labelBackgroundColor;
      },
      visible() {
        return true;
      }
    }];
  }

  paneViews() {
    const that = this;
    return [{
      renderer() {
        return {
          draw(target) {
            if (!that._series) return;
            
            const coordinate = that._series.priceToCoordinate(that._options.price);
            if (coordinate === null) return;
            
            const ctx = target.context;
            const width = target.bitmapSize.width;
            const yCoord = coordinate * target.verticalPixelRatio;
            const pixelRatio = target.horizontalPixelRatio;
            
            ctx.save();
            ctx.strokeStyle = that._options.color;
            ctx.lineWidth = that._options.lineWidth * pixelRatio;
            
            // Стиль линии
            switch (that._options.lineStyle) {
              case LineStyle.Dashed:
                ctx.setLineDash([6 * pixelRatio, 4 * pixelRatio]);
                break;
              case LineStyle.Dotted:
                ctx.setLineDash([2 * pixelRatio, 2 * pixelRatio]);
                break;
              case LineStyle.LargeDashed:
                ctx.setLineDash([12 * pixelRatio, 6 * pixelRatio]);
                break;
              case LineStyle.SparseDotted:
                ctx.setLineDash([2 * pixelRatio, 6 * pixelRatio]);
                break;
              default:
                ctx.setLineDash([]);
            }
            
            ctx.beginPath();
            ctx.moveTo(0, yCoord);
            ctx.lineTo(width, yCoord);
            ctx.stroke();
            ctx.restore();
          }
        };
      }
    }];
  }
}

// -------------------
// Пример использования
// -------------------

import { createChart, CandlestickSeries } from 'lightweight-charts';
import { PrecisePriceLine } from './precise-price-line.js';

const chart = createChart(document.getElementById('chart'), {
  width: 800,
  height: 400
});

const series = chart.addSeries(CandlestickSeries, {
  priceFormat: {
    type: 'price',
    precision: 5,
    minMove: 0.0005  // Шаг для торгового инструмента
  }
});

series.setData([
  { time: '2025-01-01', open: 1.1590, high: 1.1600, low: 1.1580, close: 1.1592 },
  { time: '2025-01-02', open: 1.1592, high: 1.1620, low: 1.1585, close: 1.1615 }
]);

// Создаём точные price lines
const stopLoss = new PrecisePriceLine({
  price: 1.15949,
  color: '#F23645',
  lineWidth: 2,
  lineStyle: LineStyle.Dashed,
  precision: 5,
  formatter: (p) => `SL: ${p.toFixed(5)}`
});

const takeProfit = new PrecisePriceLine({
  price: 1.16287,
  color: '#26A69A',
  lineWidth: 2,
  lineStyle: LineStyle.Dashed,
  precision: 5,
  formatter: (p) => `TP: ${p.toFixed(5)}`
});

const entryLevel = new PrecisePriceLine({
  price: 1.15923,
  color: '#2196F3',
  lineWidth: 1,
  lineStyle: LineStyle.Solid,
  labelText: 'Entry @ 1.15923'  // Статический текст
});

// Прикрепляем к серии
series.attachPrimitive(stopLoss);
series.attachPrimitive(takeProfit);
series.attachPrimitive(entryLevel);

// Динамическое обновление
document.getElementById('updateBtn').addEventListener('click', () => {
  stopLoss.updateOptions({
    price: 1.15800,
    labelText: 'SL Moved'
  });
});

// Удаление
document.getElementById('removeBtn').addEventListener('click', () => {
  series.detachPrimitive(stopLoss);
});
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Точность | Производительность | Поддержка |
|---------|--------|-----------|----------|-------------------|-----------|
| 1. Custom Formatter для серии | 4/10 | Низкая | Средняя | Высокая | ✅ Стабильная |
| 2. HTML-метка поверх графика | 7/10 | Средняя | Высокая | Средняя | ⚠️ Требует синхронизации |
| 3. **Custom Primitive** | **8/10** | Высокая | **Высокая** | **Высокая** | ✅ **Официальный API** |
| 4. Monkey-patching | 3/10 | Низкая | Высокая | Высокая | ❌ Нестабильная |
| 5. Уменьшение minMove | 5/10 | Низкая | Средняя | Высокая | ✅ Стабильная |

### Рекомендации по выбору:

- **Быстрое решение** → Решение 5 (уменьшение minMove)
- **Простое + гибкое** → Решение 2 (HTML-метки)
- **Production-ready** → Решение 3 (Custom Primitive) ✅

---

## 🔗 Источники

1. [GitHub Issue #1959](https://github.com/tradingview/lightweight-charts/issues/1959) - Оригинальный баг-репорт
2. [Lightweight Charts Plugin API](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/plugins) - Документация по Plugin API
3. [Price Format Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceFormatBuiltIn) - Опции форматирования цены
4. [GitHub Issue #1596](https://github.com/tradingview/lightweight-charts/issues/1596) - Обсуждение minMove
5. [GitHub PR #1993](https://github.com/tradingview/lightweight-charts/pull/1993) - Фикс позиционирования меток
6. [GitHub Issue #945](https://github.com/tradingview/lightweight-charts/issues/945) - Feature request для кастомных меток

---

## 📝 Примечания

### Ожидаемое официальное решение
В issue #1959 запрашивается добавление:
1. Отдельного `formatter` для price lines
2. Опции отключения snap-to-minMove для метки

До официального фикса используйте Custom Primitive как наиболее надёжное решение.

### Совместимость
- Решение 3 (Custom Primitive) работает с версией 4.1+ (Plugin API)
- Для версий 3.x используйте Решение 2 (HTML-метки)
