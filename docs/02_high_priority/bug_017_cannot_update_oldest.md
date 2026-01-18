# БАГ #17: Ошибка "Cannot update oldest data" при real-time updates

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#592](https://github.com/tradingview/lightweight-charts/issues/592)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

При вызове `series.update()` с данными, имеющими timestamp старше последней точки серии, возникает ошибка. Это блокирует обновление исторических данных и создаёт проблемы при работе с WebSocket real-time feeds.

### Симптомы:
- Ошибка: `"Cannot update oldest data"` или `"Assertion failed: data must be asc ordered by time"`
- Невозможно добавить историю "в начало" серии через `update()`
- Невозможно обновить/исправить данные с прошедшим timestamp
- Real-time updates с тем же timestamp воспринимаются как duplicate

### Техническая причина:
- `series.update()` работает **только** с последней точкой данных
- Требует, чтобы `new_time >= last_time`
- Designed для append-only операций
- Нет встроенного механизма для insert/update посередине массива

### Сценарии воспроизведения:
1. **WebSocket reconnect:** После переподключения приходят пропущенные данные с прошлым timestamp
2. **Historical pagination:** Подгрузка старых данных при scroll влево
3. **Data correction:** Попытка исправить ошибочные данные в прошлом
4. **Duplicate timestamps:** Два update в одну секунду

### Пример проблемного кода:
```javascript
const series = chart.addCandlestickSeries();

// Начальные данные
series.setData([
    { time: 1705320000, open: 45000, high: 45500, low: 44800, close: 45200 },
    { time: 1705323600, open: 45200, high: 45800, low: 45100, close: 45600 },
    { time: 1705327200, open: 45600, high: 46000, low: 45400, close: 45900 }, // Последний
]);

// ❌ ПРОБЛЕМА: попытка обновить данные старше последней точки
series.update({
    time: 1705320000, // < 1705327200 — это ПРОШЛЫЙ timestamp!
    open: 45000,
    high: 45600,      // Хотим исправить high
    low: 44800,
    close: 45200  
});
// Error: Cannot update oldest data!

// ❌ ПРОБЛЕМА: попытка добавить пропущенные данные
series.update({
    time: 1705325400, // Между существующими точками
    open: 45500,
    high: 45700,
    low: 45300,
    close: 45400
});
// Error: data must be ascending!
```

---

## 🔍 Найденные решения

### Решение 1: Data Manager с полным набором данных
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

Централизованное управление данными с использованием `setData()` для любых изменений.

**Преимущества:**
- ✅ Полный контроль над данными
- ✅ Поддержка любых операций (insert, update, delete)
- ✅ Гарантированная консистентность
- ✅ Работает для всех сценариев

**Недостатки:**
- ⚠️ Больше памяти (хранение всех данных)
- ⚠️ `setData()` медленнее `update()` для больших объёмов
- ⚠️ Требует debounce для частых обновлений

