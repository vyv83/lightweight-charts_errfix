# БАГ #42: barSpacing не обновляется при масштабировании

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#371](https://github.com/tradingview/lightweight-charts/issues/371)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Последнее обновление:** Март 2020

## 📋 Описание проблемы

### Суть проблемы

При программном или пользовательском масштабировании (zoom) графика значение `barSpacing`, получаемое через `chart.timeScale().options().barSpacing`, **не обновляется динамически** и возвращает **исходное начальное значение**, а не актуальный текущий интервал между барами.

### Детали

1. **Начальное значение vs фактическое:**
   - `barSpacing` задаётся при инициализации (по умолчанию `6` пикселей)
   - После zoom in/out реальное расстояние между барами меняется
   - Но `options().barSpacing` продолжает возвращать начальное значение

2. **Проблема для разработчиков:**
   - Невозможно точно определить текущий уровень zoom
   - Невозможно рассчитать количество баров для заполнения viewport
   - Динамическая подгрузка данных при scroll становится неточной

3. **Реальный сценарий:**
   ```javascript
   // Ожидание: barSpacing меняется при zoom
   chart.timeScale().applyOptions({ barSpacing: 10 });
   // Пользователь делает zoom out
   console.log(chart.timeScale().options().barSpacing); // Всё ещё 10!
   ```

### Сценарии возникновения

- Динамическая подгрузка данных при scroll (infinite scroll)
- Расчёт количества баров для заполнения viewport
- Синхронизация zoom между несколькими графиками
- Сохранение/восстановление состояния zoom

## 🔍 Найденные решения

### Решение 1: Расчёт barSpacing через visible range (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Наиболее точный метод

```typescript
function getCurrentBarSpacing(chart: IChartApi): number {
  const timeScale = chart.timeScale();
  const visibleRange = timeScale.getVisibleLogicalRange();
  
  if (!visibleRange) {
    // Fallback на начальное значение
    return timeScale.options().barSpacing;
  }
  
  // Получаем ширину chart area (без price scale)
  const chartElement = chart.chartElement();
  const priceScaleWidth = chart.priceScale('right').width();
  const chartWidth = chartElement.clientWidth - priceScaleWidth;
  
  // Количество видимых баров
  const visibleBars = visibleRange.to - visibleRange.from;
  
  // Фактический barSpacing
  return chartWidth / visibleBars;
}
```

**Плюсы:**
- Точное значение текущего расстояния между барами
- Работает при любом уровне zoom
- Не требует внешних библиотек

**Минусы:**
- Требует доступа к DOM для получения ширины
- Необходимо учитывать ширину price scale

---

### Решение 2: Подписка на изменение visible range

**Оценка:** ⭐⭐⭐⭐ (4/5) - Реактивный подход

```typescript
interface ZoomState {
  barSpacing: number;
  visibleBars: number;
  zoomLevel: number; // 1 = default, < 1 = zoom out, > 1 = zoom in
}

function createZoomTracker(chart: IChartApi, initialBarSpacing: number): {
  getState: () => ZoomState;
  destroy: () => void;
} {
  const state: ZoomState = {
    barSpacing: initialBarSpacing,
    visibleBars: 0,
    zoomLevel: 1,
  };
  
  const chartElement = chart.chartElement();
  
  const updateState = () => {
    const visibleRange = chart.timeScale().getVisibleLogicalRange();
    if (!visibleRange) return;
    
    const priceScaleWidth = chart.priceScale('right').width();
    const chartWidth = chartElement.clientWidth - priceScaleWidth;
    const visibleBars = visibleRange.to - visibleRange.from;
    
    state.visibleBars = visibleBars;
    state.barSpacing = chartWidth / visibleBars;
    state.zoomLevel = state.barSpacing / initialBarSpacing;
  };
  
  // Подписка на изменение visible range
  const unsubscribe = chart.timeScale().subscribeVisibleLogicalRangeChange(updateState);
  
  // Инициализация
  updateState();
  
  return {
    getState: () => ({ ...state }),
    destroy: unsubscribe,
  };
}

// Использование
const zoomTracker = createZoomTracker(chart, 10);
console.log(zoomTracker.getState().barSpacing); // Актуальное значение
```

**Плюсы:**
- Автоматическое обновление при zoom/scroll
- Отслеживание уровня zoom относительно начального
- Чистая абстракция

**Минусы:**
- Дополнительная подписка (нужно не забыть destroy)
- Overhead при частых изменениях

---

### Решение 3: Расчёт количества баров для viewport

**Оценка:** ⭐⭐⭐⭐ (4/5) - Решение для data loading

