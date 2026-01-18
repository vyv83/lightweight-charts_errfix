# БАГ #10: Деградация производительности setData() при больших датасетах

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issues:** [#838](https://github.com/tradingview/lightweight-charts/issues/838), [#1404](https://github.com/tradingview/lightweight-charts/issues/1404)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (частично улучшено в v5.1.0 с Data Conflation)  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

### Суть проблемы
Каждый вызов `setData()` перерисовывает всю серию с нуля, что приводит к значительной деградации производительности при работе с большими датасетами (>10k точек):

1. **Полная перерисовка** — при каждом `setData()` библиотека очищает и заново рендерит все данные
2. **Накопительное замедление** — при lazy loading исторических данных каждый вызов медленнее предыдущего
3. **Заметные "подвисания" UI** — блокировка главного потока на 100-500ms при >50k точек
4. **Смещение visible range** — при добавлении данных в начало смещается видимая область

### Сценарии воспроизведения

```javascript
// Проблемный код: pagination исторических данных
async function loadMoreHistory() {
    const oldData = series.getData();  // Не существует такого API!
    const newChunk = await fetchHistory(oldestTimestamp);
    
    // ❌ МЕДЛЕННО: полная перерисовка
    series.setData([...newChunk, ...currentData]);  // O(n) каждый раз
}

// Проблема: 10k + 1k = перерисовка 11k точек
// Затем: 11k + 1k = перерисовка 12k точек
// И так далее... экспоненциальная деградация
```

### Технические причины

1. **Архитектура API** — `setData()` не умеет инкрементально добавлять данные
2. **Data transformation overhead** — конвертация данных создаёт огромные allocations (Issue #838)
3. **Отсутствие prepend API** — нет эффективного способа добавить данные в начало
4. **Visible range shifting** — библиотека пересчитывает видимую область при каждом `setData()`

---

## 🔍 Найденные решения

### Решение 1: Использование Data Conflation (v5.1.0+)
**Оценка: 9/10** ⭐⭐⭐⭐⭐

**Источник:** [Lightweight Charts v5.1.0 Release Notes](https://github.com/tradingview/lightweight-charts/releases/tag/v5.1.0)

**Суть:** Включение автоматической агрегации данных при zoom out.

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
    width: 800,
    height: 400,
});

// Создаём серию с Data Conflation
const series = chart.addCandlestickSeries({
    // ✅ Включаем conflation для больших датасетов
    enableConflation: true,
    
    // Порог активации (по умолчанию 1.0)
    // Увеличьте для более раннего включения
    conflationThresholdFactor: 2.0,
    
    // Предварительный расчёт при инициализации
    // ⚠️ Увеличивает initial load time
    precomputeConflationOnInit: true,
    
    // Приоритет фоновых вычислений
    precomputeConflationPriority: 'user-visible'
});

// Теперь setData работает быстрее при zoom out
series.setData(hugeDataset);  // 100k+ точек
```

**Как это работает:**
- Когда bar spacing < 0.5 пикселя, библиотека автоматически объединяет точки
- Renderится меньше элементов → выше производительность
- Данные кешируются для быстрого zoom

**Плюсы:**
- ✅ Официальное решение от TradingView
- ✅ Автоматическая оптимизация
- ✅ Не требует изменения логики приложения
- ✅ Работает со всеми типами серий

**Минусы:**
- ⚠️ Требует v5.1.0+
- ⚠️ При zoom in всё равно рендерятся все данные
- ⚠️ `precomputeConflationOnInit` увеличивает initial load

---

### Решение 2: update() вместо setData() для новых данных
**Оценка: 9/10** ⭐⭐⭐⭐⭐

**Источник:** [Lightweight Charts Docs — Real-time Updates](https://tradingview.github.io/lightweight-charts/docs/api)

**Суть:** Использование `update()` для добавления новых данных в конец серии.

```typescript
// ============================================
// Правильный паттерн: update() для real-time
// ============================================

class EfficientDataManager {
    private series: ISeriesApi<SeriesType>;
    private lastTimestamp: number = 0;
    
    constructor(series: ISeriesApi<SeriesType>) {
        this.series = series;
    }
    
    /**
     * Первоначальная загрузка — используем setData()
     */
    public initialize(data: SeriesData[]): void {
        this.series.setData(data);
        if (data.length > 0) {
            this.lastTimestamp = data[data.length - 1].time as number;
        }
    }
    
    /**
     * Добавление нового бара — используем update()
     * ✅ Очень быстро, O(1)
     */
    public appendBar(bar: SeriesData): void {
        const barTime = bar.time as number;
        
        if (barTime > this.lastTimestamp) {
            // Новый бар
            this.series.update(bar);
            this.lastTimestamp = barTime;
        } else if (barTime === this.lastTimestamp) {
            // Обновление последнего бара (intrabar update)
            this.series.update(bar);
        } else {
            // ⚠️ Исторические данные - придётся использовать setData()
            console.warn('Cannot update historical data with update()');
        }
    }
    
    /**
     * Batch обновление последнего бара
     * Для high-frequency WebSocket
     */
    public batchUpdate(bars: SeriesData[]): void {
        // Фильтруем только новый бар и обновления последнего
        const validBars = bars.filter(
            bar => (bar.time as number) >= this.lastTimestamp
        );
        
        for (const bar of validBars) {
            this.series.update(bar);
        }
        
        if (validBars.length > 0) {
            this.lastTimestamp = validBars[validBars.length - 1].time as number;
        }
    }
}

// Использование
const manager = new EfficientDataManager(candleSeries);
manager.initialize(historicalData);

// WebSocket real-time updates
socket.onmessage = (event) => {
    const tick = JSON.parse(event.data);
    manager.appendBar(tick);  // ✅ O(1) операция!
};
```

**Плюсы:**
- ✅ O(1) сложность для append-операций
- ✅ Нет перерисовки всего графика
- ✅ Идеально для real-time данных
- ✅ Работает во всех версиях

**Минусы:**
- ⚠️ Нельзя добавлять данные в начало (prepend)
- ⚠️ Нельзя обновлять исторические точки
- ⚠️ Требует tracking последнего timestamp

---

### Решение 3: Виртуальное окно данных (Data Windowing)
**Оценка: 8/10** ⭐⭐⭐⭐

**Источник:** Паттерн из high-performance charting libraries (SciChart, ZingChart)

**Суть:** Держать в графике только видимые данные + буфер.

```typescript
// ============================================
// Виртуальное окно данных
// ============================================

interface DataPoint {
    time: number;
    open: number;
    high: number;
    low: number;
    close: number;
}

class VirtualDataWindow {
    private allData: DataPoint[] = [];
    private series: ISeriesApi<'Candlestick'>;
    private chart: IChartApi;
    
    // Размер окна
    private readonly VISIBLE_BUFFER = 500;    // Дополнительные точки за пределами экрана
    private readonly MIN_VISIBLE = 50;        // Минимум видимых точек
    
    private loadedRange: { from: number; to: number } = { from: 0, to: 0 };
    private isLoading: boolean = false;
    
    constructor(chart: IChartApi, series: ISeriesApi<'Candlestick'>) {
        this.chart = chart;
        this.series = series;
        this.setupScrollListener();
    }
    
    /**
     * Загрузка полного датасета в память (не в график!)
     */
    public setFullDataset(data: DataPoint[]): void {
        this.allData = data;
        
        // Показываем только последние N точек
        const startIndex = Math.max(0, data.length - this.VISIBLE_BUFFER * 2);
        this.loadWindow(startIndex, data.length);
    }
    
    /**
     * Загрузка окна данных в график
     */
    private loadWindow(fromIndex: number, toIndex: number): void {
        const from = Math.max(0, fromIndex);
        const to = Math.min(this.allData.length, toIndex);
        
        if (from === this.loadedRange.from && to === this.loadedRange.to) {
            return; // Окно не изменилось
        }
        
        const windowData = this.allData.slice(from, to);
        this.series.setData(windowData);
        this.loadedRange = { from, to };
        
        console.log(`Loaded window: ${from} - ${to} (${to - from} points)`);
    }
    
    /**
     * Отслеживание scroll для подгрузки данных
     */
    private setupScrollListener(): void {
        this.chart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
            if (!range || this.isLoading) return;
            
            this.checkAndExpandWindow(range);
        });
    }
    
    private checkAndExpandWindow(range: LogicalRange): void {
        const visibleFrom = Math.floor(range.from);
        const visibleTo = Math.ceil(range.to);
        
        // Рассчитываем индексы в allData
        const currentWindowSize = this.loadedRange.to - this.loadedRange.from;
        const dataVisibleFrom = visibleFrom;
        const dataVisibleTo = visibleTo;
        
        // Проверяем, нужно ли расширять окно
        const needsExpansionLeft = dataVisibleFrom < this.VISIBLE_BUFFER / 2;
        const needsExpansionRight = dataVisibleTo > currentWindowSize - this.VISIBLE_BUFFER / 2;
        
        if (needsExpansionLeft && this.loadedRange.from > 0) {
            this.isLoading = true;
            
            // Расширяем влево
            const newFrom = Math.max(0, this.loadedRange.from - this.VISIBLE_BUFFER);
            
            // Сохраняем visible range
            const currentRange = this.chart.timeScale().getVisibleLogicalRange();
            
            this.loadWindow(newFrom, this.loadedRange.to);
            
            // Восстанавливаем visible range со смещением
            if (currentRange) {
                const offset = this.loadedRange.from - newFrom;
                this.chart.timeScale().setVisibleLogicalRange({
                    from: currentRange.from + offset,
                    to: currentRange.to + offset
                });
            }
            
            this.isLoading = false;
        }
    }
    
    /**
     * Получение статистики
     */
    public getStats(): { total: number; loaded: number; efficiency: number } {
        const loaded = this.loadedRange.to - this.loadedRange.from;
        return {
            total: this.allData.length,
            loaded,
            efficiency: loaded / this.allData.length
        };
    }
}