```javascript
class ChartDataManager {
    constructor(series) {
        this.series = series;
        this.data = [];          // Хранит все данные
        this.dataMap = new Map(); // Быстрый lookup по времени
        this.pendingUpdate = false;
    }
    
    // Добавить/обновить одну точку
    upsert(point) {
        const existing = this.dataMap.get(point.time);
        
        if (existing) {
            // Обновляем существующую точку
            Object.assign(existing, point);
        } else {
            // Добавляем новую
            this.data.push(point);
            this.dataMap.set(point.time, point);
            
            // Сортируем по времени
            this.data.sort((a, b) => a.time - b.time);
        }
        
        this._scheduleRender();
    }
    
    // Добавить множество точек
    upsertBatch(points) {
        for (const point of points) {
            const existing = this.dataMap.get(point.time);
            if (existing) {
                Object.assign(existing, point);
            } else {
                this.data.push(point);
                this.dataMap.set(point.time, point);
            }
        }
        
        // Одна сортировка для всего batch
        this.data.sort((a, b) => a.time - b.time);
        
        this._scheduleRender();
    }
    
    // Добавить только новую точку (real-time, оптимизированно)
    appendLatest(point) {
        const lastTime = this.data.length > 0 
            ? this.data[this.data.length - 1].time 
            : 0;
        
        if (point.time >= lastTime) {
            // Используем быстрый update()
            if (point.time === lastTime || this.data.length === 0) {
                if (this.data.length > 0) {
                    Object.assign(this.data[this.data.length - 1], point);
                } else {
                    this.data.push(point);
                    this.dataMap.set(point.time, point);
                }
            } else {
                this.data.push(point);
                this.dataMap.set(point.time, point);
            }
            
            this.series.update(point);
            return true;
        }
        
        // Fallback на upsert для старых данных
        this.upsert(point);
        return false;
    }
    
    // Prepend исторических данных
    prependHistory(history) {
        // Предполагаем что history отсортирован
        const newData = [...history];
        
        for (const point of newData) {
            if (!this.dataMap.has(point.time)) {
                this.dataMap.set(point.time, point);
            } else {
                // Merge с существующими
                Object.assign(this.dataMap.get(point.time), point);
            }
        }
        
        // Пересобираем массив из Map
        this.data = Array.from(this.dataMap.values())
            .sort((a, b) => a.time - b.time);
        
        this._scheduleRender();
    }
    
    // Удалить точку
    remove(time) {
        if (!this.dataMap.has(time)) return false;
        
        this.dataMap.delete(time);
        this.data = this.data.filter(d => d.time !== time);
        
        this._scheduleRender();
        return true;
    }
    
    // Получить данные
    getData() {
        return [...this.data];
    }
    
    // Получить точку по времени
    get(time) {
        return this.dataMap.get(time);
    }
    
    // Очистить все
    clear() {
        this.data = [];
        this.dataMap.clear();
        this.series.setData([]);
    }
    
    // Debounced render
    _scheduleRender() {
        if (this.pendingUpdate) return;
        
        this.pendingUpdate = true;
        requestAnimationFrame(() => {
            this.series.setData(this.data);
            this.pendingUpdate = false;
        });
    }
}

// Использование
const dataManager = new ChartDataManager(series);

// Начальные данные
dataManager.upsertBatch(initialData);

// Real-time update (оптимизированно)
websocket.onmessage = (event) => {
    const newBar = JSON.parse(event.data);
    dataManager.appendLatest(newBar);
};

// После reconnect — получаем пропущенные данные
const missedData = await fetchMissedData(lastTime);
dataManager.upsertBatch(missedData);  // Автоматически сортирует и применяет
```

---

### Решение 2: Timestamp validation перед update
**Оценка: 8/10**

Проверка timestamp перед вызовом `update()` с fallback на `setData()`.

**Преимущества:**
- ✅ Максимальная производительность для append
- ✅ Automatic fallback для edge cases
- ✅ Простая интеграция

**Недостатки:**
- ⚠️ `setData()` fallback может быть медленным
- ⚠️ Нужно хранить данные для fallback

