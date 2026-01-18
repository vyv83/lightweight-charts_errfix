# БАГ #13: Маркеры точек исчезают после точки с другим цветом

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#2017](https://github.com/tradingview/lightweight-charts/issues/2017), [#1926](https://github.com/tradingview/lightweight-charts/issues/1926)  
> **Версии:** v5.0+  
> **Статус:** 🔴 Open (ноябрь 2025, подтверждено как баг)  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

При использовании **per-point colors** (индивидуальные цвета для каждой точки) в `LineSeries`, маркеры (`pointMarkers`) **перестают рендериться** после первой точки с изменённым цветом.

### Симптомы:
- Маркеры отображаются только до первой точки со сменой цвета
- После точки с другим цветом все последующие маркеры исчезают
- Линия рисуется корректно, но маркеры — нет
- Цвет линии может обновляться на один бар позже ожидаемого

### Техническая причина:
Текущая реализация использует **"Forward-looking"** подход для стилизации:
- Стиль точки определяет стиль **следующего** сегмента линии
- Для маркеров ожидается **"Backward-looking"** подход
- Это несоответствие вызывает прекращение рендеринга маркеров

### Сценарии воспроизведения:
1. **Highlight аномалий:** Разноцветные точки для индикации аномальных значений
2. **Threshold-based coloring:** Цвет зависит от пересечения порогов
3. **Sentiment coloring:** Зелёные/красные точки по настроению

### Пример проблемного кода:
```javascript
const lineSeries = chart.addLineSeries({
    lineWidth: 2,
    pointMarkersVisible: true,
    pointMarkersRadius: 4,
});

// Данные с per-point colors
lineSeries.setData([
    { time: '2024-01-01', value: 100 },                    // ✅ маркер есть
    { time: '2024-01-02', value: 105 },                    // ✅ маркер есть
    { time: '2024-01-03', value: 98, color: '#ff0000' },   // ✅ маркер есть (последний!)
    { time: '2024-01-04', value: 102 },                    // ❌ маркер ИСЧЕЗ
    { time: '2024-01-05', value: 110 },                    // ❌ маркер ИСЧЕЗ
    { time: '2024-01-06', value: 115, color: '#00ff00' },  // ❌ маркер ИСЧЕЗ
]);
// После точки с color: '#ff0000' все маркеры пропадают!
```

---

## 🔍 Найденные решения

### Решение 1: Отдельная серия для маркеров (Scatter overlay)
**Оценка: 8/10** ⭐ РЕКОМЕНДУЕМОЕ

Использование отдельной baseline/line серии ТОЛЬКО для отображения маркеров.

**Преимущества:**
- ✅ Полностью обходит баг
- ✅ Независимый контроль над маркерами
- ✅ Можно использовать разные цвета маркеров
- ✅ Работает в текущих версиях

**Недостатки:**
- ⚠️ Две серии вместо одной
- ⚠️ Требуется синхронизация данных
- ⚠️ Небольшой overhead по памяти

```javascript
const chart = createChart(container);

// Основная серия - линия без маркеров
const lineSeries = chart.addLineSeries({
    lineWidth: 2,
    pointMarkersVisible: false,  // Отключаем встроенные маркеры!
    lastValueVisible: true,
    priceLineVisible: true,
});

// Серия-overlay только для маркеров
const markerSeries = chart.addLineSeries({
    lineWidth: 0,                // Линия невидима
    pointMarkersVisible: true,   // Только маркеры
    pointMarkersRadius: 5,
    lastValueVisible: false,
    priceLineVisible: false,
    crosshairMarkerVisible: false,
});

// Подготовка данных
const lineData = [
    { time: '2024-01-01', value: 100 },
    { time: '2024-01-02', value: 105 },
    { time: '2024-01-03', value: 98 },
    { time: '2024-01-04', value: 102 },
    { time: '2024-01-05', value: 110 },
];

// Данные маркеров с индивидуальными цветами
const markerData = lineData.map((item, index) => ({
    ...item,
    color: item.value > 100 ? '#26a69a' : '#ef5350'  // Зелёный/красный по значению
}));

lineSeries.setData(lineData);
markerSeries.setData(markerData);

// Оба на одной price scale
lineSeries.priceScale().applyOptions({ scaleMargins: { top: 0.1, bottom: 0.1 } });
```