// Использование
const virtualWindow = new VirtualDataWindow(chart, series);
virtualWindow.setFullDataset(massiveDataset);  // 500k точек

// В графике только ~1000 точек в любой момент времени
```

**Плюсы:**
- ✅ Постоянный размер данных в графике
- ✅ Подходит для миллионов точек
- ✅ Минимальное потребление памяти GraphicsGL
- ✅ Плавный scroll

**Минусы:**
- ⚠️ Сложная реализация
- ⚠️ Все данные должны быть в памяти приложения
- ⚠️ Возможны "прыжки" при resync visible range

---

### Решение 4: LTTB Downsampling для обзорного вида
**Оценка: 8/10** ⭐⭐⭐⭐

**Источник:** [Largest Triangle Three Buckets Algorithm](https://skemman.is/handle/1946/15343)

**Суть:** Уменьшение количества точек с сохранением визуальных характеристик.

```typescript
// ============================================
// LTTB Downsampling Implementation
// ============================================

interface Point {
    time: number;
    value: number;
}

/**
 * Largest Triangle Three Buckets (LTTB) Algorithm
 * Sveinn Steinarsson, 2013
 */
function lttbDownsample(data: Point[], threshold: number): Point[] {
    if (threshold >= data.length || threshold <= 2) {
        return data;
    }
    
    const sampled: Point[] = [];
    
    // Bucket size
    const every = (data.length - 2) / (threshold - 2);
    
    // Always add first point
    sampled.push(data[0]);
    
    let a = 0;  // Previously selected point index
    
    for (let i = 0; i < threshold - 2; i++) {
        // Calculate bucket boundaries
        const avgRangeStart = Math.floor((i + 1) * every) + 1;
        const avgRangeEnd = Math.floor((i + 2) * every) + 1;
        const avgRangeLength = Math.min(avgRangeEnd, data.length) - avgRangeStart;
        
        // Calculate average point in next bucket (for area calculation)
        let avgX = 0;
        let avgY = 0;
        for (let j = avgRangeStart; j < avgRangeStart + avgRangeLength; j++) {
            avgX += data[j].time;
            avgY += data[j].value;
        }
        avgX /= avgRangeLength;
        avgY /= avgRangeLength;
        
        // Current bucket boundaries
        const rangeOffs = Math.floor((i + 0) * every) + 1;
        const rangeTo = Math.floor((i + 1) * every) + 1;
        
        // Find point with largest triangle area
        const pointAX = data[a].time;
        const pointAY = data[a].value;
        
        let maxArea = -1;
        let maxAreaPoint = rangeOffs;
        
        for (let j = rangeOffs; j < rangeTo; j++) {
            // Calculate triangle area
            const area = Math.abs(
                (pointAX - avgX) * (data[j].value - pointAY) -
                (pointAX - data[j].time) * (avgY - pointAY)
            ) * 0.5;
            
            if (area > maxArea) {
                maxArea = area;
                maxAreaPoint = j;
            }
        }
        
        sampled.push(data[maxAreaPoint]);
        a = maxAreaPoint;
    }
    
    // Always add last point
    sampled.push(data[data.length - 1]);
    
    return sampled;
}