```javascript
class SmartSeriesUpdater {
    constructor(series) {
        this.series = series;
        this.data = [];
        this.lastTime = null;
    }
    
    setData(data) {
        this.data = [...data].sort((a, b) => a.time - b.time);
        this.lastTime = this.data.length > 0 
            ? this.data[this.data.length - 1].time 
            : null;
        this.series.setData(this.data);
    }
    
    update(point) {
        // Проверяем можно ли использовать update()
        if (this.lastTime === null || point.time >= this.lastTime) {
            // ✅ Можно безопасно вызывать update()
            if (point.time === this.lastTime && this.data.length > 0) {
                // Обновляем последнюю точку
                this.data[this.data.length - 1] = point;
            } else {
                // Добавляем новую
                this.data.push(point);
            }
            
            this.lastTime = point.time;
            this.series.update(point);
            
            return { method: 'update', success: true };
        }
        
        // ❌ Нужен fallback на setData()
        console.warn(`Timestamp ${point.time} is older than last ${this.lastTime}. Using setData() fallback.`);
        
        // Находим место для вставки
        const insertIndex = this._findInsertIndex(point.time);
        
        if (insertIndex < this.data.length && this.data[insertIndex].time === point.time) {
            // Update existing
            this.data[insertIndex] = point;
        } else {
            // Insert new
            this.data.splice(insertIndex, 0, point);
        }
        
        this.series.setData(this.data);
        
        return { method: 'setData', success: true };
    }
    
    _findInsertIndex(time) {
        let left = 0;
        let right = this.data.length;
        
        while (left < right) {
            const mid = (left + right) >>> 1;
            if (this.data[mid].time < time) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        
        return left;
    }
    
    // Batch insert с автоматической сортировкой
    batchInsert(points) {
        for (const point of points) {
            const idx = this._findInsertIndex(point.time);
            if (idx < this.data.length && this.data[idx].time === point.time) {
                this.data[idx] = point;
            } else {
                this.data.splice(idx, 0, point);
            }
        }
        
        this.lastTime = this.data.length > 0 
            ? this.data[this.data.length - 1].time 
            : null;
        
        this.series.setData(this.data);
    }
}

// Использование
const updater = new SmartSeriesUpdater(series);

updater.setData(initialData);

// Real-time — производительно
updater.update({ time: 1705330800, open: 46000, high: 46200, low: 45900, close: 46100 });

// Historical — автоматически использует setData
updater.update({ time: 1705320000, open: 45000, high: 45600, low: 44800, close: 45200 });
```

---

### Решение 3: Dual-buffer для real-time + history
**Оценка: 8/10**

Разделение логики для real-time updates и исторических изменений.

**Преимущества:**
- ✅ Оптимизировано для разных сценариев
- ✅ Чёткое разделение ответственности
- ✅ Поддержка infinite scroll

**Недостатки:**
- ⚠️ Сложнее логика
- ⚠️ Нужна синхронизация буферов

```javascript
class DualBufferManager {
    constructor(series, options = {}) {
        this.series = series;
        this.maxDataPoints = options.maxDataPoints || 10000;
        
        // Основной буфер данных
        this.data = [];
        this.dataIndex = new Map();
        
        // Pending historical data (будет применён 1 раз)
        this.pendingHistory = [];
        this.historyPending = false;
        
        // Real-time buffer для быстрых updates
        this.realtimeBuffer = [];
        this.realtimePending = false;
    }
    
    // Начальная загрузка
    initialize(data) {
        this.data = [...data].sort((a, b) => a.time - b.time);
        this._rebuildIndex();
        this.series.setData(this.data);
    }
    
    // Real-time tick (оптимизированно)
    tick(point) {
        const lastTime = this.data.length > 0 
            ? this.data[this.data.length - 1].time 
            : 0;
        
        if (point.time >= lastTime) {
            // Fast path: использум update()
            if (point.time === lastTime && this.data.length > 0) {
                Object.assign(this.data[this.data.length - 1], point);
            } else {
                this.data.push(point);
                this.dataIndex.set(point.time, this.data.length - 1);
            }
            
            this.series.update(point);
            this._trimOldData();
            return 'realtime';
        }
        
        // Slow path: добавляем в pending
        this.realtimeBuffer.push(point);
        this._scheduleRealtimeFlush();
        return 'buffered';
    }
    
    // Добавить исторические данные (prepend)
    addHistory(history) {
        this.pendingHistory.push(...history);
        this._scheduleHistoryFlush();
    }
    
    _scheduleRealtimeFlush() {
        if (this.realtimePending) return;
        
        this.realtimePending = true;
        requestAnimationFrame(() => {
            this._flushRealtime();
            this.realtimePending = false;
        });
    }
    
    _scheduleHistoryFlush() {
        if (this.historyPending) return;
        
        this.historyPending = true;
        // Откладываем дольше чтобы не мешать realtime
        setTimeout(() => {
            this._flushHistory();
            this.historyPending = false;
        }, 100);
    }
    
    _flushRealtime() {
        if (this.realtimeBuffer.length === 0) return;
        
        for (const point of this.realtimeBuffer) {
            const existingIdx = this.dataIndex.get(point.time);
            if (existingIdx !== undefined) {
                Object.assign(this.data[existingIdx], point);
            } else {
                this.data.push(point);
            }
        }
        
        this.realtimeBuffer = [];
        this.data.sort((a, b) => a.time - b.time);
        this._rebuildIndex();
        this.series.setData(this.data);
    }
    
    _flushHistory() {
        if (this.pendingHistory.length === 0) return;
        
        const saveRange = this._getVisibleRange();
        
        for (const point of this.pendingHistory) {
            if (!this.dataIndex.has(point.time)) {
                this.data.push(point);
            }
        }
        
        this.pendingHistory = [];
        this.data.sort((a, b) => a.time - b.time);
        this._rebuildIndex();
        this.series.setData(this.data);
        
        // Восстанавливаем visible range
        if (saveRange) {
            this._restoreVisibleRange(saveRange);
        }
    }
    
    _rebuildIndex() {
        this.dataIndex.clear();
        this.data.forEach((point, idx) => {
            this.dataIndex.set(point.time, idx);
        });
    }
    
    _trimOldData() {
        if (this.data.length > this.maxDataPoints) {
            const removeCount = this.data.length - this.maxDataPoints;
            this.data.splice(0, removeCount);
            this._rebuildIndex();
        }
    }
    
    _getVisibleRange() {
        // Реализация зависит от контекста
        return null;
    }
    
    _restoreVisibleRange(range) {
        // Реализация зависит от контекста
    }
}

// Использование
const buffer = new DualBufferManager(series, { maxDataPoints: 5000 });

// Начальные данные
buffer.initialize(historicalData);

// WebSocket real-time
ws.onmessage = (e) => {
    const tick = JSON.parse(e.data);
    buffer.tick(tick);
};

// Infinite scroll — загрузка истории
chart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
    if (range.from < 10) {  // Близко к левому краю
        const oldestTime = buffer.data[0]?.time;
        loadMoreHistory(oldestTime).then(history => {
            buffer.addHistory(history);
        });
    }
});
```