---

### Решение 2: Series Primitives для кастомных маркеров
**Оценка: 9/10**

Использование Series Primitives API для полного контроля над рендерингом маркеров.

**Преимущества:**
- ✅ Полный контроль над отрисовкой
- ✅ Любые формы и стили маркеров
- ✅ Не зависит от внутренних багов
- ✅ Максимальная гибкость

**Недостатки:**
- ⚠️ Требует написания кода рендеринга
- ⚠️ Выше сложность реализации
- ⚠️ Нужно понимание Canvas API

```javascript
// Custom Primitive для маркеров
class PointMarkersPrimitive {
    constructor(data) {
        this._data = data;
    }

    updateData(data) {
        this._data = data;
    }

    updateAllViews() {
        // Вызывается при обновлении графика
    }

    paneViews() {
        return [new PointMarkersPaneView(this._data)];
    }
}

class PointMarkersPaneView {
    constructor(data) {
        this._data = data;
        this._renderer = new PointMarkersRenderer();
    }

    update() {
        this._renderer.setData(this._data);
    }

    renderer() {
        return this._renderer;
    }
}

class PointMarkersRenderer {
    constructor() {
        this._data = [];
    }

    setData(data) {
        this._data = data;
    }

    draw(target, priceConverter) {
        const ctx = target.context;
        const timeScale = target.chart.timeScale();
        
        for (const point of this._data) {
            const x = timeScale.timeToCoordinate(point.time);
            const y = priceConverter.priceToCoordinate(point.value);
            
            if (x === null || y === null) continue;
            
            // Рисуем маркер
            ctx.beginPath();
            ctx.arc(x, y, point.radius || 5, 0, Math.PI * 2);
            ctx.fillStyle = point.color || '#2196f3';
            ctx.fill();
            
            // Опционально: обводка
            if (point.borderColor) {
                ctx.strokeStyle = point.borderColor;
                ctx.lineWidth = point.borderWidth || 1;
                ctx.stroke();
            }
        }
    }
}

// Использование
const lineSeries = chart.addLineSeries({
    pointMarkersVisible: false  // Отключаем встроенные
});

const markerData = data.map(item => ({
    ...item,
    color: item.value > threshold ? '#26a69a' : '#ef5350',
    radius: 5
}));

const markersPrimitive = new PointMarkersPrimitive(markerData);
lineSeries.attachPrimitive(markersPrimitive);
```

---

### Решение 3: setMarkers() вместо pointMarkers
**Оценка: 6/10**

Использование API `setMarkers()` для отображения маркеров.

**Преимущества:**
- ✅ Официальный API
- ✅ Поддержка форм (арелУ, circle, square)
- ✅ Текстовые метки

**Недостатки:**
- ❌ Маркеры рисуются над/под свечами, не на линии
- ❌ Ограниченные позиции (aboveBar, belowBar, inBar)
- ❌ Не полностью решает задачу

```javascript
const lineSeries = chart.addLineSeries({
    pointMarkersVisible: false
});

lineSeries.setData(data);

// Создаём маркеры для каждой точки
const markers = data.map((item, index) => ({
    time: item.time,
    position: 'inBar',
    color: item.value > 100 ? '#26a69a' : '#ef5350',
    shape: 'circle',
    size: 1,
}));

lineSeries.setMarkers(markers);
```

---

### Решение 4: Единый цвет для всех точек (обход проблемы)
**Оценка: 3/10**

Отказ от per-point colors в пользу единого цвета.

**Преимущества:**
- ✅ Нулевые изменения в коде
- ✅ Маркеры работают

**Недостатки:**
- ❌ Потеря функциональности
- ❌ Невозможно подсвечивать отдельные точки
- ❌ Не решение, а отступление

```javascript
// Просто не используем per-point colors
const lineSeries = chart.addLineSeries({
    color: '#2196f3',  // Единый цвет
    pointMarkersVisible: true,
    pointMarkersRadius: 4,
});

lineSeries.setData(data.map(item => ({
    time: item.time,
    value: item.value,
    // НЕ добавляем color!
})));
```

