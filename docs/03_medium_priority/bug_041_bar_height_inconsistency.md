# БАГ #41: Непостоянная высота баров между положительными и отрицательными значениями

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1860](https://github.com/tradingview/lightweight-charts/issues/1860)  
> **Версии:** Все версии  
> **Статус:** ⚠️ Working as Intended (закрыт)

---

## 📋 Описание проблемы

### Суть проблемы
В histogram series высота баров для положительных и отрицательных значений **не симметрична** относительно базовой линии (zero line). Это воспринимается пользователями как визуальное несоответствие, особенно при отображении индикаторов типа MACD, осцилляторов или данных с положительными и отрицательными компонентами.

### Проявления проблемы

1. **Асимметричное масштабирование:**
   - Бар с значением `+10` и бар с `-10` могут иметь разную визуальную высоту
   - Нулевая линия не находится точно посередине

2. **Неочевидное поведение:**
   - Отрицательные значения отображаются как бары, направленные вниз
   - Масштабирование зависит от диапазона видимых данных

3. **Технический контекст:**
   ```javascript
   // Пример данных
   const data = [
       { time: '2024-01-01', value: 10, color: '#26a69a' },
       { time: '2024-01-02', value: -10, color: '#ef5350' },
       // Ожидание: одинаковая визуальная высота
       // Реальность: может отличаться из-за автомасштабирования
   ];
   ```

### Дизайн-решение библиотеки

Issue #1860 был **закрыт как "Working as Intended"**. Это означает:

1. Библиотека использует автомасштабирование на основе видимого диапазона данных
2. Визуальная высота бара пропорциональна его значению относительно текущего масштаба
3. Нулевая линия не гарантированно находится в центре области графика

### Затронутые сценарии

- **MACD индикатор** — гистограмма с положительными и отрицательными компонентами
- **RSI deviation** — отклонения от средней линии
- **Volume delta** — разница между buy и sell volume
- **Любые двусторонние индикаторы**

---

## 🔍 Найденные решения

### Решение 1: Использование абсолютных значений с кастомной окраской (⭐ 8/10)

**Описание:**  
Преобразование всех значений в абсолютные (`Math.abs(value)`) и использование отдельного поля `color` для индикации направления (положительное/отрицательное).

**Преимущества:**
- ✅ Все бары имеют одинаковую визуальную пропорцию
- ✅ Простая реализация
- ✅ Цвет передаёт информацию о знаке

**Недостатки:**
- ❌ Теряется визуальное направление (вверх/вниз)
- ❌ Может быть менее интуитивно

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Преобразует данные для histogram с абсолютными значениями
 */
function convertToAbsoluteValues(data, options = {}) {
    const {
        positiveColor = '#26a69a',
        negativeColor = '#ef5350',
        zeroColor = '#888888',
    } = options;

    return data.map(point => {
        const value = point.value;
        let color;
        
        if (value > 0) {
            color = positiveColor;
        } else if (value < 0) {
            color = negativeColor;
        } else {
            color = zeroColor;
        }
        
        return {
            ...point,
            value: Math.abs(value),
            color,
            _originalValue: value, // Сохраняем оригинальное значение
        };
    });
}

// Использование
const chart = createChart(container, { width: 800, height: 400 });

const histogramSeries = chart.addHistogramSeries({
    priceFormat: {
        type: 'volume',
    },
    priceLineVisible: false,
});

const rawData = [
    { time: '2024-01-01', value: 10 },
    { time: '2024-01-02', value: -5 },
    { time: '2024-01-03', value: 15 },
    { time: '2024-01-04', value: -20 },
];

const processedData = convertToAbsoluteValues(rawData);
histogramSeries.setData(processedData);
```

---

### Решение 2: Две отдельные серии для положительных и отрицательных значений (⭐ 9/10)

**Описание:**  
Создание двух histogram серий — одной для положительных значений, другой для отрицательных. Это обеспечивает визуальное разделение и корректное направление баров.

**Преимущества:**
- ✅ Сохраняется визуальное направление (вверх/вниз)
- ✅ Независимое масштабирование каждой серии
- ✅ Чёткое разделение положительных и отрицательных значений

**Недостатки:**
- ❌ Две серии вместо одной
- ❌ Требуется синхронизация данных

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Создаёт двойную histogram серию для симметричного отображения
 */
class SymmetricHistogramManager {
    constructor(chart, options = {}) {
        this.chart = chart;
        this.options = {
            positiveColor: options.positiveColor || '#26a69a',
            negativeColor: options.negativeColor || '#ef5350',
            priceScaleId: options.priceScaleId || 'histogram',
            scaleMargins: options.scaleMargins || { top: 0.8, bottom: 0 },
        };
        
        this._createSeries();
    }

    _createSeries() {
        // Серия для положительных значений
        this.positiveSeries = this.chart.addHistogramSeries({
            color: this.options.positiveColor,
            priceScaleId: this.options.priceScaleId,
            priceLineVisible: false,
            lastValueVisible: false,
        });

        // Серия для отрицательных значений
        this.negativeSeries = this.chart.addHistogramSeries({
            color: this.options.negativeColor,
            priceScaleId: this.options.priceScaleId,
            priceLineVisible: false,
            lastValueVisible: false,
        });

        // Настройка общей шкалы
        this.chart.priceScale(this.options.priceScaleId).applyOptions({
            scaleMargins: this.options.scaleMargins,
        });
    }

    /**
     * Устанавливает данные для histogram
     */
    setData(data) {
        const positiveData = [];
        const negativeData = [];

        data.forEach(point => {
            const value = point.value;
            
            if (value >= 0) {
                positiveData.push({
                    time: point.time,
                    value: value,
                    color: this.options.positiveColor,
                });
                // Добавляем whitespace в отрицательную серию
                negativeData.push({ time: point.time });
            } else {
                negativeData.push({
                    time: point.time,
                    value: value,
                    color: this.options.negativeColor,
                });
                // Добавляем whitespace в положительную серию
                positiveData.push({ time: point.time });
            }
        });

        this.positiveSeries.setData(positiveData);
        this.negativeSeries.setData(negativeData);
    }

    /**
     * Обновляет последнюю точку
     */
    update(point) {
        const value = point.value;
        
        if (value >= 0) {
            this.positiveSeries.update({
                time: point.time,
                value: value,
                color: this.options.positiveColor,
            });
            this.negativeSeries.update({ time: point.time });
        } else {
            this.negativeSeries.update({
                time: point.time,
                value: value,
                color: this.options.negativeColor,
            });
            this.positiveSeries.update({ time: point.time });
        }
    }

    /**
     * Удаляет серии
     */
    remove() {
        this.chart.removeSeries(this.positiveSeries);
        this.chart.removeSeries(this.negativeSeries);
    }
}

// ============ Использование ============

const chart = createChart(container, {
    width: 800,
    height: 400,
    layout: {
        background: { type: 'solid', color: '#1e222d' },
        textColor: '#d1d4dc',
    },
});

// Добавляем основную серию (например, свечи)
const candleSeries = chart.addCandlestickSeries();
candleSeries.setData(priceData);

// Создаём симметричную histogram
const histogramManager = new SymmetricHistogramManager(chart, {
    positiveColor: '#26a69a',
    negativeColor: '#ef5350',
    scaleMargins: { top: 0.7, bottom: 0 },
});

// MACD histogram данные
const macdHistogram = [
    { time: '2024-01-01', value: 0.5 },
    { time: '2024-01-02', value: 0.8 },
    { time: '2024-01-03', value: -0.3 },
    { time: '2024-01-04', value: -0.7 },
    { time: '2024-01-05', value: 0.2 },
];

histogramManager.setData(macdHistogram);
```

---

### Решение 3: Принудительная установка симметричного диапазона (⭐ 8/10)

**Описание:**  
Вычисление максимального абсолютного значения и установка симметричного диапазона для price scale (`-max` до `+max`).

**Преимущества:**
- ✅ Нулевая линия точно посередине
- ✅ Симметричное визуальное распределение
- ✅ Работает с одной серией

**Недостатки:**
- ❌ Требует пересчёта при каждом обновлении данных
- ❌ Отключает автомасштабирование

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Менеджер симметричного масштабирования
 */
class SymmetricScaleManager {
    constructor(chart, series, priceScaleId = 'right') {
        this.chart = chart;
        this.series = series;
        this.priceScaleId = priceScaleId;
        this.currentData = [];
    }

    /**
     * Устанавливает данные с симметричным масштабированием
     */
    setData(data) {
        this.currentData = data;
        this.series.setData(data);
        this._applySymmetricScale();
    }

    /**
     * Обновляет данные
     */
    update(point) {
        // Находим или добавляем точку
        const existingIndex = this.currentData.findIndex(
            d => d.time === point.time
        );
        
        if (existingIndex >= 0) {
            this.currentData[existingIndex] = point;
        } else {
            this.currentData.push(point);
        }
        
        this.series.update(point);
        this._applySymmetricScale();
    }

    _applySymmetricScale() {
        // Находим максимальное абсолютное значение
        const maxAbsValue = this.currentData.reduce((max, point) => {
            const absValue = Math.abs(point.value || 0);
            return Math.max(max, absValue);
        }, 0);

        if (maxAbsValue === 0) return;

        // Добавляем небольшой padding (10%)
        const padding = maxAbsValue * 0.1;
        const range = maxAbsValue + padding;

        // Отключаем автомасштабирование и устанавливаем диапазон
        this.chart.priceScale(this.priceScaleId).applyOptions({
            autoScale: false,
        });

        // Устанавливаем симметричный диапазон
        // Примечание: Lightweight Charts не имеет прямого метода setMinMax,
        // поэтому используем workaround через scaleMargins
        this.series.applyOptions({
            autoscaleInfoProvider: () => ({
                priceRange: {
                    minValue: -range,
                    maxValue: range,
                },
                margins: {
                    above: 0.1,
                    below: 0.1,
                },
            }),
        });
    }

    /**
     * Включает автомасштабирование обратно
     */
    enableAutoScale() {
        this.chart.priceScale(this.priceScaleId).applyOptions({
            autoScale: true,
        });
        this.series.applyOptions({
            autoscaleInfoProvider: undefined,
        });
    }
}

// ============ Использование ============

const chart = createChart(container, { width: 800, height: 400 });

const histogramSeries = chart.addHistogramSeries({
    color: '#2962FF',
    priceLineVisible: false,
});

const scaleManager = new SymmetricScaleManager(chart, histogramSeries);

const data = [
    { time: '2024-01-01', value: 10, color: '#26a69a' },
    { time: '2024-01-02', value: -5, color: '#ef5350' },
    { time: '2024-01-03', value: 15, color: '#26a69a' },
    { time: '2024-01-04', value: -20, color: '#ef5350' },
];

scaleManager.setData(data);
```

---

### Решение 4: Custom autoscaleInfoProvider (⭐ 9/10)

**Описание:**  
Использование `autoscaleInfoProvider` для принудительной симметрии относительно нуля.

**Преимущества:**
- ✅ Нативный API
- ✅ Автоматически адаптируется к данным
- ✅ Не требует внешнего управления

**Недостатки:**
- ❌ Может влиять на другие серии на той же шкале

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Создаёт autoscaleInfoProvider для симметричного масштабирования
 */
function createSymmetricAutoscaleProvider(seriesData) {
    return (original) => {
        // Используем оригинальный расчёт как базу
        const res = original();
        
        if (!res || !res.priceRange) {
            return res;
        }

        const { minValue, maxValue } = res.priceRange;
        
        // Вычисляем максимальное абсолютное значение
        const maxAbs = Math.max(Math.abs(minValue), Math.abs(maxValue));
        
        // Возвращаем симметричный диапазон
        return {
            priceRange: {
                minValue: -maxAbs,
                maxValue: maxAbs,
            },
            margins: res.margins,
        };
    };
}

// ============ Использование ============

const chart = createChart(container, {
    width: 800,
    height: 400,
    layout: {
        background: { type: 'solid', color: '#1e222d' },
        textColor: '#d1d4dc',
    },
});

const histogramSeries = chart.addHistogramSeries({
    color: '#2962FF',
    priceLineVisible: false,
    autoscaleInfoProvider: createSymmetricAutoscaleProvider(),
});

const data = [
    { time: '2024-01-01', value: 10, color: '#26a69a' },
    { time: '2024-01-02', value: -5, color: '#ef5350' },
    { time: '2024-01-03', value: 15, color: '#26a69a' },
    { time: '2024-01-04', value: -20, color: '#ef5350' },
    { time: '2024-01-05', value: 8, color: '#26a69a' },
];

histogramSeries.setData(data);
chart.timeScale().fitContent();
```

---

### Решение 5: Baseline Series вместо Histogram (⭐ 7/10)

**Описание:**  
Использование `BaselineSeries` вместо `HistogramSeries` для визуализации данных с базовой линией.

**Преимущества:**
- ✅ Встроенная поддержка baseline
- ✅ Разные цвета выше и ниже линии
- ✅ Визуально привлекательно

**Недостатки:**
- ❌ Это line/area, а не бары
- ❌ Может не подходить для histogram-style визуализации

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
    width: 800,
    height: 400,
    layout: {
        background: { type: 'solid', color: '#1e222d' },
        textColor: '#d1d4dc',
    },
});

