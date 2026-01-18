# БАГ #24: График сжимается при смещении назад

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1428](https://github.com/tradingview/lightweight-charts/issues/1428)  
> **Версии:** v4.0+  
> **Платформы:** Все браузеры  
> **Статус:** 🟢 Закрыт (исправлен в v4.1.1), но связанные проблемы остаются

---

## 📋 Описание проблемы

### Суть проблемы
При добавлении и обновлении нескольких линий (серий) на графике, он может неожиданно "сжиматься" или перемещаться в противоположном направлении. Также при прокрутке назад (влево) может изменяться масштаб (`barSpacing`).

### Типичные сценарии

1. **Добавление нескольких серий одновременно:**
   ```typescript
   // Добавляем несколько линий
   const series1 = chart.addLineSeries();
   const series2 = chart.addLineSeries();
   const series3 = chart.addLineSeries();
   
   series1.setData(data1);
   series2.setData(data2);
   series3.setData(data3);
   // ❌ График может "прыгнуть" или сжаться
   ```

2. **Прокрутка к историческим данным:**
   ```typescript
   // Пользователь прокручивает график влево
   // ❌ barSpacing неожиданно изменяется
   // ❌ График "сжимается", показывая больше данных чем ожидалось
   ```

3. **Использование fixLeftEdge/fixRightEdge:**
   ```typescript
   const chart = createChart(container, {
     timeScale: {
       fixLeftEdge: true,
       fixRightEdge: true,
     },
   });
   // ❌ Некорректный начальный barSpacing
   // ❌ rightOffset игнорируется
   ```

### Связанные проблемы

| Проблема | Описание | Статус |
|----------|----------|--------|
| #1428 | Chart moves in opposite direction | ✅ Исправлено |
| Initial barSpacing с fix edges | Некорректный начальный зум | ✅ v3.6 |
| rightOffset ignored | rightOffset не работает с fixRightEdge | ⚠️ Частично |
| lockVisibleTimeRangeOnResize | Не работает с fixLeftEdge | ✅ v4.0 |

---

## 🔍 Найденные решения

### Решение 1: Обновление до последней версии

**Оценка: 9/10**

Многие проблемы со сжатием были исправлены в версиях 4.0 и 4.1.1:

```bash
npm update lightweight-charts
# или
npm install lightweight-charts@latest
```

**Что было исправлено:**
- v4.0: Глitches при сбросе time scale во время прокрутки
- v4.1.1: shiftVisibleRangeOnNewBar для real-time обновлений
- v4.1.1: Новая опция `allowShiftVisibleRangeOnWhitespaceReplacement`

**Плюсы:**
- Официальные fixes
- Минимум усилий

**Минусы:**
- Может потребовать миграции кода
- Не решает все edge cases

---

### Решение 2: Контроль barSpacing при прокрутке

**Оценка: 8/10**

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

/**
 * Wrapper для контроля barSpacing во время прокрутки
 */
class StableBarSpacingChart {
  private chart: IChartApi;
  private targetBarSpacing: number;
  private isScrolling: boolean = false;

  constructor(container: HTMLElement, initialBarSpacing: number = 6) {
    this.targetBarSpacing = initialBarSpacing;
    
    this.chart = createChart(container, {
      timeScale: {
        barSpacing: initialBarSpacing,
        minBarSpacing: 1,
        maxBarSpacing: 50,
      },
    });

    this.setupBarSpacingControl();
  }

  private setupBarSpacingControl(): void {
    const timeScale = this.chart.timeScale();
    
    // Отслеживаем изменения barSpacing
    let lastBarSpacing = this.targetBarSpacing;
    
    // Подписываемся на изменение visible range
    timeScale.subscribeVisibleLogicalRangeChange(() => {
      if (this.isScrolling) {
        // Во время прокрутки принудительно сохраняем barSpacing
        const currentOptions = timeScale.options();
        if (Math.abs(currentOptions.barSpacing - this.targetBarSpacing) > 0.1) {
          timeScale.applyOptions({ barSpacing: this.targetBarSpacing });
        }
      }
    });

    // Отслеживаем события прокрутки
    const chartElement = this.chart.chartElement();
    
    chartElement.addEventListener('wheel', (e) => {
      if (!e.ctrlKey && !e.metaKey) {
        // Горизонтальная прокрутка (без zoom)
        this.isScrolling = true;
        setTimeout(() => { this.isScrolling = false; }, 100);
      } else {
        // Zoom - обновляем target barSpacing
        setTimeout(() => {
          this.targetBarSpacing = timeScale.options().barSpacing;
        }, 50);
      }
    });
  }

  /**
   * Устанавливает желаемый barSpacing
   */
  setBarSpacing(spacing: number): void {
    this.targetBarSpacing = spacing;
    this.chart.timeScale().applyOptions({ barSpacing: spacing });
  }

  /**
   * Получить текущий barSpacing
   */
  getBarSpacing(): number {
    return this.chart.timeScale().options().barSpacing;
  }

  getChart(): IChartApi {
    return this.chart;
  }
}

// Использование
const stableChart = new StableBarSpacingChart(container, 8);
const series = stableChart.getChart().addLineSeries();
series.setData(data);
```

**Плюсы:**
- Стабильный barSpacing при прокрутке
- Zoom работает как ожидается
- Полный контроль

**Минусы:**
- Дополнительная логика
- Может конфликтовать с некоторыми операциями

---

### Решение 3: Правильная конфигурация fixEdge опций

**Оценка: 7/10**

```typescript
import { createChart } from 'lightweight-charts';

// ✅ Правильная конфигурация с fix edges
const chart = createChart(container, {
  width: 800,
  height: 400,
  timeScale: {
    // НЕ используйте оба fix edge одновременно если возможно
    fixLeftEdge: false,  // Позволяем прокрутку влево
    fixRightEdge: true,  // Ограничиваем справа
    
    // Явно указываем barSpacing
    barSpacing: 8,
    minBarSpacing: 0.5,
    maxBarSpacing: 50,
    
    // Контроль отступов
    rightOffset: 10,
    
    // Блокировка при resize
    lockVisibleTimeRangeOnResize: true,
  },
});

// После инициализации - принудительный перерендер
// для решения проблемы пустого графика
requestAnimationFrame(() => {
  chart.timeScale().applyOptions({});
});
```

**Важные рекомендации:**

```typescript
// ❌ НЕ рекомендуется: оба fix edge сразу
const badConfig = {
  timeScale: {
    fixLeftEdge: true,
    fixRightEdge: true,
  },
};

// ✅ Рекомендуется: один fix edge
const goodConfig = {
  timeScale: {
    fixLeftEdge: false,
    fixRightEdge: true,  // Обычно это нужнее
    rightOffset: 5,
  },
};

// ✅ Или: динамическое управление
function setFixedBounds(chart: IChartApi, data: any[]) {
  if (data.length > 0) {
    const firstTime = data[0].time;
    const lastTime = data[data.length - 1].time;
    
    chart.timeScale().setVisibleRange({
      from: firstTime,
      to: lastTime,
    });
  }
}
```

---

### Решение 4: Отложенное добавление серий

**Оценка: 8/10**

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  SeriesType
} from 'lightweight-charts';

/**
 * Менеджер для добавления серий без "прыжков"
 */
class SeriesManager {
  private chart: IChartApi;
  private pendingSeries: Array<{
    type: SeriesType;
    options: any;
    data: any[];
  }> = [];
  private isInitialized: boolean = false;

  constructor(chart: IChartApi) {
    this.chart = chart;
  }

  /**
   * Добавляет серию в очередь
   */
  queueSeries<T extends SeriesType>(
    type: T,
    options: any,
    data: any[]
  ): void {
    this.pendingSeries.push({ type, options, data });
  }

  /**
   * Применяет все добавленные серии одновременно
   */
  applyAll(): ISeriesApi<SeriesType>[] {
    // Сохраняем текущий range
    const range = this.chart.timeScale().getVisibleLogicalRange();
    const barSpacing = this.chart.timeScale().options().barSpacing;

    const series: ISeriesApi<SeriesType>[] = [];

    // Добавляем все серии
    for (const pending of this.pendingSeries) {
      const s = this.chart.addSeries(pending.type as any, pending.options);
      s.setData(pending.data);
      series.push(s);
    }

    this.pendingSeries = [];

    // Восстанавливаем состояние
    requestAnimationFrame(() => {
      this.chart.timeScale().applyOptions({ barSpacing });
      if (range) {
        this.chart.timeScale().setVisibleLogicalRange(range);
      }
    });

    this.isInitialized = true;
    return series;
  }

  /**
   * Добавляет одну серию (после инициализации)
   */
  addSeries<T extends SeriesType>(
    type: T,
    options: any,
    data: any[]
  ): ISeriesApi<T> {
    const range = this.chart.timeScale().getVisibleLogicalRange();
    const barSpacing = this.chart.timeScale().options().barSpacing;

    const series = this.chart.addSeries(type as any, options) as ISeriesApi<T>;
    series.setData(data);

    // Восстанавливаем состояние
    requestAnimationFrame(() => {
      this.chart.timeScale().applyOptions({ barSpacing });
      if (range) {
        this.chart.timeScale().setVisibleLogicalRange(range);
      }
    });

    return series;
  }
}

// Использование
const chart = createChart(container);
const manager = new SeriesManager(chart);

// Добавляем все серии в очередь
manager.queueSeries('Line', { color: '#2962FF' }, data1);
manager.queueSeries('Line', { color: '#E91E63' }, data2);
manager.queueSeries('Histogram', { color: '#26a69a' }, volumeData);

// Применяем все сразу
const [line1, line2, histogram] = manager.applyAll();

// После инициализации можно добавлять по одной
const newLine = manager.addSeries('Line', { color: '#FF9800' }, newData);
```

---

### Решение 5: Batch обновление с синхронизацией

**Оценка: 9/10**

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  LogicalRange
} from 'lightweight-charts';

