# БАГ #16: Мерцание при синхронизации crosshair между графиками

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#1608](https://github.com/tradingview/lightweight-charts/issues/1608)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

При использовании `setCrosshairPosition()` для синхронизации кроссхэра между несколькими графиками возникают визуальные glitches — мерцание, рывки или смещение кроссхэра при scroll.

### Симптомы:
- **Jitter/Glitches:** Кроссхэр дёргается или мерцает при движении мыши
- **Drift при scroll:** При прокрутке вертикальная линия кроссхэра отклоняется от позиции мыши
- **Infinite loop:** При неправильной реализации — бесконечный цикл синхронизации
- **sourceEvent undefined:** `setCrosshairPosition()` вызывается с `event.sourceEvent === undefined` во время scroll
- **Рывки:** Crosshair прыгает между позициями

### Техническая причина:
- Scroll события вызывают `subscribeCrosshairMove` с `sourceEvent = undefined`
- Программный вызов `setCrosshairPosition()` триггерит повторный `crosshairMove` event
- Race condition между событиями разных графиков
- Отсутствие throttling приводит к лавине обновлений

### Сценарии воспроизведения:
1. **Multi-chart layout:** Два или более графиков с синхронизированным кроссхэром
2. **Scroll + hover:** Навести курсор на график → начать прокрутку колесом мыши
3. **Fast mouse movement:** Быстрое перемещение мыши между графиками

### Пример проблемного кода:
```javascript
const chart1 = createChart(container1);
const chart2 = createChart(container2);

const series1 = chart1.addCandlestickSeries();
const series2 = chart2.addCandlestickSeries();

// ❌ ПРОБЛЕМНЫЙ КОД — приводит к glitches!
chart1.subscribeCrosshairMove((param) => {
    if (param.time) {
        const data = param.seriesData.get(series1);
        // Во время scroll sourceEvent = undefined → drift
        chart2.setCrosshairPosition(data?.close, param.time, series2);
    }
});

chart2.subscribeCrosshairMove((param) => {
    if (param.time) {
        const data = param.seriesData.get(series2);
        // ❌ БЕЗ ФЛАГА = потенциальный infinite loop!
        chart1.setCrosshairPosition(data?.close, param.time, series1);
    }
});

// При scroll:
// 1. Crosshair начинает дрейфовать от позиции мыши
// 2. Мерцание между графиками
// 3. Возможный infinite loop
```

---

## 🔍 Найденные решения

### Решение 1: Check sourceEvent + syncing flag
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

Проверка `sourceEvent` и использование флага для предотвращения infinite loop.

**Преимущества:**
- ✅ Решает проблему drift при scroll
- ✅ Предотвращает infinite loop
- ✅ Официально рекомендуемый подход
- ✅ Минимальные изменения кода

**Недостатки:**
- ⚠️ Crosshair не синхронизируется во время scroll (by design)
- ⚠️ Нужен аккуратный lifecycle management

```javascript
class CrosshairSynchronizer {
    constructor(charts) {
        this.charts = charts;     // Array of { chart, series }
        this.isSyncing = false;   // Флаг для предотвращения loop
        
        this._setupSync();
    }
    
    _setupSync() {
        this.charts.forEach(({ chart, series }, index) => {
            chart.subscribeCrosshairMove((param) => {
                // 🔑 КЛЮЧЕВОЕ: Проверяем sourceEvent
                // undefined означает что это scroll, а не движение мыши
                if (!param.sourceEvent) {
                    return; // Игнорируем scroll события
                }
                
                // 🔑 Проверяем флаг синхронизации
                if (this.isSyncing) {
                    return; // Уже синхронизируем, выходим
                }
                
                this._syncCrosshair(param, series, index);
            });
        });
    }
    
    _syncCrosshair(param, sourceSeries, sourceIndex) {
        // Устанавливаем флаг
        this.isSyncing = true;
        
        try {
            this.charts.forEach(({ chart, series }, index) => {
                if (index === sourceIndex) return; // Пропускаем источник
                
                if (param.point && param.time) {
                    const data = param.seriesData?.get(sourceSeries);
                    const price = data?.close ?? data?.value;
                    
                    if (price !== undefined) {
                        chart.setCrosshairPosition(price, param.time, series);
                    }
                } else {
                    chart.clearCrosshairPosition();
                }
            });
        } finally {
            // Сбрасываем флаг в следующем tick
            requestAnimationFrame(() => {
                this.isSyncing = false;
            });
        }
    }
    
    destroy() {
        this.isSyncing = false;
        this.charts = [];
    }
}

// Использование
const charts = [
    { chart: chart1, series: series1 },
    { chart: chart2, series: series2 },
];

const synchronizer = new CrosshairSynchronizer(charts);
```