```typescript
/**
 * Рассчитывает количество баров для заполнения viewport
 * Учитывает текущий уровень zoom
 */
function estimateBarsToFillViewport(
  chart: IChartApi,
  options?: {
    padding?: number; // Дополнительные бары для scroll buffer
    accountForPriceScale?: boolean;
  }
): number {
  const { padding = 50, accountForPriceScale = true } = options || {};
  
  const timeScale = chart.timeScale();
  const visibleRange = timeScale.getVisibleLogicalRange();
  
  if (visibleRange) {
    // Если есть visible range - используем его для точного расчёта
    return Math.ceil(visibleRange.to - visibleRange.from) + padding;
  }
  
  // Fallback: расчёт по начальному barSpacing
  const chartElement = chart.chartElement();
  let chartWidth = chartElement.clientWidth;
  
  if (accountForPriceScale) {
    // Примерная ширина price scale (точную можно узнать только после рендера)
    chartWidth -= 60; // Примерно 60px для price scale
  }
  
  const barSpacing = timeScale.options().barSpacing;
  return Math.ceil(chartWidth / barSpacing) + padding;
}

// Использование для lazy loading
async function loadInitialData(chart: IChartApi, series: ISeriesApi<'Candlestick'>) {
  const barsNeeded = estimateBarsToFillViewport(chart, { padding: 100 });
  const data = await fetchHistoricalData(barsNeeded);
  series.setData(data);
}
```

**Плюсы:**
- Практичное решение для data loading
- Включает buffer для плавного scroll
- Работает до и после инициализации данных

**Минусы:**
- До загрузки данных использует приблизительное значение
- Price scale width известна точно только после рендера

---

### Решение 4: Использование setVisibleLogicalRange для контроля zoom

**Оценка:** ⭐⭐⭐⭐ (4/5) - Программное управление zoom

```typescript
/**
 * Устанавливает zoom через barSpacing с реальным обновлением
 */
function setZoomLevel(
  chart: IChartApi,
  targetBarSpacing: number
): void {
  const chartElement = chart.chartElement();
  const priceScaleWidth = chart.priceScale('right').width();
  const chartWidth = chartElement.clientWidth - priceScaleWidth;
  
  // Рассчитываем количество баров для целевого barSpacing
  const targetVisibleBars = chartWidth / targetBarSpacing;
  
  // Получаем текущий центр view
  const currentRange = chart.timeScale().getVisibleLogicalRange();
  if (!currentRange) return;
  
  const center = (currentRange.from + currentRange.to) / 2;
  const halfRange = targetVisibleBars / 2;
  
  // Устанавливаем новый range (что эквивалентно zoom)
  chart.timeScale().setVisibleLogicalRange({
    from: center - halfRange,
    to: center + halfRange,
  });
}

/**
 * Получает текущий barSpacing после setVisibleLogicalRange
 */
function getActualBarSpacing(chart: IChartApi): number | null {
  const visibleRange = chart.timeScale().getVisibleLogicalRange();
  if (!visibleRange) return null;
  
  const chartElement = chart.chartElement();
  const priceScaleWidth = chart.priceScale('right').width();
  const chartWidth = chartElement.clientWidth - priceScaleWidth;
  
  return chartWidth / (visibleRange.to - visibleRange.from);
}
```

**Плюсы:**
- Полный контроль над zoom level
- Точное позиционирование
- Можно анимировать

**Минусы:**
- Требует пересчёта при изменении размера chart

---

### Решение 5: Fork библиотеки с методом getBarSpacing()

**Оценка:** ⭐⭐ (2/5) - Радикальное решение

Существует форк `lightweight-charts-with-getbarspacing` на npm, добавляющий метод `getBarSpacing()`.

```bash
npm install lightweight-charts-with-getbarspacing
```

```javascript
import { createChart } from 'lightweight-charts-with-getbarspacing';

const chart = createChart(container);
// После zoom
const actualSpacing = chart.timeScale().getBarSpacing(); // Реальное значение
```

**Плюсы:**
- Простой API
- Прямое решение проблемы

**Минусы:**
- Неофициальный форк
- Может отставать от основной версии
- Риски безопасности и поддержки

## ✅ Рекомендуемое решение

Для большинства случаев рекомендуется **комбинация решений 1 и 2** — создание обёртки для отслеживания zoom:

