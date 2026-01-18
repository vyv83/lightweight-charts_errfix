# БАГ #23: setData нарушает visible range

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1875](https://github.com/tradingview/lightweight-charts/issues/1875), [#549](https://github.com/tradingview/lightweight-charts/issues/549), [#1395](https://github.com/tradingview/lightweight-charts/issues/1395)  
> **Версии:** Все версии  
> **Платформы:** Все браузеры  
> **Статус:** 🔴 Open (задокументировано как expected, но unexpected для users)

---

## 📋 Описание проблемы

### Суть проблемы
При вызове `series.setData()` график неожиданно изменяет видимую область (visible range), "прыгая" к другой позиции. Это нарушает пользовательский опыт, особенно при:
- Подгрузке исторических данных (infinite scroll)
- Переключении timeframes
- Обновлении данных в реальном времени

### Типичные сценарии

1. **Пользователь прокручивает график влево (в прошлое):**
   ```typescript
   // Пользователь смотрит на данные за прошлый месяц
   // Подгружаем исторические данные
   const allData = [...historicalData, ...currentData];
   series.setData(allData);
   // ❌ График "прыгает" к последним данным
   ```

2. **Переключение timeframe:**
   ```typescript
   // Пользователь смотрит на определённый период
   // Меняем timeframe с 1h на 1d
   series.setData(dailyData);
   // ❌ Видимая область сбрасывается
   ```

3. **Обновление данных во время взаимодействия:**
   ```typescript
   // Пользователь зумирует график
   // Приходит обновление через WebSocket
   series.setData(updatedData);
   // ❌ Зум сбрасывается, график "дёргается"
   ```

### Ожидаемое vs Фактическое поведение

| Ожидание | Реальность |
|----------|------------|
| Данные обновляются, видимая область сохраняется | Видимая область сбрасывается к последним данным |
| Плавный UX при подгрузке истории | "Прыжок" графика после setData |
| Сохранение zoom уровня | Zoom может измениться |

### Почему это критично

- **UX проблема**: Пользователи теряют контекст при каждом обновлении
- **Infinite scroll**: Невозможно реализовать без workaround
- **Real-time apps**: "Дёрганье" графика при частых обновлениях

---

## 🔍 Найденные решения

### Решение 1: Сохранение и восстановление Logical Range

**Оценка: 8/10**

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  LogicalRange,
  LineData
} from 'lightweight-charts';

/**
 * Обновляет данные серии с сохранением видимой области
 */
function setDataPreservingRange<T extends LineData>(
  chart: IChartApi,
  series: ISeriesApi<'Line'>,
  data: T[]
): void {
  // 1. Сохраняем текущий visible range
  const currentRange = chart.timeScale().getVisibleLogicalRange();
  
  // 2. Обновляем данные
  series.setData(data);
  
  // 3. Восстанавливаем visible range
  if (currentRange !== null) {
    chart.timeScale().setVisibleLogicalRange(currentRange);
  }
}

// Использование
setDataPreservingRange(chart, lineSeries, newData);
```

**Плюсы:**
- Простая реализация
- Работает в большинстве случаев
- Не требует модификации библиотеки

**Минусы:**
- Может создавать "jitter" если пользователь активно взаимодействует с графиком
- Небольшая задержка между setData и восстановлением range

---

### Решение 2: Debounced обновление с блокировкой

**Оценка: 7/10**

```typescript
import { 
  IChartApi, 
  ISeriesApi,
  LogicalRange,
  SeriesType
} from 'lightweight-charts';

class ChartDataManager<T extends SeriesType> {
  private chart: IChartApi;
  private series: ISeriesApi<T>;
  private isUpdating: boolean = false;
  private pendingData: any[] | null = null;
  private updateTimeout: number | null = null;

  constructor(chart: IChartApi, series: ISeriesApi<T>) {
    this.chart = chart;
    this.series = series;
  }

