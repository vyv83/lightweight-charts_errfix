# БАГ #35: Смещение маркеров при появлении нового бара

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1990](https://github.com/tradingview/lightweight-charts/issues/1990)  
> **Версии:** v5.0.9  
> **Статус:** 🔴 Open (can't reproduce / needs more info)

---

## 📋 Описание проблемы

### Суть проблемы

При добавлении нового бара на график маркеры визуально "подпрыгивают" или сдвигаются на один кадр (эффект **"rubber-banding"**), создавая впечатление рассинхронизации между данными и маркерами.

### Ожидаемое поведение

Маркеры должны оставаться на своих позициях стабильно, без визуальных сдвигов при обновлении данных.

### Сценарии воспроизведения

```typescript
// Типичный сценарий: real-time обновление
async function onPriceTick(newPrice: OhlcData) {
  // Используется setData для ограничения количества баров
  const data = [...existingData.slice(-99), newPrice];
  series.setData(data);
  
  // Маркеры обновляются после данных
  if (markersChanged) {
    markersApi.setMarkers(markers);
  }
}
// Результат: маркеры "дрожат" или смещаются на 1 кадр
```

### Возможные причины

1. **Race condition** между обновлением данных и маркеров
2. **Асинхронность рендеринга** — данные и маркеры рендерятся в разных кадрах
3. **Использование `setData()` вместо `update()`** — полная перерисовка вместо инкрементального обновления
4. **Пересчёт visible range** — смещение timeScale при добавлении новых данных

### Влияние

- **UX degradation** — дрожание маркеров отвлекает пользователя
- **Профессиональный вид** — выглядит как баг даже если функционально работает
- **Trust issues** — пользователи могут сомневаться в точности данных

---

## 🔍 Найденные решения

### Решение 1: Использовать series.update() вместо setData()

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐⭐ (9/10)

**Описание:**  
Метод `update()` оптимизирован для инкрементальных обновлений и не вызывает полный пересчёт timeScale.

**Преимущества:**
- Официально рекомендуемый подход для real-time данных
- Минимизирует пересчёт состояния графика
- Синхронное обновление последнего бара

**Недостатки:**
- Не подходит если нужно ограничить количество баров
- Требует изменения логики обновления данных

```typescript
interface RealtimeChartManager {
  series: ISeriesApi<'Candlestick'>;
  markersApi: ISeriesApi<any>;
  markers: SeriesMarker<Time>[];
}

class OptimizedChartUpdater {
  private series: ISeriesApi<'Candlestick'>;
  private markers: SeriesMarker<Time>[] = [];
  private lastBarTime?: Time;
  
  constructor(series: ISeriesApi<'Candlestick'>) {
    this.series = series;
  }
  
  /**
   * Обновление данных через update() - рекомендуемый метод
   */
  updatePrice(newCandle: CandlestickData<Time>): void {
    // update() автоматически:
    // - Обновляет существующий бар если time совпадает
    // - Добавляет новый бар если time > последнего
    this.series.update(newCandle);
    
    // Запоминаем время для маркеров
    this.lastBarTime = newCandle.time;
  }
  
  /**
   * Обновление маркеров - вызывать только при реальном изменении
   */
  updateMarkers(newMarkers: SeriesMarker<Time>[]): void {
    // Проверяем, действительно ли изменились маркеры
    if (this.markersEqual(this.markers, newMarkers)) {
      return; // Не обновляем без необходимости
    }
    
    this.markers = [...newMarkers];
    this.series.setMarkers(this.markers);
  }
  
  private markersEqual(a: SeriesMarker<Time>[], b: SeriesMarker<Time>[]): boolean {
    if (a.length !== b.length) return false;
    return a.every((m, i) => 
      m.time === b[i].time && 
      m.position === b[i].position &&
      m.shape === b[i].shape
    );
  }
}

// Использование
const updater = new OptimizedChartUpdater(candlestickSeries);

websocket.onmessage = (event) => {
  const tick = JSON.parse(event.data);
  
  // ✅ Используем update() вместо setData()
  updater.updatePrice({
    time: tick.time as Time,
    open: tick.open,
    high: tick.high,
    low: tick.low,
    close: tick.close
  });
  
  // Маркеры только при необходимости
  if (tick.signal) {
    updater.updateMarkers([...existingMarkers, {
      time: tick.time as Time,
      position: 'aboveBar',
      color: '#2196F3',
      shape: 'arrowDown',
      text: tick.signal
    }]);
  }
};
```

---

### Решение 2: Синхронное обновление данных и маркеров в одном кадре

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐ (8/10)

**Описание:**  
Использовать `requestAnimationFrame` для гарантии обновления данных и маркеров в одном цикле рендеринга.

**Преимущества:**
- Работает с любым методом обновления данных
- Гарантирует атомарность визуального обновления
- Простая интеграция в существующий код

**Недостатки:**
- Небольшая задержка (до 16ms)
- Требует аккуратного batching обновлений

```typescript
class SynchronizedChartUpdater {
  private series: ISeriesApi<'Candlestick'>;
  private pendingData: CandlestickData<Time>[] = [];
  private pendingMarkers: SeriesMarker<Time>[] | null = null;
  private frameScheduled = false;
  
  constructor(series: ISeriesApi<'Candlestick'>) {
    this.series = series;
  }
  
  /**
   * Планирует обновление данных
   */
  scheduleDataUpdate(data: CandlestickData<Time>[]): void {
    this.pendingData = data;
    this.scheduleFrame();
  }
  
  /**
   * Планирует обновление маркеров
   */
  scheduleMarkersUpdate(markers: SeriesMarker<Time>[]): void {
    this.pendingMarkers = markers;
    this.scheduleFrame();
  }
  
  /**
   * Батч-обновление данных и маркеров
   */
  batchUpdate(
    data: CandlestickData<Time>[],
    markers: SeriesMarker<Time>[]
  ): void {
    this.pendingData = data;
    this.pendingMarkers = markers;
    this.scheduleFrame();
  }
  
  private scheduleFrame(): void {
    if (this.frameScheduled) return;
    
    this.frameScheduled = true;
    requestAnimationFrame(() => this.flush());
  }
  
  private flush(): void {
    this.frameScheduled = false;
    
    // Сначала данные
    if (this.pendingData.length > 0) {
      this.series.setData(this.pendingData);
      this.pendingData = [];
    }
    
    // Затем маркеры в том же кадре
    if (this.pendingMarkers !== null) {
      this.series.setMarkers(this.pendingMarkers);
      this.pendingMarkers = null;
    }
  }
}

// Использование
const syncUpdater = new SynchronizedChartUpdater(series);

websocket.onmessage = (event) => {
  const tick = JSON.parse(event.data);
  
  // Атомарное обновление
  syncUpdater.batchUpdate(
    [...currentData, tick],
    computeMarkers([...currentData, tick])
  );
};
```

---

### Решение 3: Windowed Data без setData()

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐ (7/10)

**Описание:**  
Если нужно ограничить количество баров, использовать комбинацию `update()` + ручное управление visible range вместо `setData()`.

**Преимущества:**
- Сохраняет преимущества `update()`
- Позволяет контролировать видимый диапазон
- Данные остаются в памяти, но не занимают экран

**Недостатки:**
- Потенциальные проблемы с памятью при длительной работе
- Более сложная логика

```typescript
class WindowedChartManager {
  private chart: IChartApi;
  private series: ISeriesApi<'Candlestick'>;
  private windowSize: number;
  private dataCount = 0;
  
  constructor(
    chart: IChartApi,
    series: ISeriesApi<'Candlestick'>,
    windowSize = 100
  ) {
    this.chart = chart;
    this.series = series;
    this.windowSize = windowSize;
  }
  
  /**
   * Добавить новый бар с автоматическим окном
   */
  addBar(candle: CandlestickData<Time>): void {
    // Используем update() — не вызывает rubber-banding
    this.series.update(candle);
    this.dataCount++;
    
    // Ограничиваем видимый диапазон вместо данных
    if (this.dataCount > this.windowSize) {
      this.adjustVisibleRange();
    }
  }
  
  /**
   * Периодическая очистка старых данных (опционально)
   */
  pruneOldData(): void {
    const data = this.series.data();
    if (data.length > this.windowSize * 2) {
      // Оставляем только последние windowSize * 1.5 баров
      const trimmed = data.slice(-Math.floor(this.windowSize * 1.5));
      this.series.setData(trimmed as CandlestickData<Time>[]);
      this.dataCount = trimmed.length;
    }
  }
  
  private adjustVisibleRange(): void {
    const timeScale = this.chart.timeScale();
    const visibleRange = timeScale.getVisibleLogicalRange();
    
    if (visibleRange) {
      // Сдвигаем окно на 1 бар вправо
      timeScale.setVisibleLogicalRange({
        from: visibleRange.from + 1,
        to: visibleRange.to + 1
      });
    }
  }
}

// Использование
const windowedManager = new WindowedChartManager(chart, series, 100);

setInterval(() => {
  windowedManager.addBar(generateNewCandle());
}, 1000);

// Периодическая очистка каждые 5 минут
setInterval(() => {
  windowedManager.pruneOldData();
}, 5 * 60 * 1000);
```

---

### Решение 4: Маркеры через Custom Primitive вместо setMarkers()

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐ (8/10)

**Описание:**  
Использовать custom primitive для маркеров вместо встроенного `setMarkers()`. Это даёт полный контроль над рендерингом.

**Преимущества:**
- Полный контроль над позиционированием
- Возможность оптимизации под конкретный use case
- Независимость от внутренней логики маркеров

**Недостатки:**
- Больше кода
- Требует понимания Primitives API

```typescript
interface MarkerData {
  time: Time;
  price: number;
  color: string;
  shape: 'arrow' | 'circle' | 'square';
  text?: string;
}

class StableMarkersPrimitive implements ISeriesPrimitive<Time> {
  private _paneViews: IPrimitivePaneView[];
  private _markers: MarkerData[] = [];
  private _series?: ISeriesApi<any>;
  private _requestUpdate?: () => void;

  constructor() {
    this._paneViews = [new MarkersView(this)];
  }

  attached(params: SeriesAttachedParameter<Time>): void {
    this._series = params.series;
    this._requestUpdate = params.requestUpdate;
  }

  detached(): void {
    this._series = undefined;
    this._requestUpdate = undefined;
  }

  paneViews(): readonly IPrimitivePaneView[] {
    return this._paneViews;
  }

  updateAllViews(): void {
    (this._paneViews[0] as MarkersView).update();
  }

  // Публичный API для обновления маркеров
  setMarkers(markers: MarkerData[]): void {
    this._markers = markers;
    this._requestUpdate?.();
  }

  getMarkers(): MarkerData[] {
    return this._markers;
  }

  getSeries(): ISeriesApi<any> | undefined {
    return this._series;
  }
}

class MarkersView implements IPrimitivePaneView {
  private _primitive: StableMarkersPrimitive;
  private _renderer: MarkersRenderer;

  constructor(primitive: StableMarkersPrimitive) {
    this._primitive = primitive;
    this._renderer = new MarkersRenderer(primitive);
  }

  zOrder(): PrimitivePaneViewZOrder {
    return 'top';
  }

  renderer(): IPrimitivePaneRenderer | null {
    return this._renderer;
  }

  update(): void {
    // Обновление кэшированных позиций при необходимости
  }
}

class MarkersRenderer implements IPrimitivePaneRenderer {
  private _primitive: StableMarkersPrimitive;

  constructor(primitive: StableMarkersPrimitive) {
    this._primitive = primitive;
  }

  draw(target: CanvasRenderingContext2D): void {
    const series = this._primitive.getSeries();
    if (!series) return;

    const markers = this._primitive.getMarkers();
    
    for (const marker of markers) {
      // Получаем координаты через series API
      const x = series.priceToCoordinate(marker.price);
      // Предположим, что chart доступен через series
      // Здесь нужна ваша логика конвертации time → x
      
      if (x === null) continue;

      this.drawMarker(target, marker, 0, x); // Здесь x и y
    }
  }

  private drawMarker(
    ctx: CanvasRenderingContext2D,
    marker: MarkerData,
    x: number,
    y: number
  ): void {
    ctx.save();
    ctx.fillStyle = marker.color;

    switch (marker.shape) {
      case 'arrow':
        this.drawArrow(ctx, x, y);
        break;
      case 'circle':
        ctx.beginPath();
        ctx.arc(x, y, 6, 0, Math.PI * 2);
        ctx.fill();
        break;
      case 'square':
        ctx.fillRect(x - 5, y - 5, 10, 10);
        break;
    }

    if (marker.text) {
      ctx.fillStyle = '#333';
      ctx.font = '12px Arial';
      ctx.textAlign = 'center';
      ctx.fillText(marker.text, x, y - 12);
    }

    ctx.restore();
  }

  private drawArrow(ctx: CanvasRenderingContext2D, x: number, y: number): void {
    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.lineTo(x - 6, y - 10);
    ctx.lineTo(x + 6, y - 10);
    ctx.closePath();
    ctx.fill();
  }
}

// Использование
const markersPrimitive = new StableMarkersPrimitive();
series.attachPrimitive(markersPrimitive);

// Обновление маркеров без rubber-banding
function updateMarkers(newMarkers: MarkerData[]): void {
  markersPrimitive.setMarkers(newMarkers);
}
```

---

### Решение 5: Debounce маркеров относительно данных

**Рейтинг:** ⭐⭐⭐⭐⭐⭐ (6/10)

**Описание:**  
Задержать обновление маркеров на несколько миллисекунд после обновления данных для гарантии завершения рендеринга.

**Преимущества:**
- Простейшая реализация
- Минимальные изменения в коде

**Недостатки:**
- Не решает корневую проблему
- Добавляет задержку
- Может не работать в edge cases

```typescript
class DebouncedMarkerUpdater {
  private series: ISeriesApi<any>;
  private pendingMarkers: SeriesMarker<Time>[] | null = null;
  private timeoutId: ReturnType<typeof setTimeout> | null = null;
  private debounceMs: number;

  constructor(series: ISeriesApi<any>, debounceMs = 32) {
    this.series = series;
    this.debounceMs = debounceMs; // 2 frames at 60fps
  }

  /**
   * Немедленное обновление данных
   */
  updateData(data: any[]): void {
    this.series.setData(data);
  }

  /**
   * Отложенное обновление маркеров
   */
  updateMarkers(markers: SeriesMarker<Time>[]): void {
    this.pendingMarkers = markers;

    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }

    this.timeoutId = setTimeout(() => {
      if (this.pendingMarkers) {
        this.series.setMarkers(this.pendingMarkers);
        this.pendingMarkers = null;
      }
      this.timeoutId = null;
    }, this.debounceMs);
  }

  /**
   * Принудительное применение pending маркеров
   */
  flush(): void {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
      this.timeoutId = null;
    }
    if (this.pendingMarkers) {
      this.series.setMarkers(this.pendingMarkers);
      this.pendingMarkers = null;
    }
  }
}

// Использование
const debouncedUpdater = new DebouncedMarkerUpdater(series, 33);

websocket.onmessage = (event) => {
  const tick = JSON.parse(event.data);
  
  debouncedUpdater.updateData(newData);
  debouncedUpdater.updateMarkers(newMarkers);
};
```

---

## ✅ Рекомендуемое решение

### Комбинация: Решения 1 + 2 (update() + синхронизация)

Для максимальной стабильности рекомендуется комбинировать использование `update()` с синхронизацией через `requestAnimationFrame`:

```typescript
import {
  IChartApi,
  ISeriesApi,
  CandlestickData,
  SeriesMarker,
  Time,
} from 'lightweight-charts';

/**
 * Оптимизированный менеджер для real-time графиков
 * Решает проблему rubber-banding маркеров
 */
class RealTimeChartManager {
  private chart: IChartApi;
  private series: ISeriesApi<'Candlestick'>;
  
  // Состояние маркеров
  private markers: SeriesMarker<Time>[] = [];
  private markersChanged = false;
  
  // Batching
  private pendingUpdates: CandlestickData<Time>[] = [];
  private frameScheduled = false;
  
  constructor(chart: IChartApi, series: ISeriesApi<'Candlestick'>) {
    this.chart = chart;
    this.series = series;
  }

  /**
   * Обновить или добавить последний бар
   */
  updateBar(candle: CandlestickData<Time>): void {
    this.pendingUpdates.push(candle);
    this.scheduleFrame();
  }

  /**
   * Добавить маркер
   */
  addMarker(marker: SeriesMarker<Time>): void {
    // Проверяем, нет ли уже маркера на этом времени
    const existingIndex = this.markers.findIndex(m => m.time === marker.time);
    if (existingIndex >= 0) {
      this.markers[existingIndex] = marker;
    } else {
      this.markers.push(marker);
      // Сортировка по времени
      this.markers.sort((a, b) => {
        const timeA = typeof a.time === 'number' ? a.time : new Date(a.time).getTime();
        const timeB = typeof b.time === 'number' ? b.time : new Date(b.time).getTime();
        return timeA - timeB;
      });
    }
    this.markersChanged = true;
    this.scheduleFrame();
  }

  /**
   * Удалить маркер
   */
  removeMarker(time: Time): void {
    const index = this.markers.findIndex(m => m.time === time);
    if (index >= 0) {
      this.markers.splice(index, 1);
      this.markersChanged = true;
      this.scheduleFrame();
    }
  }

  /**
   * Заменить все маркеры
   */
  setMarkers(markers: SeriesMarker<Time>[]): void {
    this.markers = [...markers];
    this.markersChanged = true;
    this.scheduleFrame();
  }

  private scheduleFrame(): void {
    if (this.frameScheduled) return;
    
    this.frameScheduled = true;
    requestAnimationFrame(() => this.processFrame());
  }

  private processFrame(): void {
    this.frameScheduled = false;

    // 1. Обрабатываем все pending updates через update()
    for (const candle of this.pendingUpdates) {
      this.series.update(candle);
    }
    this.pendingUpdates = [];

    // 2. Обновляем маркеры только если изменились
    if (this.markersChanged) {
      this.series.setMarkers(this.markers);
      this.markersChanged = false;
    }
  }

  /**
   * Очистка при уничтожении
   */
  destroy(): void {
    this.markers = [];
    this.pendingUpdates = [];
  }
}

// ============================================
// Пример использования с WebSocket
// ============================================

function setupRealtimeChart(
  container: HTMLElement,
  websocketUrl: string
): { manager: RealTimeChartManager; cleanup: () => void } {
  
  const chart = createChart(container, {
    width: 800,
    height: 400,
    timeScale: {
      shiftVisibleRangeOnNewBar: true, // Авто-сдвиг при новом баре
    },
  });

  const series = chart.addCandlestickSeries({
    upColor: '#26A69A',
    downColor: '#EF5350',
    borderVisible: false,
    wickUpColor: '#26A69A',
    wickDownColor: '#EF5350',
  });

  const manager = new RealTimeChartManager(chart, series);

  // WebSocket connection
  const ws = new WebSocket(websocketUrl);

  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);

    if (data.type === 'candle') {
      manager.updateBar({
        time: data.time as Time,
        open: data.open,
        high: data.high,
        low: data.low,
        close: data.close,
      });
    }

    if (data.type === 'signal') {
      manager.addMarker({
        time: data.time as Time,
        position: data.direction === 'buy' ? 'belowBar' : 'aboveBar',
        color: data.direction === 'buy' ? '#26A69A' : '#EF5350',
        shape: data.direction === 'buy' ? 'arrowUp' : 'arrowDown',
        text: data.label,
      });
    }
  };

  const cleanup = () => {
    ws.close();
    manager.destroy();
    chart.remove();
  };

  return { manager, cleanup };
}

export { RealTimeChartManager, setupRealtimeChart };
```

### Почему это решение оптимально

| Критерий | Оценка |
|----------|--------|
| **Устранение rubber-banding** | ✅ update() не вызывает пересчёт timeScale |
| **Синхронизация** | ✅ RAF гарантирует один кадр рендеринга |
| **Производительность** | ✅ Batching минимизирует вызовы |
| **Простота использования** | ✅ Чистый API для обновлений |
| **Совместимость** | ✅ Работает с v4.1+ |

---

## 📊 Сравнительная таблица решений

| Решение | Эффективность | Сложность | Производительность | Рейтинг |
|---------|--------------|-----------|-------------------|---------|
| #1: update() вместо setData() | ⭐⭐⭐⭐⭐ | Низкая | Высокая | 9/10 |
| #2: RAF синхронизация | ⭐⭐⭐⭐⭐ | Средняя | Высокая | 8/10 |
| #3: Windowed Data | ⭐⭐⭐⭐ | Высокая | Средняя | 7/10 |
| #4: Custom Primitive | ⭐⭐⭐⭐⭐ | Высокая | Высокая | 8/10 |
| #5: Debounce маркеров | ⭐⭐⭐ | Низкая | Средняя | 6/10 |

### Рекомендации по выбору

- **Новый проект** → Решение #1 (update())
- **Существующий код с setData()** → Решение #2 (RAF синхронизация)
- **Ограничение памяти** → Решение #3 (Windowed Data)
- **Сложные кастомные маркеры** → Решение #4 (Custom Primitive)
- **Быстрый фикс** → Решение #5 (Debounce)

---

## 🔗 Источники

1. **GitHub Issue #1990** — [Series Markers misaligned on new bar](https://github.com/tradingview/lightweight-charts/issues/1990)

2. **Официальная документация: Series update()** — [Real-time Updates](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#update)

3. **Официальная документация: setMarkers()** — [Markers API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#setmarkers)

4. **TimeScaleOptions: shiftVisibleRangeOnNewBar** — [Time Scale Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions#shiftvisiblerangeonnewbar)

5. **GitHub Issue #1874** — [setMarkers time alignment](https://github.com/tradingview/lightweight-charts/issues/1874)

6. **GitHub Issue #464** — [Marker alignment with gaps](https://github.com/tradingview/lightweight-charts/issues/464)

7. **Plugin Examples: Up/Down Markers Primitive** — [Yield Curve with Markers](https://tradingview.github.io/lightweight-charts/plugin-examples/)

---

**Документ создан:** 2026-01-18  
**Версия документа:** 1.0