const baselineSeries = chart.addBaselineSeries({
    baseValue: { type: 'price', price: 0 },
    topLineColor: 'rgba(38, 166, 154, 1)',
    topFillColor1: 'rgba(38, 166, 154, 0.28)',
    topFillColor2: 'rgba(38, 166, 154, 0.05)',
    bottomLineColor: 'rgba(239, 83, 80, 1)',
    bottomFillColor1: 'rgba(239, 83, 80, 0.05)',
    bottomFillColor2: 'rgba(239, 83, 80, 0.28)',
});

const data = [
    { time: '2024-01-01', value: 10 },
    { time: '2024-01-02', value: -5 },
    { time: '2024-01-03', value: 15 },
    { time: '2024-01-04', value: -20 },
    { time: '2024-01-05', value: 8 },
];

baselineSeries.setData(data);
chart.timeScale().fitContent();
```

---

### Решение 6: Custom Series для полного контроля (⭐ 6/10)

**Описание:**  
Создание Custom Series (доступно с v4.1) для полного контроля над рендерингом баров.

**Преимущества:**
- ✅ Полный контроль над отрисовкой
- ✅ Любая кастомизация

**Недостатки:**
- ❌ Высокая сложность реализации
- ❌ Требует глубокого понимания Canvas API

**Реализация (упрощённый пример):**
```javascript
import { createChart, customSeriesDefaultOptions } from 'lightweight-charts';

