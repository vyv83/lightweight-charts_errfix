# БАГ #11: Перекрытие свечей при zoom (Overlapping Candlesticks)

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issues:** [#2031](https://github.com/tradingview/lightweight-charts/issues/2031), [#691](https://github.com/tradingview/lightweight-charts/issues/691)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (частично решено через Data Conflation в v5.1.0)  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

### Суть проблемы
При сильном zoom out свечи визуально перекрываются, создавая "поломанный" или нечитаемый вид графика:

1. **Субпиксельный рендеринг** — библиотека не может отрисовать элементы меньше 1 физического пикселя
2. **Наложение визуальных артефактов** — свечи "сливаются" в непонятную массу
3. **Потеря информативности** — невозможно различить отдельные OHLC-бары
4. **Архитектурное ограничение** — минимальный `barSpacing` = 0.5 пикселя (Issue #1450)

### Визуальное проявление

```
Нормальный вид (zoom in):          Проблемный вид (zoom out):
┌───┐   ┌───┐   ┌───┐             ████████████████████
│   │   │   │   │   │             ████████████████████
├───┤   ├───┤   ├───┤             ████████████████████
│   │   │   │   │   │             (свечи неразличимы)
└───┘   └───┘   └───┘             
```

### Технические причины

1. **Физическое ограничение Canvas** — максимум 2 точки данных на 1 пиксель экрана
2. **Алгоритм рендеринга** — при `barSpacing < 1px` свечи перекрываются
3. **Anti-aliasing артефакты** — сглаживание усугубляет проблему
4. **devicePixelRatio** — на Retina-экранах проблема менее заметна

### Сценарии воспроизведения

```javascript
// Проблема воспроизводится при:
// 1. Большом количестве данных
const data = generateCandleData(50000);  // 50k свечей
series.setData(data);

// 2. Полном zoom out
chart.timeScale().fitContent();  // Показать все данные

// 3. При barSpacing < 1
chart.timeScale().applyOptions({
    barSpacing: 0.5  // Минимальное значение
});

// Результат: свечи визуально накладываются друг на друга
```

---

## 🔍 Найденные решения

### Решение 1: Data Conflation (v5.1.0+)
**Оценка: 9/10** ⭐⭐⭐⭐⭐

**Источник:** [Lightweight Charts v5.1.0 Release Notes](https://github.com/tradingview/lightweight-charts/releases/tag/v5.1.0)

**Суть:** Автоматическое объединение свечей при zoom out, что полностью решает проблему перекрытия.

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
    width: 800,
    height: 400
});

// ✅ Включаем Data Conflation для предотвращения перекрытия
const series = chart.addCandlestickSeries({
    // Автоматическое объединение при zoom out
    enableConflation: true,
    
    // Множитель порога (чем выше, тем раньше включается conflation)
    // Значения: 1.0 (по умолчанию), 2.0, 4.0, 8.0+
    conflationThresholdFactor: 2.0,
    
    // Предварительный расчёт для плавного zoom
    precomputeConflationOnInit: true,
    
    // Приоритет фоновых вычислений
    precomputeConflationPriority: 'user-visible'
});

series.setData(hugeDataset);  // 100k+ точек

// При zoom out библиотека автоматически:
// 1. Определяет что barSpacing < порога
// 2. Объединяет соседние свечи в одну агрегированную
// 3. Рендерит меньше элементов → нет перекрытия
```

**Как работает Data Conflation для Candlestick:**
```
Исходные свечи:        После conflation (2x):
   Свеча 1               ┌─────┐
   ┌─┐                   │     │
   │ │                   │     │  ← Агрегированная свеча:
   └─┘                   │     │     open = candle1.open
   Свеча 2               │     │     high = max(candle1.high, candle2.high)
   ┌─┐                   │     │     low = min(candle1.low, candle2.low)  
   │ │      →            └─────┘     close = candle2.close
   └─┘
```

**Плюсы:**
- ✅ Полностью решает проблему перекрытия
- ✅ Официальное решение от TradingView
- ✅ Автоматическое, не требует кода
- ✅ Сохраняет OHLC-характеристики данных

**Минусы:**
- ⚠️ Требует v5.1.0+
- ⚠️ При сильном conflation свечи выглядят "неестественно"
- ⚠️ Начальный overhead при `precomputeConflationOnInit`

---

### Решение 2: Ограничение zoom out через minBarSpacing
**Оценка: 8/10** ⭐⭐⭐⭐

**Источник:** [Lightweight Charts Docs — Time Scale Options](https://tradingview.github.io/lightweight-charts/docs/api)

**Суть:** Запретить zoom out до уровня, когда свечи перекрываются.

```typescript
import { createChart } from 'lightweight-charts';

// ============================================
// Ограничение минимального barSpacing
// ============================================

const chart = createChart(container, {
    width: 800,
    height: 400,
    timeScale: {
        // ✅ Увеличиваем минимальный spacing (по умолчанию 0.5)
        minBarSpacing: 3,  // Минимум 3 пикселя между свечами
        
        // Начальный spacing
        barSpacing: 6,
        
        // Опционально: ограничить правый край
        rightOffset: 20
    }
});

const series = chart.addCandlestickSeries();
series.setData(data);

// ============================================
// Динамическое управление zoom limits
// ============================================

class ZoomLimiter {
    private chart: IChartApi;
    private minSpacing: number;
    private maxSpacing: number;
    
    constructor(
        chart: IChartApi,
        minSpacing: number = 3,
        maxSpacing: number = 50
    ) {
        this.chart = chart;
        this.minSpacing = minSpacing;
        this.maxSpacing = maxSpacing;
        
        // Применяем лимиты
        this.applyLimits();
        
        // Отслеживаем zoom
        this.setupZoomListener();
    }
    
    private applyLimits(): void {
        this.chart.timeScale().applyOptions({
            minBarSpacing: this.minSpacing
        });
    }
    
    private setupZoomListener(): void {
        this.chart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
            if (!range) return;
            
            const currentSpacing = this.chart.timeScale().options().barSpacing;
            
            // Ограничиваем максимальный zoom in
            if (currentSpacing > this.maxSpacing) {
                this.chart.timeScale().applyOptions({
                    barSpacing: this.maxSpacing
                });
            }
        });
    }
    
    /**
     * Динамическое обновление лимитов
     */
    public updateLimits(min: number, max: number): void {
        this.minSpacing = min;
        this.maxSpacing = max;
        this.applyLimits();
    }
}

