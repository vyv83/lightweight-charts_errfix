# БАГ #15: hoveredObjectId сбрасывается при scroll

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#1523](https://github.com/tradingview/lightweight-charts/issues/1523)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

При горизонтальной прокрутке графика свойство `hoveredObjectId` (в callback от `subscribeCrosshairMove`) сбрасывается в `undefined`, даже если курсор мыши остаётся неподвижным и визуально находится над маркером.

### Симптомы:
- `hoveredObjectId` становится `undefined` при горизонтальном scroll
- Курсор **остаётся над маркером**, но библиотека не обновляет `hoveredObjectId` на новый элемент под курсором
- В отличие от `logicalIndex` и `time`, которые обновляются при scroll, `hoveredObjectId` — НЕТ
- Приводит к "мерцанию" или исчезновению tooltips при прокрутке
- Нарушает UX интерактивных элементов

### Техническая причина:
- Scroll event не вызывает re-hit-test для маркеров
- Внутренний hover state не синхронизирован с визуальным состоянием
- `hoveredObjectId` обновляется только при **движении мыши**, а не при scroll

### Сценарии воспроизведения:
1. **Hover + Scroll:** Навести курсор на маркер → прокрутить колёсиком мыши → `hoveredObjectId` = `undefined`
2. **Sticky tooltips:** При scroll пользователь ожидает что tooltip будет следовать за маркером или обновляться
3. **Multi-chart sync:** При синхронизации scroll между графиками hover state теряется

### Пример проблемного кода:
```javascript
const series = chart.addCandlestickSeries();
series.setData(candleData);

series.setMarkers([
    {
        time: '2024-01-15',
        position: 'aboveBar',
        color: '#f68e2e',
        shape: 'circle',
        id: 'marker-1',
        text: 'Signal'
    }
]);

chart.subscribeCrosshairMove((param) => {
    console.log('hoveredObjectId:', param.hoveredObjectId);
    
    // ❌ ПРОБЛЕМА: При scroll hoveredObjectId становится undefined
    // Даже если курсор визуально остаётся над маркером
    
    // ❌ Tooltip мерцает или исчезает при прокрутке!
    if (param.hoveredObjectId) {
        showTooltip(param.hoveredObjectId);
    } else {
        hideTooltip();
    }
});

// При scroll мышью - hoveredObjectId = undefined 😢
```

---

## 🔍 Найденные решения

### Решение 1: Re-check on scroll с кэшированием позиции мыши
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

Отслеживание последней позиции мыши и повторный hit-test при scroll.

**Преимущества:**
- ✅ Полностью решает проблему
- ✅ hover state сохраняется при scroll
- ✅ Работает для всех типов маркеров
- ✅ Минимальное влияние на производительность

**Недостатки:**
- ⚠️ Требует хранения последней позиции мыши
- ⚠️ Дополнительный обработчик событий

