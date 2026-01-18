# БАГ #18: Плагины не реагируют на обновление данных серии

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#1491](https://github.com/tradingview/lightweight-charts/issues/1491)  
> **Версии:** v4.1+  
> **Статус:** 🔴 Open  
> **Последнее обновление:** январь 2026

---

## 📋 Описание проблемы

### Суть бага
Custom primitives и плагины, расширяющие `PluginBase`, **не перерисовываются** автоматически при вызове:
- `series.update()`
- `series.setData()`

### Техническая причина
Проблема связана с **некорректным binding контекста `this`** в методе `_fireDataUpdated` в `plugin-base.ts`:

```typescript
// Проблемный код в PluginBase (строка 25)
this._series.subscribeDataChanged(this._fireDataUpdated);
// ↑ При вызове callback `this` указывает на wrong context,
//   и `this.dataUpdated` становится undefined
```

### Симптомы
1. ❌ Примитив "застревает" в старом состоянии после `series.update()`
2. ❌ Callback `dataUpdated()` не вызывается при изменении данных серии
3. ❌ Визуальное отображение примитива не соответствует актуальным данным
4. ❌ Индикаторы на базе примитивов показывают устаревшие значения

### Сценарии воспроизведения
- Любые custom primitives, зависящие от данных серии
- Indicator plugins (MA, RSI, MACD и т.д.)
- Custom annotations, привязанные к данным
- Session highlighting примитивы

### Частота
**100%** для всех примитивов, не реализующих manual refresh logic

---

## 🔍 Найденные решения

### Решение 1: Arrow Function Binding
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

Использование стрелочной функции для сохранения контекста `this`:

```typescript
class MyPrimitive extends PluginBase {
  constructor() {
    super();
  }

  attached(params: ISeriesPrimitivePaneViewParams) {
    super.attached(params);
    
    // ✅ Правильный binding через arrow function
    this._series.subscribeDataChanged((scope) => this._fireDataUpdated(scope));
  }

  dataUpdated(scope: DataChangedScope) {
    // Этот метод теперь будет вызываться корректно
    this.requestUpdate();
    console.log('Data updated:', scope);
  }
}
```

**Преимущества:**
- ✅ Простой и понятный синтаксис
- ✅ Идиоматичный JavaScript/TypeScript код
- ✅ Автоматическое сохранение контекста

**Недостатки:**
- ⚠️ Создаёт новую функцию при каждом вызове attached

---

### Решение 2: Explicit .bind(this)
**Оценка: 8/10**

Явное связывание контекста через `.bind()`:

```typescript
class MyPrimitive extends PluginBase {
  private _boundFireDataUpdated: (scope: DataChangedScope) => void;

  constructor() {
    super();
    // Создаём bound версию один раз в конструкторе
    this._boundFireDataUpdated = this._fireDataUpdated.bind(this);
  }

  attached(params: ISeriesPrimitivePaneViewParams) {
    super.attached(params);
    
    // ✅ Используем заранее созданную bound функцию
    this._series.subscribeDataChanged(this._boundFireDataUpdated);
  }
}
```

**Преимущества:**
- ✅ Оптимизация памяти (функция создаётся один раз)
- ✅ Можно использовать для unsubscribe
- ✅ Явное указание на решение проблемы binding

**Недостатки:**
- ⚠️ Требует дополнительного поля в классе
- ⚠️ Более verbose код

---

### Решение 3: Manual requestUpdate Pattern
**Оценка: 7/10**

Ручной вызов `requestUpdate` после изменения данных на стороне приложения:

```typescript
// В компоненте/приложении
class ChartManager {
  private chart: IChartApi;
  private series: ISeriesApi<'Candlestick'>;
  private primitive: MyPrimitive;
  private requestPrimitiveUpdate: () => void;

  setupPrimitive() {
    this.primitive = new MyPrimitive();
    
    // Сохраняем callback из attached lifecycle
    this.primitive.attached({
      chart: this.chart,
      series: this.series,
      requestUpdate: () => {
        this.requestPrimitiveUpdate?.();
      }
    });
    
    this.requestPrimitiveUpdate = () => {
      this.primitive.requestUpdate();
    };
  }

  updateData(newData: CandlestickData[]) {
    this.series.setData(newData);
    
    // ✅ Явный вызов обновления примитива
    this.requestPrimitiveUpdate();
  }
}
```

