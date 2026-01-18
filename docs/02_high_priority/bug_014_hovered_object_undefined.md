# БАГ #14: hoveredObjectId undefined для маркеров

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issues:** [#1503](https://github.com/tradingview/lightweight-charts/issues/1503), [#1513](https://github.com/tradingview/lightweight-charts/issues/1513), [#1371](https://github.com/tradingview/lightweight-charts/issues/1371), [#1270](https://github.com/tradingview/lightweight-charts/issues/1270)  
> **Версии:** Все версии с markers  
> **Статус:** 🔴 Open (множественные related issues)  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

При использовании `subscribeCrosshairMove` или `subscribeClick` свойство `hoveredObjectId` возвращает `undefined` при наведении на маркеры, вместо ожидаемого ID маркера.

### Симптомы:
- `hoveredObjectId` возвращает `undefined` при наведении на маркеры
- Непостоянное поведение — иногда работает при **медленном** движении мыши
- **Не работает** для маркеров, пересечённых price line (#1513)
- При **остановке движения мыши** ID теряется
- **Не работает** для текста маркера в `subscribeClick`
- На **мобильных устройствах** часто undefined (#1593)

### Техническая причина:
- Hit-test маркеров использует маленькую область обнаружения
- Race condition между событиями мыши и обновлением состояния
- Конфликт hit-test с другими элементами (price lines)

### Сценарии воспроизведения:
1. **Interactive tooltips:** Попытка показать tooltip при hover на маркер
2. **Click handlers:** Обработка клика по маркеру для показа деталей
3. **Highlighting:** Подсветка маркера при наведении

### Пример проблемного кода:
```javascript
const series = chart.addCandlestickSeries();
series.setData(candleData);

// Добавляем маркеры с ID
series.setMarkers([
    {
        time: '2024-01-15',
        position: 'aboveBar',
        color: '#f68e2e',
        shape: 'circle',
        id: 'signal-1',  // 🔑 ID для идентификации
        text: 'Buy Signal'
    },
    {
        time: '2024-01-20',
        position: 'belowBar',
        color: '#e91e63',
        shape: 'circle',
        id: 'signal-2',
        text: 'Sell Signal'
    }
]);

chart.subscribeCrosshairMove((param) => {
    // ❌ ПРОБЛЕМА: часто undefined даже при наведении на маркер!
    console.log('hoveredObjectId:', param.hoveredObjectId);
    
    if (param.hoveredObjectId) {
        showTooltip(param.hoveredObjectId);  // Редко срабатывает
    } else {
        hideTooltip();
    }
});
```

---

## 🔍 Найденные решения

### Решение 1: Time-based маркер lookup (по времени под курсором)
**Оценка: 8/10** ⭐ РЕКОМЕНДУЕМОЕ

Определение маркера по `param.time` вместо `hoveredObjectId`.

**Преимущества:**
- ✅ Надёжно работает — time всегда доступен
- ✅ Простая реализация
- ✅ Не зависит от hit-test багов
- ✅ Работает на мобильных устройствах

**Недостатки:**
- ⚠️ Требует предварительного индексирования маркеров
- ⚠️ Не различает маркеры на одном timestamp
- ⚠️ Срабатывает на весь bar, не только на визуальную область маркера

```javascript
// Создаём индекс маркеров по времени
const markersIndex = new Map();

const markers = [
    { time: '2024-01-15', position: 'aboveBar', color: '#f68e2e', shape: 'circle', id: 'signal-1', text: 'Buy' },
    { time: '2024-01-20', position: 'belowBar', color: '#e91e63', shape: 'circle', id: 'signal-2', text: 'Sell' },
];

// Индексируем маркеры
markers.forEach(marker => {
    const timeKey = typeof marker.time === 'object' 
        ? `${marker.time.year}-${marker.time.month}-${marker.time.day}`
        : String(marker.time);
    
    if (!markersIndex.has(timeKey)) {
        markersIndex.set(timeKey, []);
    }
    markersIndex.get(timeKey).push(marker);
});

series.setMarkers(markers);

// Обработчик hover
chart.subscribeCrosshairMove((param) => {
    if (!param.time) {
        hideTooltip();
        return;
    }
    
    // Получаем время в нужном формате
    const timeKey = typeof param.time === 'object'
        ? `${param.time.year}-${param.time.month}-${param.time.day}`
        : String(param.time);
    
    // Ищем маркеры по времени
    const markersAtTime = markersIndex.get(timeKey);
    
    if (markersAtTime && markersAtTime.length > 0) {
        // Нашли маркер(ы)!
        showTooltip(markersAtTime[0]);  // Показываем первый
    } else {
        hideTooltip();
    }
});
```

---

### Решение 2: Coordinate-based hit detection (по координатам)
**Оценка: 9/10**

Ручной hit-test по пиксельным координатам курсора и позициям маркеров.

**Преимущества:**
- ✅ Точное определение hover области
- ✅ Работает для любых overlapping элементов
- ✅ Можно настроить размер hit-области
- ✅ Независимость от встроенного hit-test

**Недостатки:**
- ⚠️ Требует вычисления координат маркеров
- ⚠️ Больше кода
- ⚠️ Нужно пересчитывать при scroll/zoom

```javascript
class MarkerHitDetector {
    constructor(chart, series, markers, options = {}) {
        this.chart = chart;
        this.series = series;
        this.markers = markers;
        this.hitRadius = options.hitRadius || 10;  // Радиус hit-области в пикселях
        
        this._markerCoords = new Map();
        this._updateCoords();
        
        // Обновляем координаты при scroll/zoom
        chart.timeScale().subscribeVisibleLogicalRangeChange(() => this._updateCoords());
    }
    
    _updateCoords() {
        this._markerCoords.clear();
        const timeScale = this.chart.timeScale();
        
        for (const marker of this.markers) {
            const x = timeScale.timeToCoordinate(marker.time);
            if (x === null) continue;
            
            // Получаем цену для позиции маркера
            const barData = this._getBarAtTime(marker.time);
            if (!barData) continue;
            
            let price;
            if (marker.position === 'aboveBar') {
                price = barData.high;
            } else if (marker.position === 'belowBar') {
                price = barData.low;
            } else {
                price = barData.close;
            }
            
            const y = this.series.priceToCoordinate(price);
            if (y === null) continue;
            
            // Корректируем Y для offset маркера
            const offsetY = marker.position === 'aboveBar' ? -15 : 15;
            
            this._markerCoords.set(marker.id, {
                marker,
                x,
                y: y + offsetY,
            });
        }
    }
    
    _getBarAtTime(time) {
        // Получаем данные бара по времени
        // Реализация зависит от структуры данных
        return this._dataMap?.get(time);
    }
    
    setData(data) {
        this._dataMap = new Map();
        for (const bar of data) {
            this._dataMap.set(bar.time, bar);
        }
        this._updateCoords();
    }
    
    hitTest(x, y) {
        for (const [id, coord] of this._markerCoords) {
            const dx = x - coord.x;
            const dy = y - coord.y;
            const distance = Math.sqrt(dx * dx + dy * dy);
            
            if (distance <= this.hitRadius) {
                return coord.marker;
            }
        }
        return null;
    }
}

// Использование
const detector = new MarkerHitDetector(chart, series, markers, { hitRadius: 15 });
detector.setData(candleData);

chart.subscribeCrosshairMove((param) => {
    if (param.point) {
        const hitMarker = detector.hitTest(param.point.x, param.point.y);
        if (hitMarker) {
            showTooltip(hitMarker);
        } else {
            hideTooltip();
        }
    }
});
```

---

### Решение 3: Custom Primitives с hitTest
**Оценка: 9/10**

Использование Series Primitives API с собственной реализацией hitTest.

**Преимущества:**
- ✅ Полный контроль над hit-test логикой
- ✅ Корректная интеграция с hoveredObjectId
- ✅ Любые формы и размеры маркеров
- ✅ Официально поддерживаемый подход

**Недостатки:**
- ⚠️ Требует реализации полного primitive
- ⚠️ Более сложная архитектура
- ⚠️ Нужно самостоятельно рисовать маркеры

```javascript
class InteractiveMarkersPrimitive {
    constructor(markers, options = {}) {
        this._markers = markers;
        this._hitRadius = options.hitRadius || 12;
        this._markerRadius = options.markerRadius || 8;
        this._coords = [];
    }

    updateAllViews() {
        this._updateCoords();
    }

    _updateCoords() {
        if (!this._chart || !this._series) return;
        
        const timeScale = this._chart.timeScale();
        this._coords = [];
        
        for (const marker of this._markers) {
            const x = timeScale.timeToCoordinate(marker.time);
            if (x === null) continue;
            
            const y = this._series.priceToCoordinate(marker.price);
            if (y === null) continue;
            
            this._coords.push({
                marker,
                x,
                y,
            });
        }
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

    // 🔑 КЛЮЧЕВОЙ МЕТОД: hitTest
    hitTest(x, y) {
        for (const coord of this._coords) {
            const dx = x - coord.x;
            const dy = y - coord.y;
            const distance = Math.sqrt(dx * dx + dy * dy);
            
            if (distance <= this._hitRadius) {
                return {
                    cursorStyle: 'pointer',
                    externalId: coord.marker.id,  // 🔑 Будет в hoveredObjectId!
                };
            }
        }
        return null;
    }

    paneViews() {
        return [new InteractiveMarkersPaneView(this)];
    }
}

class InteractiveMarkersPaneView {
    constructor(source) {
        this._source = source;
    }

    update() {
        this._source._updateCoords();
    }

    renderer() {
        return {
            draw: (target) => this._draw(target),
        };
    }

    _draw(target) {
        const ctx = target.context;
        const radius = this._source._markerRadius;
        
        for (const coord of this._source._coords) {
            const { marker, x, y } = coord;
            
            ctx.beginPath();
            ctx.arc(x, y, radius, 0, Math.PI * 2);
            ctx.fillStyle = marker.color || '#2196f3';
            ctx.fill();
            
            // Обводка
            ctx.strokeStyle = '#ffffff';
            ctx.lineWidth = 2;
            ctx.stroke();
            
            // Текст (опционально)
            if (marker.text) {
                ctx.font = '12px Arial';
                ctx.fillStyle = '#ffffff';
                ctx.textAlign = 'center';
                ctx.fillText(marker.text, x, y - radius - 5);
            }
        }
    }
}

// Использование
const markersPrimitive = new InteractiveMarkersPrimitive([
    { time: 1705320000, price: 45000, color: '#26a69a', id: 'buy-1', text: 'BUY' },
    { time: 1705406400, price: 44000, color: '#ef5350', id: 'sell-1', text: 'SELL' },
]);

series.attachPrimitive(markersPrimitive);

// Теперь hoveredObjectId будет работать!
chart.subscribeCrosshairMove((param) => {
    if (param.hoveredObjectId) {
        console.log('Hovered marker:', param.hoveredObjectId);
        showTooltip(param.hoveredObjectId);
    } else {
        hideTooltip();
    }
});
```

---

### Решение 4: Debounced hover с fallback
**Оценка: 6/10**

Использование debounce и fallback на time-based lookup.

**Преимущества:**
- ✅ Использует встроенный hoveredObjectId когда работает
- ✅ Fallback на время когда не работает
- ✅ Минимальные изменения

**Недостатки:**
- ⚠️ Не решает проблему полностью
- ⚠️ Задержка из-за debounce
- ⚠️ Непредсказуемое поведение

```javascript
function debounce(fn, delay) {
    let timeout;
    return (...args) => {
        clearTimeout(timeout);
        timeout = setTimeout(() => fn(...args), delay);
    };
}

const markersIndex = new Map();
markers.forEach(m => markersIndex.set(m.time, m));

let lastHoveredMarkerId = null;

const handleCrosshairMove = debounce((param) => {
    let foundMarker = null;
    
    // Пробуем hoveredObjectId
    if (param.hoveredObjectId) {
        foundMarker = markers.find(m => m.id === param.hoveredObjectId);
    }
    
    // Fallback на time-based lookup
    if (!foundMarker && param.time) {
        foundMarker = markersIndex.get(param.time);
    }
    
    const markerId = foundMarker?.id || null;
    
    if (markerId !== lastHoveredMarkerId) {
        lastHoveredMarkerId = markerId;
        
        if (foundMarker) {
            showTooltip(foundMarker);
        } else {
            hideTooltip();
        }
    }
}, 50);  // 50ms debounce

chart.subscribeCrosshairMove(handleCrosshairMove);
```

---

## ✅ Рекомендуемое решение

### Полный рабочий пример: Надёжное определение маркеров

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// РЕШЕНИЕ: Надёжное определение hover маркеров
// ============================================

class ReliableMarkerHover {
    constructor(chart, series, options = {}) {
        this.chart = chart;
        this.series = series;
        this.markers = [];
        this.markersIndex = new Map();  // time -> markers[]
        this.coordsCache = new Map();   // id -> {x, y, marker}
        this.hitRadius = options.hitRadius || 15;
        
        this._hoveredMarkerId = null;
        this._onMarkerHover = options.onMarkerHover || (() => {});
        this._onMarkerLeave = options.onMarkerLeave || (() => {});
        this._onMarkerClick = options.onMarkerClick || (() => {});
        
        this._dataMap = new Map();
        
        this._setupSubscriptions();
    }
    
    _setupSubscriptions() {
        // Обновляем координаты при scroll/zoom
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            this._updateCoords();
        });
        
        // Hover detection
        this.chart.subscribeCrosshairMove((param) => {
            this._handleCrosshairMove(param);
        });
        
        // Click detection
        this.chart.subscribeClick((param) => {
            this._handleClick(param);
        });
    }
    
    setData(data) {
        this._dataMap.clear();
        for (const bar of data) {
            const timeKey = this._normalizeTime(bar.time);
            this._dataMap.set(timeKey, bar);
        }
        return this;
    }
    
    setMarkers(markers) {
        this.markers = markers;
        this.markersIndex.clear();
        
        // Индексируем по времени
        for (const marker of markers) {
            const timeKey = this._normalizeTime(marker.time);
            
            if (!this.markersIndex.has(timeKey)) {
                this.markersIndex.set(timeKey, []);
            }
            this.markersIndex.get(timeKey).push(marker);
        }
        
        // Устанавливаем на серию
        this.series.setMarkers(markers);
        
        // Обновляем координаты
        this._updateCoords();
        
        return this;
    }
    
    _normalizeTime(time) {
        if (typeof time === 'object' && time !== null) {
            return `${time.year}-${String(time.month).padStart(2, '0')}-${String(time.day).padStart(2, '0')}`;
        }
        return String(time);
    }
    
    _updateCoords() {
        this.coordsCache.clear();
        const timeScale = this.chart.timeScale();
        
        for (const marker of this.markers) {
            const x = timeScale.timeToCoordinate(marker.time);
            if (x === null) continue;
            
            // Получаем данные бара
            const timeKey = this._normalizeTime(marker.time);
            const barData = this._dataMap.get(timeKey);
            if (!barData) continue;
            
            // Определяем цену в зависимости от позиции
            let price;
            switch (marker.position) {
                case 'aboveBar':
                    price = barData.high || barData.value;
                    break;
                case 'belowBar':
                    price = barData.low || barData.value;
                    break;
                case 'inBar':
                default:
                    price = barData.close || barData.value;
            }
            
            const y = this.series.priceToCoordinate(price);
            if (y === null) continue;
            
            // Offset для позиции маркера
            const offsetY = marker.position === 'aboveBar' ? -20 
                          : marker.position === 'belowBar' ? 20 
                          : 0;
            
            this.coordsCache.set(marker.id, {
                marker,
                x,
                y: y + offsetY,
            });
        }
    }
    
    _hitTest(x, y) {
        for (const [id, coord] of this.coordsCache) {
            const dx = x - coord.x;
            const dy = y - coord.y;
            const distance = Math.sqrt(dx * dx + dy * dy);
            
            if (distance <= this.hitRadius) {
                return coord.marker;
            }
        }
        return null;
    }
    
    _handleCrosshairMove(param) {
        let foundMarker = null;
        
        // Стратегия 1: Попробуем встроенный hoveredObjectId
        if (param.hoveredObjectId) {
            foundMarker = this.markers.find(m => m.id === param.hoveredObjectId);
        }
        
        // Стратегия 2: Coordinate-based hit test
        if (!foundMarker && param.point) {
            foundMarker = this._hitTest(param.point.x, param.point.y);
        }
        
        // Стратегия 3: Time-based lookup (fallback)
        if (!foundMarker && param.time) {
            const timeKey = this._normalizeTime(param.time);
            const markersAtTime = this.markersIndex.get(timeKey);
            if (markersAtTime && markersAtTime.length > 0) {
                foundMarker = markersAtTime[0];
            }
        }
        
        const newHoveredId = foundMarker?.id || null;
        
        // Если состояние изменилось
        if (newHoveredId !== this._hoveredMarkerId) {
            // Leave previous
            if (this._hoveredMarkerId) {
                const prevMarker = this.markers.find(m => m.id === this._hoveredMarkerId);
                if (prevMarker) {
                    this._onMarkerLeave(prevMarker);
                }
            }
            
            // Enter new
            this._hoveredMarkerId = newHoveredId;
            
            if (foundMarker) {
                this._onMarkerHover(foundMarker, param);
            }
        }
    }
    
    _handleClick(param) {
        let clickedMarker = null;
        
        // Те же стратегии что и для hover
        if (param.hoveredObjectId) {
            clickedMarker = this.markers.find(m => m.id === param.hoveredObjectId);
        }
        
        if (!clickedMarker && param.point) {
            clickedMarker = this._hitTest(param.point.x, param.point.y);
        }
        
        if (!clickedMarker && param.time) {
            const timeKey = this._normalizeTime(param.time);
            const markersAtTime = this.markersIndex.get(timeKey);
            if (markersAtTime && markersAtTime.length > 0) {
                clickedMarker = markersAtTime[0];
            }
        }
        
        if (clickedMarker) {
            this._onMarkerClick(clickedMarker, param);
        }
    }
    
    destroy() {
        this.markers = [];
        this.markersIndex.clear();
        this.coordsCache.clear();
    }
}

// ============================================
// ПРИМЕР ИСПОЛЬЗОВАНИЯ
// ============================================

const container = document.getElementById('chart');
const chart = createChart(container, {
    width: 1000,
    height: 500,
    layout: {
        background: { type: 'solid', color: '#1e222d' },
        textColor: '#d1d4dc',
    },
    grid: {
        vertLines: { color: '#2B2B43' },
        horzLines: { color: '#2B2B43' },
    },
});

const series = chart.addCandlestickSeries({
    upColor: '#26a69a',
    downColor: '#ef5350',
    borderVisible: false,
    wickUpColor: '#26a69a',
    wickDownColor: '#ef5350',
});

// Данные
const candleData = generateCandleData(100, 44000, 48000);
series.setData(candleData);

// Маркеры
const markers = [
    { time: candleData[20].time, position: 'belowBar', color: '#26a69a', shape: 'arrowUp', id: 'buy-1', text: 'BUY' },
    { time: candleData[40].time, position: 'aboveBar', color: '#ef5350', shape: 'arrowDown', id: 'sell-1', text: 'SELL' },
    { time: candleData[60].time, position: 'belowBar', color: '#26a69a', shape: 'arrowUp', id: 'buy-2', text: 'BUY' },
    { time: candleData[80].time, position: 'aboveBar', color: '#ef5350', shape: 'arrowDown', id: 'sell-2', text: 'SELL' },
];

// Tooltip элемент
const tooltip = document.createElement('div');
tooltip.style.cssText = `
    position: absolute;
    background: #1e222d;
    border: 1px solid #2B2B43;
    border-radius: 4px;
    padding: 8px 12px;
    color: #d1d4dc;
    font-size: 12px;
    pointer-events: none;
    z-index: 1000;
    display: none;
`;
container.appendChild(tooltip);

// Создаём hover handler
const markerHover = new ReliableMarkerHover(chart, series, {
    hitRadius: 20,
    
    onMarkerHover: (marker, param) => {
        console.log('Hover on marker:', marker.id);
        
        tooltip.innerHTML = `
            <strong>${marker.text}</strong><br>
            ID: ${marker.id}<br>
            Type: ${marker.shape}
        `;
        tooltip.style.display = 'block';
        tooltip.style.left = `${param.point.x + 10}px`;
        tooltip.style.top = `${param.point.y - 50}px`;
    },
    
    onMarkerLeave: (marker) => {
        console.log('Left marker:', marker.id);
        tooltip.style.display = 'none';
    },
    
    onMarkerClick: (marker, param) => {
        console.log('Clicked marker:', marker.id);
        alert(`Clicked: ${marker.text} (${marker.id})`);
    },
});

// Устанавливаем данные
markerHover.setData(candleData);
markerHover.setMarkers(markers);

chart.timeScale().fitContent();

// ============================================
// ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ
// ============================================

function generateCandleData(count, minPrice, maxPrice) {
    const data = [];
    let time = Math.floor(Date.now() / 1000) - count * 86400;
    let price = (minPrice + maxPrice) / 2;
    
    for (let i = 0; i < count; i++) {
        const volatility = (Math.random() - 0.5) * 500;
        const open = price;
        const close = price + volatility;
        const high = Math.max(open, close) + Math.random() * 200;
        const low = Math.min(open, close) - Math.random() * 200;
        
        data.push({
            time: time + i * 86400,
            open: Math.max(minPrice, Math.min(maxPrice, open)),
            high: Math.max(minPrice, Math.min(maxPrice, high)),
            low: Math.max(minPrice, Math.min(maxPrice, low)),
            close: Math.max(minPrice, Math.min(maxPrice, close))
        });
        
        price = close;
    }
    
    return data;
}
```

---

## 📊 Сравнительная таблица

| Решение | Оценка | Надёжность | Точность | Сложность |
|---------|--------|------------|----------|-----------|
| **Time-based lookup** | 8/10 ⭐ | ✅ Высокая | ⚠️ По бару | Низкая |
| Coordinate hit-test | 9/10 | ✅ Высокая | ✅ Точная | Средняя |
| Custom Primitives | 9/10 | ✅ Высокая | ✅ Точная | Высокая |
| Debounced fallback | 6/10 | ⚠️ Средняя | ⚠️ Средняя | Низкая |

### Рекомендации по выбору:

- **Быстрый fix** → Time-based lookup (Решение 1)
- **Точное определение** → Coordinate hit-test (Решение 2)
- **Полный контроль** → Custom Primitives (Решение 3)
- **Минимум изменений** → Debounced fallback (Решение 4)

---

## 🔗 Источники

1. **GitHub Issue #1503** - [hoveredObjectId undefined on markers](https://github.com/tradingview/lightweight-charts/issues/1503)
2. **GitHub Issue #1513** - [hoveredObjectId не работает когда маркер пересечён price line](https://github.com/tradingview/lightweight-charts/issues/1513)
3. **GitHub Issue #1371** - [Related marker interaction issues](https://github.com/tradingview/lightweight-charts/issues/1371)
4. **GitHub Issue #1270** - [Click on marker ID issues](https://github.com/tradingview/lightweight-charts/issues/1270)
5. **GitHub Issue #1523** - [hoveredObjectId сбрасывается при scroll](https://github.com/tradingview/lightweight-charts/issues/1523)
6. **API Reference: hitTest** - [ISeriesPrimitiveBase.hitTest](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesPrimitiveBase#hittest)
7. **Series Primitives Docs** - [Series Primitives Guide](https://tradingview.github.io/lightweight-charts/tutorials/how_to/series-primitives)

---

## 💡 Дополнительные советы

1. **Кэширование координат:** Обновляйте coordsCache только при scroll/zoom, не на каждое движение мыши
2. **Множественные маркеры:** При нескольких маркерах на одном баре показывайте dropdown со списком
3. **Touch устройства:** Увеличьте hitRadius до 25+ для мобильных
4. **Price lines конфликт:** При использовании price lines предпочитайте coordinate-based hit-test
5. **Z-index маркеров:** Primitive маркеры рисуются поверх встроенных — используйте для приоритета