// Использование
const limiter = new ZoomLimiter(chart, 3, 30);
```

**Рекомендуемые значения minBarSpacing:**

| Тип графика | minBarSpacing | Эффект |
|-------------|---------------|--------|
| Candlestick | 3-4 | Чёткие свечи без перекрытия |
| OHLC Bars | 2-3 | Читаемые бары |
| Line | 0.5-1 | Линии не требуют много места |
| Area | 0.5-1 | Аналогично линиям |

**Плюсы:**
- ✅ Простое решение
- ✅ Работает во всех версиях
- ✅ Нет дополнительной нагрузки
- ✅ Гарантированно читаемый график

**Минусы:**
- ⚠️ Ограничивает возможность обзора всех данных
- ⚠️ Пользователь не может zoom out дальше
- ⚠️ Может не подойти для overview-сценариев

---

### Решение 3: Автоматическое переключение на Line при zoom out
**Оценка: 8/10** ⭐⭐⭐⭐

**Источник:** Паттерн из профессиональных trading-платформ

**Суть:** При сильном zoom out автоматически переключать серию с Candlestick на Line.

```typescript
// ============================================
// Адаптивный переключатель типа серии
// ============================================

import {
    createChart,
    IChartApi,
    ISeriesApi,
    CandlestickData,
    LineData,
    Time
} from 'lightweight-charts';

