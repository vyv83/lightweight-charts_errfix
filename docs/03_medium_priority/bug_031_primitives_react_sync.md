# БАГ #31: Примитивы не синхронизируются при scroll/zoom в React

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1920](https://github.com/tradingview/lightweight-charts/issues/1920)  
> **Версии:** v5.0.8+  
> **Статус:** ✅ Closed (Исправлено в v5.0.9+)  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы
При использовании custom primitives в React-приложении примитивы не синхронизируются с графиком при прокрутке и масштабировании:

- Примитивы "плавают" отдельно от данных графика
- Координаты примитивов становятся устаревшими (stale)
- Визуальные элементы отрисовываются с задержкой или в неправильных позициях

### Пример проблемы

```javascript
// Примитив с проблемой синхронизации
class DebugPrimitive implements ISeriesPrimitive<Time> {
  private _param: SeriesAttachedParameter<Time> | null = null;
  
  attached(param: SeriesAttachedParameter<Time>) {
    this._param = param;  // ⚠️ Ссылка может стать устаревшей
  }
  
  paneViews() {
    return [this._paneView];
  }
  
  // ⚠️ Пустая реализация - координаты не обновляются!
  updateAllViews() {}
}

// Результат:
// ❌ При scroll/zoom примитив "плавает" отдельно от свечей
// ❌ Координаты вычисляются из устаревших ссылок
```

### Визуальное проявление
- Примитив (маркер, линия, область) остаётся на экранных координатах первоначальной отрисовки
- При прокрутке графика примитив не следует за данными
- После zoom примитив оказывается в неверной позиции

### Воспроизведение

```tsx
import React, { useEffect, useRef } from 'react';
import { 
  createChart, Time, ISeriesPrimitive, 
  SeriesAttachedParameter, IPrimitivePaneView, 
  CandlestickSeries, CandlestickData 
} from 'lightweight-charts';

const candleData: CandlestickData<Time>[] = [
  { time: 1737032400 as Time, open: 238.0, high: 238.6, low: 237.5, close: 238.55 },
  { time: 1737036000 as Time, open: 238.55, high: 239.0, low: 237.8, close: 238.2 },
  { time: 1737039600 as Time, open: 238.2, high: 238.8, low: 236.5, close: 237.0 },
];

const singleTrade = {
  entry_time: "2025-01-16T13:00:00.000Z",
  entry_price: 238.55,
};

const parseTime = (isoString: string) => 
  (new Date(isoString).getTime() / 1000) as Time;

// Проблемный примитив
class DebugPaneView implements IPrimitivePaneView {
  private readonly _primitive: DebugPrimitive;
  
  constructor(primitive: DebugPrimitive) {
    this._primitive = primitive;
  }
  
  renderer() {
    const paneView = this;
    return {
      draw: (target: any) => {
        target.useBitmapCoordinateSpace((scope: any) => {
          const param = paneView._primitive.param();
          if (!param) return;
          
          const { series, chart } = param;
          const ctx = scope.context;
          
          // ⚠️ Эти координаты становятся устаревшими при scroll/zoom
          const x = chart.timeScale().timeToCoordinate(
            parseTime(singleTrade.entry_time)
          );
          const y = series.priceToCoordinate(singleTrade.entry_price);
          
          if (x !== null && y !== null) {
            ctx.fillStyle = 'blue';
            ctx.fillRect(x - 5, y - 5, 10, 10);
          }
        });
      }
    };
  }
}

class DebugPrimitive implements ISeriesPrimitive<Time> {
  private _param: SeriesAttachedParameter<Time> | null = null;
  private readonly _paneView: DebugPaneView;
  
  constructor() {
    this._paneView = new DebugPaneView(this);
  }
  
  attached(param: SeriesAttachedParameter<Time>) {
    this._param = param;
  }
  
  detached() {
    this._param = null;
  }
  
  paneViews() {
    return [this._paneView];
  }
  
  param() {
    return this._param;
  }
  
  // ⚠️ ПРОБЛЕМА: Пустая реализация!
  updateAllViews() {}
}

export default function App() {
  const chartContainerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (!chartContainerRef.current) return;
    
    const chart = createChart(chartContainerRef.current);
    const candleSeries = chart.addSeries(CandlestickSeries, {});
    candleSeries.setData(candleData);
    
    const primitive = new DebugPrimitive();
    candleSeries.attachPrimitive(primitive);
    
    chart.timeScale().fitContent();
    
    return () => { chart.remove(); };
  }, []);
  
  return <div ref={chartContainerRef} style={{ width: '800px', height: '600px' }} />;
}
```

---

## 🔍 Найденные решения

### Решение 1: Обновление библиотеки до v5.0.9+
**Оценка: 10/10** ⭐ ОФИЦИАЛЬНЫЙ ФИКС

Issue #1920 была закрыта 12 июля 2025 как "completed". Обновление до последней версии решает проблему.

```bash
npm update lightweight-charts
# или
npm install lightweight-charts@latest
```

**Плюсы:**
- Официальное решение
- Не требует изменения кода
- Работает "из коробки"

**Минусы:**
- Требует обновления зависимости
- Может потребовать проверки совместимости

---

### Решение 2: Корректная реализация updateAllViews()
**Оценка: 8/10**

Правильная реализация метода `updateAllViews()` для обновления состояния примитива.

```typescript
class SyncedPrimitive implements ISeriesPrimitive<Time> {
  private _param: SeriesAttachedParameter<Time> | null = null;
  private _requestUpdate: (() => void) | null = null;
  private readonly _paneViews: SyncedPaneView[];
  
  // Данные примитива
  private _coordinates: { x: number | null; y: number | null } = { x: null, y: null };
  
  constructor(private _time: Time, private _price: number) {
    this._paneViews = [new SyncedPaneView(this)];
  }
  
  attached(param: SeriesAttachedParameter<Time>) {
    this._param = param;
    this._requestUpdate = param.requestUpdate;
    
    // Начальный расчёт координат
    this._updateCoordinates();
  }
  
  detached() {
    this._param = null;
    this._requestUpdate = null;
  }
  
  paneViews() {
    return this._paneViews;
  }
  
  // ✅ КЛЮЧЕВОЙ МЕТОД: Вызывается при каждом обновлении viewport
  updateAllViews() {
    this._updateCoordinates();
  }
  
  private _updateCoordinates() {
    if (!this._param) {
      this._coordinates = { x: null, y: null };
      return;
    }
    
    const { series, chart } = this._param;
    this._coordinates = {
      x: chart.timeScale().timeToCoordinate(this._time),
      y: series.priceToCoordinate(this._price)
    };
  }
  
  // Геттер для paneView
  getCoordinates() {
    return this._coordinates;
  }
  
  // Метод для внешнего обновления данных
  setPosition(time: Time, price: number) {
    this._time = time;
    this._price = price;
    this._updateCoordinates();
    this._requestUpdate?.();
  }
}

class SyncedPaneView implements IPrimitivePaneView {
  constructor(private _primitive: SyncedPrimitive) {}
  
  renderer() {
    const primitive = this._primitive;
    return {
      draw: (target: any) => {
        target.useBitmapCoordinateSpace((scope: any) => {
          const { x, y } = primitive.getCoordinates();
          if (x === null || y === null) return;
          
          const ctx = scope.context;
          const ratio = scope.horizontalPixelRatio;
          
          ctx.fillStyle = 'blue';
          ctx.beginPath();
          ctx.arc(x * ratio, y * ratio, 5 * ratio, 0, 2 * Math.PI);
          ctx.fill();
        });
      }
    };
  }
}
```

**Плюсы:**
- Соответствует документации API
- Координаты всегда актуальны
- Правильная архитектура

**Минусы:**
- Требует рефакторинга существующего кода
- Сложность при миграции больших проектов

---

### Решение 3: Подписка на события timeScale
**Оценка: 6/10**

Ручная подписка на изменения временной шкалы для принудительного обновления.

```typescript
class ManualSyncPrimitive implements ISeriesPrimitive<Time> {
  private _param: SeriesAttachedParameter<Time> | null = null;
  private _unsubscribe: (() => void) | null = null;
  
  attached(param: SeriesAttachedParameter<Time>) {
    this._param = param;
    
    // Подписываемся на изменения visible range
    const handler = () => {
      // Форсируем перерисовку
      param.requestUpdate();
    };
    
    param.chart.timeScale().subscribeVisibleTimeRangeChange(handler);
    param.chart.timeScale().subscribeVisibleLogicalRangeChange(handler);
    
    this._unsubscribe = () => {
      param.chart.timeScale().unsubscribeVisibleTimeRangeChange(handler);
      param.chart.timeScale().unsubscribeVisibleLogicalRangeChange(handler);
    };
  }
  
  detached() {
    this._unsubscribe?.();
    this._unsubscribe = null;
    this._param = null;
  }
  
  updateAllViews() {
    // Обновляем состояние
  }
  
  paneViews() {
    return [/* pane views */];
  }
}
```

**Плюсы:**
- Гарантированное обновление при scroll/zoom
- Не зависит от внутренней логики библиотеки

**Минусы:**
- Избыточные вызовы requestUpdate
- Потенциальные проблемы с производительностью
- Дополнительная сложность с cleanup

---

### Решение 4: Использование useRef для хранения актуальных ссылок в React
**Оценка: 7/10**

Паттерн React для предотвращения stale closures.

```tsx
import React, { useEffect, useRef, useCallback } from 'react';
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

function ChartWithPrimitive({ data, primitiveData }) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  // ✅ Используем refs для хранения актуальных ссылок
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Candlestick'> | null>(null);
  const primitiveRef = useRef<MyPrimitive | null>(null);
  
  // Класс примитива с доступом к refs
  class MyPrimitive implements ISeriesPrimitive<Time> {
    private _param: SeriesAttachedParameter<Time> | null = null;
    
    attached(param: SeriesAttachedParameter<Time>) {
      this._param = param;
    }
    
    detached() {
      this._param = null;
    }
    
    updateAllViews() {
      // Обновление логики
    }
    
    paneViews() {
      return [{
        renderer: () => ({
          draw: (target: any) => {
            // ✅ Используем АКТУАЛЬНЫЕ ссылки из refs
            const chart = chartRef.current;
            const series = seriesRef.current;
            
            if (!chart || !series) return;
            
            target.useBitmapCoordinateSpace((scope: any) => {
              const x = chart.timeScale().timeToCoordinate(primitiveData.time);
              const y = series.priceToCoordinate(primitiveData.price);
              
              if (x !== null && y !== null) {
                const ctx = scope.context;
                ctx.fillStyle = 'green';
                ctx.fillRect(x - 5, y - 5, 10, 10);
              }
            });
          }
        })
      }];
    }
  }
  
  useEffect(() => {
    if (!containerRef.current) return;
    
    const chart = createChart(containerRef.current, { width: 800, height: 600 });
    chartRef.current = chart;
    
    const series = chart.addSeries(CandlestickSeries);
    seriesRef.current = series;
    series.setData(data);
    
    const primitive = new MyPrimitive();
    primitiveRef.current = primitive;
    series.attachPrimitive(primitive);
    
    chart.timeScale().fitContent();
    
    return () => {
      chart.remove();
      chartRef.current = null;
      seriesRef.current = null;
      primitiveRef.current = null;
    };
  }, []);
  
  // Обновление при изменении primitiveData
  useEffect(() => {
    if (primitiveRef.current && chartRef.current) {
      // Форсируем обновление
      chartRef.current.timeScale().applyOptions({});
    }
  }, [primitiveData]);
  
  return <div ref={containerRef} />;
}
```

**Плюсы:**
- Идиоматичный React-подход
- Избегает stale closures
- Хорошо интегрируется с React lifecycle

**Минусы:**
- Дополнительная сложность кода
- Необходимо управлять refs вручную
- Может быть сложно при глубокой вложенности

---

### Решение 5: Использование React-обёртки
**Оценка: 7/10**

Использование community React-обёрток, которые решают проблемы интеграции.

```bash
npm install lightweight-charts-react-components
```

```tsx
import { Chart, CandlestickSeries, CustomPrimitive } from 'lightweight-charts-react-components';

function TradingChart({ data, markerPosition }) {
  return (
    <Chart width={800} height={600}>
      <CandlestickSeries data={data}>
        <CustomPrimitive
          time={markerPosition.time}
          price={markerPosition.price}
          render={({ x, y, ctx }) => {
            if (x !== null && y !== null) {
              ctx.fillStyle = 'blue';
              ctx.fillRect(x - 5, y - 5, 10, 10);
            }
          }}
        />
      </CandlestickSeries>
    </Chart>
  );
}
```

**Плюсы:**
- Декларативный API
- Управление lifecycle из коробки
- Меньше boilerplate

**Минусы:**
- Дополнительная зависимость
- Меньше контроля
- Возможные ограничения функционала

---

## ✅ Рекомендуемое решение

### Для новых проектов: Обновление + Правильная реализация

Комбинация обновления библиотеки и корректной реализации примитивов.

```typescript
// synced-primitive.ts
import { 
  ISeriesPrimitive, 
  SeriesAttachedParameter, 
  IPrimitivePaneView,
  Time 
} from 'lightweight-charts';

interface MarkerData {
  time: Time;
  price: number;
  color?: string;
  radius?: number;
}

/**
 * Правильно синхронизированный примитив для маркера
 */
export class SyncedMarkerPrimitive implements ISeriesPrimitive<Time> {
  private _param: SeriesAttachedParameter<Time> | null = null;
  private _requestUpdate: (() => void) | null = null;
  private _paneView: SyncedMarkerPaneView;
  
  // Кэшированные координаты
  private _x: number | null = null;
  private _y: number | null = null;
  
  constructor(private _data: MarkerData) {
    this._paneView = new SyncedMarkerPaneView(this);
  }
  
  // === Lifecycle ===
  
  attached(param: SeriesAttachedParameter<Time>) {
    this._param = param;
    this._requestUpdate = param.requestUpdate;
    this._recalculateCoordinates();
  }
  
  detached() {
    this._param = null;
    this._requestUpdate = null;
  }
  
  // === Core API ===
  
  paneViews(): readonly IPrimitivePaneView[] {
    return [this._paneView];
  }
  
  /**
   * ✅ КРИТИЧЕСКИ ВАЖНЫЙ МЕТОД
   * Вызывается библиотекой при каждом обновлении viewport (scroll, zoom)
   */
  updateAllViews(): void {
    this._recalculateCoordinates();
  }
  
  // === Internal ===
  
  private _recalculateCoordinates(): void {
    if (!this._param) {
      this._x = null;
      this._y = null;
      return;
    }
    
    const { chart, series } = this._param;
    this._x = chart.timeScale().timeToCoordinate(this._data.time);
    this._y = series.priceToCoordinate(this._data.price);
  }
  
  // === Public API ===
  
  getX(): number | null { return this._x; }
  getY(): number | null { return this._y; }
  getData(): MarkerData { return this._data; }
  
  setData(data: Partial<MarkerData>): void {
    Object.assign(this._data, data);
    this._recalculateCoordinates();
    this._requestUpdate?.();
  }
}

class SyncedMarkerPaneView implements IPrimitivePaneView {
  constructor(private _primitive: SyncedMarkerPrimitive) {}
  
  renderer() {
    const primitive = this._primitive;
    
    return {
      draw(target: any) {
        target.useBitmapCoordinateSpace((scope: any) => {
          const x = primitive.getX();
          const y = primitive.getY();
          
          if (x === null || y === null) return;
          
          const data = primitive.getData();
          const ctx = scope.context;
          const ratio = scope.horizontalPixelRatio;
          
          const scaledX = x * ratio;
          const scaledY = y * ratio;
          const scaledRadius = (data.radius ?? 5) * ratio;
          
          ctx.save();
          ctx.fillStyle = data.color ?? '#2196F3';
          ctx.beginPath();
          ctx.arc(scaledX, scaledY, scaledRadius, 0, 2 * Math.PI);
          ctx.fill();
          ctx.restore();
        });
      }
    };
  }
}

// === Использование в React ===

// App.tsx
import React, { useEffect, useRef } from 'react';
import { createChart, CandlestickSeries, Time } from 'lightweight-charts';
import { SyncedMarkerPrimitive } from './synced-primitive';

export function ChartWithSyncedPrimitive() {
  const containerRef = useRef<HTMLDivElement>(null);
  const primitiveRef = useRef<SyncedMarkerPrimitive | null>(null);
  
  useEffect(() => {
    if (!containerRef.current) return;
    
    const chart = createChart(containerRef.current, {
      width: 800,
      height: 600,
      layout: { background: { color: '#1e222d' }, textColor: '#d1d4dc' }
    });
    
    const series = chart.addSeries(CandlestickSeries, {
      upColor: '#26a69a',
      downColor: '#ef5350',
      borderVisible: false,
      wickUpColor: '#26a69a',
      wickDownColor: '#ef5350'
    });
    
    series.setData([
      { time: '2025-01-01', open: 100, high: 105, low: 98, close: 103 },
      { time: '2025-01-02', open: 103, high: 108, low: 102, close: 107 },
      { time: '2025-01-03', open: 107, high: 110, low: 105, close: 109 },
      { time: '2025-01-04', open: 109, high: 112, low: 106, close: 108 },
      { time: '2025-01-05', open: 108, high: 115, low: 107, close: 114 }
    ]);
    
    // ✅ Создаём синхронизированный примитив
    const primitive = new SyncedMarkerPrimitive({
      time: '2025-01-03' as Time,
      price: 109,
      color: '#00e676',
      radius: 8
    });
    
    primitiveRef.current = primitive;
    series.attachPrimitive(primitive);
    
    chart.timeScale().fitContent();
    
    return () => {
      chart.remove();
    };
  }, []);
  
  // Пример динамического обновления
  const handleMoveMarker = () => {
    primitiveRef.current?.setData({
      time: '2025-01-04' as Time,
      price: 110,
      color: '#ff9800'
    });
  };
  
  return (
    <div>
      <div ref={containerRef} />
      <button onClick={handleMoveMarker}>Move Marker</button>
    </div>
  );
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Надёжность | Производительность | Совместимость |
|---------|--------|-----------|------------|-------------------|---------------|
| 1. Обновление библиотеки | **10/10** | Низкая | ✅ Высокая | ✅ Оптимальная | ✅ v5.0.9+ |
| 2. Корректный updateAllViews | 8/10 | Средняя | ✅ Высокая | ✅ Оптимальная | ✅ Все версии |
| 3. Подписка на timeScale | 6/10 | Средняя | ⚠️ Средняя | ⚠️ Избыточная | ✅ Все версии |
| 4. useRef в React | 7/10 | Высокая | ✅ Высокая | ✅ Хорошая | ✅ Все версии |
| 5. React-обёртка | 7/10 | Низкая | ✅ Высокая | ✅ Хорошая | ⚠️ Зависит |

### Рекомендации:

- **Быстрое решение** → Решение 1 (обновление библиотеки) ✅
- **Если нельзя обновить** → Решение 2 (правильный updateAllViews)
- **React-проекты** → Решение 1 + Решение 4 (комбинация)

---

## 🔗 Источники

1. [GitHub Issue #1920](https://github.com/tradingview/lightweight-charts/issues/1920) - Оригинальный баг-репорт (CLOSED)
2. [Primitives Documentation](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/plugins) - Официальная документация
3. [ISeriesPrimitive Interface](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesPrimitive) - API Reference
4. [React Integration Guide](https://tradingview.github.io/lightweight-charts/tutorials/react/simple) - Интеграция с React
5. [lightweight-charts-react-components](https://www.npmjs.com/package/lightweight-charts-react-components) - Community React wrapper
6. [StackBlitz Reproduction](https://stackblitz.com/edit/react-ts-vite-fty5m5pe) - Пример воспроизведения

---

## 📝 Примечания

### Статус исправления
Issue #1920 была закрыта как "completed" 12 июля 2025. Баг исправлен в версии **v5.0.9** и выше.

### Ключевые моменты для разработчиков

1. **Всегда реализуйте `updateAllViews()`** — этот метод вызывается при каждом обновлении viewport
2. **Вычисляйте координаты в `updateAllViews()`**, а не только в `attached()`
3. **Используйте `requestUpdate()`** при изменении внутреннего состояния примитива
4. **В React используйте `useRef`** для хранения ссылок на chart/series

### Чек-лист для примитивов

- [ ] Метод `attached()` сохраняет `param` и `requestUpdate`
- [ ] Метод `detached()` очищает ссылки
- [ ] Метод `updateAllViews()` пересчитывает координаты
- [ ] Renderer использует кэшированные координаты из `updateAllViews()`
- [ ] При изменении данных вызывается `requestUpdate()`