```javascript
class ScrollAwareHoverTracker {
    constructor(chart, series, markers, options = {}) {
        this.chart = chart;
        this.series = series;
        this.markers = markers;
        this.hitRadius = options.hitRadius || 15;
        
        // Кэш последней позиции мыши
        this._lastMousePos = null;
        this._hoveredMarkerId = null;
        this._dataMap = new Map();
        this._coordsCache = new Map();
        
        // Индексируем маркеры
        this._markersIndex = new Map();
        markers.forEach(m => {
            const key = this._normalizeTime(m.time);
            if (!this._markersIndex.has(key)) {
                this._markersIndex.set(key, []);
            }
            this._markersIndex.get(key).push(m);
        });
        
        this._callbacks = {
            onHover: options.onHover || (() => {}),
            onLeave: options.onLeave || (() => {}),
        };
        
        this._setupListeners();
    }
    
    _normalizeTime(time) {
        if (typeof time === 'object' && time !== null) {
            return `${time.year}-${String(time.month).padStart(2, '0')}-${String(time.day).padStart(2, '0')}`;
        }
        return String(time);
    }
    
    setData(data) {
        this._dataMap.clear();
        data.forEach(bar => {
            this._dataMap.set(this._normalizeTime(bar.time), bar);
        });
        this._updateCoords();
    }
    
    _updateCoords() {
        this._coordsCache.clear();
        const timeScale = this.chart.timeScale();
        
        for (const marker of this.markers) {
            const x = timeScale.timeToCoordinate(marker.time);
            if (x === null) continue;
            
            const timeKey = this._normalizeTime(marker.time);
            const barData = this._dataMap.get(timeKey);
            if (!barData) continue;
            
            let price;
            switch (marker.position) {
                case 'aboveBar':
                    price = barData.high || barData.value;
                    break;
                case 'belowBar':
                    price = barData.low || barData.value;
                    break;
                default:
                    price = barData.close || barData.value;
            }
            
            const y = this.series.priceToCoordinate(price);
            if (y === null) continue;
            
            const offsetY = marker.position === 'aboveBar' ? -20 
                          : marker.position === 'belowBar' ? 20 : 0;
            
            this._coordsCache.set(marker.id, {
                marker,
                x,
                y: y + offsetY
            });
        }
    }
    
    _hitTest(x, y) {
        if (x === null || y === null) return null;
        
        for (const [id, coord] of this._coordsCache) {
            const dx = x - coord.x;
            const dy = y - coord.y;
            if (Math.sqrt(dx * dx + dy * dy) <= this.hitRadius) {
                return coord.marker;
            }
        }
        return null;
    }
    
    _setupListeners() {
        // Основной обработчик crosshair
        this.chart.subscribeCrosshairMove((param) => {
            if (param.point) {
                this._lastMousePos = { ...param.point };
            }
            this._checkHover(param.point);
        });
        
        // 🔑 КЛЮЧЕВОЕ: обновляем при scroll!
        this.chart.timeScale().subscribeVisibleTimeRangeChange(() => {
            this._updateCoords();
            
            // Re-check hover с кэшированной позицией мыши
            if (this._lastMousePos) {
                this._checkHover(this._lastMousePos);
            }
        });
        
        // Резервный вариант: subscribeVisibleLogicalRangeChange
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            this._updateCoords();
            
            if (this._lastMousePos) {
                // Небольшая задержка для стабильности
                requestAnimationFrame(() => {
                    this._checkHover(this._lastMousePos);
                });
            }
        });
    }
    
    _checkHover(point) {
        if (!point) {
            if (this._hoveredMarkerId) {
                const prev = this.markers.find(m => m.id === this._hoveredMarkerId);
                if (prev) this._callbacks.onLeave(prev);
                this._hoveredMarkerId = null;
            }
            return;
        }
        
        const hitMarker = this._hitTest(point.x, point.y);
        const newId = hitMarker?.id || null;
        
        if (newId !== this._hoveredMarkerId) {
            // Leave old
            if (this._hoveredMarkerId) {
                const prev = this.markers.find(m => m.id === this._hoveredMarkerId);
                if (prev) this._callbacks.onLeave(prev);
            }
            
            // Enter new
            this._hoveredMarkerId = newId;
            if (hitMarker) {
                this._callbacks.onHover(hitMarker, point);
            }
        }
    }
    
    destroy() {
        this._lastMousePos = null;
        this._hoveredMarkerId = null;
    }
}

// Использование
const tracker = new ScrollAwareHoverTracker(chart, series, markers, {
    hitRadius: 20,
    onHover: (marker, point) => {
        showTooltip(marker, point);
    },
    onLeave: (marker) => {
        hideTooltip();
    }
});

tracker.setData(candleData);
series.setMarkers(markers);
```

---

### Решение 2: Custom Primitive с независимым hover tracking
**Оценка: 8/10**

Использование Series Primitives API с собственным управлением hover state.

**Преимущества:**
- ✅ Полная независимость от встроенного hover
- ✅ Корректная работа hitTest возвращает externalId
- ✅ Официально поддерживаемый способ

**Недостатки:**
- ⚠️ Требует полной реализации primitive
- ⚠️ Больше boilerplate кода
- ⚠️ Нужно самому рисовать маркеры