interface ChartState {
  barSpacing: number;
  logicalRange: LogicalRange | null;
}

/**
 * Выполняет batch операции с гарантией стабильности
 */
class ChartBatchUpdater {
  private chart: IChartApi;
  private isBatching: boolean = false;
  private savedState: ChartState | null = null;

  constructor(chart: IChartApi) {
    this.chart = chart;
  }

  /**
   * Начинает batch операцию
   */
  beginBatch(): void {
    if (this.isBatching) {
      console.warn('Batch already in progress');
      return;
    }

    this.isBatching = true;
    this.savedState = this.captureState();
  }

  /**
   * Завершает batch операцию и восстанавливает состояние
   */
  endBatch(options: {
    restoreBarSpacing?: boolean;
    restoreRange?: boolean;
    animate?: boolean;
  } = {}): void {
    const { 
      restoreBarSpacing = true, 
      restoreRange = true,
      animate = false 
    } = options;

    if (!this.isBatching || !this.savedState) {
      console.warn('No batch in progress');
      return;
    }

    const state = this.savedState;
    
    const restore = () => {
      if (restoreBarSpacing) {
        this.chart.timeScale().applyOptions({ 
          barSpacing: state.barSpacing 
        });
      }
      
      if (restoreRange && state.logicalRange) {
        this.chart.timeScale().setVisibleLogicalRange(state.logicalRange);
      }
    };

    if (animate) {
      requestAnimationFrame(restore);
    } else {
      restore();
    }

    this.isBatching = false;
    this.savedState = null;
  }