  /**
   * Обновляет данные с debounce и сохранением range
   */
  setData(data: any[], debounceMs: number = 100): void {
    this.pendingData = data;

    if (this.updateTimeout) {
      clearTimeout(this.updateTimeout);
    }

    this.updateTimeout = window.setTimeout(() => {
      this.performUpdate();
    }, debounceMs);
  }

  private performUpdate(): void {
    if (this.isUpdating || !this.pendingData) return;

    this.isUpdating = true;

    // Сохраняем range
    const range = this.chart.timeScale().getVisibleLogicalRange();

    // Обновляем данные
    this.series.setData(this.pendingData);
    this.pendingData = null;

    // Восстанавливаем range на следующем кадре
    requestAnimationFrame(() => {
      if (range !== null) {
        this.chart.timeScale().setVisibleLogicalRange(range);
      }
      this.isUpdating = false;
    });
  }

  /**
   * Принудительное немедленное обновление
   */
  forceUpdate(data: any[]): void {
    if (this.updateTimeout) {
      clearTimeout(this.updateTimeout);
    }
    this.pendingData = data;
    this.performUpdate();
  }
}

// Использование
const dataManager = new ChartDataManager(chart, lineSeries);
dataManager.setData(newData);
```

**Плюсы:**
- Debounce предотвращает множественные обновления
- Использует requestAnimationFrame для плавности
- Предотвращает race conditions

**Минусы:**
- Более сложная реализация
- Задержка debounce может быть заметна

---

### Решение 3: Использование series.update() вместо setData()

**Оценка: 9/10**

```typescript
import { 
  ISeriesApi,
  LineData,
  UTCTimestamp
} from 'lightweight-charts';

/**
 * Инкрементальное обновление данных без нарушения visible range
 */
class IncrementalDataUpdater {
  private series: ISeriesApi<'Line'>;
  private currentData: Map<UTCTimestamp, LineData> = new Map();

  constructor(series: ISeriesApi<'Line'>) {
    this.series = series;
  }

  /**
   * Инициализация с начальными данными
   */
  initialize(data: LineData[]): void {
    this.currentData.clear();
    data.forEach(item => {
      this.currentData.set(item.time as UTCTimestamp, item);
    });
    this.series.setData(data);
  }

  /**
   * Добавление нового бара (не нарушает visible range)
   */
  appendBar(bar: LineData): void {
    this.currentData.set(bar.time as UTCTimestamp, bar);
    this.series.update(bar);
  }

  /**
   * Обновление существующего бара
   */
  updateBar(bar: LineData): void {
    const time = bar.time as UTCTimestamp;
    if (this.currentData.has(time)) {
      this.currentData.set(time, bar);
      this.series.update(bar);
    }
  }

  /**
   * Добавление исторических данных (для infinite scroll)
   * С сохранением visible range
   */
  prependHistory(
    chart: import('lightweight-charts').IChartApi,
    historicalData: LineData[]
  ): void {
    // Сохраняем range
    const range = chart.timeScale().getVisibleLogicalRange();

    // Добавляем исторические данные в начало
    historicalData.forEach(item => {
      this.currentData.set(item.time as UTCTimestamp, item);
    });

    // Сортируем по времени
    const allData = Array.from(this.currentData.values())
      .sort((a, b) => (a.time as number) - (b.time as number));

    // Вычисляем смещение (количество добавленных баров)
    const offset = historicalData.length;

    // Обновляем данные
    this.series.setData(allData);

    // Восстанавливаем range со смещением
    if (range !== null) {
      chart.timeScale().setVisibleLogicalRange({
        from: range.from + offset,
        to: range.to + offset,
      });
    }
  }
}

// Использование
const updater = new IncrementalDataUpdater(lineSeries);
updater.initialize(initialData);

// Для real-time обновлений
updater.appendBar(newBar); // ✅ Не нарушает visible range