```javascript
class ScrollSafeMarkersPrimitive {
    constructor(markers, options = {}) {
        this._markers = markers;
        this._options = {
            hitRadius: options.hitRadius || 15,
            markerRadius: options.markerRadius || 8
        };
        this._coords = [];
        this._chart = null;
        this._series = null;
        this._requestUpdate = null;
        
        // Кэширование позиции мыши
        this._lastMouseX = null;
        this._lastMouseY = null;
        this._lastHoveredId = null;
    }
    
    attached({ chart, series, requestUpdate }) {
        this._chart = chart;
        this._series = series;
        this._requestUpdate = requestUpdate;
        
        // Подписываемся на scroll для обновления координат
        this._scrollHandler = () => {
            this._updateCoords();
            
            // 🔑 Проверяем hover после scroll
            if (this._lastMouseX !== null && this._lastMouseY !== null) {
                requestAnimationFrame(() => requestUpdate());
            }
        };
        
        chart.timeScale().subscribeVisibleLogicalRangeChange(this._scrollHandler);
    }
    
    detached() {
        if (this._chart) {
            this._chart.timeScale().unsubscribeVisibleLogicalRangeChange(this._scrollHandler);
        }
        this._chart = null;
        this._series = null;
    }
    
    updateAllViews() {
        this._updateCoords();
    }
    
    _updateCoords() {
        if (!this._chart || !this._series) return;
        
        this._coords = [];
        const timeScale = this._chart.timeScale();
        
        for (const marker of this._markers) {
            const x = timeScale.timeToCoordinate(marker.time);
            if (x === null) continue;
            
            const y = this._series.priceToCoordinate(marker.price);
            if (y === null) continue;
            
            this._coords.push({ marker, x, y });
        }
    }
    
    // 🔑 Hit test - вызывается при каждом движении мыши
    hitTest(x, y) {
        // Сохраняем позицию мыши для re-check при scroll
        this._lastMouseX = x;
        this._lastMouseY = y;
        
        for (const coord of this._coords) {
            const dx = x - coord.x;
            const dy = y - coord.y;
            if (Math.sqrt(dx * dx + dy * dy) <= this._options.hitRadius) {
                this._lastHoveredId = coord.marker.id;
                return {
                    cursorStyle: 'pointer',
                    externalId: coord.marker.id
                };
            }
        }
        
        this._lastHoveredId = null;
        return null;
    }
    
    paneViews() {
        return [new ScrollSafeMarkersPaneView(this)];
    }
    
    // Публичный метод для получения текущего hovered маркера
    getHoveredMarker() {
        if (!this._lastHoveredId) return null;
        return this._markers.find(m => m.id === this._lastHoveredId);
    }
}

class ScrollSafeMarkersPaneView {
    constructor(source) {
        this._source = source;
    }
    
    update() {
        this._source._updateCoords();
    }
    
    renderer() {
        return {
            draw: (target) => this._draw(target)
        };
    }
    
    _draw(target) {
        const ctx = target.context;
        const radius = this._source._options.markerRadius;
        
        for (const coord of this._source._coords) {
            const { marker, x, y } = coord;
            const isHovered = marker.id === this._source._lastHoveredId;
            
            ctx.beginPath();
            ctx.arc(x, y, isHovered ? radius * 1.3 : radius, 0, Math.PI * 2);
            ctx.fillStyle = marker.color || '#2196f3';
            ctx.fill();
            
            ctx.strokeStyle = isHovered ? '#ffffff' : 'rgba(255,255,255,0.5)';
            ctx.lineWidth = isHovered ? 3 : 1;
            ctx.stroke();
        }
    }
}

// Использование
const primitive = new ScrollSafeMarkersPrimitive([
    { time: 1705320000, price: 45000, color: '#26a69a', id: 'm1' },
    { time: 1705406400, price: 44500, color: '#ef5350', id: 'm2' }
], { hitRadius: 20 });

series.attachPrimitive(primitive);

// Теперь hoveredObjectId будет работать при scroll!
chart.subscribeCrosshairMove((param) => {
    // externalId от hitTest будет в hoveredObjectId
    if (param.hoveredObjectId) {
        showTooltip(param.hoveredObjectId);
    } else {
        hideTooltip();
    }
});
```