**Преимущества:**
- ✅ Полный контроль над обновлениями
- ✅ Можно объединять batch updates
- ✅ Не требует патча PluginBase

**Недостатки:**
- ⚠️ Требует discipline от разработчика
- ⚠️ Легко забыть вызвать requestUpdate
- ⚠️ Code duplication в каждом месте обновления

---

### Решение 4: Wrapper Class с подпиской на данные
**Оценка: 8/10**

Создание wrapper-класса с корректной реализацией подписки:

```typescript
// Исправленный PluginBase
export abstract class FixedPluginBase implements ISeriesPrimitive<HorzScaleItem> {
  protected _chart: IChartApi | undefined;
  protected _series: ISeriesApi<SeriesType> | undefined;
  protected _requestUpdate?: () => void;

  attached(params: ISeriesPrimitivePaneViewParams) {
    this._chart = params.chart;
    this._series = params.series;
    this._requestUpdate = params.requestUpdate;
    
    // ✅ Корректный binding
    this._series.subscribeDataChanged((scope) => {
      this.dataUpdated(scope);
    });
  }

  detached() {
    this._chart = undefined;
    this._series = undefined;
    this._requestUpdate = undefined;
  }

  requestUpdate() {
    this._requestUpdate?.();
  }

  // Метод для переопределения в наследниках
  protected dataUpdated(scope: DataChangedScope): void {
    // Override in subclass
    this.requestUpdate();
  }

  abstract paneViews(): readonly IPrimitivePaneView[];
}
```

**Преимущества:**
- ✅ Централизованное решение
- ✅ Чистый API для наследников
- ✅ Правильное управление lifecycle

**Недостатки:**
- ⚠️ Дублирование кода библиотеки
- ⚠️ Нужно следить за совместимостью с обновлениями

---

### Решение 5: Proxy Pattern для Series
**Оценка: 6/10**

Обёртка над series API для автоматического уведомления примитивов:

```typescript
class SeriesProxy<T extends SeriesType> {
  private _series: ISeriesApi<T>;
  private _primitives: Set<ISeriesPrimitive<any>> = new Set();

  constructor(series: ISeriesApi<T>) {
    this._series = series;
  }

  registerPrimitive(primitive: ISeriesPrimitive<any>) {
    this._primitives.add(primitive);
  }

  unregisterPrimitive(primitive: ISeriesPrimitive<any>) {
    this._primitives.delete(primitive);
  }

  setData(data: SeriesDataItemTypeMap[T][]) {
    this._series.setData(data);
    this._notifyPrimitives('full');
  }

  update(data: SeriesDataItemTypeMap[T]) {
    this._series.update(data);
    this._notifyPrimitives('update');
  }

  private _notifyPrimitives(scope: DataChangedScope) {
    this._primitives.forEach(primitive => {
      if ('dataUpdated' in primitive && typeof primitive.dataUpdated === 'function') {
        primitive.dataUpdated(scope);
      }
    });
  }

  // Proxy other methods...
  get original(): ISeriesApi<T> {
    return this._series;
  }
}
```

**Преимущества:**
- ✅ Централизованное управление обновлениями
- ✅ Автоматическая синхронизация
- ✅ Поддержка множественных примитивов

**Недостатки:**
- ⚠️ Сложная архитектура
- ⚠️ Нужно использовать proxy везде вместо series
- ⚠️ Overhead на каждое обновление

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Arrow Function + FixedPluginBase

Для максимальной надёжности рекомендуется:
1. Использовать исправленный базовый класс
2. Применять arrow function для подписок
3. Реализовать cleanup в detached