/**
 * Кастомная симметричная histogram серия
 */
class SymmetricHistogramSeries {
    constructor() {
        this._options = {
            ...customSeriesDefaultOptions,
            positiveColor: '#26a69a',
            negativeColor: '#ef5350',
            barWidth: 0.8, // Процент от доступного пространства
        };
    }

    priceValueBuilder(plotRow) {
        return [plotRow.value];
    }

    isWhitespace(data) {
        return data.value === undefined;
    }

    renderer() {
        return new SymmetricHistogramRenderer();
    }

    update(options) {
        this._options = { ...this._options, ...options };
    }

    defaultOptions() {
        return this._options;
    }
}

class SymmetricHistogramRenderer {
    draw(target, priceConverter) {
        target.useBitmapCoordinateSpace((scope) => {
            const ctx = scope.context;
            const data = scope.seriesData;
            
            if (!data || data.length === 0) return;

            // Находим максимальное абсолютное значение для симметричного масштабирования
            const maxAbs = data.reduce((max, bar) => {
                return Math.max(max, Math.abs(bar.originalValue || bar.value || 0));
            }, 0);

            data.forEach((bar) => {
                if (bar.value === undefined) return;

                const x = bar.x;
                const width = bar.columnWidth * 0.8;
                
                // Рассчитываем высоту относительно симметричного диапазона
                const normalizedValue = bar.originalValue / maxAbs;
                const centerY = scope.bitmapSize.height / 2;
                const maxHeight = scope.bitmapSize.height / 2;
                const barHeight = normalizedValue * maxHeight;

                // Определяем цвет
                const color = bar.originalValue >= 0 
                    ? this._options.positiveColor 
                    : this._options.negativeColor;

                ctx.fillStyle = color;
                
                if (bar.originalValue >= 0) {
                    ctx.fillRect(
                        x - width / 2,
                        centerY - barHeight,
                        width,
                        barHeight
                    );
                } else {
                    ctx.fillRect(
                        x - width / 2,
                        centerY,
                        width,
                        Math.abs(barHeight)
                    );
                }
            });

            // Рисуем нулевую линию
            ctx.strokeStyle = '#888888';
            ctx.lineWidth = 1;
            ctx.beginPath();
            ctx.moveTo(0, scope.bitmapSize.height / 2);
            ctx.lineTo(scope.bitmapSize.width, scope.bitmapSize.height / 2);
            ctx.stroke();
        });
    }
}