---

### Решение 3: Debounced tooltip с sticky behavior
**Оценка: 7/10**

Tooltip не скрывается мгновенно при scroll, а использует timeout.

**Преимущества:**
- ✅ Простая реализация
- ✅ Визуально сглаживает проблему
- ✅ Minimal code changes

**Недостатки:**
- ⚠️ Не решает проблему полностью
- ⚠️ Может показывать неактуальную информацию
- ⚠️ UX compromise

```javascript
class StickyTooltip {
    constructor(container, options = {}) {
        this.container = container;
        this.hideDelay = options.hideDelay || 200; // ms
        this.element = this._createTooltipElement();
        this._hideTimeout = null;
        this._currentMarkerId = null;
    }
    
    _createTooltipElement() {
        const el = document.createElement('div');
        el.className = 'marker-tooltip';
        el.style.cssText = `
            position: absolute;
            background: rgba(30, 34, 45, 0.95);
            border: 1px solid #2B2B43;
            border-radius: 6px;
            padding: 10px 14px;
            color: #d1d4dc;
            font-size: 13px;
            pointer-events: none;
            z-index: 1000;
            display: none;
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            transition: opacity 0.15s ease;
        `;
        this.container.appendChild(el);
        return el;
    }
    
    show(marker, point) {
        // Отменяем отложенное скрытие
        if (this._hideTimeout) {
            clearTimeout(this._hideTimeout);
            this._hideTimeout = null;
        }
        
        this._currentMarkerId = marker.id;
        
        this.element.innerHTML = `
            <strong>${marker.text || marker.id}</strong><br>
            <span style="color: ${marker.color}">●</span> ${marker.shape}
        `;
        this.element.style.display = 'block';
        this.element.style.opacity = '1';
        this.element.style.left = `${point.x + 15}px`;
        this.element.style.top = `${point.y - 50}px`;
    }
    
    hide() {
        // 🔑 Отложенное скрытие — даёт время на scroll
        if (this._hideTimeout) return;
        
        this._hideTimeout = setTimeout(() => {
            this.element.style.opacity = '0';
            setTimeout(() => {
                this.element.style.display = 'none';
                this._currentMarkerId = null;
            }, 150);
            this._hideTimeout = null;
        }, this.hideDelay);
    }
    
    // Вернуть true если tooltip для этого маркера уже показан
    isShowing(markerId) {
        return this._currentMarkerId === markerId;
    }
}

// Использование
const tooltip = new StickyTooltip(chartContainer, { hideDelay: 300 });

chart.subscribeCrosshairMove((param) => {
    if (param.hoveredObjectId) {
        const marker = markers.find(m => m.id === param.hoveredObjectId);
        if (marker && param.point) {
            tooltip.show(marker, param.point);
        }
    } else {
        // Скроется с задержкой, давая время на scroll
        tooltip.hide();
    }
});
```

---

### Решение 4: Wheel event interception
**Оценка: 6/10**

Перехват wheel event для сохранения hover state.

**Преимущества:**
- ✅ Прямой контроль над scroll поведением
- ✅ Можно блокировать hide при scroll

**Недостатки:**
- ⚠️ Требует правильного timing
- ⚠️ Сложно синхронизировать с библиотекой
- ⚠️ Может конфликтовать с другими обработчиками