---

### Решение 2: Throttle + sourceEvent check
**Оценка: 8/10**

Throttling обновлений кроссхэра для предотвращения jitter.

**Преимущества:**
- ✅ Сглаживает jitter при быстром движении мыши
- ✅ Снижает нагрузку на рендеринг
- ✅ Работает вместе с sourceEvent check

**Недостатки:**
- ⚠️ Небольшая задержка синхронизации (by design)
- ⚠️ Нужен баланс между плавностью и отзывчивостью

```javascript
function throttle(fn, delay) {
    let lastCall = 0;
    let scheduled = false;
    let lastArgs = null;
    
    return function(...args) {
        const now = Date.now();
        lastArgs = args;
        
        if (now - lastCall >= delay) {
            fn.apply(this, args);
            lastCall = now;
        } else if (!scheduled) {
            scheduled = true;
            setTimeout(() => {
                fn.apply(this, lastArgs);
                lastCall = Date.now();
                scheduled = false;
            }, delay - (now - lastCall));
        }
    };
}

class ThrottledCrosshairSync {
    constructor(charts, options = {}) {
        this.charts = charts;
        this.throttleMs = options.throttleMs || 16; // ~60fps
        this.isSyncing = false;
        
        this._syncHandler = throttle(
            this._doSync.bind(this), 
            this.throttleMs
        );
        
        this._setupSync();
    }
    
    _setupSync() {
        this.charts.forEach(({ chart, series }, index) => {
            chart.subscribeCrosshairMove((param) => {
                // Игнорируем scroll и уже синхронизируемые события
                if (!param.sourceEvent || this.isSyncing) return;
                
                this._syncHandler(param, series, index);
            });
        });
    }
    
    _doSync(param, sourceSeries, sourceIndex) {
        this.isSyncing = true;
        
        this.charts.forEach(({ chart, series }, index) => {
            if (index === sourceIndex) return;
            
            if (param.point && param.time) {
                const data = param.seriesData?.get(sourceSeries);
                const price = data?.close ?? data?.value;
                
                if (price !== undefined) {
                    chart.setCrosshairPosition(price, param.time, series);
                }
            } else {
                chart.clearCrosshairPosition();
            }
        });
        
        // Сбрасываем флаг после обработки
        requestAnimationFrame(() => {
            this.isSyncing = false;
        });
    }
}

// Использование
const sync = new ThrottledCrosshairSync([
    { chart: chart1, series: series1 },
    { chart: chart2, series: series2 },
], { throttleMs: 16 });
```

---

### Решение 3: Event source tracking (полный контроль)
**Оценка: 9/10**

Отслеживание источника события для полного контроля над синхронизацией.

**Преимущества:**
- ✅ Полный контроль над flow событий
- ✅ Поддержка любого количества графиков
- ✅ Предотвращает все edge cases

**Недостатки:**
- ⚠️ Больший объём кода
- ⚠️ Требует централизованного управления

```javascript
class ChartSyncManager {
    constructor() {
        this.charts = new Map();  // id -> { chart, series, subscribed }
        this.activeSource = null;
        this.lastPosition = null;
    }
    
    register(id, chart, series) {
        this.charts.set(id, { chart, series, subscribed: false });
        this._subscribe(id);
        return this;
    }
    
    unregister(id) {
        this.charts.delete(id);
        return this;
    }
    
    _subscribe(id) {
        const entry = this.charts.get(id);
        if (!entry || entry.subscribed) return;
        
        entry.chart.subscribeCrosshairMove((param) => {
            this._handleCrosshairMove(id, param, entry.series);
        });
        
        entry.subscribed = true;
    }
    
    _handleCrosshairMove(sourceId, param, sourceSeries) {
        // 🔑 Игнорируем scroll события
        if (!param.sourceEvent) return;
        
        // 🔑 Если мы уже обрабатываем событие от другого источника — выходим
        if (this.activeSource !== null && this.activeSource !== sourceId) {
            return;
        }
        
        // Начинаем синхронизацию
        this.activeSource = sourceId;
        
        if (!param.point || !param.time) {
            this._clearAll(sourceId);
            this._releaseSource();
            return;
        }
        
        // Получаем данные
        const data = param.seriesData?.get(sourceSeries);
        const price = data?.close ?? data?.value ?? null;
        
        if (price === null) {
            this._releaseSource();
            return;
        }
        
        // Сохраняем для переиспользования
        this.lastPosition = { time: param.time, price };
        
        // Синхронизируем остальные графики
        this.charts.forEach((entry, chartId) => {
            if (chartId === sourceId) return;
            
            entry.chart.setCrosshairPosition(price, param.time, entry.series);
        });
        
        // Освобождаем в следующем frame
        requestAnimationFrame(() => this._releaseSource());
    }
    
    _clearAll(exceptId) {
        this.charts.forEach((entry, chartId) => {
            if (chartId !== exceptId) {
                entry.chart.clearCrosshairPosition();
            }
        });
    }
    
    _releaseSource() {
        this.activeSource = null;
    }
    
    // Принудительно установить crosshair на всех графиках
    syncTo(time, price) {
        if (this.activeSource !== null) return;
        
        this.charts.forEach((entry) => {
            entry.chart.setCrosshairPosition(price, time, entry.series);
        });
    }
    
    clearAll() {
        this.charts.forEach((entry) => {
            entry.chart.clearCrosshairPosition();
        });
        this.lastPosition = null;
    }
}

// Использование
const syncManager = new ChartSyncManager();

syncManager
    .register('main', chart1, series1)
    .register('volume', chart2, volumeSeries)
    .register('indicators', chart3, indicatorSeries);
```