// Использование
const chart = createChart(container, { width: 800, height: 400 });

const customSeries = chart.addCustomSeries(new SymmetricHistogramSeries(), {
    positiveColor: '#26a69a',
    negativeColor: '#ef5350',
});

const data = [
    { time: '2024-01-01', value: 10 },
    { time: '2024-01-02', value: -5 },
    { time: '2024-01-03', value: 15 },
    { time: '2024-01-04', value: -20 },
];

// Преобразуем данные с сохранением оригинального значения
const processedData = data.map(d => ({
    time: d.time,
    value: Math.abs(d.value),
    originalValue: d.value,
}));

customSeries.setData(processedData);
```

---

## ✅ Рекомендуемое решение

### Комбинация: autoscaleInfoProvider + раздельные серии

Для достижения симметричного отображения histogram с положительными и отрицательными значениями рекомендуется использовать `autoscaleInfoProvider` с раздельными сериями:

```javascript
import { createChart } from 'lightweight-charts';

/**
 * Полный менеджер симметричной histogram
 */
class AdvancedSymmetricHistogram {
    constructor(chart, options = {}) {
        this.chart = chart;
        this.options = {
            positiveColor: options.positiveColor || '#26a69a',
            negativeColor: options.negativeColor || '#ef5350',
            priceScaleId: options.priceScaleId || 'histogram',
            scaleMargins: options.scaleMargins || { top: 0.7, bottom: 0 },
        };
        
        this.positiveSeries = null;
        this.negativeSeries = null;
        this.data = [];
        
        this._init();
    }