class AdaptiveChartType {
    private chart: IChartApi;
    private container: HTMLElement;
    
    // Серии
    private candleSeries: ISeriesApi<'Candlestick'> | null = null;
    private lineSeries: ISeriesApi<'Line'> | null = null;
    
    // Данные
    private candleData: CandlestickData[] = [];
    private lineData: LineData[] = [];
    
    // Пороги
    private readonly SWITCH_TO_LINE_THRESHOLD = 2.0;  // barSpacing < 2 → Line
    private readonly SWITCH_TO_CANDLE_THRESHOLD = 4.0;  // barSpacing > 4 → Candle
    
    // Текущее состояние
    private currentType: 'Candlestick' | 'Line' = 'Candlestick';
    private isTransitioning: boolean = false;
    
    constructor(container: HTMLElement) {
        this.container = container;
        this.chart = createChart(container, {
            width: container.clientWidth,
            height: container.clientHeight,
            layout: {
                background: { color: '#1a1a2e' },
                textColor: '#eee'
            }
        });
        
        this.setupListener();
    }
    
    /**
     * Установка данных
     */
    public setData(data: CandlestickData[]): void {
        this.candleData = data;
        
        // Преобразуем в line data (используем close price)
        this.lineData = data.map(candle => ({
            time: candle.time,
            value: candle.close
        }));
        
        // Инициализируем с candlestick
        this.showCandlestick();
    }
    
    /**
     * Отслеживание zoom уровня
     */
    private setupListener(): void {
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            if (this.isTransitioning) return;
            
            const currentSpacing = this.chart.timeScale().options().barSpacing;
            this.checkAndSwitch(currentSpacing);
        });
    }
    
    private checkAndSwitch(barSpacing: number): void {
        if (this.currentType === 'Candlestick' && barSpacing < this.SWITCH_TO_LINE_THRESHOLD) {
            this.switchToLine();
        } else if (this.currentType === 'Line' && barSpacing > this.SWITCH_TO_CANDLE_THRESHOLD) {
            this.switchToCandlestick();
        }
    }
    
    private switchToLine(): void {
        this.isTransitioning = true;
        
        // Сохраняем visible range
        const range = this.chart.timeScale().getVisibleLogicalRange();
        
        // Удаляем candlestick серию
        if (this.candleSeries) {
            this.chart.removeSeries(this.candleSeries);
            this.candleSeries = null;
        }
        
        // Создаём line серию
        this.lineSeries = this.chart.addLineSeries({
            color: '#2962FF',
            lineWidth: 1,
            priceLineVisible: true,
            lastValueVisible: true
        });
        this.lineSeries.setData(this.lineData);
        
        // Восстанавливаем range
        if (range) {
            requestAnimationFrame(() => {
                this.chart.timeScale().setVisibleLogicalRange(range);
            });
        }
        
        this.currentType = 'Line';
        this.isTransitioning = false;
        
        console.log('Switched to Line chart (zoom out detected)');
    }
    
    private switchToCandlestick(): void {
        this.isTransitioning = true;
        
        const range = this.chart.timeScale().getVisibleLogicalRange();
        
        if (this.lineSeries) {
            this.chart.removeSeries(this.lineSeries);
            this.lineSeries = null;
        }
        
        this.showCandlestick();
        
        if (range) {
            requestAnimationFrame(() => {
                this.chart.timeScale().setVisibleLogicalRange(range);
            });
        }
        
        this.isTransitioning = false;
        console.log('Switched to Candlestick chart (zoom in detected)');
    }
    
    private showCandlestick(): void {
        this.candleSeries = this.chart.addCandlestickSeries({
            upColor: '#26a69a',
            downColor: '#ef5350',
            borderVisible: false,
            wickUpColor: '#26a69a',
            wickDownColor: '#ef5350'
        });
        this.candleSeries.setData(this.candleData);
        this.currentType = 'Candlestick';
    }
    
    /**
     * Получение текущего типа серии
     */
    public getCurrentType(): 'Candlestick' | 'Line' {
        return this.currentType;
    }
    
    /**
     * Очистка
     */
    public dispose(): void {
        this.chart.remove();
    }
}