---

### Решение 4: Queue-based real-time handler
**Оценка: 7/10**

Очередь для обработки входящих данных с правильной сортировкой.

**Преимущества:**
- ✅ Гарантированный порядок обработки
- ✅ Защита от race conditions
- ✅ Batching для производительности

**Недостатки:**
- ⚠️ Задержка обработки (queueing)
- ⚠️ Сложность отладки

```javascript
class UpdateQueue {
    constructor(series, processInterval = 50) {
        this.series = series;
        this.queue = [];
        this.data = [];
        this.processing = false;
        this.processInterval = processInterval;
        
        this._startProcessing();
    }
    
    enqueue(update) {
        this.queue.push({
            ...update,
            receivedAt: Date.now()
        });
    }
    
    enqueueBatch(updates) {
        const now = Date.now();
        for (const update of updates) {
            this.queue.push({ ...update, receivedAt: now });
        }
    }
    
    _startProcessing() {
        setInterval(() => this._processQueue(), this.processInterval);
    }
    
    _processQueue() {
        if (this.queue.length === 0 || this.processing) return;
        
        this.processing = true;
        
        // Берём все текущие элементы очереди
        const batch = this.queue.splice(0);
        
        // Сортируем по timestamp
        batch.sort((a, b) => a.time - b.time);
        
        let needsFullRefresh = false;
        const lastTime = this.data.length > 0 
            ? this.data[this.data.length - 1].time 
            : null;
        
        for (const update of batch) {
            // Проверяем нужен ли полный refresh
            if (lastTime !== null && update.time < lastTime) {
                needsFullRefresh = true;
            }
            
            // Upsert в наш массив
            const existingIdx = this.data.findIndex(d => d.time === update.time);
            if (existingIdx >= 0) {
                this.data[existingIdx] = update;
            } else {
                this.data.push(update);
            }
        }
        
        if (needsFullRefresh) {
            this.data.sort((a, b) => a.time - b.time);
            this.series.setData(this.data);
        } else if (batch.length > 0) {
            // Можем использовать update для последнего элемента
            const lastUpdate = batch[batch.length - 1];
            this.series.update(lastUpdate);
        }
        
        this.processing = false;
    }
    
    setInitialData(data) {
        this.data = [...data].sort((a, b) => a.time - b.time);
        this.series.setData(this.data);
    }
    
    destroy() {
        this.queue = [];
        this.data = [];
    }
}

// Использование
const queue = new UpdateQueue(series, 50); // Процессинг каждые 50ms

queue.setInitialData(initialData);

// WebSocket — все updates идут в очередь
ws.onmessage = (e) => {
    const data = JSON.parse(e.data);
    
    if (Array.isArray(data)) {
        queue.enqueueBatch(data);
    } else {
        queue.enqueue(data);
    }
};
```