// ============================================
// Адаптация для Candlestick данных
// ============================================

interface CandleData {
    time: number;
    open: number;
    high: number;
    low: number;
    close: number;
}

function downsampleCandles(data: CandleData[], threshold: number): CandleData[] {
    if (threshold >= data.length || threshold <= 2) {
        return data;
    }
    
    const bucketSize = Math.ceil(data.length / threshold);
    const result: CandleData[] = [];
    
    for (let i = 0; i < data.length; i += bucketSize) {
        const bucket = data.slice(i, Math.min(i + bucketSize, data.length));
        
        // Агрегируем bucket в одну свечу
        const aggregated: CandleData = {
            time: bucket[0].time,
            open: bucket[0].open,
            high: Math.max(...bucket.map(c => c.high)),
            low: Math.min(...bucket.map(c => c.low)),
            close: bucket[bucket.length - 1].close
        };
        
        result.push(aggregated);
    }
    
    return result;
}

// ============================================
// Умный менеджер с автоматическим downsampling
// ============================================

class SmartDataManager {
    private fullData: CandleData[] = [];
    private series: ISeriesApi<'Candlestick'>;
    private chart: IChartApi;
    
    // Пороги для downsampling
    private readonly DOWNSAMPLE_THRESHOLD = 5000;
    private readonly TARGET_POINTS = 2000;
    
