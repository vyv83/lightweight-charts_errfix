# БАГ #34: Несогласованное поведение примитивов при разных способах привязки

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1883](https://github.com/tradingview/lightweight-charts/issues/1883)  
> **Версии:** v5.0+  
> **Статус:** 🔴 Open (needs investigation)

---

## 📋 Описание проблемы

### Суть проблемы

При привязке примитивов (primitives) к серии (`attachPrimitive` на series) и к pane (`attachPrimitive` на chart) наблюдается **разное поведение рендеринга**:

- **Примитив, привязанный к серии** — рендерится **над** графиком (поверх свечей/линий)
- **Примитив, привязанный к pane** — рендерится **под** графиком (за свечами/линиями)

### Ожидаемое поведение

Примитивы должны рендериться в соответствии с их `zOrder` и типом view, независимо от способа привязки. Ожидается, что примитив на pane будет отображаться **спереди** графика при использовании стандартных настроек.

### Сценарии воспроизведения

```typescript
// Примитив, привязанный к серии - рендерится СВЕРХУ
const seriesPrimitive = new TextPrimitive();
series.attachPrimitive(seriesPrimitive);

// Примитив, привязанный к pane - рендерится СНИЗУ (неожиданно!)
const panePrimitive = new TextPrimitive();
chart.attachPrimitive(panePrimitive);
```

### Влияние на разработку

1. **Непредсказуемый z-order** — разработчик не может гарантировать видимость примитива
2. **Несогласованность API** — одинаковый код даёт разные результаты
3. **Сложность миграции** — при переносе примитива между series и pane меняется визуальное поведение
4. **Проблемы с watermarks/annotations** — pane-примитивы скрываются за данными

### Связанные проблемы

- **Issue #1594** — График не обновляется при detach примитива
- **Issue #1920** — Примитивы не синхронизируются при scroll/zoom в React
- **Issue #2000** — Утечки памяти при добавлении/удалении примитивов

---

## 🔍 Найденные решения

### Решение 1: Использовать только ISeriesPrimitive для критичного z-order

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐ (8/10)

**Описание:**  
Всегда привязывать примитивы к серии через `series.attachPrimitive()` вместо `chart.attachPrimitive()`, чтобы гарантировать рендеринг поверх графика.

**Преимущества:**
- Консистентное поведение z-order
- Доступ к price/time scale рендерингу
- Поддержка `autoscaleInfo()` для влияния на масштаб
- Полный контроль через `ISeriesPrimitiveAxisView`

**Недостатки:**
- Требуется "фиктивная" серия для chart-wide аннотаций
- Не подходит для watermarks, которые должны быть под данными

```typescript
// ✅ Рекомендуемый подход - привязка к серии
class OverlayPrimitive implements ISeriesPrimitive<Time> {
  private _paneViews: IPrimitivePaneView[];

  constructor() {
    this._paneViews = [new OverlayPaneView()];
  }

  paneViews(): readonly IPrimitivePaneView[] {
    return this._paneViews;
  }
  
  updateAllViews(): void {
    this._paneViews.forEach(view => view.update());
  }
}

// Привязываем к любой серии
const primitive = new OverlayPrimitive();
mainSeries.attachPrimitive(primitive);
```

---

### Решение 2: Явное управление zOrder через IPrimitivePaneView

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐⭐ (9/10)

**Описание:**  
Использовать свойство `zOrder` интерфейса `IPrimitivePaneView` для явного указания слоя рендеринга.

**Преимущества:**
- Стандартный API для управления z-order
- Работает для обоих типов привязки
- Гибкость в позиционировании относительно других элементов

**Доступные значения zOrder:**

| Значение | Описание | Позиция |
|----------|----------|---------|
| `'bottom'` | Под всеми элементами | Самый нижний слой |
| `'normal'` | Стандартный уровень | Между grid и серией |
| `'top'` | Над сериями | Поверх данных |

```typescript
class CustomPaneView implements IPrimitivePaneView {
  // Явно указываем z-order
  zOrder(): PrimitivePaneViewZOrder {
    return 'top'; // Рендерить поверх всего
  }

  renderer(): IPrimitivePaneRenderer | null {
    return new CustomRenderer();
  }

  // Опционально: для рендеринга ПОД элементами с тем же zOrder
  drawBackground(context: CanvasRenderingContext2D): void {
    // Рисуем фон
  }
}

class OverlayPrimitive implements ISeriesPrimitive<Time> {
  paneViews(): readonly IPrimitivePaneView[] {
    return [new CustomPaneView()];
  }
}
```

---

### Решение 3: Создание "невидимой" серии для pane-подобных примитивов

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐ (7/10)

**Описание:**  
Для chart-wide аннотаций создать прозрачную линейную серию и привязывать примитивы к ней.

**Преимущества:**
- Стабильное поведение z-order
- Полный доступ к ISeriesPrimitive API
- Не влияет на autoscale (при правильной настройке)

**Недостатки:**
- "Хак" с фиктивной серией
- Дополнительный overhead

```typescript
// Создаём невидимую серию для примитивов
const invisibleSeries = chart.addLineSeries({
  visible: false,
  autoscaleInfoProvider: () => null, // Не влияет на autoscale
  priceLineVisible: false,
  lastValueVisible: false,
});

// Добавляем минимальные данные
invisibleSeries.setData([
  { time: '2024-01-01', value: 0 }
]);

// Теперь привязываем примитив
const watermark = new WatermarkPrimitive('DEMO');
invisibleSeries.attachPrimitive(watermark);
```

---

### Решение 4: Паттерн Adapter для унификации привязок

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐ (8/10)

**Описание:**  
Создать абстракцию-обёртку, которая инкапсулирует логику привязки и обеспечивает консистентное поведение.

**Преимущества:**
- Единый API для всех примитивов
- Скрывает сложность выбора типа привязки
- Легко адаптировать при исправлении бага в библиотеке

```typescript
enum PrimitiveLayer {
  Background = 'background',
  BelowData = 'below',
  AboveData = 'above',
  Overlay = 'overlay'
}

class PrimitiveManager {
  private chart: IChartApi;
  private helperSeries?: ISeriesApi<any>;
  
  constructor(chart: IChartApi) {
    this.chart = chart;
  }
  
  attachPrimitive(
    primitive: ISeriesPrimitive<Time>,
    layer: PrimitiveLayer,
    series?: ISeriesApi<any>
  ): void {
    // Для слоёв выше данных - всегда через series
    if (layer === PrimitiveLayer.AboveData || layer === PrimitiveLayer.Overlay) {
      if (series) {
        series.attachPrimitive(primitive);
      } else {
        this.getHelperSeries().attachPrimitive(primitive);
      }
    } else {
      // Для фоновых слоёв можно использовать chart
      this.chart.attachPrimitive(primitive);
    }
  }
  
  private getHelperSeries(): ISeriesApi<any> {
    if (!this.helperSeries) {
      this.helperSeries = this.chart.addLineSeries({
        visible: false,
        priceLineVisible: false,
        lastValueVisible: false,
      });
    }
    return this.helperSeries;
  }
  
  detachPrimitive(
    primitive: ISeriesPrimitive<Time>,
    series?: ISeriesApi<any>
  ): void {
    try {
      if (series) {
        series.detachPrimitive(primitive);
      } else if (this.helperSeries) {
        this.helperSeries.detachPrimitive(primitive);
      }
    } catch {
      // Fallback: попытка отвязать от chart
      this.chart.detachPrimitive?.(primitive as any);
    }
    
    // Принудительное обновление
    this.chart.timeScale().fitContent();
  }
}
```

---

### Решение 5: Workaround через requestUpdate в lifecycle hooks

**Рейтинг:** ⭐⭐⭐⭐⭐⭐ (6/10)

**Описание:**  
Принудительно вызывать `requestUpdate()` в lifecycle методах примитива для обеспечения корректного рендеринга.

**Преимущества:**
- Простая реализация
- Не требует изменения архитектуры

**Недостатки:**
- Не решает корневую проблему z-order
- Дополнительные вызовы рендеринга

```typescript
class RobustPrimitive implements ISeriesPrimitive<Time> {
  private _requestUpdate?: () => void;
  private _paneViews: IPrimitivePaneView[];

  constructor() {
    this._paneViews = [new MyPaneView()];
  }

  attached(options: SeriesAttachedParameter<Time>): void {
    this._requestUpdate = options.requestUpdate;
    // Принудительно запрашиваем обновление после привязки
    requestAnimationFrame(() => {
      this._requestUpdate?.();
    });
  }

  detached(): void {
    // Важно: вызываем update перед отвязкой
    this._requestUpdate?.();
    this._requestUpdate = undefined;
  }

  paneViews(): readonly IPrimitivePaneView[] {
    return this._paneViews;
  }

  updateAllViews(): void {
    this._paneViews.forEach(v => v.update?.());
    this._requestUpdate?.();
  }
}
```

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Решения 2 + 4

Для наиболее надёжной работы рекомендуется **комбинация явного zOrder и PrimitiveManager**:

```typescript
import {
  IChartApi,
  ISeriesApi,
  ISeriesPrimitive,
  IPrimitivePaneView,
  IPrimitivePaneRenderer,
  SeriesAttachedParameter,
  PrimitivePaneViewZOrder,
  Time,
  CanvasRenderingContext2D,
} from 'lightweight-charts';

// ============================================
// 1. Базовый класс для примитивов с zOrder
// ============================================

abstract class BasePrimitive implements ISeriesPrimitive<Time> {
  protected _requestUpdate?: () => void;
  protected _series?: ISeriesApi<any>;
  protected abstract _paneViews: IPrimitivePaneView[];

  attached(params: SeriesAttachedParameter<Time>): void {
    this._requestUpdate = params.requestUpdate;
    this._series = params.series;
    this.onAttached();
  }

  detached(): void {
    this._requestUpdate?.();
    this._requestUpdate = undefined;
    this._series = undefined;
    this.onDetached();
  }

  paneViews(): readonly IPrimitivePaneView[] {
    return this._paneViews;
  }

  updateAllViews(): void {
    this._paneViews.forEach(view => view.update?.());
  }

  requestUpdate(): void {
    this._requestUpdate?.();
  }

  protected onAttached(): void {}
  protected onDetached(): void {}
}

// ============================================
// 2. PaneView с явным zOrder
// ============================================

class ControlledZOrderPaneView implements IPrimitivePaneView {
  private _zOrder: PrimitivePaneViewZOrder;
  private _renderer: IPrimitivePaneRenderer;

  constructor(
    renderer: IPrimitivePaneRenderer,
    zOrder: PrimitivePaneViewZOrder = 'top'
  ) {
    this._renderer = renderer;
    this._zOrder = zOrder;
  }

  zOrder(): PrimitivePaneViewZOrder {
    return this._zOrder;
  }

  renderer(): IPrimitivePaneRenderer | null {
    return this._renderer;
  }

  update(): void {
    // Обновление состояния view при необходимости
  }
}

// ============================================
// 3. Пример: Текстовый примитив-оверлей
// ============================================

interface TextOverlayOptions {
  text: string;
  x: number;
  y: number;
  color?: string;
  fontSize?: number;
  zOrder?: PrimitivePaneViewZOrder;
}

class TextOverlayRenderer implements IPrimitivePaneRenderer {
  private _options: TextOverlayOptions;

  constructor(options: TextOverlayOptions) {
    this._options = options;
  }

  draw(context: CanvasRenderingContext2D): void {
    const { text, x, y, color = '#333', fontSize = 14 } = this._options;
    
    context.save();
    context.font = `${fontSize}px Arial, sans-serif`;
    context.fillStyle = color;
    context.textAlign = 'left';
    context.textBaseline = 'top';
    context.fillText(text, x, y);
    context.restore();
  }

  update(options: Partial<TextOverlayOptions>): void {
    Object.assign(this._options, options);
  }
}

class TextOverlayPrimitive extends BasePrimitive {
  protected _paneViews: IPrimitivePaneView[];
  private _renderer: TextOverlayRenderer;

  constructor(options: TextOverlayOptions) {
    super();
    this._renderer = new TextOverlayRenderer(options);
    this._paneViews = [
      new ControlledZOrderPaneView(
        this._renderer,
        options.zOrder ?? 'top'
      )
    ];
  }

  updateText(text: string): void {
    this._renderer.update({ text });
    this.requestUpdate();
  }
}

// ============================================
// 4. PrimitiveManager - унифицированное управление
// ============================================

enum RenderLayer {
  Background = 'background',
  BelowChart = 'below',
  AboveChart = 'above',
  Overlay = 'overlay'
}

interface ManagedPrimitive {
  primitive: ISeriesPrimitive<Time>;
  layer: RenderLayer;
  attachedTo: ISeriesApi<any> | IChartApi;
}

class ChartPrimitiveManager {
  private _chart: IChartApi;
  private _helperSeries?: ISeriesApi<'Line'>;
  private _primitives: Map<ISeriesPrimitive<Time>, ManagedPrimitive> = new Map();

  constructor(chart: IChartApi) {
    this._chart = chart;
  }

  /**
   * Привязать примитив с гарантированным слоем рендеринга
   */
  attach(
    primitive: ISeriesPrimitive<Time>,
    layer: RenderLayer,
    targetSeries?: ISeriesApi<any>
  ): void {
    let attachTo: ISeriesApi<any> | IChartApi;

    switch (layer) {
      case RenderLayer.Background:
        // Фоновые элементы - через chart (но с zOrder: 'bottom')
        attachTo = this._chart;
        break;
        
      case RenderLayer.BelowChart:
        // Под данными - через helper series с zOrder: 'bottom'
        attachTo = targetSeries ?? this.getHelperSeries();
        break;
        
      case RenderLayer.AboveChart:
      case RenderLayer.Overlay:
        // Над данными - через series (гарантирует рендеринг сверху)
        attachTo = targetSeries ?? this.getHelperSeries();
        break;
        
      default:
        attachTo = targetSeries ?? this.getHelperSeries();
    }

    // Привязываем
    if ('attachPrimitive' in attachTo) {
      (attachTo as ISeriesApi<any>).attachPrimitive(primitive);
    }

    // Сохраняем для управления
    this._primitives.set(primitive, {
      primitive,
      layer,
      attachedTo: attachTo
    });
  }

  /**
   * Отвязать примитив с корректной очисткой
   */
  detach(primitive: ISeriesPrimitive<Time>): boolean {
    const managed = this._primitives.get(primitive);
    if (!managed) return false;

    try {
      if ('detachPrimitive' in managed.attachedTo) {
        (managed.attachedTo as ISeriesApi<any>).detachPrimitive(primitive);
      }
    } catch (e) {
      console.warn('Primitive detach warning:', e);
    }

    this._primitives.delete(primitive);
    
    // Принудительное обновление графика
    this._chart.timeScale().fitContent();
    
    return true;
  }

  /**
   * Отвязать все примитивы
   */
  detachAll(): void {
    for (const primitive of this._primitives.keys()) {
      this.detach(primitive);
    }
  }

  /**
   * Получить helper-серию для примитивов
   */
  private getHelperSeries(): ISeriesApi<'Line'> {
    if (!this._helperSeries) {
      this._helperSeries = this._chart.addLineSeries({
        visible: false,
        priceLineVisible: false,
        lastValueVisible: false,
        autoscaleInfoProvider: () => null,
      });

      // Добавляем минимальные данные
      const now = Math.floor(Date.now() / 1000) as Time;
      this._helperSeries.setData([{ time: now, value: 0 }]);
    }
    return this._helperSeries;
  }

  /**
   * Очистка при уничтожении
   */
  destroy(): void {
    this.detachAll();
    if (this._helperSeries) {
      this._chart.removeSeries(this._helperSeries);
      this._helperSeries = undefined;
    }
  }
}

// ============================================
// 5. Использование
// ============================================

function example(chart: IChartApi, mainSeries: ISeriesApi<'Candlestick'>): void {
  const manager = new ChartPrimitiveManager(chart);

  // Текст НАД графиком (гарантированно)
  const overlayText = new TextOverlayPrimitive({
    text: 'Important Annotation',
    x: 50,
    y: 30,
    color: '#FF5722',
    fontSize: 16,
    zOrder: 'top'
  });
  manager.attach(overlayText, RenderLayer.Overlay);

  // Текст привязанный к основной серии
  const seriesAnnotation = new TextOverlayPrimitive({
    text: 'Series Note',
    x: 100,
    y: 60,
    color: '#2196F3',
    zOrder: 'top'
  });
  manager.attach(seriesAnnotation, RenderLayer.AboveChart, mainSeries);

  // При размонтировании компонента
  // manager.destroy();
}

export {
  BasePrimitive,
  ControlledZOrderPaneView,
  TextOverlayPrimitive,
  ChartPrimitiveManager,
  RenderLayer
};
```

### Почему это решение оптимально

| Критерий | Оценка |
|----------|--------|
| **Надёжность** | ✅ Использует series-привязку для гарантии z-order |
| **Гибкость** | ✅ PrimitiveManager инкапсулирует логику выбора |
| **Поддерживаемость** | ✅ Легко адаптировать при исправлении бага |
| **Производительность** | ⚠️ Небольшой overhead от helper-серии |
| **Совместимость** | ✅ Работает с v5.0+ |

---

## 📊 Сравнительная таблица решений

| Решение | Надёжность | Сложность | Производительность | Рейтинг |
|---------|------------|-----------|-------------------|---------|
| #1: Только ISeriesPrimitive | ⭐⭐⭐⭐⭐ | Низкая | Высокая | 8/10 |
| #2: Явный zOrder | ⭐⭐⭐⭐⭐ | Низкая | Высокая | 9/10 |
| #3: Невидимая серия | ⭐⭐⭐⭐ | Средняя | Средняя | 7/10 |
| #4: PrimitiveManager | ⭐⭐⭐⭐⭐ | Высокая | Средняя | 8/10 |
| #5: requestUpdate hooks | ⭐⭐⭐ | Низкая | Низкая | 6/10 |

### Рекомендации по выбору

- **Простой проект** → Решение #1 или #2
- **Сложное приложение с множеством примитивов** → Решение #4 (PrimitiveManager)
- **Нужны фоновые watermarks** → Комбинация #2 + #3
- **Быстрый workaround** → Решение #5

---

## 🔗 Источники

1. **GitHub Issue #1883** — [Inconsistent behavior when rendering primitive attached to series vs pane](https://github.com/tradingview/lightweight-charts/issues/1883)

2. **Официальная документация: Plugins** — [Series Primitives](https://tradingview.github.io/lightweight-charts/docs/plugins/series-primitives)

3. **Официальная документация: ISeriesPrimitive** — [API Reference](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesPrimitive)

4. **Официальная документация: IPrimitivePaneView** — [Pane Views](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IPrimitivePaneView)

5. **GitHub Issue #1594** — [Chart not updating after detaching primitive](https://github.com/tradingview/lightweight-charts/issues/1594)

6. **GitHub Issue #1920** — [Primitives not syncing with scroll/zoom in React](https://github.com/tradingview/lightweight-charts/issues/1920)

7. **Plugin Examples** — [Official Examples](https://tradingview.github.io/lightweight-charts/plugin-examples/)

---

**Документ создан:** 2026-01-18  
**Версия документа:** 1.0