---

### Решение 4: RAF-based sync (оптимальная производительность)
**Оценка: 8/10**

Использование requestAnimationFrame для батчевой синхронизации.

**Преимущества:**
- ✅ Максимальная производительность
- ✅ Автоматическое выравнивание с render cycle
- ✅ Предотвращает перерисовки

**Недостатки:**
- ⚠️ Сложнее дебажить
- ⚠️ Небольшая inherent задержка

```javascript
class RAFCrosshairSync {
    constructor(charts) {
        this.charts = charts;
        this.pendingSync = null;
        this.rafId = null;
        this.isSyncing = false;
        
        this._setupSync();
    }
    
    _setupSync() {
        this.charts.forEach(({ chart, series }, index) => {
            chart.subscribeCrosshairMove((param) => {
                if (!param.sourceEvent || this.isSyncing) return;
                
                // Сохраняем pendingSync и планируем RAF
                this.pendingSync = { param, series, sourceIndex: index };
                this._scheduleSync();
            });
        });
    }
    
    _scheduleSync() {
        if (this.rafId !== null) return; // Уже запланировано
        
        this.rafId = requestAnimationFrame(() => {
            this.rafId = null;
            this._performSync();
        });
    }
    
    _performSync() {
        if (!this.pendingSync) return;
        
        const { param, series: sourceSeries, sourceIndex } = this.pendingSync;
        this.pendingSync = null;
        
        this.isSyncing = true;
        
        this.charts.forEach(({ chart, series }, index) => {
            if (index === sourceIndex) return;
            
            if (param.point && param.time) {
                const data = param.seriesData?.get(sourceSeries);
                const price = data?.close ?? data?.value;
                
                if (price !== undefined) {
                    chart.setCrosshairPosition(price, param.time, series);
                }
            } else {
                chart.clearCrosshairPosition();
            }
        });
        
        // Reset sync flag в следующем frame
        requestAnimationFrame(() => {
            this.isSyncing = false;
        });
    }
    
    destroy() {
        if (this.rafId !== null) {
            cancelAnimationFrame(this.rafId);
        }
        this.pendingSync = null;
        this.charts = [];
    }
}

// Использование
const sync = new RAFCrosshairSync([
    { chart: chart1, series: candleSeries1 },
    { chart: chart2, series: candleSeries2 },
]);
```

---

## ✅ Рекомендуемое решение

### Полный рабочий пример: Robust Multi-Chart Crosshair Sync

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// ROBUST CROSSHAIR SYNCHRONIZATION
// ============================================

class RobustCrosshairSync {
    constructor(options = {}) {
        this.charts = new Map();
        this.options = {
            throttleMs: options.throttleMs || 16,
            ignoreScroll: options.ignoreScroll !== false, // true by default
            debug: options.debug || false,
        };
        
        this._activeSource = null;
        this._pendingUpdate = null;
        this._rafId = null;
        this._lastThrottleTime = 0;
    }
    
    /**
     * Register a chart for synchronization
     */
    add(id, chart, series) {
        if (this.charts.has(id)) {
            this._log(`Chart ${id} already registered`);
            return this;
        }
        
        const handler = this._createHandler(id, series);
        chart.subscribeCrosshairMove(handler);
        
        this.charts.set(id, {
            id,
            chart,
            series,
            handler,
        });
        
        this._log(`Registered chart: ${id}`);
        return this;
    }
    
    /**
     * Remove a chart from synchronization
     */
    remove(id) {
        this.charts.delete(id);
        this._log(`Removed chart: ${id}`);
        return this;
    }
    