    constructor(chart: IChartApi, series: ISeriesApi<'Candlestick'>) {
        this.chart = chart;
        this.series = series;
        this.setupZoomListener();
    }
    
    public setData(data: CandleData[]): void {
        this.fullData = data;
        this.updateSeriesData();
    }
    
    private updateSeriesData(): void {
        const visibleRange = this.chart.timeScale().getVisibleLogicalRange();
        
        if (!visibleRange) {
            // Первая загрузка — используем downsampling если данных много
            if (this.fullData.length > this.DOWNSAMPLE_THRESHOLD) {
                const downsampled = downsampleCandles(this.fullData, this.TARGET_POINTS);
                this.series.setData(downsampled);
                console.log(`Downsampled: ${this.fullData.length} → ${downsampled.length}`);
            } else {
                this.series.setData(this.fullData);
            }
            return;
        }
        
        // Рассчитываем видимый диапазон
        const visibleCount = Math.ceil(visibleRange.to - visibleRange.from);
        
        if (visibleCount > this.DOWNSAMPLE_THRESHOLD) {
            // Сильный zoom-out — нужен downsampling
            const downsampled = downsampleCandles(this.fullData, this.TARGET_POINTS);
            this.series.setData(downsampled);
        } else {
            // Близкий зум — полные данные
            this.series.setData(this.fullData);
        }
    }
    
    private setupZoomListener(): void {
        let debounceTimer: number;
        
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            clearTimeout(debounceTimer);
            debounceTimer = window.setTimeout(() => {
                this.updateSeriesData();
            }, 150);
        });
    }
}
```

**Плюсы:**
- ✅ Научно-обоснованный алгоритм
- ✅ Сохраняет визуальные характеристики данных
- ✅ Значительное уменьшение точек (100k → 2k)
- ✅ Работает со всеми версиями

**Минусы:**
- ⚠️ Дополнительные вычисления при zoom
- ⚠️ Потеря точности при сильном downsampling
- ⚠️ Нужна адаптация для OHLC данных

---

### Решение 5: Сохранение visible range при prepend
**Оценка: 7/10** ⭐⭐⭐⭐

**Источник:** [GitHub Issue #1875](https://github.com/tradingview/lightweight-charts/issues/1875)

**Суть:** Сохранение и восстановление видимой области при добавлении данных.

```typescript
// ============================================
// Стабильный prepend с сохранением range
// ============================================

class StablePrependManager {
    private series: ISeriesApi<SeriesType>;
    private chart: IChartApi;
    private currentData: SeriesData[] = [];
    