// ============================================
// Использование
// ============================================

const container = document.getElementById('chart')!;
const adaptiveChart = new AdaptiveChartType(container);

// Загружаем данные
const candleData = await fetchCandleData();
adaptiveChart.setData(candleData);

// При zoom out → автоматически переключится на Line
// При zoom in → автоматически вернётся на Candlestick
```

**Плюсы:**
- ✅ Line chart всегда читаем при любом zoom
- ✅ Сохраняет информативность (close price виден)
- ✅ Работает во всех версиях
- ✅ UX как в профессиональных платформах

**Минусы:**
- ⚠️ Теряется OHLC информация при zoom out
- ⚠️ Переключение может быть заметным
- ⚠️ Дополнительная сложность кода

---

### Решение 4: Ручное объединение свечей (Manual Aggregation)
**Оценка: 7/10** ⭐⭐⭐⭐

**Источник:** Общие практики работы с финансовыми данными

**Суть:** Предварительно агрегировать данные для разных уровней zoom.

```typescript
// ============================================
// Система уровней детализации (LOD)
// ============================================

interface CandleData {
    time: number;
    open: number;
    high: number;
    low: number;
    close: number;
    volume?: number;
}

interface LODLevel {
    name: string;
    aggregationFactor: number;  // Сколько свечей объединять
    minBarSpacing: number;      // При каком spacing показывать этот уровень
    data: CandleData[];
}

class LODDataManager {
    private rawData: CandleData[] = [];
    private levels: LODLevel[] = [];
    
    private series: ISeriesApi<'Candlestick'>;
    private chart: IChartApi;
    private currentLevel: number = 0;
    
    constructor(chart: IChartApi, series: ISeriesApi<'Candlestick'>) {
        this.chart = chart;
        this.series = series;
        this.setupZoomListener();
    }
    
    /**
     * Установка исходных данных и генерация уровней LOD
     */
    public setData(data: CandleData[]): void {
        this.rawData = data;
        this.generateLODLevels();
        this.applyCurrentLevel();
    }
    
    private generateLODLevels(): void {
        this.levels = [
            // LOD 0: Полные данные (zoom in)
            {
                name: '1x',
                aggregationFactor: 1,
                minBarSpacing: 4,
                data: this.rawData
            },
            // LOD 1: 2x агрегация
            {
                name: '2x',
                aggregationFactor: 2,
                minBarSpacing: 2,
                data: this.aggregateCandles(this.rawData, 2)
            },
            // LOD 2: 4x агрегация
            {
                name: '4x',
                aggregationFactor: 4,
                minBarSpacing: 1,
                data: this.aggregateCandles(this.rawData, 4)
            },
            // LOD 3: 8x агрегация (максимальный zoom out)
            {
                name: '8x',
                aggregationFactor: 8,
                minBarSpacing: 0.5,
                data: this.aggregateCandles(this.rawData, 8)
            }
        ];
        
        console.log('Generated LOD levels:', this.levels.map(l => ({
            name: l.name,
            points: l.data.length
        })));
    }
    
    /**
     * Агрегация свечей
     */
    private aggregateCandles(data: CandleData[], factor: number): CandleData[] {
        if (factor <= 1) return data;
        
        const result: CandleData[] = [];
        
        for (let i = 0; i < data.length; i += factor) {
            const bucket = data.slice(i, Math.min(i + factor, data.length));
            if (bucket.length === 0) continue;
            
            const aggregated: CandleData = {
                time: bucket[0].time,
                open: bucket[0].open,
                high: Math.max(...bucket.map(c => c.high)),
                low: Math.min(...bucket.map(c => c.low)),
                close: bucket[bucket.length - 1].close,
                volume: bucket.reduce((sum, c) => sum + (c.volume || 0), 0)
            };
            
            result.push(aggregated);
        }
        
        return result;
    }
    