    _init() {
        const symmetricProvider = this._createSymmetricProvider();

        // Серия для положительных значений
        this.positiveSeries = this.chart.addHistogramSeries({
            color: this.options.positiveColor,
            priceScaleId: this.options.priceScaleId,
            priceLineVisible: false,
            lastValueVisible: false,
            autoscaleInfoProvider: symmetricProvider,
        });

        // Серия для отрицательных значений
        this.negativeSeries = this.chart.addHistogramSeries({
            color: this.options.negativeColor,
            priceScaleId: this.options.priceScaleId,
            priceLineVisible: false,
            lastValueVisible: false,
            autoscaleInfoProvider: symmetricProvider,
        });

        // Настройка шкалы
        this.chart.priceScale(this.options.priceScaleId).applyOptions({
            scaleMargins: this.options.scaleMargins,
        });
    }

    _createSymmetricProvider() {
        const self = this;
        
        return (original) => {
            const res = original();
            
            if (!res || !res.priceRange) return res;

            // Находим максимальное абсолютное значение в данных
            const maxAbs = self.data.reduce((max, point) => {
                return Math.max(max, Math.abs(point.value || 0));
            }, 0);

            if (maxAbs === 0) return res;

            // Добавляем 10% padding
            const paddedMax = maxAbs * 1.1;

            return {
                priceRange: {
                    minValue: -paddedMax,
                    maxValue: paddedMax,
                },
                margins: {
                    above: 0.05,
                    below: 0.05,
                },
            };
        };
    }