  /**
   * Выполняет функцию в batch режиме
   */
  batch<T>(
    fn: () => T,
    options: {
      restoreBarSpacing?: boolean;
      restoreRange?: boolean;
    } = {}
  ): T {
    this.beginBatch();
    try {
      return fn();
    } finally {
      this.endBatch(options);
    }
  }

  private captureState(): ChartState {
    return {
      barSpacing: this.chart.timeScale().options().barSpacing,
      logicalRange: this.chart.timeScale().getVisibleLogicalRange(),
    };
  }
}

// Использование
const chart = createChart(container);
const batchUpdater = new ChartBatchUpdater(chart);

// Batch добавление серий
batchUpdater.batch(() => {
  const series1 = chart.addLineSeries({ color: '#2962FF' });
  const series2 = chart.addLineSeries({ color: '#E91E63' });
  
  series1.setData(data1);
  series2.setData(data2);
  
  return [series1, series2];
}, { restoreBarSpacing: true, restoreRange: true });

// Или с ручным контролем
batchUpdater.beginBatch();

const line = chart.addLineSeries();
line.setData(data);

const histogram = chart.addHistogramSeries();
histogram.setData(volumeData);

batchUpdater.endBatch({ restoreBarSpacing: true });
```

---

## ✅ Рекомендуемое решение

### Комбинированный подход

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

/**
 * Стабильный Chart wrapper
 */
export function createStableChart(
  container: HTMLElement,
  options: Parameters<typeof createChart>[1] = {}
) {
  // 1. Правильная начальная конфигурация
  const chart = createChart(container, {
    ...options,
    timeScale: {
      barSpacing: 8,
      minBarSpacing: 1,
      maxBarSpacing: 50,
      rightOffset: 10,
      fixRightEdge: true,
      fixLeftEdge: false,
      lockVisibleTimeRangeOnResize: true,
      ...options.timeScale,
    },
  });

  // 2. Fix для пустого графика при инициализации
  requestAnimationFrame(() => {
    chart.timeScale().applyOptions({});
  });

  // 3. Wrapper для стабильных операций
  return {
    chart,
    
    /**
     * Добавляет серии без "прыжков"
     */
    addSeriesSafe<T extends 'Line' | 'Candlestick' | 'Histogram'>(
      type: T,
      seriesOptions: any,
      data: any[]
    ): ISeriesApi<T> {
      const state = {
        barSpacing: chart.timeScale().options().barSpacing,
        range: chart.timeScale().getVisibleLogicalRange(),
      };

      const series = chart.addSeries(type as any, seriesOptions) as ISeriesApi<T>;
      series.setData(data);

      requestAnimationFrame(() => {
        chart.timeScale().applyOptions({ barSpacing: state.barSpacing });
        if (state.range) {
          chart.timeScale().setVisibleLogicalRange(state.range);
        }
      });

      return series;
    },

    /**
     * Удаляет график
     */
    remove(): void {
      chart.remove();
    },
  };
}

// Пример использования
const stableChart = createStableChart(container, {
  width: 800,
  height: 400,
});

// Добавление серий без сжатия
const candlesticks = stableChart.addSeriesSafe('Candlestick', {
  upColor: '#26a69a',
  downColor: '#ef5350',
}, candlestickData);

const volume = stableChart.addSeriesSafe('Histogram', {
  color: '#385263',
  priceScaleId: 'volume',
}, volumeData);
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Multi-series | Scroll stable | Fix edges |
|---------|--------|-----------|--------------|---------------|-----------|
| Update версии | 9/10 | ⭐ | ✅ | ⚠️ | ⚠️ |
| **BarSpacing control** | **8/10** | **⭐⭐** | ⚠️ | **✅** | ⚠️ |
| Fix edge config | 7/10 | ⭐ | ⚠️ | ⚠️ | ✅ |
| **Delayed series** | **8/10** | **⭐⭐** | **✅** | ⚠️ | ⚠️ |
| **Batch updater** | **9/10** | **⭐⭐⭐** | **✅** | **✅** | **✅** |

---

## 🔗 Источники

1. [GitHub Issue #1428 - Chart moves in opposite direction](https://github.com/tradingview/lightweight-charts/issues/1428)
2. [Lightweight Charts v4.1.1 Release Notes](https://tradingview.github.io/lightweight-charts/docs/release-notes)
3. [TimeScale Options Documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions)
4. [Lightweight Charts v4.0 Release Notes](https://tradingview.github.io/lightweight-charts/docs/release-notes)
5. [Stack Overflow - barSpacing issues](https://stackoverflow.com/questions/tagged/lightweight-charts)

---

## 📝 Changelog проблемы

| Дата | Версия LC | Изменение |
|------|-----------|-----------|
| 2021 | v3.6 | Исправлен barSpacing с fix edges |
| 2023 | v4.0 | Исправлены глitches при scroll reset |
| 2023 | v4.1.1 | Исправлен shiftVisibleRangeOnNewBar |
| 2023 | v4.1.1 | Добавлен allowShiftVisibleRangeOnWhitespaceReplacement |