```javascript
class WheelAwareHover {
    constructor(chart, container) {
        this.chart = chart;
        this.container = container;
        this.isScrolling = false;
        this.scrollTimeout = null;
        this._lastHoveredId = null;
        
        // Перехватываем wheel события
        container.addEventListener('wheel', this._onWheel.bind(this), { passive: true });
    }
    
    _onWheel(e) {
        this.isScrolling = true;
        
        if (this.scrollTimeout) {
            clearTimeout(this.scrollTimeout);
        }
        
        // Считаем scroll завершённым через 100ms без событий
        this.scrollTimeout = setTimeout(() => {
            this.isScrolling = false;
        }, 100);
    }
    
    // Использовать в subscribeCrosshairMove
    shouldHide(hoveredObjectId) {
        if (hoveredObjectId) {
            this._lastHoveredId = hoveredObjectId;
            return false;
        }
        
        // Если scrolling — сохраняем предыдущий hover
        if (this.isScrolling && this._lastHoveredId) {
            return false;  // Не скрываем
        }
        
        this._lastHoveredId = null;
        return true;  // Можно скрыть
    }
    
    getLastHoveredId() {
        return this._lastHoveredId;
    }
}

// Использование
const wheelHover = new WheelAwareHover(chart, chartContainer);

chart.subscribeCrosshairMove((param) => {
    if (param.hoveredObjectId) {
        showTooltip(param.hoveredObjectId, param.point);
    } else if (wheelHover.shouldHide(param.hoveredObjectId)) {
        hideTooltip();
    }
    // Иначе — сохраняем tooltip видимым
});
```

---

## ✅ Рекомендуемое решение

### Полный рабочий пример: Scroll-safe hover tracking

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// РЕШЕНИЕ: Полноценный Scroll-safe hover
// ============================================

class ScrollSafeHoverManager {
    constructor(chart, series, options = {}) {
        this.chart = chart;
        this.series = series;
        this.markers = [];
        this.hitRadius = options.hitRadius || 18;
        
        // State
        this._lastMousePos = null;
        this._hoveredMarker = null;
        this._markersIndex = new Map();
        this._coordsCache = new Map();
        this._dataMap = new Map();
        
        // Callbacks
        this._onHover = options.onHover || (() => {});
        this._onLeave = options.onLeave || (() => {});
        this._onClick = options.onClick || (() => {});
        
        // Debounce для scroll
        this._scrollDebounce = null;
        
        this._init();
    }
    
    _init() {
        // Track mouse position
        this.chart.subscribeCrosshairMove((param) => {
            if (param.point) {
                this._lastMousePos = { x: param.point.x, y: param.point.y };
                this._checkHover();
            } else {
                this._lastMousePos = null;
                this._updateHoverState(null);
            }
        });
        
        // Track clicks
        this.chart.subscribeClick((param) => {
            if (this._hoveredMarker) {
                this._onClick(this._hoveredMarker, param.point);
            }
        });
        
        // 🔑 Re-check on scroll
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            // Debounce для производительности
            if (this._scrollDebounce) {
                cancelAnimationFrame(this._scrollDebounce);
            }
            
            this._scrollDebounce = requestAnimationFrame(() => {
                this._updateCoords();
                if (this._lastMousePos) {
                    this._checkHover();
                }
            });
        });
    }
    
    setData(data) {
        this._dataMap.clear();
        for (const bar of data) {
            const key = this._timeKey(bar.time);
            this._dataMap.set(key, bar);
        }
        this._updateCoords();
        return this;
    }
    
    setMarkers(markers) {
        this.markers = markers;
        
        // Index by time
        this._markersIndex.clear();
        for (const m of markers) {
            const key = this._timeKey(m.time);
            if (!this._markersIndex.has(key)) {
                this._markersIndex.set(key, []);
            }
            this._markersIndex.get(key).push(m);
        }
        
        // Apply to series
        this.series.setMarkers(markers);
        
        this._updateCoords();
        return this;
    }
    
    _timeKey(time) {
        if (typeof time === 'object' && time !== null) {
            const y = time.year;
            const m = String(time.month).padStart(2, '0');
            const d = String(time.day).padStart(2, '0');
            return `${y}-${m}-${d}`;
        }
        return String(time);
    }
    
    _updateCoords() {
        this._coordsCache.clear();
        const ts = this.chart.timeScale();
        
        for (const marker of this.markers) {
            const x = ts.timeToCoordinate(marker.time);
            if (x === null) continue;
            
            const key = this._timeKey(marker.time);
            const bar = this._dataMap.get(key);
            if (!bar) continue;
            
            let price;
            switch (marker.position) {
                case 'aboveBar': price = bar.high ?? bar.value; break;
                case 'belowBar': price = bar.low ?? bar.value; break;
                default: price = bar.close ?? bar.value;
            }
            
            const y = this.series.priceToCoordinate(price);
            if (y === null) continue;
            
            const offsetY = marker.position === 'aboveBar' ? -22 
                          : marker.position === 'belowBar' ? 22 : 0;
            
            this._coordsCache.set(marker.id, {
                marker,
                x,
                y: y + offsetY
            });
        }
    }
    
    _checkHover() {
        if (!this._lastMousePos) {
            this._updateHoverState(null);
            return;
        }
        
        const { x, y } = this._lastMousePos;
        let found = null;
        
        // Coordinate-based hit test
        for (const [id, coord] of this._coordsCache) {
            const dx = x - coord.x;
            const dy = y - coord.y;
            if (Math.sqrt(dx * dx + dy * dy) <= this.hitRadius) {
                found = coord.marker;
                break;
            }
        }
        
        this._updateHoverState(found);
    }
    
    _updateHoverState(marker) {
        const newId = marker?.id ?? null;
        const oldId = this._hoveredMarker?.id ?? null;
        
        if (newId === oldId) return;
        
        // Leave
        if (this._hoveredMarker) {
            this._onLeave(this._hoveredMarker);
        }
        
        // Enter
        this._hoveredMarker = marker;
        if (marker) {
            this._onHover(marker, this._lastMousePos);
        }
    }
    
    destroy() {
        if (this._scrollDebounce) {
            cancelAnimationFrame(this._scrollDebounce);
        }
        this._lastMousePos = null;
        this._hoveredMarker = null;
    }
}