// Для подгрузки истории
updater.prependHistory(chart, historicalBars);
```

**Плюсы:**
- `update()` специально разработан для real-time данных
- Не нарушает visible range при append
- Для prepend - корректное смещение range

**Минусы:**
- Более сложная логика для prepend
- Нужно отслеживать данные отдельно

---

### Решение 4: TimeScale опции для контроля поведения

**Оценка: 6/10**

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  timeScale: {
    // Отключаем автоматический сдвиг при новых барах
    shiftVisibleRangeOnNewBar: false,
    
    // Контроль правого отступа
    rightOffset: 5,
    
    // Минимальный отступ справа (в барах)
    minBarSpacing: 0.5,
    
    // Фиксируем правую границу
    fixRightEdge: false,
    fixLeftEdge: false,
  },
});

// Дополнительно: подписываемся на изменение visible range
chart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
  if (range !== null) {
    // Можем сохранять текущий range для последующего восстановления
    localStorage.setItem('chartRange', JSON.stringify(range));
  }
});
```

**Плюсы:**
- Встроенные опции библиотеки
- Просто настроить

**Минусы:**
- `shiftVisibleRangeOnNewBar` не полностью решает проблему с setData
- Не работает для всех сценариев

---

### Решение 5: Wrapper с полным контролем состояния

**Оценка: 9/10**

```typescript
import {
  createChart,
  IChartApi,
  ISeriesApi,
  SeriesType,
  LogicalRange,
  Time,
  SingleValueData,
  CandlestickData,
} from 'lightweight-charts';

interface ChartState {
  visibleRange: LogicalRange | null;
  isUserInteracting: boolean;
  lastUpdateTime: number;
}

class StableChart<T extends SeriesType = 'Line'> {
  private chart: IChartApi;
  private series: ISeriesApi<T>;
  private state: ChartState = {
    visibleRange: null,
    isUserInteracting: false,
    lastUpdateTime: 0,
  };
  private interactionTimeout: number | null = null;

  constructor(
    container: HTMLElement,
    chartOptions: Parameters<typeof createChart>[1],
    seriesType: T
  ) {
    this.chart = createChart(container, {
      ...chartOptions,
      timeScale: {
        ...chartOptions?.timeScale,
        shiftVisibleRangeOnNewBar: false,
      },
    });

    // Создаём серию (упрощённо для примера)
    this.series = this.chart.addSeries(seriesType as any) as ISeriesApi<T>;

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    // Отслеживаем изменения visible range
    this.chart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
      if (!this.state.isUserInteracting) {
        this.state.visibleRange = range;
      }
    });

    // Отслеживаем взаимодействие пользователя
    const container = this.chart.chartElement();
    
    container.addEventListener('mousedown', () => {
      this.state.isUserInteracting = true;
      this.clearInteractionTimeout();
    });

    container.addEventListener('mouseup', () => {
      this.scheduleInteractionEnd();
    });

    container.addEventListener('wheel', () => {
      this.state.isUserInteracting = true;
      this.scheduleInteractionEnd();
    });

    container.addEventListener('touchstart', () => {
      this.state.isUserInteracting = true;
      this.clearInteractionTimeout();
    });

    container.addEventListener('touchend', () => {
      this.scheduleInteractionEnd();
    });
  }

  private clearInteractionTimeout(): void {
    if (this.interactionTimeout !== null) {
      clearTimeout(this.interactionTimeout);
      this.interactionTimeout = null;
    }
  }

  private scheduleInteractionEnd(): void {
    this.clearInteractionTimeout();
    this.interactionTimeout = window.setTimeout(() => {
      this.state.isUserInteracting = false;
      // Обновляем сохранённый range после окончания взаимодействия
      this.state.visibleRange = this.chart.timeScale().getVisibleLogicalRange();
    }, 200);
  }

  /**
   * Устанавливает данные с гарантированным сохранением visible range
   */
  setData(data: Parameters<ISeriesApi<T>['setData']>[0]): void {
    // Сохраняем текущий range
    const rangeToRestore = this.state.isUserInteracting
      ? this.chart.timeScale().getVisibleLogicalRange()
      : this.state.visibleRange;

    // Обновляем данные
    this.series.setData(data);
    this.state.lastUpdateTime = Date.now();

    // Восстанавливаем range
    if (rangeToRestore !== null) {
      // Используем requestAnimationFrame для гарантии
      requestAnimationFrame(() => {
        this.chart.timeScale().setVisibleLogicalRange(rangeToRestore);
      });
    }
  }

  /**
   * Добавляет исторические данные с корректным смещением
   */
  prependData(
    existingData: Parameters<ISeriesApi<T>['setData']>[0],
    newHistoricalData: Parameters<ISeriesApi<T>['setData']>[0]
  ): void {
    const range = this.chart.timeScale().getVisibleLogicalRange();
    const offset = (newHistoricalData as any[]).length;

    // Объединяем данные
    const allData = [...(newHistoricalData as any[]), ...(existingData as any[])];
    
    this.series.setData(allData as any);

    // Восстанавливаем range со смещением
    if (range !== null) {
      requestAnimationFrame(() => {
        this.chart.timeScale().setVisibleLogicalRange({
          from: range.from + offset,
          to: range.to + offset,
        });
      });
    }
  }

  /**
   * Обновляет последний бар (не нарушает range)
   */
  update(data: Parameters<ISeriesApi<T>['update']>[0]): void {
    this.series.update(data);
  }

  /**
   * Получить API графика для прямого доступа
   */
  getChart(): IChartApi {
    return this.chart;
  }

  /**
   * Получить API серии
   */
  getSeries(): ISeriesApi<T> {
    return this.series;
  }

  /**
   * Очистка ресурсов
   */
  destroy(): void {
    this.clearInteractionTimeout();
    this.chart.remove();
  }
}

// Использование
const stableChart = new StableChart(container, {
  width: 800,
  height: 400,
}, 'Line');

// Установка данных - visible range сохранится
stableChart.setData(data);

// Подгрузка истории - range сместится корректно
stableChart.prependData(currentData, historicalData);

// Real-time update - range не затрагивается
stableChart.update(newBar);
```