    /**
     * Устанавливает данные
     */
    setData(data) {
        this.data = data;
        
        const positiveData = [];
        const negativeData = [];

        data.forEach(point => {
            const value = point.value;
            
            if (value >= 0) {
                positiveData.push({
                    time: point.time,
                    value: value,
                    color: point.color || this.options.positiveColor,
                });
                negativeData.push({ time: point.time });
            } else {
                negativeData.push({
                    time: point.time,
                    value: value,
                    color: point.color || this.options.negativeColor,
                });
                positiveData.push({ time: point.time });
            }
        });

        this.positiveSeries.setData(positiveData);
        this.negativeSeries.setData(negativeData);
    }

    /**
     * Обновляет последнюю точку
     */
    update(point) {
        // Обновляем внутренний массив данных
        const existingIndex = this.data.findIndex(d => d.time === point.time);
        if (existingIndex >= 0) {
            this.data[existingIndex] = point;
        } else {
            this.data.push(point);
        }

        const value = point.value;
        
        if (value >= 0) {
            this.positiveSeries.update({
                time: point.time,
                value: value,
                color: point.color || this.options.positiveColor,
            });
            this.negativeSeries.update({ time: point.time });
        } else {
            this.negativeSeries.update({
                time: point.time,
                value: value,
                color: point.color || this.options.negativeColor,
            });
            this.positiveSeries.update({ time: point.time });
        }
    }

    /**
     * Применяет градиентную окраску по интенсивности
     */
    applyGradientColors(basePositiveColor, baseNegativeColor) {
        this.data = this.data.map(point => {
            const maxAbs = Math.max(...this.data.map(d => Math.abs(d.value)));
            const intensity = Math.abs(point.value) / maxAbs;
            
            // Вычисляем цвет с учётом интенсивности
            const baseColor = point.value >= 0 ? basePositiveColor : baseNegativeColor;
            const color = this._adjustColorIntensity(baseColor, intensity);
            
            return { ...point, color };
        });
        
        this.setData(this.data);
    }

    _adjustColorIntensity(hexColor, intensity) {
        // Преобразуем hex в RGB
        const r = parseInt(hexColor.slice(1, 3), 16);
        const g = parseInt(hexColor.slice(3, 5), 16);
        const b = parseInt(hexColor.slice(5, 7), 16);
        
        // Применяем интенсивность (от 0.3 до 1.0)
        const adjustedIntensity = 0.3 + intensity * 0.7;
        
        const newR = Math.round(r * adjustedIntensity);
        const newG = Math.round(g * adjustedIntensity);
        const newB = Math.round(b * adjustedIntensity);
        
        return `rgb(${newR}, ${newG}, ${newB})`;
    }

    /**
     * Удаляет серии
     */
    remove() {
        this.chart.removeSeries(this.positiveSeries);
        this.chart.removeSeries(this.negativeSeries);
    }
}

// ============ Полный пример использования ============

const container = document.getElementById('chart');

const chart = createChart(container, {
    width: 1000,
    height: 500,
    layout: {
        background: { type: 'solid', color: '#131722' },
        textColor: '#d1d4dc',
    },
    grid: {
        vertLines: { color: 'rgba(42, 46, 57, 0.6)' },
        horzLines: { color: 'rgba(42, 46, 57, 0.6)' },
    },
});