    _createHandler(sourceId, sourceSeries) {
        return (param) => {
            // 🔑 KEY FIX #1: Ignore scroll events
            if (this.options.ignoreScroll && !param.sourceEvent) {
                this._log(`Ignoring scroll event from ${sourceId}`);
                return;
            }
            
            // 🔑 KEY FIX #2: Prevent infinite loop
            if (this._activeSource !== null && this._activeSource !== sourceId) {
                this._log(`Blocking recursive sync from ${sourceId}`);
                return;
            }
            
            // Throttle check
            const now = Date.now();
            if (now - this._lastThrottleTime < this.options.throttleMs) {
                // Schedule for later
                this._scheduleUpdate(sourceId, param, sourceSeries);
                return;
            }
            
            this._lastThrottleTime = now;
            this._syncFromSource(sourceId, param, sourceSeries);
        };
    }
    
    _scheduleUpdate(sourceId, param, sourceSeries) {
        this._pendingUpdate = { sourceId, param, sourceSeries };
        
        if (this._rafId !== null) return;
        
        this._rafId = requestAnimationFrame(() => {
            this._rafId = null;
            
            if (this._pendingUpdate) {
                const { sourceId, param, sourceSeries } = this._pendingUpdate;
                this._pendingUpdate = null;
                this._syncFromSource(sourceId, param, sourceSeries);
            }
        });
    }
    
    _syncFromSource(sourceId, param, sourceSeries) {
        this._activeSource = sourceId;
        
        try {
            // No crosshair position
            if (!param.point || !param.time) {
                this._clearOthers(sourceId);
                return;
            }
            
            // Get price from series data
            const data = param.seriesData?.get(sourceSeries);
            const price = this._extractPrice(data);
            
            if (price === null) {
                this._log(`No price data at ${param.time}`);
                return;
            }
            
            // Sync to all other charts
            this.charts.forEach((entry, chartId) => {
                if (chartId === sourceId) return;
                
                try {
                    entry.chart.setCrosshairPosition(price, param.time, entry.series);
                } catch (e) {
                    this._log(`Error syncing to ${chartId}: ${e.message}`);
                }
            });
            
        } finally {
            // Release lock in next frame
            requestAnimationFrame(() => {
                this._activeSource = null;
            });
        }
    }
    
    _extractPrice(data) {
        if (!data) return null;
        
        // Handle different series types
        if ('close' in data) return data.close;
        if ('value' in data) return data.value;
        if ('high' in data && 'low' in data) {
            return (data.high + data.low) / 2;
        }
        
        return null;
    }
    
    _clearOthers(exceptId) {
        this.charts.forEach((entry, chartId) => {
            if (chartId !== exceptId) {
                entry.chart.clearCrosshairPosition();
            }
        });
    }
    
    /**
     * Manually set crosshair on all charts
     */
    setAll(time, price) {
        if (this._activeSource !== null) return;
        
        this.charts.forEach((entry) => {
            entry.chart.setCrosshairPosition(price, time, entry.series);
        });
    }
    
    /**
     * Clear crosshair on all charts
     */
    clearAll() {
        this.charts.forEach((entry) => {
            entry.chart.clearCrosshairPosition();
        });
    }
    
    _log(msg) {
        if (this.options.debug) {
            console.log(`[CrosshairSync] ${msg}`);
        }
    }
    
    destroy() {
        if (this._rafId !== null) {
            cancelAnimationFrame(this._rafId);
        }
        this.charts.clear();
        this._pendingUpdate = null;
    }
}

// ============================================
// FULL WORKING EXAMPLE
// ============================================

const container1 = document.getElementById('chart1');
const container2 = document.getElementById('chart2');

// Style containers
[container1, container2].forEach(c => {
    c.style.width = '100%';
    c.style.height = '300px';
    c.style.marginBottom = '10px';
});

// Create charts with same options
const chartOptions = {
    layout: {
        background: { type: 'solid', color: '#131722' },
        textColor: '#d1d4dc',
    },
    grid: {
        vertLines: { color: '#1f2937' },
        horzLines: { color: '#1f2937' },
    },
    crosshair: {
        mode: 0, // Normal
        vertLine: {
            color: '#6366f1',
            width: 1,
            style: 0, // Solid
            labelBackgroundColor: '#6366f1',
        },
        horzLine: {
            color: '#6366f1',
            width: 1,
            style: 0,
            labelBackgroundColor: '#6366f1',
        },
    },
    timeScale: {
        borderColor: '#2B2B43',
    },
    rightPriceScale: {
        borderColor: '#2B2B43',
    },
};