---

## ✅ Рекомендуемое решение

### Полный рабочий пример: Overlay серия для маркеров

```javascript
import { createChart, LineStyle } from 'lightweight-charts';

// ============================================
// РЕШЕНИЕ: Colored Point Markers без бага
// ============================================

class ColoredLineWithMarkers {
    constructor(container, options = {}) {
        this.chart = createChart(container, {
            width: options.width || 800,
            height: options.height || 400,
            layout: {
                background: { type: 'solid', color: '#1e222d' },
                textColor: '#d1d4dc',
            },
            grid: {
                vertLines: { color: '#2B2B43' },
                horzLines: { color: '#2B2B43' },
            },
            crosshair: {
                mode: 0, // Normal
            },
        });

        this._colorFn = options.colorFn || (() => '#2196f3');
        this._initSeries();
    }

    _initSeries() {
        // Основная линия (без маркеров)
        this.lineSeries = this.chart.addLineSeries({
            color: '#2196f3',
            lineWidth: 2,
            pointMarkersVisible: false,  // 🔑 Отключаем встроенные маркеры
            lastValueVisible: true,
            priceLineVisible: true,
            crosshairMarkerVisible: true,
        });

        // Overlay серия для маркеров (невидимая линия)
        this.markerSeries = this.chart.addLineSeries({
            color: 'transparent',
            lineWidth: 0,                // 🔑 Линия невидима
            pointMarkersVisible: true,   // 🔑 Только маркеры
            pointMarkersRadius: 6,
            lastValueVisible: false,
            priceLineVisible: false,
            crosshairMarkerVisible: false,
        });
    }

    setData(data) {
        // Линия - чистые данные
        this.lineSeries.setData(data.map(item => ({
            time: item.time,
            value: item.value,
        })));

        // Маркеры - с цветами
        this.markerSeries.setData(data.map(item => ({
            time: item.time,
            value: item.value,
            color: this._colorFn(item),  // 🔑 Индивидуальные цвета
        })));

        return this;
    }

    updateData(newPoint) {
        this.lineSeries.update({
            time: newPoint.time,
            value: newPoint.value,
        });

        this.markerSeries.update({
            time: newPoint.time,
            value: newPoint.value,
            color: this._colorFn(newPoint),
        });

        return this;
    }

    setColorFunction(fn) {
        this._colorFn = fn;
        return this;
    }

    fitContent() {
        this.chart.timeScale().fitContent();
        return this;
    }

    destroy() {
        this.chart.remove();
    }
}

// ============================================
// ПРИМЕР ИСПОЛЬЗОВАНИЯ
// ============================================

const container = document.getElementById('chart');

// Функция определения цвета по значению
const colorByThreshold = (point) => {
    if (point.value >= 110) return '#26a69a';      // Зелёный - выше
    if (point.value <= 95) return '#ef5350';       // Красный - ниже
    return '#ffeb3b';                               // Жёлтый - средний
};

// Создаём график
const chart = new ColoredLineWithMarkers(container, {
    width: 1000,
    height: 500,
    colorFn: colorByThreshold
});

// Генерируем данные
const data = generateLineData(100, 90, 120);
chart.setData(data).fitContent();

// Real-time обновления
setInterval(() => {
    const lastTime = data[data.length - 1].time;
    const newPoint = {
        time: lastTime + 3600,
        value: 100 + (Math.random() - 0.5) * 30
    };
    data.push(newPoint);
    chart.updateData(newPoint);
}, 2000);

// ============================================
// АЛЬТЕРНАТИВА: Series Primitive (полный контроль)
// ============================================

class CustomPointMarkersPrimitive {
    constructor(options = {}) {
        this._data = [];
        this._options = {
            defaultRadius: options.radius || 5,
            defaultColor: options.color || '#2196f3',
            borderColor: options.borderColor || null,
            borderWidth: options.borderWidth || 1,
        };
        this._paneViews = [new CustomPointMarkersPaneView(this)];
    }

    updateData(data) {
        this._data = data;
        this._paneViews[0].update();
    }

    paneViews() {
        return this._paneViews;
    }

    attached({ chart, series, requestUpdate }) {
        this._chart = chart;
        this._series = series;
        this._requestUpdate = requestUpdate;
    }

    detached() {
        this._chart = null;
        this._series = null;
    }

    requestUpdate() {
        if (this._requestUpdate) {
            this._requestUpdate();
        }
    }
}

class CustomPointMarkersPaneView {
    constructor(source) {
        this._source = source;
    }

    update() {
        // Вызывается при изменении данных
    }

    renderer() {
        return {
            draw: (target) => this._draw(target),
        };
    }

    _draw(target) {
        const ctx = target.context;
        const timeScale = this._source._chart.timeScale();
        const series = this._source._series;
        const options = this._source._options;

        for (const point of this._source._data) {
            const x = timeScale.timeToCoordinate(point.time);
            const y = series.priceToCoordinate(point.value);

            if (x === null || y === null) continue;

            const radius = point.radius || options.defaultRadius;
            const color = point.color || options.defaultColor;

            // Основной круг
            ctx.beginPath();
            ctx.arc(x, y, radius, 0, Math.PI * 2);
            ctx.fillStyle = color;
            ctx.fill();

            // Обводка
            if (options.borderColor) {
                ctx.strokeStyle = options.borderColor;
                ctx.lineWidth = options.borderWidth;
                ctx.stroke();
            }
        }
    }
}

// Использование Primitive:
// const primitive = new CustomPointMarkersPrimitive({ radius: 6 });
// lineSeries.attachPrimitive(primitive);
// primitive.updateData(markerData);

// ============================================
// ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ
// ============================================

function generateLineData(count, minValue, maxValue) {
    const data = [];
    let time = Math.floor(Date.now() / 1000) - count * 3600;
    let value = (minValue + maxValue) / 2;

    for (let i = 0; i < count; i++) {
        value += (Math.random() - 0.5) * 5;
        value = Math.max(minValue, Math.min(maxValue, value));

        data.push({
            time: time + i * 3600,
            value: Math.round(value * 100) / 100,
        });
    }

    return data;
}
```