// Основная серия (свечи)
const candleSeries = chart.addCandlestickSeries({
    upColor: '#26a69a',
    downColor: '#ef5350',
    borderVisible: false,
    wickUpColor: '#26a69a',
    wickDownColor: '#ef5350',
});

candleSeries.setData(priceData);

// Создаём симметричную MACD histogram
const macdHistogram = new AdvancedSymmetricHistogram(chart, {
    positiveColor: '#26a69a',
    negativeColor: '#ef5350',
    scaleMargins: { top: 0.75, bottom: 0 },
});

// Данные MACD histogram
const macdData = [
    { time: '2024-01-01', value: 0.5 },
    { time: '2024-01-02', value: 0.8 },
    { time: '2024-01-03', value: 1.2 },
    { time: '2024-01-04', value: 0.6 },
    { time: '2024-01-05', value: -0.3 },
    { time: '2024-01-06', value: -0.7 },
    { time: '2024-01-07', value: -1.1 },
    { time: '2024-01-08', value: -0.5 },
    { time: '2024-01-09', value: 0.2 },
    { time: '2024-01-10', value: 0.9 },
];

macdHistogram.setData(macdData);
chart.timeScale().fitContent();

// Real-time обновления
function simulateRealtimeUpdate() {
    const lastTime = macdData[macdData.length - 1].time;
    const nextDay = new Date(lastTime);
    nextDay.setDate(nextDay.getDate() + 1);
    
    const newValue = (Math.random() - 0.5) * 2.4;
    
    macdHistogram.update({
        time: nextDay.toISOString().split('T')[0],
        value: newValue,
    });
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Симметрия | Направление | Real-time |
|---------|--------|-----------|-----------|-------------|-----------|
| **autoscaleInfoProvider** | ⭐ 9/10 | Низкая | ✅ Да | ✅ Да | ✅ Да |
| Две серии | ⭐ 9/10 | Средняя | ✅ Да | ✅ Да | ✅ Да |
| Принудительный диапазон | ⭐ 8/10 | Средняя | ✅ Да | ✅ Да | ⚠️ Пересчёт |
| Абсолютные значения | ⭐ 8/10 | Низкая | ✅ Да | ❌ Нет | ✅ Да |
| Baseline Series | ⭐ 7/10 | Низкая | ✅ Да | ⚠️ Area | ✅ Да |
| Custom Series | ⭐ 6/10 | Высокая | ✅ Да | ✅ Да | ✅ Да |

---

## ⚠️ Важные замечания

1. **Issue #1860 закрыт как "Working as Intended"** — это не баг, а архитектурное решение библиотеки.

2. **`autoscaleInfoProvider`** — самый элегантный способ достижения симметричного масштабирования.

3. **Две серии vs одна** — для полного контроля лучше использовать две серии (положительная + отрицательная).

4. **Производительность** — все решения имеют минимальное влияние на производительность.

---

## 🔗 Источники

1. [GitHub Issue #1860 - Height of bars are not consistent between positive and negative values](https://github.com/tradingview/lightweight-charts/issues/1860)
2. [Lightweight Charts - Histogram Series](https://tradingview.github.io/lightweight-charts/docs/series-types/histogram)
3. [Lightweight Charts - autoscaleInfoProvider](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/SeriesOptionsCommon#autoscaleinfoprovider)
4. [Lightweight Charts - Custom Series](https://tradingview.github.io/lightweight-charts/docs/plugins/custom_series)
5. [Stack Overflow - Histogram with negative values](https://stackoverflow.com/questions/tagged/lightweight-charts+histogram)

---

## 📅 История изменений

| Дата | Версия | Изменения |
|------|--------|-----------|
| 2026-01-18 | 1.0 | Создание документа |

---

*Документация подготовлена для проекта lightweight-charts error tracking*