const chart1 = createChart(container1, chartOptions);
const chart2 = createChart(container2, chartOptions);

// Add series
const series1 = chart1.addCandlestickSeries({
    upColor: '#22c55e',
    downColor: '#ef4444',
    borderVisible: false,
    wickUpColor: '#22c55e',
    wickDownColor: '#ef4444',
});

const series2 = chart2.addLineSeries({
    color: '#3b82f6',
    lineWidth: 2,
});

// Generate sample data
const candleData = generateCandleData(150);
series1.setData(candleData);

// Line series data (e.g., indicator)
const lineData = candleData.map(d => ({
    time: d.time,
    value: (d.high + d.low + d.close) / 3, // Typical price
}));
series2.setData(lineData);

// ✅ SETUP SYNCHRONIZATION
const crosshairSync = new RobustCrosshairSync({
    throttleMs: 16,
    ignoreScroll: true,
    debug: true,
});

crosshairSync
    .add('main', chart1, series1)
    .add('indicator', chart2, series2);

// Sync time scales as well (for scroll/zoom)
const syncTimeScales = () => {
    const logicalRange = chart1.timeScale().getVisibleLogicalRange();
    if (logicalRange) {
        chart2.timeScale().setVisibleLogicalRange(logicalRange);
    }
};

chart1.timeScale().subscribeVisibleLogicalRangeChange(syncTimeScales);

// Fit content
chart1.timeScale().fitContent();
chart2.timeScale().fitContent();

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

// Cleanup on unmount
window.addEventListener('beforeunload', () => {
    crosshairSync.destroy();
    chart1.remove();
    chart2.remove();
});
```

---

## 📊 Сравнительная таблица

| Решение | Оценка | Scroll Fix | Loop Prevention | Производительность | Сложность |
|---------|--------|------------|-----------------|-------------------|-----------|
| **sourceEvent + flag** | 9/10 ⭐ | ✅ Да | ✅ Да | ✅ Отличная | Низкая |
| **Throttle + check** | 8/10 | ✅ Да | ✅ Да | ✅ Отличная | Средняя |
| **Event source tracking** | 9/10 | ✅ Да | ✅ Да | ✅ Хорошая | Высокая |
| **RAF-based sync** | 8/10 | ✅ Да | ✅ Да | ✅ Отличная | Средняя |

---

## 📝 Quick Reference: Ключевые правила

### ✅ DO (Делать):
```javascript
// 1. Проверять sourceEvent
if (!param.sourceEvent) return;

// 2. Использовать флаг синхронизации
if (this.isSyncing) return;
this.isSyncing = true;
try { /* sync */ } finally { this.isSyncing = false; }

// 3. Сбрасывать флаг в следующем frame
requestAnimationFrame(() => { this.isSyncing = false; });

// 4. Использовать throttle для высокой частоты событий
const handler = throttle(syncHandler, 16);
```

### ❌ DON'T (Не делать):
```javascript
// 1. Синхронизировать без проверок
chart.subscribeCrosshairMove((p) => {
    otherChart.setCrosshairPosition(...); // ❌ Infinite loop!
});

// 2. Синхронизировать при scrolling
if (!param.sourceEvent) {
    // ❌ Это scroll event — не синхронизировать!
}

// 3. Забывать про cleanup
// ❌ Memory leaks!
```

---

## 🔗 Источники

1. **GitHub Issue #1608** — [Glitches when syncing crosshair during scroll via setCrosshairPosition](https://github.com/tradingview/lightweight-charts/issues/1608)
2. **Lightweight Charts Docs** — [setCrosshairPosition](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#setCrosshairPosition)
3. **Lightweight Charts Docs** — [clearCrosshairPosition](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#clearCrosshairPosition)
4. **Lightweight Charts Docs** — [subscribeCrosshairMove](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#subscribeCrosshairMove)
5. **Chart Synchronization Examples** — [Multi-chart sync patterns](https://tradingview.github.io/lightweight-charts/tutorials/how_to/two-price-scales)
6. **Stack Overflow** — Debounce/throttle patterns for chart events

---

## 🔄 Связанные баги

- [БАГ #15: hoveredObjectId сбрасывается при scroll](./bug_015_hovered_reset_scroll.md) — похожая проблема с scroll
- [БАГ #14: hoveredObjectId undefined для маркеров](./bug_014_hovered_object_undefined.md) — hit-test проблемы
- [БАГ #8: Деградация производительности при WebSocket](./bug_008_websocket_degradation.md) — важно для real-time sync

---

*Документ создан: 2026-01-18*  
*Последнее обновление: 2026-01-18*