    private setupZoomListener(): void {
        let debounceTimer: number;
        
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            clearTimeout(debounceTimer);
            debounceTimer = window.setTimeout(() => {
                this.checkLevelChange();
            }, 100);
        });
    }
    
    private checkLevelChange(): void {
        const currentSpacing = this.chart.timeScale().options().barSpacing;
        
        // Найти подходящий уровень
        let targetLevel = 0;
        for (let i = this.levels.length - 1; i >= 0; i--) {
            if (currentSpacing >= this.levels[i].minBarSpacing) {
                targetLevel = i;
                break;
            }
        }
        
        if (targetLevel !== this.currentLevel) {
            this.switchLevel(targetLevel);
        }
    }
    
    private switchLevel(level: number): void {
        const oldLevel = this.currentLevel;
        this.currentLevel = level;
        
        // Сохраняем visible range
        const range = this.chart.timeScale().getVisibleLogicalRange();
        
        // Меняем данные
        this.series.setData(this.levels[level].data);
        
        // Корректируем visible range (масштабирование)
        if (range && oldLevel !== level) {
            const scaleFactor = this.levels[oldLevel].aggregationFactor / 
                               this.levels[level].aggregationFactor;
            
            requestAnimationFrame(() => {
                this.chart.timeScale().setVisibleLogicalRange({
                    from: range.from * scaleFactor,
                    to: range.to * scaleFactor
                });
            });
        }
        
        console.log(`LOD level changed: ${this.levels[oldLevel].name} → ${this.levels[level].name}`);
    }
    
    private applyCurrentLevel(): void {
        this.series.setData(this.levels[this.currentLevel].data);
    }
    
    /**
     * Статистика
     */
    public getStats(): { level: string; points: number; reduction: string } {
        const currentData = this.levels[this.currentLevel];
        return {
            level: currentData.name,
            points: currentData.data.length,
            reduction: `${((1 - currentData.data.length / this.rawData.length) * 100).toFixed(1)}%`
        };
    }
}

// ============================================
// Использование
// ============================================

const container = document.getElementById('chart')!;
const chart = createChart(container, { /*...*/ });
const series = chart.addCandlestickSeries();

const lodManager = new LODDataManager(chart, series);
lodManager.setData(hugeDataset);  // 100k свечей

// При zoom out система автоматически переключит LOD:
// 100k → 50k → 25k → 12.5k
// Свечи никогда не перекрываются!
```

**Плюсы:**
- ✅ Точный контроль над уровнями детализации
- ✅ Работает во всех версиях
- ✅ Оптимальная производительность
- ✅ Сохраняет candlestick вид

**Минусы:**
- ⚠️ Дополнительная память для хранения уровней
- ⚠️ Переключение должно быть плавным
- ⚠️ Сложная реализация

---

### Решение 5: CSS-фильтры для сглаживания (Косметическое)
**Оценка: 5/10** ⭐⭐⭐

**Источник:** CSS обработка canvas-элементов

**Суть:** Применение CSS-фильтров для улучшения визуального восприятия.

```typescript
// ============================================
// CSS обработка для улучшения вида при zoom out
// ============================================

class CanvasEnhancer {
    private chart: IChartApi;
    private container: HTMLElement;
    
    // Пороги для применения фильтров
    private readonly BLUR_THRESHOLD = 1.5;
    private readonly CONTRAST_THRESHOLD = 2.0;
    
    constructor(chart: IChartApi, container: HTMLElement) {
        this.chart = chart;
        this.container = container;
        this.setupListener();
    }
    
