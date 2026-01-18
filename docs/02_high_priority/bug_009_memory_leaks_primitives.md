# БАГ #9: Утечки памяти при добавлении/удалении примитивов

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issues:** [#946](https://github.com/tradingview/lightweight-charts/issues/946), [#2000](https://github.com/tradingview/lightweight-charts/issues/2000)  
> **Версии:** v5.0+  
> **Статус:** 🔴 Open (октябрь 2025)  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

### Суть проблемы
При частом добавлении и удалении custom primitives через `detachPrimitive()` происходит постепенное увеличение потребления памяти. Память не освобождается корректно, что приводит к:

1. **Накоплению данных в internal buffers** — примитивы остаются в памяти после удаления
2. **Персистентной задержке рендеринга** — даже после удаления всех примитивов график работает медленнее
3. **Деградации FPS** — время кадра увеличивается с 16ms до 60-80ms (с 60 FPS до 10-15 FPS)
4. **Невосстанавливаемой производительности** — лаг становится "новой нормой"

### Сценарии воспроизведения

```javascript
// Проблемный код — частое создание/удаление примитивов
for (let i = 0; i < 100; i++) {
    const primitive = new MyCustomPrimitive();
    series.attachPrimitive(primitive);
    
    // Позже...
    series.detachPrimitive(primitive);  // ❌ Утечка памяти!
}
```

### Корневые причины (из Issue #2000)

1. **`detachPrimitive()` не очищает internal references** — ссылки на примитив сохраняются внутри библиотеки
2. **Rendering pipeline продолжает обрабатывать detached primitives** — цикл рендеринга итерируется по "призракам"
3. **Reference cleanup failures** — detached primitives не попадают в garbage collection
4. **Invalidation cascades** — каскадные инвалидации в rendering loop

---

## 🔍 Найденные решения

### Решение 1: Отключение вместо удаления (Официальный workaround)
**Оценка: 9/10** ⭐⭐⭐⭐⭐

**Источник:** [GitHub Issue #2000](https://github.com/tradingview/lightweight-charts/issues/2000)

**Суть:** Вместо вызова `detachPrimitive()` — отключить примитив, очистив его состояние.

```typescript
class SafeCustomPrimitive implements ISeriesPrimitive<Time> {
    private _isActive: boolean = true;
    private _paneViews: IPrimitivePaneView[] = [];
    
    // Метод для "безопасного удаления"
    public disable(): void {
        this._isActive = false;
        this._paneViews = [];  // Очищаем views
        // НЕ вызываем detachPrimitive()!
    }
    
    // Возвращаем пустой массив когда отключены
    public paneViews(): readonly IPrimitivePaneView[] {
        return this._isActive ? this._paneViews : [];
    }
    
    public priceAxisViews(): readonly ISeriesPrimitiveAxisView[] {
        return this._isActive ? this._priceAxisViews : [];
    }
    
    public timeAxisViews(): readonly ISeriesPrimitiveAxisView[] {
        return this._isActive ? this._timeAxisViews : [];
    }
}
```

**Плюсы:**
- ✅ Официальная рекомендация от разработчиков
- ✅ Полностью избегает проблемы с `detachPrimitive()`
- ✅ Примитив не рендерится (пустые views)
- ✅ Минимальные изменения кода

**Минусы:**
- ⚠️ Примитив остаётся в internal state библиотеки
- ⚠️ Небольшой overhead памяти от "спящих" примитивов

---

### Решение 2: Object Pooling Pattern
**Оценка: 8/10** ⭐⭐⭐⭐

**Источник:** [web.dev — Object Pooling](https://web.dev/), [Medium — Memory Optimization](https://medium.com/)

**Суть:** Переиспользование объектов примитивов вместо создания новых.

```typescript
class PrimitivePool<T extends ISeriesPrimitive<Time>> {
    private pool: T[] = [];
    private activeCount: number = 0;
    private factory: () => T;
    
    constructor(factory: () => T, initialSize: number = 10) {
        this.factory = factory;
        // Предварительное создание объектов
        for (let i = 0; i < initialSize; i++) {
            this.pool.push(factory());
        }
    }
    
    public acquire(): T {
        if (this.pool.length > 0) {
            const primitive = this.pool.pop()!;
            this.activeCount++;
            return primitive;
        }
        // Расширяем пул при необходимости
        this.activeCount++;
        return this.factory();
    }
    
    public release(primitive: T): void {
        // Сбрасываем состояние примитива
        if ('reset' in primitive && typeof primitive.reset === 'function') {
            (primitive as any).reset();
        }
        this.pool.push(primitive);
        this.activeCount--;
    }
    
    public get stats(): { active: number; pooled: number } {
        return {
            active: this.activeCount,
            pooled: this.pool.length
        };
    }
}

// Использование
const linePool = new PrimitivePool(() => new TrendLinePrimitive(), 20);

// Получить примитив из пула
const line = linePool.acquire();
series.attachPrimitive(line);

// Вернуть в пул вместо удаления
linePool.release(line);
// line остаётся attached, но "сброшен"
```

**Плюсы:**
- ✅ Нулевое создание новых объектов после инициализации
- ✅ Нет вызовов `detachPrimitive()`
- ✅ Предсказуемое потребление памяти
- ✅ Снижение нагрузки на Garbage Collector

**Минусы:**
- ⚠️ Требует архитектурных изменений
- ⚠️ Необходимо правильно реализовать `reset()`
- ⚠️ Начальный overhead памяти от пула

---

### Решение 3: Lazy Detachment с батчингом
**Оценка: 7/10** ⭐⭐⭐⭐

**Источник:** Общие практики оптимизации canvas-библиотек

**Суть:** Накопление примитивов для удаления и batch-удаление с принудительной очисткой.

```typescript
class PrimitiveManager {
    private series: ISeriesApi<SeriesType>;
    private activePrimitives: Map<string, ISeriesPrimitive<Time>> = new Map();
    private pendingRemoval: ISeriesPrimitive<Time>[] = [];
    private cleanupScheduled: boolean = false;
    
    constructor(series: ISeriesApi<SeriesType>) {
        this.series = series;
    }
    
    public add(id: string, primitive: ISeriesPrimitive<Time>): void {
        this.activePrimitives.set(id, primitive);
        this.series.attachPrimitive(primitive);
    }
    
    public remove(id: string): void {
        const primitive = this.activePrimitives.get(id);
        if (primitive) {
            this.activePrimitives.delete(id);
            
            // Отключаем рендеринг немедленно
            if ('disable' in primitive) {
                (primitive as any).disable();
            }
            
            // Добавляем в очередь на удаление
            this.pendingRemoval.push(primitive);
            this.scheduleCleanup();
        }
    }
    
    private scheduleCleanup(): void {
        if (this.cleanupScheduled) return;
        this.cleanupScheduled = true;
        
        // Отложенная очистка для батчинга
        requestIdleCallback(() => {
            this.performCleanup();
        }, { timeout: 5000 });
    }
    
    private performCleanup(): void {
        this.cleanupScheduled = false;
        
        // Batch detach
        for (const primitive of this.pendingRemoval) {
            try {
                this.series.detachPrimitive(primitive);
            } catch (e) {
                console.warn('Failed to detach primitive:', e);
            }
        }
        
        // Очищаем ссылки
        this.pendingRemoval = [];
        
        // Принудительный GC hint (только для профилирования)
        if (typeof gc === 'function') {
            gc();
        }
    }
    
    public dispose(): void {
        this.activePrimitives.clear();
        this.pendingRemoval = [];
    }
}
```

**Плюсы:**
- ✅ Немедленное отключение рендеринга
- ✅ Отложенное удаление снижает нагрузку
- ✅ Централизованное управление

**Минусы:**
- ⚠️ Всё ещё использует `detachPrimitive()` (риск утечки)
- ⚠️ Сложнее в реализации
- ⚠️ `requestIdleCallback` не везде поддерживается

---

### Решение 4: WeakRef + FinalizationRegistry
**Оценка: 6/10** ⭐⭐⭐

**Источник:** [MDN — WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef)

**Суть:** Использование слабых ссылок для автоматической очистки.

```typescript
class WeakPrimitiveTracker {
    private registry: FinalizationRegistry<string>;
    private primitiveRefs: Map<string, WeakRef<ISeriesPrimitive<Time>>> = new Map();
    
    constructor(onCleanup: (id: string) => void) {
        this.registry = new FinalizationRegistry((id: string) => {
            console.log(`Primitive ${id} was garbage collected`);
            this.primitiveRefs.delete(id);
            onCleanup(id);
        });
    }
    
    public track(id: string, primitive: ISeriesPrimitive<Time>): void {
        const weakRef = new WeakRef(primitive);
        this.primitiveRefs.set(id, weakRef);
        this.registry.register(primitive, id, primitive);
    }
    
    public untrack(id: string, primitive: ISeriesPrimitive<Time>): void {
        this.registry.unregister(primitive);
        this.primitiveRefs.delete(id);
    }
    
    public get(id: string): ISeriesPrimitive<Time> | undefined {
        const ref = this.primitiveRefs.get(id);
        return ref?.deref();
    }
}

// Использование
const tracker = new WeakPrimitiveTracker((id) => {
    console.log(`Cleanup triggered for ${id}`);
});

const primitive = new MyPrimitive();
tracker.track('line-1', primitive);

// Позже: просто удаляем сильную ссылку
primitive = null;  // GC автоматически очистит
```

**Плюсы:**
- ✅ Автоматическая очистка при GC
- ✅ Современный подход
- ✅ Нет ручного управления lifecycle

**Минусы:**
- ⚠️ GC непредсказуем — нельзя гарантировать время очистки
- ⚠️ Не работает в старых браузерах (Node 14+, Chrome 84+)
- ⚠️ Не решает проблему internal references в библиотеке

---

### Решение 5: Пересоздание графика при достижении порога
**Оценка: 5/10** ⭐⭐⭐

**Источник:** Практический workaround из community

**Суть:** Полное пересоздание chart instance после определённого количества операций.

```typescript
class ChartInstanceManager {
    private container: HTMLElement;
    private options: ChartOptions;
    private chart: IChartApi | null = null;
    private operationCount: number = 0;
    private readonly THRESHOLD: number = 500;  // Пересоздать после 500 операций
    
    constructor(container: HTMLElement, options: ChartOptions) {
        this.container = container;
        this.options = options;
        this.createChart();
    }
    
    private createChart(): void {
        if (this.chart) {
            this.chart.remove();
        }
        this.chart = createChart(this.container, this.options);
        this.operationCount = 0;
    }
    
    public getChart(): IChartApi {
        if (!this.chart) throw new Error('Chart not initialized');
        return this.chart;
    }
    
    public trackOperation(): void {
        this.operationCount++;
        if (this.operationCount >= this.THRESHOLD) {
            console.log('Threshold reached, recreating chart...');
            this.saveState();
            this.createChart();
            this.restoreState();
        }
    }
    
    private saveState(): void {
        // Сохранить visible range, данные серий, etc.
    }
    
    private restoreState(): void {
        // Восстановить состояние
    }
    
    public dispose(): void {
        this.chart?.remove();
        this.chart = null;
    }
}
```

**Плюсы:**
- ✅ Гарантированная очистка всей памяти
- ✅ Простой концептуально

**Минусы:**
- ⚠️ Визуальное "мерцание" при пересоздании
- ⚠️ Потеря состояния (нужно сохранять/восстанавливать)
- ⚠️ Плохой UX при частых операциях

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Disable + Object Pooling

Для production-приложений рекомендуется комбинация **Решения 1** и **Решения 2**:

```typescript
// ============================================
// Полная реализация: SafePrimitiveManager
// ============================================

import {
    ISeriesApi,
    ISeriesPrimitive,
    IPrimitivePaneView,
    ISeriesPrimitiveAxisView,
    Time,
    SeriesType
} from 'lightweight-charts';

/**
 * Базовый класс для безопасных примитивов
 * Реализует паттерн disable вместо detach
 */
abstract class SafePrimitive implements ISeriesPrimitive<Time> {
    protected _isActive: boolean = true;
    protected _paneViews: IPrimitivePaneView[] = [];
    protected _priceAxisViews: ISeriesPrimitiveAxisView[] = [];
    protected _timeAxisViews: ISeriesPrimitiveAxisView[] = [];
    
    /**
     * Отключает примитив без удаления из библиотеки
     * Это БЕЗОПАСНЫЙ способ "удаления"
     */
    public disable(): void {
        this._isActive = false;
        // Очищаем views для предотвращения рендеринга
        this._paneViews = [];
        this._priceAxisViews = [];
        this._timeAxisViews = [];
    }
    
    /**
     * Повторно активирует примитив
     */
    public enable(): void {
        this._isActive = true;
        this.rebuildViews();
    }
    
    /**
     * Сбрасывает примитив в начальное состояние (для pooling)
     */
    public reset(): void {
        this._isActive = false;
        this._paneViews = [];
        this._priceAxisViews = [];
        this._timeAxisViews = [];
    }
    
    /**
     * Восстанавливает views после enable
     */
    protected abstract rebuildViews(): void;
    
    // ISeriesPrimitive implementation
    public paneViews(): readonly IPrimitivePaneView[] {
        return this._isActive ? this._paneViews : [];
    }
    
    public priceAxisViews(): readonly ISeriesPrimitiveAxisView[] {
        return this._isActive ? this._priceAxisViews : [];
    }
    
    public timeAxisViews(): readonly ISeriesPrimitiveAxisView[] {
        return this._isActive ? this._timeAxisViews : [];
    }
    
    public updateAllViews(): void {
        if (!this._isActive) return;
        // Обновление views
    }
    
    public attached?(params: { chart: unknown; series: unknown }): void;
    public detached?(): void;
}

/**
 * Пул для переиспользования примитивов
 */
class PrimitivePool<T extends SafePrimitive> {
    private pool: T[] = [];
    private factory: () => T;
    private maxSize: number;
    
    constructor(factory: () => T, initialSize: number = 10, maxSize: number = 100) {
        this.factory = factory;
        this.maxSize = maxSize;
        
        // Предварительное заполнение пула
        for (let i = 0; i < initialSize; i++) {
            const primitive = factory();
            primitive.disable();
            this.pool.push(primitive);
        }
    }
    
    /**
     * Получить примитив из пула или создать новый
     */
    public acquire(): T {
        let primitive: T;
        
        if (this.pool.length > 0) {
            primitive = this.pool.pop()!;
        } else {
            primitive = this.factory();
        }
        
        primitive.enable();
        return primitive;
    }
    
    /**
     * Вернуть примитив в пул
     */
    public release(primitive: T): void {
        primitive.reset();
        
        if (this.pool.length < this.maxSize) {
            this.pool.push(primitive);
        }
        // Если пул переполнен, просто оставляем disabled
    }
    
    /**
     * Статистика пула
     */
    public getStats(): { poolSize: number; maxSize: number } {
        return {
            poolSize: this.pool.length,
            maxSize: this.maxSize
        };
    }
}

/**
 * Менеджер примитивов с поддержкой pooling и безопасного удаления
 */
class SafePrimitiveManager<T extends SafePrimitive> {
    private series: ISeriesApi<SeriesType>;
    private pool: PrimitivePool<T>;
    private activePrimitives: Map<string, T> = new Map();
    private attachedToSeries: Set<T> = new Set();
    
    constructor(
        series: ISeriesApi<SeriesType>,
        factory: () => T,
        poolSize: number = 20
    ) {
        this.series = series;
        this.pool = new PrimitivePool(factory, poolSize);
    }
    
    /**
     * Добавить примитив (из пула или новый)
     */
    public add(id: string): T {
        // Проверяем, не существует ли уже
        if (this.activePrimitives.has(id)) {
            throw new Error(`Primitive with id '${id}' already exists`);
        }
        
        const primitive = this.pool.acquire();
        this.activePrimitives.set(id, primitive);
        
        // Attach к серии только если ещё не attached
        if (!this.attachedToSeries.has(primitive)) {
            this.series.attachPrimitive(primitive);
            this.attachedToSeries.add(primitive);
        }
        
        return primitive;
    }
    
    /**
     * Удалить примитив (безопасно, без detach)
     */
    public remove(id: string): void {
        const primitive = this.activePrimitives.get(id);
        if (!primitive) return;
        
        this.activePrimitives.delete(id);
        this.pool.release(primitive);
        // НЕ вызываем detachPrimitive!
    }
    
    /**
     * Получить примитив по ID
     */
    public get(id: string): T | undefined {
        return this.activePrimitives.get(id);
    }
    
    /**
     * Количество активных примитивов
     */
    public get activeCount(): number {
        return this.activePrimitives.size;
    }
    
    /**
     * Полная очистка (для unmount компонента)
     */
    public dispose(): void {
        // Отключаем все активные примитивы
        for (const primitive of this.activePrimitives.values()) {
            primitive.disable();
        }
        this.activePrimitives.clear();
    }
}

// ============================================
// Пример использования
// ============================================

// Конкретная реализация примитива
class TrendLinePrimitive extends SafePrimitive {
    private startPoint: { time: Time; price: number } | null = null;
    private endPoint: { time: Time; price: number } | null = null;
    
    public setPoints(
        start: { time: Time; price: number },
        end: { time: Time; price: number }
    ): void {
        this.startPoint = start;
        this.endPoint = end;
        this.rebuildViews();
    }
    
    protected rebuildViews(): void {
        if (!this.startPoint || !this.endPoint) {
            this._paneViews = [];
            return;
        }
        // Создание views для отрисовки линии
        // this._paneViews = [new TrendLinePaneView(...)];
    }
    
    public reset(): void {
        super.reset();
        this.startPoint = null;
        this.endPoint = null;
    }
}

// Использование в React компоненте
function ChartComponent() {
    const chartContainerRef = useRef<HTMLDivElement>(null);
    const chartRef = useRef<IChartApi | null>(null);
    const managerRef = useRef<SafePrimitiveManager<TrendLinePrimitive> | null>(null);
    
    useEffect(() => {
        if (!chartContainerRef.current) return;
        
        // Создаём график
        const chart = createChart(chartContainerRef.current, {
            width: 800,
            height: 400
        });
        chartRef.current = chart;
        
        // Создаём серию
        const series = chart.addCandlestickSeries();
        
        // Создаём менеджер примитивов с пулом
        const manager = new SafePrimitiveManager(
            series,
            () => new TrendLinePrimitive(),
            20  // Размер пула
        );
        managerRef.current = manager;
        
        // Cleanup
        return () => {
            manager.dispose();
            chart.remove();
        };
    }, []);
    
    const addTrendLine = (id: string, start: any, end: any) => {
        const line = managerRef.current?.add(id);
        line?.setPoints(start, end);
    };
    
    const removeTrendLine = (id: string) => {
        managerRef.current?.remove(id);  // Безопасно!
    };
    
    return <div ref={chartContainerRef} />;
}
```

---

## 📊 Сравнительная таблица

| Критерий | Disable Pattern | Object Pooling | Lazy Detach | WeakRef | Recreate Chart |
|----------|-----------------|----------------|-------------|---------|----------------|
| **Оценка** | 9/10 | 8/10 | 7/10 | 6/10 | 5/10 |
| **Сложность** | Низкая | Средняя | Средняя | Высокая | Низкая |
| **Эффективность** | Высокая | Очень высокая | Средняя | Низкая | Высокая |
| **Риск утечек** | Минимальный | Минимальный | Присутствует | Присутствует | Нулевой |
| **Влияние на UX** | Нет | Нет | Нет | Нет | Мерцание |
| **Browser support** | Все | Все | Chrome 47+ | Chrome 84+ | Все |
| **Production ready** | ✅ Да | ✅ Да | ⚠️ С оговорками | ❌ Нет | ⚠️ Крайний случай |

---

## 🔗 Источники

1. [GitHub Issue #2000 — Persistent Lag When Detaching Primitives](https://github.com/tradingview/lightweight-charts/issues/2000)
2. [GitHub Issue #946 — Performance degradation with frequent updates](https://github.com/tradingview/lightweight-charts/issues/946)
3. [TradingView Docs — Memory Leak Troubleshooting](https://tradingview.com/charting-library-docs/latest/troubleshooting/memory-leak/)
4. [MDN — WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef)
5. [web.dev — Object Pooling](https://web.dev/articles/static-memory-javascript)
6. [Lightweight Charts — Plugins and Primitives](https://tradingview.github.io/lightweight-charts/docs/plugins/intro)
7. [MDN — Canvas Best Practices](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)

---

## 📅 Дата документирования
**18 января 2026**

## ✍️ Статус
Документировано ✅