---

## ✅ Рекомендуемое решение

### Полный рабочий пример: Universal Data Manager

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// UNIVERSAL DATA MANAGER
// Решает проблему "Cannot update oldest data"
// ============================================

class UniversalDataManager {
    constructor(chart, series, options = {}) {
        this.chart = chart;
        this.series = series;
        this.data = [];
        this.dataMap = new Map();
        
        this.options = {
            maxPoints: options.maxPoints || 10000,
            batchDelay: options.batchDelay || 16,  // ~60fps
            onDataChange: options.onDataChange || (() => {}),
        };
        
        this._pendingUpdates = [];
        this._updateScheduled = false;
    }
    
    // ============================================
    // PUBLIC API
    // ============================================
    
    /**
     * Установить начальные данные
     */
    setData(data) {
        this.data = [...data].sort((a, b) => a.time - b.time);
        this._rebuildMap();
        this.series.setData(this.data);
        this.options.onDataChange(this.data);
        return this;
    }
    
    /**
     * Real-time update (оптимизированно для последней точки)
     * Автоматически определяет нужен ли update() или setData()
     */
    update(point) {
        const lastTime = this._getLastTime();
        
        // Fast path: можно использовать series.update()
        if (lastTime === null || point.time >= lastTime) {
            return this._fastUpdate(point);
        }
        
        // Slow path: нужен insert в прошлое
        return this._slowUpdate(point);
    }
    
    /**
     * Batch update для множества точек
     */
    updateBatch(points) {
        if (!points || points.length === 0) return this;
        
        // Сортируем входящие точки
        const sorted = [...points].sort((a, b) => a.time - b.time);
        
        // Определяем нужен ли полный refresh
        const lastTime = this._getLastTime();
        const needsRefresh = sorted[0].time < lastTime;
        
        // Применяем все изменения
        for (const point of sorted) {
            const existing = this.dataMap.get(point.time);
            if (existing) {
                Object.assign(existing, point);
            } else {
                this.data.push(point);
                this.dataMap.set(point.time, point);
            }
        }
        
        if (needsRefresh) {
            this.data.sort((a, b) => a.time - b.time);
            this._scheduleRender();
        } else {
            // Можем использовать update для последней точки
            this.series.update(sorted[sorted.length - 1]);
        }
        
        this._trimData();
        this.options.onDataChange(this.data);
        return this;
    }
    
    /**
     * Prepend исторических данных
     */
    prependHistory(history) {
        if (!history || history.length === 0) return this;
        
        const saveRange = this.chart.timeScale().getVisibleLogicalRange();
        const offset = history.length;
        
        for (const point of history) {
            if (!this.dataMap.has(point.time)) {
                this.dataMap.set(point.time, point);
            }
        }
        
        this.data = Array.from(this.dataMap.values())
            .sort((a, b) => a.time - b.time);
        
        this.series.setData(this.data);
        
        // Корректируем visible range чтобы не прыгало
        if (saveRange) {
            this.chart.timeScale().setVisibleLogicalRange({
                from: saveRange.from + offset,
                to: saveRange.to + offset,
            });
        }
        
        this.options.onDataChange(this.data);
        return this;
    }
    