// ============================================
// ПРИМЕР ИСПОЛЬЗОВАНИЯ
// ============================================

const container = document.getElementById('chart');
container.style.position = 'relative';

const chart = createChart(container, {
    width: 1000,
    height: 500,
    layout: {
        background: { type: 'solid', color: '#131722' },
        textColor: '#d1d4dc',
    },
    grid: {
        vertLines: { color: '#1f2937' },
        horzLines: { color: '#1f2937' },
    },
    crosshair: {
        mode: 0, // Normal mode
    },
    handleScroll: true,
    handleScale: true,
});

const series = chart.addCandlestickSeries({
    upColor: '#22c55e',
    downColor: '#ef4444',
    borderVisible: false,
    wickUpColor: '#22c55e',
    wickDownColor: '#ef4444',
});

// Generate sample data
const candleData = generateCandleData(150);
series.setData(candleData);

// Create markers
const markers = [
    { 
        time: candleData[30].time, 
        position: 'belowBar', 
        color: '#22c55e', 
        shape: 'arrowUp', 
        id: 'buy-1', 
        text: 'Long Entry' 
    },
    { 
        time: candleData[50].time, 
        position: 'aboveBar', 
        color: '#ef4444', 
        shape: 'arrowDown', 
        id: 'sell-1', 
        text: 'Exit' 
    },
    { 
        time: candleData[80].time, 
        position: 'belowBar', 
        color: '#22c55e', 
        shape: 'arrowUp', 
        id: 'buy-2', 
        text: 'Long Entry' 
    },
    { 
        time: candleData[110].time, 
        position: 'aboveBar', 
        color: '#ef4444', 
        shape: 'arrowDown', 
        id: 'sell-2', 
        text: 'Exit' 
    },
];

// Tooltip element
const tooltip = document.createElement('div');
tooltip.style.cssText = `
    position: absolute;
    background: rgba(19, 23, 34, 0.95);
    border: 1px solid #3b82f6;
    border-radius: 8px;
    padding: 12px 16px;
    color: #e5e7eb;
    font-size: 13px;
    font-family: -apple-system, BlinkMacSystemFont, sans-serif;
    pointer-events: none;
    z-index: 1000;
    display: none;
    box-shadow: 0 8px 24px rgba(0,0,0,0.4);
    min-width: 140px;
`;
container.appendChild(tooltip);