**Плюсы:**
- Полный контроль над состоянием
- Учитывает взаимодействие пользователя
- Работает для всех сценариев
- Предотвращает "jitter"

**Минусы:**
- Наиболее сложная реализация
- Дополнительная абстракция над библиотекой

---

## ✅ Рекомендуемое решение

### Комбинированный подход с инкрементальными обновлениями

Для большинства случаев рекомендуется **Решение 3** (использование `update()`) для real-time данных и **Решение 5** (StableChart wrapper) для полного контроля.

#### Минимальный рабочий пример

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  LineData,
  LogicalRange 
} from 'lightweight-charts';

// ========================================
// Утилиты для сохранения visible range
// ========================================

/**
 * Выполняет setData с сохранением visible range
 */
export function setDataSafe<T extends LineData>(
  chart: IChartApi,
  series: ISeriesApi<'Line'>,
  data: T[],
  options: {
    offsetBars?: number;  // Смещение при prepend
    animate?: boolean;    // Анимировать переход
  } = {}
): void {
  const { offsetBars = 0, animate = false } = options;
  
  // Сохраняем текущий range
  const range = chart.timeScale().getVisibleLogicalRange();
  
  // Обновляем данные
  series.setData(data);
  
  // Восстанавливаем range
  if (range !== null) {
    const newRange: LogicalRange = {
      from: range.from + offsetBars,
      to: range.to + offsetBars,
    };
    
    if (animate) {
      // Плавный переход (требует requestAnimationFrame)
      requestAnimationFrame(() => {
        chart.timeScale().setVisibleLogicalRange(newRange);
      });
    } else {
      chart.timeScale().setVisibleLogicalRange(newRange);
    }
  }
}

/**
 * Hook для React с автоматическим сохранением range
 */