    constructor(chart: IChartApi, series: ISeriesApi<SeriesType>) {
        this.chart = chart;
        this.series = series;
    }
    
    /**
     * Инициализация с исходными данными
     */
    public initialize(data: SeriesData[]): void {
        this.currentData = [...data];
        this.series.setData(this.currentData);
    }
    
    /**
     * Добавление данных в начало с сохранением visible range
     */
    public prepend(newData: SeriesData[]): void {
        // 1. Сохраняем текущий visible range
        const timeScale = this.chart.timeScale();
        const currentLogicalRange = timeScale.getVisibleLogicalRange();
        
        if (!currentLogicalRange) {
            // Нет visible range — просто добавляем данные
            this.currentData = [...newData, ...this.currentData];
            this.series.setData(this.currentData);
            return;
        }
        
        // 2. Запоминаем количество новых точек
        const addedCount = newData.length;
        
        // 3. Обновляем данные
        this.currentData = [...newData, ...this.currentData];
        this.series.setData(this.currentData);
        
        // 4. Восстанавливаем visible range со смещением
        requestAnimationFrame(() => {
            timeScale.setVisibleLogicalRange({
                from: currentLogicalRange.from + addedCount,
                to: currentLogicalRange.to + addedCount
            });
        });
    }
    
    /**
     * Добавление данных в конец (wrapper над update)
     */
    public append(bar: SeriesData): void {
        this.currentData.push(bar);
        this.series.update(bar);
    }
}

// ============================================
// Infinite scroll с preload
// ============================================

class InfiniteScrollLoader {
    private manager: StablePrependManager;
    private chart: IChartApi;
    private fetchHistory: (before: Time) => Promise<SeriesData[]>;
    private isLoading: boolean = false;
    private hasMoreData: boolean = true;
    private readonly PRELOAD_THRESHOLD = -20;  // Logical index от начала
    
    constructor(
        chart: IChartApi,
        series: ISeriesApi<SeriesType>,
        fetchHistory: (before: Time) => Promise<SeriesData[]>
    ) {
        this.chart = chart;
        this.manager = new StablePrependManager(chart, series);
        this.fetchHistory = fetchHistory;
        this.setupScrollListener();
    }
    
    public async initialize(initialData: SeriesData[]): Promise<void> {
        this.manager.initialize(initialData);
        this.hasMoreData = initialData.length > 0;
    }
    
    private setupScrollListener(): void {
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(async (range) => {
            if (!range || this.isLoading || !this.hasMoreData) return;
            
            // Проверяем, близко ли пользователь к началу данных
            if (range.from < this.PRELOAD_THRESHOLD) {
                await this.loadMoreHistory();
            }
        });
    }
    
    private async loadMoreHistory(): Promise<void> {
        this.isLoading = true;
        
        try {
            const oldestTime = this.getOldestTime();
            const newData = await this.fetchHistory(oldestTime);
            
            if (newData.length === 0) {
                this.hasMoreData = false;
                console.log('No more historical data available');
                return;
            }
            
            this.manager.prepend(newData);
            console.log(`Loaded ${newData.length} historical bars`);
            
        } catch (error) {
            console.error('Failed to load history:', error);
        } finally {
            this.isLoading = false;
        }
    }
    
    private getOldestTime(): Time {
        // Получаем время первой точки
        // Это упрощённая версия, реальная реализация зависит от вашего API
        return 0 as Time;
    }
}
```

**Плюсы:**
- ✅ Пользователь не замечает "прыжков"
- ✅ Работает с существующим API
- ✅ Поддержка infinite scroll

**Минусы:**
- ⚠️ `setData()` всё равно перерисовывает всё
- ⚠️ Возможен "jitter" при быстром scroll
- ⚠️ Требует `requestAnimationFrame` хак

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Conflation + update() + Downsampling

Для оптимальной производительности рекомендуется комбинация решений:

```typescript
// ============================================
// Полная реализация: HighPerformanceDataManager
// ============================================

import {
    createChart,
    IChartApi,
    ISeriesApi,
    CandlestickData,
    Time,
    LogicalRange
} from 'lightweight-charts';

interface DataManagerConfig {
    // Data Conflation (v5.1+)
    enableConflation: boolean;
    conflationThresholdFactor: number;
    precomputeConflation: boolean;
    
    // Downsampling
    downsampleThreshold: number;
    targetPoints: number;
    