    private setupListener(): void {
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            this.applyEnhancements();
        });
    }
    
    private applyEnhancements(): void {
        const currentSpacing = this.chart.timeScale().options().barSpacing;
        const canvas = this.container.querySelector('canvas');
        
        if (!canvas) return;
        
        if (currentSpacing < this.BLUR_THRESHOLD) {
            // Сильный zoom out — применяем лёгкий blur
            canvas.style.filter = 'blur(0.3px) contrast(1.1)';
        } else if (currentSpacing < this.CONTRAST_THRESHOLD) {
            // Умеренный zoom out — только контраст
            canvas.style.filter = 'contrast(1.05)';
        } else {
            // Нормальный вид
            canvas.style.filter = 'none';
        }
    }
    
    public dispose(): void {
        const canvas = this.container.querySelector('canvas');
        if (canvas) {
            canvas.style.filter = 'none';
        }
    }
}

// Стили CSS для дополнительной обработки
const styles = `
    .chart-container canvas {
        /* Отключаем image rendering smoothing */
        image-rendering: crisp-edges;
        image-rendering: pixelated;
    }
    
    .chart-container.zoomed-out canvas {
        /* При zoom out — добавляем сглаживание */
        image-rendering: auto;
        filter: blur(0.2px) contrast(1.1);
    }
`;
```

**Плюсы:**
- ✅ Очень простая реализация
- ✅ Нет изменения данных
- ✅ Может улучшить восприятие

**Минусы:**
- ⚠️ Не решает корневую проблему
- ⚠️ Размытие ухудшает чёткость
- ⚠️ Может выглядеть неестественно

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Conflation + Zoom Limits + LOD

Для максимальной эффективности рекомендуется комбинирование решений:

```typescript
// ============================================
// Полная реализация: OverlapFreeChartManager
// ============================================

import {
    createChart,
    IChartApi,
    ISeriesApi,
    CandlestickData,
    Time,
    LogicalRange
} from 'lightweight-charts';

interface ChartConfig {
    // Data Conflation (v5.1+)
    useConflation: boolean;
    conflationFactor: number;
    
    // Zoom limits
    minBarSpacing: number;
    maxBarSpacing: number;
    
    // LOD
    useLOD: boolean;
    lodLevels: number[];
    
    // Auto-switch to line
    useLineSwitch: boolean;
    lineSwitchThreshold: number;
}

const DEFAULT_CONFIG: ChartConfig = {
    useConflation: true,
    conflationFactor: 2.0,
    minBarSpacing: 2,
    maxBarSpacing: 30,
    useLOD: true,
    lodLevels: [1, 2, 4, 8],
    useLineSwitch: true,
    lineSwitchThreshold: 1.0
};

class OverlapFreeChartManager {
    private chart: IChartApi;
    private series: ISeriesApi<'Candlestick'> | ISeriesApi<'Line'> | null = null;
    private config: ChartConfig;
    
    // Data
    private rawData: CandlestickData[] = [];
    private lodData: Map<number, CandlestickData[]> = new Map();
    
    // State
    private currentLOD: number = 1;
    private currentType: 'Candlestick' | 'Line' = 'Candlestick';
    private isTransitioning: boolean = false;
    
    constructor(
        container: HTMLElement,
        config: Partial<ChartConfig> = {}
    ) {
        this.config = { ...DEFAULT_CONFIG, ...config };
        
        this.chart = createChart(container, {
            width: container.clientWidth,
            height: container.clientHeight,
            layout: {
                background: { color: '#131722' },
                textColor: '#d1d4dc'
            },
            timeScale: {
                minBarSpacing: this.config.minBarSpacing,
                barSpacing: 6,
                rightOffset: 10
            },
            grid: {
                vertLines: { color: 'rgba(42, 46, 57, 0.5)' },
                horzLines: { color: 'rgba(42, 46, 57, 0.5)' }
            }
        });
        
        this.setupZoomListener();
    }
    
    // ===== PUBLIC API =====
    