export function useChartData(
  chartRef: React.RefObject<IChartApi | null>,
  seriesRef: React.RefObject<ISeriesApi<'Line'> | null>
) {
  const rangeRef = React.useRef<LogicalRange | null>(null);
  
  // Сохраняем range периодически
  React.useEffect(() => {
    const chart = chartRef.current;
    if (!chart) return;
    
    const unsubscribe = chart.timeScale()
      .subscribeVisibleLogicalRangeChange((range) => {
        rangeRef.current = range;
      });
    
    return unsubscribe;
  }, [chartRef]);
  
  const setData = React.useCallback((data: LineData[]) => {
    const chart = chartRef.current;
    const series = seriesRef.current;
    if (!chart || !series) return;
    
    series.setData(data);
    
    if (rangeRef.current) {
      requestAnimationFrame(() => {
        chart.timeScale().setVisibleLogicalRange(rangeRef.current!);
      });
    }
  }, [chartRef, seriesRef]);
  
  const prependData = React.useCallback((
    existingData: LineData[],
    historicalData: LineData[]
  ) => {
    const chart = chartRef.current;
    const series = seriesRef.current;
    if (!chart || !series) return;
    
    const allData = [...historicalData, ...existingData];
    const offset = historicalData.length;
    
    series.setData(allData);
    
    if (rangeRef.current) {
      requestAnimationFrame(() => {
        chart.timeScale().setVisibleLogicalRange({
          from: rangeRef.current!.from + offset,
          to: rangeRef.current!.to + offset,
        });
      });
    }
  }, [chartRef, seriesRef]);
  
  return { setData, prependData };
}

// ========================================
// Пример использования
// ========================================

function TradingChart() {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Line'> | null>(null);
  
  const { setData, prependData } = useChartData(chartRef, seriesRef);
  
  useEffect(() => {
    if (!containerRef.current) return;
    
    const chart = createChart(containerRef.current, {
      width: 800,
      height: 400,
      timeScale: {
        shiftVisibleRangeOnNewBar: false,
      },
    });
    
    const series = chart.addLineSeries();
    
    chartRef.current = chart;
    seriesRef.current = series;
    
    // Загружаем начальные данные
    series.setData(initialData);
    
    return () => chart.remove();
  }, []);
  
  // Подгрузка истории при скролле влево
  const handleLoadMore = async () => {
    const historicalData = await fetchHistoricalData();
    prependData(currentData, historicalData);
  };
  
  return <div ref={containerRef} />;
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Real-time | Infinite Scroll | Без jitter |
|---------|--------|-----------|-----------|-----------------|------------|
| Save/Restore Range | 8/10 | ⭐ Простая | ✅ | ⚠️ Частично | ⚠️ |
| Debounced Update | 7/10 | ⭐⭐ Средняя | ✅ | ✅ | ⚠️ |
| **Incremental Update** | **9/10** | **⭐⭐ Средняя** | **✅** | **✅** | **✅** |
| TimeScale Options | 6/10 | ⭐ Простая | ⚠️ | ❌ | ⚠️ |
| **StableChart Wrapper** | **9/10** | **⭐⭐⭐ Сложная** | **✅** | **✅** | **✅** |

---

## 🔗 Источники

1. [GitHub Issue #1875 - setData visible range](https://github.com/tradingview/lightweight-charts/issues/1875)
2. [GitHub Issue #549 - timeScale changes visibleRange on setData](https://github.com/tradingview/lightweight-charts/issues/549)
3. [GitHub Issue #1395 - TimeScale option to stop visible range changing](https://github.com/tradingview/lightweight-charts/issues/1395)
4. [Lightweight Charts - Time Scale API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ITimeScaleApi)
5. [Lightweight Charts - Logical Range](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/LogicalRange)
6. [Stack Overflow - Prevent setData from changing visible range](https://stackoverflow.com/questions/tagged/lightweight-charts)

---

## 📝 Рекомендации по выбору решения

| Сценарий | Рекомендуемое решение |
|----------|----------------------|
| Simple real-time chart | Решение 1 (Save/Restore) |
| High-frequency updates | Решение 3 (Incremental) |
| Infinite scroll | Решение 3 или 5 |
| Complex trading app | Решение 5 (StableChart) |
| React/Vue component | Решение 3 + custom hook |