    // Infinite scroll
    preloadThreshold: number;
}

const DEFAULT_CONFIG: DataManagerConfig = {
    enableConflation: true,
    conflationThresholdFactor: 2.0,
    precomputeConflation: false,
    downsampleThreshold: 10000,
    targetPoints: 3000,
    preloadThreshold: -30
};

class HighPerformanceDataManager {
    private chart: IChartApi;
    private series: ISeriesApi<'Candlestick'>;
    private config: DataManagerConfig;
    
    // Data storage
    private fullData: CandlestickData[] = [];
    private lastTimestamp: number = 0;
    
    // Loading state
    private isLoading: boolean = false;
    private hasMoreData: boolean = true;
    
    // Callbacks
    private fetchHistory?: (before: Time) => Promise<CandlestickData[]>;
    
    constructor(
        container: HTMLElement,
        config: Partial<DataManagerConfig> = {}
    ) {
        this.config = { ...DEFAULT_CONFIG, ...config };
        
        // Создаём chart
        this.chart = createChart(container, {
            width: container.clientWidth,
            height: container.clientHeight,
            layout: {
                background: { color: '#1a1a2e' },
                textColor: '#eee'
            }
        });
        
        // Создаём серию с оптимизациями
        this.series = this.chart.addCandlestickSeries({
            // Data Conflation
            enableConflation: this.config.enableConflation,
            conflationThresholdFactor: this.config.conflationThresholdFactor,
            precomputeConflationOnInit: this.config.precomputeConflation
        });
        
        this.setupListeners();
    }
    
    // ===== PUBLIC API =====
    
    /**
     * Первоначальная загрузка данных
     */
    public setInitialData(data: CandlestickData[]): void {
        this.fullData = [...data];
        
        if (data.length > 0) {
            this.lastTimestamp = data[data.length - 1].time as number;
        }
        
        // Применяем downsampling если нужно
        const displayData = this.getOptimizedData();
        this.series.setData(displayData);
        
        console.log(`Initial load: ${data.length} points, displayed: ${displayData.length}`);
    }
    
    /**
     * Real-time update (новый бар или обновление последнего)
     * ✅ O(1) операция
     */
    public updateBar(bar: CandlestickData): void {
        const barTime = bar.time as number;
        
        if (barTime >= this.lastTimestamp) {
            this.series.update(bar);
            
            // Обновляем локальное хранилище
            if (barTime > this.lastTimestamp) {
                this.fullData.push(bar);
                this.lastTimestamp = barTime;
            } else {
                // Обновление последнего бара
                this.fullData[this.fullData.length - 1] = bar;
            }
        }
    }
    
    /**
     * Регистрация callback для загрузки истории
     */
    public setHistoryFetcher(
        fetcher: (before: Time) => Promise<CandlestickData[]>
    ): void {
        this.fetchHistory = fetcher;
    }
    
    /**
     * Получение статистики
     */
    public getStats(): {
        totalPoints: number;
        displayedPoints: number;
        hasMoreData: boolean;
    } {
        return {
            totalPoints: this.fullData.length,
            displayedPoints: this.series.data().length,
            hasMoreData: this.hasMoreData
        };
    }
    
    /**
     * Освобождение ресурсов
     */
    public dispose(): void {
        this.chart.remove();
    }
    
    // ===== PRIVATE METHODS =====
    
    private setupListeners(): void {
        // Infinite scroll
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(
            this.handleVisibleRangeChange.bind(this)
        );
    }
    
    private async handleVisibleRangeChange(range: LogicalRange | null): Promise<void> {
        if (!range || this.isLoading || !this.hasMoreData || !this.fetchHistory) {
            return;
        }
        
        // Проверяем, нужно ли загрузить больше истории
        if (range.from < this.config.preloadThreshold) {
            await this.loadMoreHistory();
        }
    }
    