    /**
     * Удалить точку
     */
    remove(time) {
        if (!this.dataMap.has(time)) return false;
        
        this.dataMap.delete(time);
        this.data = this.data.filter(d => d.time !== time);
        this._scheduleRender();
        this.options.onDataChange(this.data);
        return true;
    }
    
    /**
     * Получить все данные
     */
    getData() {
        return [...this.data];
    }
    
    /**
     * Получить точку по времени
     */
    get(time) {
        return this.dataMap.get(time);
    }
    
    /**
     * Количество точек
     */
    get length() {
        return this.data.length;
    }
    
    /**
     * Очистить все данные
     */
    clear() {
        this.data = [];
        this.dataMap.clear();
        this.series.setData([]);
        this.options.onDataChange([]);
        return this;
    }
    
    // ============================================
    // PRIVATE METHODS
    // ============================================
    
    _fastUpdate(point) {
        const lastPoint = this.data[this.data.length - 1];
        
        if (lastPoint && point.time === lastPoint.time) {
            // Обновляем существующую точку
            Object.assign(lastPoint, point);
        } else {
            // Добавляем новую
            this.data.push(point);
            this.dataMap.set(point.time, point);
        }
        
        // Используем быстрый series.update()
        this.series.update(point);
        
        this._trimData();
        this.options.onDataChange(this.data);
        return { method: 'update', point };
    }
    
    _slowUpdate(point) {
        const existing = this.dataMap.get(point.time);
        
        if (existing) {
            Object.assign(existing, point);
        } else {
            this.data.push(point);
            this.dataMap.set(point.time, point);
        }
        
        this.data.sort((a, b) => a.time - b.time);
        this._scheduleRender();
        this.options.onDataChange(this.data);
        return { method: 'setData', point };
    }
    
    _scheduleRender() {
        if (this._updateScheduled) return;
        
        this._updateScheduled = true;
        setTimeout(() => {
            this.series.setData(this.data);
            this._updateScheduled = false;
        }, this.options.batchDelay);
    }
    
    _rebuildMap() {
        this.dataMap.clear();
        for (const point of this.data) {
            this.dataMap.set(point.time, point);
        }
    }
    
    _getLastTime() {
        if (this.data.length === 0) return null;
        return this.data[this.data.length - 1].time;
    }
    
    _trimData() {
        if (this.data.length > this.options.maxPoints) {
            const removeCount = this.data.length - this.options.maxPoints;
            const removed = this.data.splice(0, removeCount);
            
            for (const point of removed) {
                this.dataMap.delete(point.time);
            }
        }
    }
}

// ============================================
// EXAMPLE USAGE
// ============================================

const container = document.getElementById('chart');

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
    timeScale: {
        timeVisible: true,
        secondsVisible: false,
    },
});

const series = chart.addCandlestickSeries({
    upColor: '#22c55e',
    downColor: '#ef4444',
    borderVisible: false,
    wickUpColor: '#22c55e',
    wickDownColor: '#ef4444',
});

// Создаём Data Manager
const dataManager = new UniversalDataManager(chart, series, {
    maxPoints: 5000,
    batchDelay: 16,
    onDataChange: (data) => {
        console.log(`📊 Data points: ${data.length}`);
    },
});

// Начальные данные
const initialData = generateCandleData(100);
dataManager.setData(initialData);

// Симуляция WebSocket
let currentTime = initialData[initialData.length - 1].time;

setInterval(() => {
    currentTime += 60; // +1 минута
    
    const lastClose = dataManager.get(currentTime - 60)?.close || 45000;
    const change = (Math.random() - 0.5) * 200;
    
    const newCandle = {
        time: currentTime,
        open: lastClose,
        high: lastClose + Math.random() * 100,
        low: lastClose - Math.random() * 100,
        close: lastClose + change,
    };
    
    // ✅ Работает без ошибок!
    const result = dataManager.update(newCandle);
    console.log(`Update method: ${result.method}`);
}, 1000);