```typescript
import {
  IChartApi,
  ISeriesApi,
  SeriesType,
  ISeriesPrimitive,
  DataChangedScope,
  IPrimitivePaneView,
  ISeriesPrimitivePaneViewParams,
  IPrimitivePaneRenderer,
} from 'lightweight-charts';

/**
 * Исправленный базовый класс для примитивов с корректной подпиской
 * на изменения данных серии.
 * 
 * Решает проблему: https://github.com/tradingview/lightweight-charts/issues/1491
 */
export abstract class FixedPluginBase<HorzScaleItem = number> 
  implements ISeriesPrimitive<HorzScaleItem> {
  
  protected _chart: IChartApi | undefined;
  protected _series: ISeriesApi<SeriesType> | undefined;
  protected _requestUpdate?: () => void;
  private _dataChangedHandler: ((scope: DataChangedScope) => void) | null = null;

  /**
   * Вызывается при присоединении примитива к серии
   */
  attached(params: ISeriesPrimitivePaneViewParams): void {
    this._chart = params.chart;
    this._series = params.series;
    this._requestUpdate = params.requestUpdate;

    // ✅ КЛЮЧЕВОЕ ИСПРАВЛЕНИЕ: Arrow function для сохранения контекста
    this._dataChangedHandler = (scope: DataChangedScope) => {
      this._onDataUpdated(scope);
    };

    this._series.subscribeDataChanged(this._dataChangedHandler);
  }

  /**
   * Вызывается при отсоединении примитива
   */
  detached(): void {
    // ✅ Корректная очистка подписки
    if (this._series && this._dataChangedHandler) {
      this._series.unsubscribeDataChanged(this._dataChangedHandler);
      this._dataChangedHandler = null;
    }

    this._chart = undefined;
    this._series = undefined;
    this._requestUpdate = undefined;
  }

  /**
   * Запросить перерисовку примитива
   */
  protected requestUpdate(): void {
    this._requestUpdate?.();
  }

  /**
   * Внутренний обработчик изменения данных
   */
  private _onDataUpdated(scope: DataChangedScope): void {
    // Вызываем пользовательский обработчик
    this.dataUpdated(scope);
    
    // Автоматически запрашиваем перерисовку
    this.requestUpdate();
  }

  /**
   * Переопределите этот метод для реакции на изменение данных серии.
   * Вызывается при каждом update() или setData() на серии.
   * 
   * @param scope - Тип изменения ('full' для setData, 'update' для update)
   */
  protected dataUpdated(scope: DataChangedScope): void {
    // Базовая реализация - ничего не делает
    // Переопределите в наследнике для custom логики
  }

  /**
   * Вызывается для подготовки views перед рендерингом.
   * Здесь примитив должен обновить свои внутренние данные.
   */
  updateAllViews(): void {
    // Переопределите для обновления views
  }

  /**
   * Возвращает массив pane views для рендеринга
   */
  abstract paneViews(): readonly IPrimitivePaneView[];
}

// ============================================================
// Пример использования: Индикатор скользящей средней
// ============================================================

interface MAPoint {
  time: number;
  value: number;
}

class MovingAveragePrimitive extends FixedPluginBase {
  private _period: number;
  private _maData: MAPoint[] = [];
  private _view: MAPaneView;

  constructor(period: number = 20) {
    super();
    this._period = period;
    this._view = new MAPaneView(this);
  }

  get maData(): MAPoint[] {
    return this._maData;
  }

  /**
   * Вызывается автоматически при series.update() или series.setData()
   */
  protected dataUpdated(scope: DataChangedScope): void {
    console.log(`MA Indicator: Data ${scope}, recalculating...`);
    this._calculateMA();
  }

  private _calculateMA(): void {
    if (!this._series) return;

    const data = this._series.data();
    if (data.length < this._period) {
      this._maData = [];
      return;
    }

    this._maData = [];
    
    for (let i = this._period - 1; i < data.length; i++) {
      let sum = 0;
      for (let j = 0; j < this._period; j++) {
        const item = data[i - j] as any;
        // Для свечей используем close, для line - value
        const value = item.close ?? item.value ?? 0;
        sum += value;
      }
      
      this._maData.push({
        time: (data[i] as any).time,
        value: sum / this._period
      });
    }
  }

  paneViews(): readonly IPrimitivePaneView[] {
    return [this._view];
  }
}

class MAPaneView implements IPrimitivePaneView {
  private _primitive: MovingAveragePrimitive;
  private _renderer: MARenderer;

  constructor(primitive: MovingAveragePrimitive) {
    this._primitive = primitive;
    this._renderer = new MARenderer(() => this._primitive.maData);
  }

  renderer(): IPrimitivePaneRenderer {
    return this._renderer;
  }

  zOrder(): 'bottom' | 'normal' | 'top' {
    return 'bottom';
  }
}

class MARenderer implements IPrimitivePaneRenderer {
  private _getData: () => MAPoint[];

  constructor(getData: () => MAPoint[]) {
    this._getData = getData;
  }

  draw(target: CanvasRenderingContext2D): void {
    const data = this._getData();
    if (data.length < 2) return;

    target.beginPath();
    target.strokeStyle = '#2196F3';
    target.lineWidth = 2;

    // ... rendering logic with coordinate conversion
  }
}

// ============================================================
// Интеграция с React
// ============================================================

// React Hook для использования с FixedPluginBase
function useSeriesPrimitive<T extends FixedPluginBase>(
  series: ISeriesApi<SeriesType> | null,
  chart: IChartApi | null,
  createPrimitive: () => T
): T | null {
  const primitiveRef = React.useRef<T | null>(null);

  React.useEffect(() => {
    if (!series || !chart) return;

    const primitive = createPrimitive();
    primitiveRef.current = primitive;

    series.attachPrimitive(primitive);

    return () => {
      if (primitiveRef.current) {
        series.detachPrimitive(primitiveRef.current);
        primitiveRef.current = null;
      }
    };
  }, [series, chart, createPrimitive]);

  return primitiveRef.current;
}

// Пример использования hook
/*
function ChartWithMA() {
  const [chart, setChart] = useState<IChartApi | null>(null);
  const [series, setSeries] = useState<ISeriesApi<'Candlestick'> | null>(null);

  const maPrimitive = useSeriesPrimitive(
    series,
    chart,
    () => new MovingAveragePrimitive(20)
  );

  // При вызове series.update() или series.setData()
  // MA автоматически пересчитается и перерисуется!
  
  return <div ref={containerRef} />;
}
*/
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Надёжность | Production Ready |
|---------|--------|-----------|------------|------------------|
| **Arrow Function Binding** | 9/10 | 🟢 Низкая | ✅ Высокая | ✅ Да |
| **.bind(this)** | 8/10 | 🟢 Низкая | ✅ Высокая | ✅ Да |
| **Manual requestUpdate** | 7/10 | 🟡 Средняя | ⚠️ Средняя | ⚠️ Осторожно |
| **Wrapper Class** | 8/10 | 🟡 Средняя | ✅ Высокая | ✅ Да |
| **Proxy Pattern** | 6/10 | 🔴 Высокая | ⚠️ Средняя | ⚠️ Осторожно |

### Рекомендации по выбору:

- **Для быстрого исправления** → Решение 1 (Arrow Function)
- **Для новых проектов** → Решение 4 (Wrapper Class)
- **Для сложных систем** → Комбинация решений 1 + 4

---

## 🔗 Источники

1. **GitHub Issue #1491** - Plugin not responding to data update  
   https://github.com/tradingview/lightweight-charts/issues/1491

2. **GitHub Issue #1920** - Primitives do not synchronize with chart in React  
   https://github.com/tradingview/lightweight-charts/issues/1920

3. **GitHub Issue #1594** - Chart isn't updated when primitive is detached  
   https://github.com/tradingview/lightweight-charts/issues/1594

4. **Official Plugins Tutorial**  
   https://tradingview.github.io/lightweight-charts/tutorials/how_to/plugins

5. **Pane Primitives Documentation**  
   https://tradingview.github.io/lightweight-charts/docs/plugins/pane-primitives

6. **Series Primitives Documentation**  
   https://tradingview.github.io/lightweight-charts/docs/plugins/series-primitives

---

## 📝 Дополнительные заметки

### Связанные проблемы
- **Issue #2000**: Утечки памяти при detachPrimitive
- **Issue #1920**: Примитивы не синхронизируются с chart в React
- **Issue #1594**: Chart не обновляется при detach примитива

### Когда ожидать официальное исправление?
На момент написания (январь 2026) issue #1491 остаётся открытым. Рекомендуется использовать workaround с arrow function или создать собственный базовый класс.

### TypeScript подсказки
Для корректной типизации используйте:
```typescript
import type { 
  DataChangedScope,
  ISeriesPrimitivePaneViewParams 
} from 'lightweight-charts';
```