    private async loadMoreHistory(): Promise<void> {
        if (!this.fetchHistory) return;
        
        this.isLoading = true;
        
        try {
            const oldestTime = this.fullData[0]?.time || (Date.now() as Time);
            const currentRange = this.chart.timeScale().getVisibleLogicalRange();
            
            const newData = await this.fetchHistory(oldestTime);
            
            if (newData.length === 0) {
                this.hasMoreData = false;
                return;
            }
            
            // Prepend к полным данным
            this.fullData = [...newData, ...this.fullData];
            
            // Обновляем график с сохранением позиции
            const displayData = this.getOptimizedData();
            this.series.setData(displayData);
            
            // Восстанавливаем visible range
            if (currentRange) {
                requestAnimationFrame(() => {
                    this.chart.timeScale().setVisibleLogicalRange({
                        from: currentRange.from + newData.length,
                        to: currentRange.to + newData.length
                    });
                });
            }
            
            console.log(`Loaded ${newData.length} historical bars`);
            
        } catch (error) {
            console.error('History load failed:', error);
        } finally {
            this.isLoading = false;
        }
    }
    
    private getOptimizedData(): CandlestickData[] {
        // Если Data Conflation включён, библиотека сама оптимизирует
        if (this.config.enableConflation) {
            return this.fullData;
        }
        
        // Иначе применяем ручной downsampling
        if (this.fullData.length > this.config.downsampleThreshold) {
            return this.downsampleCandles(this.fullData, this.config.targetPoints);
        }
        
        return this.fullData;
    }
    
    private downsampleCandles(
        data: CandlestickData[],
        threshold: number
    ): CandlestickData[] {
        if (threshold >= data.length) return data;
        
        const bucketSize = Math.ceil(data.length / threshold);
        const result: CandlestickData[] = [];
        
        for (let i = 0; i < data.length; i += bucketSize) {
            const bucket = data.slice(i, Math.min(i + bucketSize, data.length));
            
            result.push({
                time: bucket[0].time,
                open: bucket[0].open,
                high: Math.max(...bucket.map(c => c.high)),
                low: Math.min(...bucket.map(c => c.low)),
                close: bucket[bucket.length - 1].close
            });
        }
        
        return result;
    }
}

// ============================================
// Пример использования
// ============================================

const container = document.getElementById('chart')!;

const manager = new HighPerformanceDataManager(container, {
    enableConflation: true,
    conflationThresholdFactor: 2.0,
    downsampleThreshold: 10000,
    targetPoints: 3000
});

// Загружаем исторические данные
const historicalData = await fetchInitialData();
manager.setInitialData(historicalData);

// Настраиваем infinite scroll
manager.setHistoryFetcher(async (before) => {
    return await api.getHistory({ before, limit: 1000 });
});

// WebSocket для real-time updates
socket.on('candle', (bar) => {
    manager.updateBar(bar);  // ✅ O(1)!
});
```

---

## 📊 Сравнительная таблица

| Критерий | Conflation | update() | Windowing | LTTB | Range Save |
|----------|------------|----------|-----------|------|------------|
| **Оценка** | 9/10 | 9/10 | 8/10 | 8/10 | 7/10 |
| **Сложность** | Низкая | Низкая | Высокая | Средняя | Средняя |
| **Real-time** | ✅ | ✅✅ | ⚠️ | ⚠️ | ✅ |
| **Prepend** | ✅ | ❌ | ✅ | ✅ | ✅ |
| **100k+ точек** | ✅ | ✅ | ✅✅ | ✅ | ⚠️ |
| **Версия** | v5.1+ | Все | Все | Все | Все |
| **Memory** | Среднее | Низкое | Высокое | Низкое | Среднее |

---

## 🔗 Источники

1. [GitHub Issue #838 — Data Transformation Performance](https://github.com/tradingview/lightweight-charts/issues/838)
2. [GitHub Issue #1404 — setData Time Scale Shifting](https://github.com/tradingview/lightweight-charts/issues/1404)
3. [GitHub Issue #1875 — Visible Range Jumps](https://github.com/tradingview/lightweight-charts/issues/1875)
4. [Lightweight Charts v5.1.0 — Data Conflation](https://github.com/tradingview/lightweight-charts/releases/tag/v5.1.0)
5. [TradingView Docs — Real-time Updates](https://tradingview.github.io/lightweight-charts/docs/api)
6. [LTTB Algorithm Paper — Sveinn Steinarsson](https://skemman.is/handle/1946/15343)
7. [npm — downsample package](https://www.npmjs.com/package/downsample)
8. [SciChart — Virtual Data Windowing](https://www.scichart.com/)

---

## 📅 Дата документирования
**18 января 2026**

## ✍️ Статус
Документировано ✅