// Create scroll-safe hover manager
const hoverManager = new ScrollSafeHoverManager(chart, series, {
    hitRadius: 22,
    
    onHover: (marker, point) => {
        console.log('✅ Hover:', marker.id);
        
        tooltip.innerHTML = `
            <div style="font-weight: 600; margin-bottom: 6px; color: ${marker.color}">
                ${marker.text}
            </div>
            <div style="color: #9ca3af; font-size: 11px;">
                ID: ${marker.id}<br>
                Signal: ${marker.shape === 'arrowUp' ? 'BUY' : 'SELL'}
            </div>
        `;
        
        tooltip.style.display = 'block';
        tooltip.style.left = `${Math.min(point.x + 15, container.clientWidth - 180)}px`;
        tooltip.style.top = `${Math.max(point.y - 80, 10)}px`;
    },
    
    onLeave: (marker) => {
        console.log('❌ Leave:', marker.id);
        tooltip.style.display = 'none';
    },
    
    onClick: (marker, point) => {
        console.log('🖱️ Click:', marker.id);
        alert(`Clicked signal: ${marker.text}`);
    },
});

// Initialize
hoverManager.setData(candleData);
hoverManager.setMarkers(markers);

chart.timeScale().fitContent();

// ============================================
// HELPER FUNCTION
// ============================================

function generateCandleData(count) {
    const data = [];
    let baseTime = Math.floor(Date.now() / 1000) - count * 3600;
    let price = 45000;
    
    for (let i = 0; i < count; i++) {
        const volatility = (Math.random() - 0.5) * 800;
        const open = price;
        const close = price + volatility;
        const high = Math.max(open, close) + Math.random() * 300;
        const low = Math.min(open, close) - Math.random() * 300;
        
        data.push({
            time: baseTime + i * 3600,
            open: Math.round(open * 100) / 100,
            high: Math.round(high * 100) / 100,
            low: Math.round(low * 100) / 100,
            close: Math.round(close * 100) / 100,
        });
        
        price = close;
    }
    
    return data;
}
```

---

## 📊 Сравнительная таблица

| Решение | Оценка | Надёжность при scroll | Сложность | Производительность |
|---------|--------|----------------------|-----------|-------------------|
| **Re-check on scroll** | 9/10 ⭐ | ✅ Полная | Средняя | ✅ Отличная |
| **Custom Primitive** | 8/10 | ✅ Полная | Высокая | ✅ Отличная |
| **Sticky tooltip** | 7/10 | ⚠️ Частичная | Низкая | ✅ Отличная |
| **Wheel interception** | 6/10 | ⚠️ Частичная | Средняя | ⚠️ Хорошая |

---

## 🔗 Источники

1. **GitHub Issue #1523** — [hoveredObjectId is not updated when chart is scrolled](https://github.com/tradingview/lightweight-charts/issues/1523)
2. **GitHub Issue #1608** — [Glitches when syncing crosshair during scroll](https://github.com/tradingview/lightweight-charts/issues/1608)
3. **Related Issue #1503** — [hoveredObjectId undefined for markers](https://github.com/tradingview/lightweight-charts/issues/1503)
4. **Lightweight Charts Docs** — [subscribeCrosshairMove](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#subscribecrosshairMove)
5. **Lightweight Charts Docs** — [subscribeVisibleLogicalRangeChange](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ITimeScaleApi#subscribeVisibleLogicalRangeChange)
6. **Plugins Documentation** — [Series Primitives](https://tradingview.github.io/lightweight-charts/docs/plugins/series-primitives)
7. **Stack Overflow** — Canvas hit testing patterns
8. **Chart.js External Tooltip** — [Best practices for scroll-aware tooltips](https://www.chartjs.org/docs/latest/configuration/tooltip.html#external-custom-tooltips)

---

## 🔄 Связанные баги

- [БАГ #14: hoveredObjectId undefined для маркеров](./bug_014_hovered_object_undefined.md) — похожая проблема, но без scroll
- [БАГ #16: Мерцание при синхронизации crosshair](./bug_016_crosshair_sync_glitch.md) — scroll вызывает glitches в multi-chart

---

*Документ создан: 2026-01-18*  
*Последнее обновление: 2026-01-18*