---

## 📊 Сравнительная таблица

| Решение | Оценка | Сложность | Per-point Colors | Production Ready |
|---------|--------|-----------|------------------|------------------|
| **Overlay серия** | 8/10 ⭐ | Низкая | ✅ Да | ✅ Да |
| Series Primitives | 9/10 | Высокая | ✅ Да | ✅ Да |
| setMarkers() | 6/10 | Низкая | ✅ Да | ⚠️ Ограничено |
| Единый цвет | 3/10 | Минимальная | ❌ Нет | ✅ Да |

### Рекомендации по выбору:

- **Простой случай** → Overlay серия (Решение 1)
- **Максимальный контроль** → Series Primitives (Решение 2)
- **Маркеры над/под линией** → setMarkers() (Решение 3)
- **Нет необходимости в цветах** → Единый цвет (Решение 4)

---

## 🔗 Источники

1. **GitHub Issue #2017** - [Point markers stop rendering with per-point colors](https://github.com/tradingview/lightweight-charts/issues/2017)
2. **GitHub Issue #1926** - [Related: Line color applied one bar later](https://github.com/tradingview/lightweight-charts/issues/1926)
3. **Series Primitives Documentation** - [Plugin Examples](https://tradingview.github.io/lightweight-charts/plugin-examples/)
4. **API Reference: setMarkers** - [Series Markers](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#setmarkers)
5. **Community Discussions** - Workarounds для marker rendering issues

---

## 💡 Дополнительные советы

1. **Производительность overlay:** При большом количестве точек (>10k) рассмотрите Series Primitives с виртуализацией
2. **Синхронизация:** Используйте один источник данных для обеих серий
3. **Tooltip:** При наведении на маркер overlay серии, tooltip покажет ту же цену
4. **zOrder:** В overlay серии markers будут над основной линией автоматически
5. **Обновление:** При real-time updates обновляйте обе серии одновременно