// Симуляция получения пропущенных данных
setTimeout(() => {
    console.log('📥 Simulating missed data from reconnect...');
    
    // Точки с timestamp в прошлом
    const missedData = [
        { time: currentTime - 180, open: 45100, high: 45200, low: 45000, close: 45150 },
        { time: currentTime - 120, open: 45150, high: 45300, low: 45100, close: 45250 },
    ];
    
    // ✅ Автоматически обрабатывает старые timestamps!
    dataManager.updateBatch(missedData);
}, 5000);

// Infinite scroll — загрузка истории
chart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
    if (range && range.from < 20) {
        console.log('📜 Loading more history...');
        
        const oldestTime = dataManager.data[0]?.time;
        if (oldestTime) {
            const moreHistory = generateCandleData(50, oldestTime - 50 * 60);
            dataManager.prependHistory(moreHistory);
        }
    }
});

chart.timeScale().fitContent();

// ============================================
// HELPER FUNCTION
// ============================================

function generateCandleData(count, startTime = null) {
    const data = [];
    let time = startTime || (Math.floor(Date.now() / 1000) - count * 60);
    let price = 45000;
    
    for (let i = 0; i < count; i++) {
        const change = (Math.random() - 0.5) * 300;
        const open = price;
        const close = price + change;
        
        data.push({
            time: time,
            open: Math.round(open * 100) / 100,
            high: Math.round((Math.max(open, close) + Math.random() * 100) * 100) / 100,
            low: Math.round((Math.min(open, close) - Math.random() * 100) * 100) / 100,
            close: Math.round(close * 100) / 100,
        });
        
        time += 60;
        price = close;
    }
    
    return data;
}
```

---

## 📊 Сравнительная таблица

| Решение | Оценка | Real-time | Historical | Memory | Complexity |
|---------|--------|-----------|------------|--------|------------|
| **Data Manager** | 9/10 ⭐ | ✅ Fast | ✅ Full | ⚠️ High | Medium |
| **Validation + Fallback** | 8/10 | ✅ Fast | ✅ Auto | Medium | Low |
| **Dual Buffer** | 8/10 | ✅ Fast | ✅ Queued | Medium | High |
| **Update Queue** | 7/10 | ⚠️ Delayed | ✅ Ordered | Low | Medium |

---

## 📝 Quick Reference

### ✅ DO (Правильно):
```javascript
// 1. Для real-time (append/update последней точки)
if (newPoint.time >= lastTime) {
    series.update(newPoint);  // ✅ Быстро
}

// 2. Для исторических данных
allData.push(historicalPoints);
allData.sort((a, b) => a.time - b.time);
series.setData(allData);  // ✅ Надёжно

// 3. Хранить данные локально
const dataStore = [...];  // Ваша копия

// 4. Используйте batch для множества точек
```

### ❌ DON'T (Неправильно):
```javascript
// 1. Не вызывайте update() с прошлым timestamp
series.update({ time: oldTimestamp, ... });  // ❌ Error!

// 2. Не полагайтесь только на series.update()
// Он работает только для последней точки

// 3. Не сортируйте данные после каждого update
// Используйте batch и debounce
```

---

## 🔗 Источники

1. **GitHub Issue #592** — [Cannot update oldest data](https://github.com/tradingview/lightweight-charts/issues/592)
2. **Lightweight Charts Docs** — [series.update()](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#update)
3. **Lightweight Charts Docs** — [series.setData()](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#setdata)
4. **Lightweight Charts Tutorial** — [Infinite History](https://tradingview.github.io/lightweight-charts/tutorials/how_to/infinite-history)
5. **Stack Overflow** — [Prepending data to lightweight-charts](https://stackoverflow.com/questions/tagged/lightweight-charts)
6. **WebSocket Best Practices** — Real-time data handling patterns

---

## 🔄 Связанные баги

- [БАГ #10: Деградация производительности setData()](./bug_010_setdata_performance.md) — оптимизация setData
- [БАГ #8: Деградация производительности при WebSocket](./bug_008_websocket_degradation.md) — real-time updates

---

*Документ создан: 2026-01-18*  
*Последнее обновление: 2026-01-18*