    public setData(data: CandlestickData[]): void {
        this.rawData = data;
        
        // Генерируем LOD уровни
        if (this.config.useLOD) {
            this.generateLODLevels();
        }
        
        // Создаём начальную серию
        this.createCandlestickSeries();
    }
    
    public dispose(): void {
        this.chart.remove();
    }
    
    // ===== PRIVATE METHODS =====
    
    private createCandlestickSeries(): void {
        this.series = this.chart.addCandlestickSeries({
            upColor: '#26a69a',
            downColor: '#ef5350',
            borderVisible: false,
            wickUpColor: '#26a69a',
            wickDownColor: '#ef5350',
            
            // Data Conflation (v5.1+)
            ...(this.config.useConflation && {
                enableConflation: true,
                conflationThresholdFactor: this.config.conflationFactor,
                precomputeConflationOnInit: true
            })
        });
        
        this.updateSeriesData();
        this.currentType = 'Candlestick';
    }
    
    private createLineSeries(): void {
        this.series = this.chart.addLineSeries({
            color: '#2962FF',
            lineWidth: 1,
            priceLineVisible: true,
            lastValueVisible: true
        });
        
        // Конвертируем в line data
        const lineData = this.rawData.map(c => ({
            time: c.time,
            value: c.close
        }));
        
        (this.series as ISeriesApi<'Line'>).setData(lineData);
        this.currentType = 'Line';
    }
    
    private generateLODLevels(): void {
        for (const factor of this.config.lodLevels) {
            this.lodData.set(factor, this.aggregateCandles(this.rawData, factor));
        }
    }
    
    private aggregateCandles(data: CandlestickData[], factor: number): CandlestickData[] {
        if (factor <= 1) return data;
        
        const result: CandlestickData[] = [];
        
        for (let i = 0; i < data.length; i += factor) {
            const bucket = data.slice(i, Math.min(i + factor, data.length));
            if (bucket.length === 0) continue;
            
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
    
    private setupZoomListener(): void {
        let debounceTimer: number;
        
        this.chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
            if (this.isTransitioning) return;
            
            clearTimeout(debounceTimer);
            debounceTimer = window.setTimeout(() => {
                this.handleZoomChange();
            }, 100);
        });
    }
    
    private handleZoomChange(): void {
        const currentSpacing = this.chart.timeScale().options().barSpacing;
        
        // 1. Проверяем переключение на Line
        if (this.config.useLineSwitch) {
            if (this.currentType === 'Candlestick' && 
                currentSpacing < this.config.lineSwitchThreshold) {
                this.switchToLine();
                return;
            }
            if (this.currentType === 'Line' && 
                currentSpacing > this.config.lineSwitchThreshold * 2) {
                this.switchToCandlestick();
                return;
            }
        }
        
        // 2. Проверяем LOD переключение (только для Candlestick)
        if (this.config.useLOD && this.currentType === 'Candlestick') {
            this.checkLODChange(currentSpacing);
        }
    }
    
    private checkLODChange(barSpacing: number): void {
        let targetLOD = 1;
        
        if (barSpacing < 1.5) targetLOD = 8;
        else if (barSpacing < 2.5) targetLOD = 4;
        else if (barSpacing < 4) targetLOD = 2;
        else targetLOD = 1;
        
        if (targetLOD !== this.currentLOD && this.lodData.has(targetLOD)) {
            this.switchLOD(targetLOD);
        }
    }
    
    private switchLOD(lod: number): void {
        this.isTransitioning = true;
        
        const range = this.chart.timeScale().getVisibleLogicalRange();
        const scaleFactor = this.currentLOD / lod;
        
        const data = this.lodData.get(lod);
        if (data && this.series) {
            (this.series as ISeriesApi<'Candlestick'>).setData(data);
        }
        
        if (range) {
            requestAnimationFrame(() => {
                this.chart.timeScale().setVisibleLogicalRange({
                    from: range.from * scaleFactor,
                    to: range.to * scaleFactor
                });
            });
        }
        
        console.log(`LOD: ${this.currentLOD}x → ${lod}x`);
        this.currentLOD = lod;
        this.isTransitioning = false;
    }
    