```typescript
import { IChartApi, LogicalRange } from 'lightweight-charts';

interface BarSpacingTracker {
  /** Получить текущий barSpacing в пикселях */
  getBarSpacing(): number;
  /** Получить текущий уровень zoom (1 = начальный) */
  getZoomLevel(): number;
  /** Получить количество видимых баров */
  getVisibleBarsCount(): number;
  /** Рассчитать баров для заполнения viewport */
  estimateBarsToFill(padding?: number): number;
  /** Очистить ресурсы */
  destroy(): void;
}

function createBarSpacingTracker(
  chart: IChartApi,
  initialBarSpacing: number = 6
): BarSpacingTracker {
  let currentBarSpacing = initialBarSpacing;
  let visibleBarsCount = 0;
  
  const chartElement = chart.chartElement();
  
  const calculateBarSpacing = (): number => {
    const visibleRange = chart.timeScale().getVisibleLogicalRange();
    if (!visibleRange) return initialBarSpacing;
    
    const bars = visibleRange.to - visibleRange.from;
    if (bars <= 0) return initialBarSpacing;
    
    // Учитываем обе price scale (если есть)
    let usedWidth = 0;
    try {
      usedWidth += chart.priceScale('right').width();
    } catch {}
    try {
      usedWidth += chart.priceScale('left').width();
    } catch {}
    
    const chartWidth = chartElement.clientWidth - usedWidth;
    return chartWidth / bars;
  };
  
  const update = (range: LogicalRange | null) => {
    if (range) {
      visibleBarsCount = range.to - range.from;
      currentBarSpacing = calculateBarSpacing();
    }
  };
  
  // Подписка на изменения
  const unsubscribe = chart.timeScale().subscribeVisibleLogicalRangeChange(update);
  
  // Начальный расчёт
  const initialRange = chart.timeScale().getVisibleLogicalRange();
  update(initialRange);
  
  return {
    getBarSpacing: () => currentBarSpacing,
    getZoomLevel: () => currentBarSpacing / initialBarSpacing,
    getVisibleBarsCount: () => visibleBarsCount,
    estimateBarsToFill: (padding = 0) => {
      if (visibleBarsCount > 0) {
        return Math.ceil(visibleBarsCount) + padding;
      }
      // Fallback
      const chartWidth = chartElement.clientWidth - 60;
      return Math.ceil(chartWidth / initialBarSpacing) + padding;
    },
    destroy: unsubscribe,
  };
}

// ==================== ПРИМЕР ИСПОЛЬЗОВАНИЯ ====================

const chart = createChart(container, {
  timeScale: {
    barSpacing: 10,
  },
});

const series = chart.addCandlestickSeries();

// Создаём tracker
const tracker = createBarSpacingTracker(chart, 10);

// Загрузка начальных данных
const initialBars = tracker.estimateBarsToFill(50);
console.log(`Нужно загрузить ${initialBars} баров`);

// После пользовательского zoom
setTimeout(() => {
  console.log('Текущий barSpacing:', tracker.getBarSpacing());
  console.log('Уровень zoom:', tracker.getZoomLevel());
  console.log('Видимых баров:', tracker.getVisibleBarsCount());
}, 5000);

// Очистка при unmount
// tracker.destroy();
```

## 📊 Сравнительная таблица решений

| Решение | Точность | Сложность | Производительность | Поддержка |
|---------|----------|-----------|-------------------|-----------|
| **#1 Расчёт через visible range** | ⭐⭐⭐⭐⭐ | Низкая | Высокая | ✅ Стабильная |
| **#2 Подписка + tracker** | ⭐⭐⭐⭐⭐ | Средняя | Высокая | ✅ Стабильная |
| **#3 estimateBarsToFillViewport** | ⭐⭐⭐⭐ | Низкая | Высокая | ✅ Стабильная |
| **#4 setVisibleLogicalRange** | ⭐⭐⭐⭐ | Средняя | Высокая | ✅ Стабильная |
| **#5 Fork библиотеки** | ⭐⭐⭐ | Низкая | Высокая | ⚠️ Неофициальная |

## 🔧 Дополнительные утилиты

### Синхронизация zoom между графиками

```typescript
function syncZoom(sourceChart: IChartApi, targetCharts: IChartApi[]): () => void {
  const handler = (range: LogicalRange | null) => {
    if (!range) return;
    targetCharts.forEach(target => {
      target.timeScale().setVisibleLogicalRange(range);
    });
  };
  
  return sourceChart.timeScale().subscribeVisibleLogicalRangeChange(handler);
}
```

### Сохранение/восстановление zoom state

```typescript
interface ZoomState {
  visibleRange: LogicalRange;
  scrollPosition: number;
}

function saveZoomState(chart: IChartApi): ZoomState | null {
  const range = chart.timeScale().getVisibleLogicalRange();
  if (!range) return null;
  
  return {
    visibleRange: range,
    scrollPosition: chart.timeScale().scrollPosition(),
  };
}

function restoreZoomState(chart: IChartApi, state: ZoomState): void {
  chart.timeScale().setVisibleLogicalRange(state.visibleRange);
}
```

## 🔗 Источники

- [GitHub Issue #371](https://github.com/tradingview/lightweight-charts/issues/371) - Оригинальный feature request
- [TimeScaleOptions API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions) - Документация barSpacing
- [Time Scale Tutorial](https://tradingview.github.io/lightweight-charts/tutorials/customization/time-scale) - Официальный туториал
- [Issue #753](https://github.com/tradingview/lightweight-charts/issues/753) - How to calculate bars in viewport
- [Issue #1404](https://github.com/tradingview/lightweight-charts/issues/1404) - Lazy loading historical data
- [StackOverflow: handle zooming](https://stackoverflow.com/questions/71243541/handle-zooming-in-lightweight-chart) - Примеры zoom control

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