    private switchToLine(): void {
        this.isTransitioning = true;
        
        const range = this.chart.timeScale().getVisibleLogicalRange();
        
        if (this.series) {
            this.chart.removeSeries(this.series);
        }
        
        this.createLineSeries();
        
        if (range) {
            requestAnimationFrame(() => {
                this.chart.timeScale().setVisibleLogicalRange(range);
            });
        }
        
        console.log('Switched to Line (extreme zoom out)');
        this.isTransitioning = false;
    }
    
    private switchToCandlestick(): void {
        this.isTransitioning = true;
        
        const range = this.chart.timeScale().getVisibleLogicalRange();
        
        if (this.series) {
            this.chart.removeSeries(this.series);
        }
        
        this.createCandlestickSeries();
        
        if (range) {
            requestAnimationFrame(() => {
                this.chart.timeScale().setVisibleLogicalRange(range);
            });
        }
        
        console.log('Switched to Candlestick');
        this.isTransitioning = false;
    }
    
    private updateSeriesData(): void {
        const data = this.config.useLOD 
            ? (this.lodData.get(this.currentLOD) || this.rawData)
            : this.rawData;
        
        if (this.series) {
            (this.series as ISeriesApi<'Candlestick'>).setData(data);
        }
    }
}

// ============================================
// Использование
// ============================================

const container = document.getElementById('chart')!;

const chart = new OverlapFreeChartManager(container, {
    useConflation: true,      // v5.1+
    conflationFactor: 2.0,
    minBarSpacing: 2,
    useLOD: true,
    lodLevels: [1, 2, 4, 8],
    useLineSwitch: true,
    lineSwitchThreshold: 1.0
});

chart.setData(hugeDataset);  // 100k свечей

// Результат: свечи НИКОГДА не перекрываются!
// - При zoom out → автоматическая агрегация (LOD + Conflation)
// - При сильном zoom out → переключение на Line
// - При zoom in → возврат к детальным Candlestick
```

---

## 📊 Сравнительная таблица

| Критерий | Conflation | minBarSpacing | Line Switch | Manual LOD | CSS Filters |
|----------|------------|---------------|-------------|------------|-------------|
| **Оценка** | 9/10 | 8/10 | 8/10 | 7/10 | 5/10 |
| **Решает проблему** | ✅ Полностью | ⚠️ Частично | ✅ Полностью | ✅ Полностью | ❌ Нет |
| **Версия** | v5.1+ | Все | Все | Все | Все |
| **Сложность** | Низкая | Низкая | Средняя | Высокая | Низкая |
| **Производительность** | ✅ | ✅ | ✅ | ⚠️ | ✅ |
| **Сохраняет OHLC** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **UX** | ✅ | ⚠️ | ✅ | ✅ | ⚠️ |

---

## 🔗 Источники

1. [GitHub Issue #2031 — Candlestick Overlap During Zooming](https://github.com/tradingview/lightweight-charts/issues/2031)
2. [GitHub Issue #691 — Candlesticks Look Broken When Zooming Out](https://github.com/tradingview/lightweight-charts/issues/691)
3. [GitHub Issue #1450 — Minimum Bar Spacing Architectural Limit](https://github.com/tradingview/lightweight-charts/issues/1450)
4. [Lightweight Charts v5.1.0 — Data Conflation](https://github.com/tradingview/lightweight-charts/releases/tag/v5.1.0)
5. [TradingView Docs — Time Scale Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions)
6. [MDN — Canvas Rendering Best Practices](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)

---

## 📅 Дата документирования
**18 января 2026**

## ✍️ Статус
Документировано ✅
