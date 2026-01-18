# БАГ #19: Утечки памяти event listeners в SPA

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** Множественные reports (общая проблема интеграции)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (требует discipline от developers)  
> **Последнее обновление:** январь 2026

---

## 📋 Описание проблемы

### Суть бага
При использовании lightweight-charts в React/Vue/Angular/Svelte Single Page Applications (SPA) происходит **накопление event listeners**, если разработчик не выполняет корректную очистку подписок при unmount компонента.

### Механизм утечки
1. ❌ Компонент подписывается на события: `subscribeClick`, `subscribeCrosshairMove`, etc.
2. ❌ При навигации компонент размонтируется **БЕЗ** вызова `unsubscribe`
3. ❌ Старые callback-функции остаются в памяти, удерживая ссылки на DOM и state
4. ❌ При повторном монтировании создаются новые подписки → leak прогрессирует
5. ❌ После нескольких переходов приложение становится "медленным"

### Типичные симптомы
- 📈 Постоянно растущее потребление RAM (видно в DevTools → Memory)
- 🐢 Постепенное замедление UI после навигации между страницами
- 🔁 Event handlers срабатывают несколько раз на одно действие
- 💥 Eventual crash из-за исчерпания памяти (в экстремальных случаях)

### Особенно актуально для
- **React 18+** с Strict Mode (двойной вызов useEffect в dev)
- **Next.js** с частой навигацией
- **Vue 3** с Composition API
- Любого SPA с frequent mount/unmount chart компонентов

### Частота
**100%** при неправильной очистке. Проблема проявляется постепенно.

---

## 🔍 Найденные решения

### Решение 1: Корректный useEffect cleanup (React)
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

Полная очистка всех ресурсов в cleanup function:

```typescript
import { useEffect, useRef } from 'react';
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  MouseEventHandler,
  TimeRangeChangeEventHandler
} from 'lightweight-charts';

function TradingChart({ data, symbol }: ChartProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Candlestick'> | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    // ============== СОЗДАНИЕ ==============
    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
      layout: { background: { color: '#1a1a1a' } }
    });
    chartRef.current = chart;

    const series = chart.addCandlestickSeries();
    seriesRef.current = series;
    series.setData(data);

    // ============== ПОДПИСКИ ==============
    // ⚠️ ВАЖНО: Сохраняем ссылки на handlers для unsubscribe
    
    const handleClick: MouseEventHandler = (param) => {
      if (param.point) {
        console.log('Clicked at:', param.point);
      }
    };

    const handleCrosshairMove: MouseEventHandler = (param) => {
      // Обновление tooltip и т.д.
    };

    const handleTimeRangeChange: TimeRangeChangeEventHandler = (range) => {
      // Lazy loading логика
    };

    const handleResize = () => {
      if (containerRef.current && chartRef.current) {
        chartRef.current.resize(
          containerRef.current.clientWidth,
          containerRef.current.clientHeight
        );
      }
    };

    // Подписываемся
    chart.subscribeClick(handleClick);
    chart.subscribeCrosshairMove(handleCrosshairMove);
    chart.timeScale().subscribeVisibleTimeRangeChange(handleTimeRangeChange);
    window.addEventListener('resize', handleResize);

    // ============== CLEANUP (КРИТИЧЕСКИ ВАЖНО!) ==============
    return () => {
      // 1. Отписываемся от всех событий chart
      chart.unsubscribeClick(handleClick);
      chart.unsubscribeCrosshairMove(handleCrosshairMove);
      chart.timeScale().unsubscribeVisibleTimeRangeChange(handleTimeRangeChange);
      
      // 2. Удаляем window listeners
      window.removeEventListener('resize', handleResize);
      
      // 3. Удаляем chart (освобождает все внутренние ресурсы)
      chart.remove();
      
      // 4. Очищаем refs
      chartRef.current = null;
      seriesRef.current = null;
    };
  }, []); // Пустой deps array - chart создаётся один раз

  // Отдельный effect для обновления данных
  useEffect(() => {
    if (seriesRef.current && data) {
      seriesRef.current.setData(data);
    }
  }, [data]);

  return <div ref={containerRef} style={{ width: '100%', height: '400px' }} />;
}
```

**Преимущества:**
- ✅ Полная очистка всех ресурсов
- ✅ Стандартный React паттерн
- ✅ Работает с Strict Mode

**Недостатки:**
- ⚠️ Verbose - много boilerplate кода
- ⚠️ Легко забыть одну подписку

---

### Решение 2: Subscription Registry Pattern
**Оценка: 9/10**

Централизованное управление подписками через registry:

```typescript
type Unsubscribe = () => void;

class SubscriptionRegistry {
  private subscriptions: Unsubscribe[] = [];

  /**
   * Добавить подписку в registry
   */
  add(unsubscribe: Unsubscribe): void {
    this.subscriptions.push(unsubscribe);
  }

  /**
   * Добавить подписку как пару subscribe/unsubscribe
   */
  track<T extends (...args: any[]) => void>(
    subscribe: (handler: T) => void,
    unsubscribe: (handler: T) => void,
    handler: T
  ): void {
    subscribe(handler);
    this.subscriptions.push(() => unsubscribe(handler));
  }

  /**
   * Отписаться от всех событий
   */
  unsubscribeAll(): void {
    this.subscriptions.forEach(unsub => {
      try {
        unsub();
      } catch (e) {
        console.warn('Failed to unsubscribe:', e);
      }
    });
    this.subscriptions = [];
  }

  /**
   * Количество активных подписок
   */
  get count(): number {
    return this.subscriptions.length;
  }
}

// Использование в компоненте
function ChartWithRegistry({ data }: Props) {
  const containerRef = useRef<HTMLDivElement>(null);
  const registryRef = useRef(new SubscriptionRegistry());

  useEffect(() => {
    if (!containerRef.current) return;

    const registry = registryRef.current;
    const chart = createChart(containerRef.current, { /* options */ });

    // Легко добавлять подписки
    registry.track(
      chart.subscribeClick.bind(chart),
      chart.unsubscribeClick.bind(chart),
      (param) => console.log('click', param)
    );

    registry.track(
      chart.subscribeCrosshairMove.bind(chart),
      chart.unsubscribeCrosshairMove.bind(chart),
      (param) => { /* crosshair logic */ }
    );

    // Window events
    const handleResize = () => chart.resize(
      containerRef.current!.clientWidth,
      containerRef.current!.clientHeight
    );
    window.addEventListener('resize', handleResize);
    registry.add(() => window.removeEventListener('resize', handleResize));

    return () => {
      registry.unsubscribeAll();
      chart.remove();
    };
  }, []);

  return <div ref={containerRef} />;
}
```

**Преимущества:**
- ✅ Централизованный cleanup
- ✅ Невозможно забыть unsubscribe
- ✅ Легко тестировать

**Недостатки:**
- ⚠️ Дополнительный класс для поддержки
- ⚠️ Небольшой overhead

---

### Решение 3: AbortController Pattern (Modern)
**Оценка: 8/10**

Современный подход с использованием AbortController:

```typescript
function ModernChart({ data }: Props) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    // AbortController для всех связанных операций
    const abortController = new AbortController();
    const { signal } = abortController;

    const chart = createChart(containerRef.current, { /* options */ });
    const series = chart.addCandlestickSeries();
    series.setData(data);

    // ============== Event Listeners с AbortSignal ==============
    
    // Window events - AbortController работает напрямую
    window.addEventListener('resize', () => {
      if (containerRef.current) {
        chart.resize(
          containerRef.current.clientWidth,
          containerRef.current.clientHeight
        );
      }
    }, { signal });

    window.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') {
        chart.timeScale().resetTimeScale();
      }
    }, { signal });

    // Chart subscriptions - нужна обёртка (AbortController native не поддерживается)
    const subscriptions: Array<() => void> = [];

    const trackChartEvent = <T extends (...args: any[]) => void>(
      subscribe: (h: T) => void,
      unsubscribe: (h: T) => void,
      handler: T
    ) => {
      // Проверяем, не отменён ли уже
      if (signal.aborted) return;
      
      subscribe(handler);
      
      // Автоматически отписываемся при abort
      signal.addEventListener('abort', () => unsubscribe(handler), { once: true });
    };

    trackChartEvent(
      chart.subscribeClick.bind(chart),
      chart.unsubscribeClick.bind(chart),
      (param) => console.log('click', param)
    );

    trackChartEvent(
      chart.subscribeCrosshairMove.bind(chart),
      chart.unsubscribeCrosshairMove.bind(chart),
      (param) => { /* ... */ }
    );

    // Cleanup через abort
    return () => {
      abortController.abort();
      chart.remove();
    };
  }, []);

  return <div ref={containerRef} />;
}
```

**Преимущества:**
- ✅ Современный browser API
- ✅ Автоматическая очистка window events
- ✅ Композируется с fetch и другими AbortController-aware APIs

**Недостатки:**
- ⚠️ lightweight-charts API не поддерживает AbortSignal нативно
- ⚠️ Требуется wrapper для chart subscriptions

---

### Решение 4: Custom Hook useLightweightChart
**Оценка: 10/10** ⭐⭐ ЛУЧШЕЕ РЕШЕНИЕ

Переиспользуемый hook, инкапсулирующий всю логику:

```typescript
import { useEffect, useRef, useCallback } from 'react';
import {
  createChart,
  IChartApi,
  ISeriesApi,
  SeriesType,
  ChartOptions,
  DeepPartial,
  MouseEventHandler,
  Time,
} from 'lightweight-charts';

interface UseLightweightChartOptions {
  container: React.RefObject<HTMLDivElement>;
  options?: DeepPartial<ChartOptions>;
  autoResize?: boolean;
}

interface ChartEventHandlers {
  onClick?: MouseEventHandler<Time>;
  onCrosshairMove?: MouseEventHandler<Time>;
  onVisibleTimeRangeChange?: (range: any) => void;
}

export function useLightweightChart({
  container,
  options = {},
  autoResize = true,
}: UseLightweightChartOptions) {
  const chartRef = useRef<IChartApi | null>(null);
  const seriesMapRef = useRef<Map<string, ISeriesApi<SeriesType>>>(new Map());
  const handlersRef = useRef<ChartEventHandlers>({});
  const cleanupRef = useRef<Array<() => void>>([]);

  // Регистрация cleanup функции
  const addCleanup = useCallback((fn: () => void) => {
    cleanupRef.current.push(fn);
  }, []);

  // Инициализация chart
  useEffect(() => {
    if (!container.current) return;

    const chart = createChart(container.current, {
      width: container.current.clientWidth,
      height: container.current.clientHeight,
      ...options,
    });
    chartRef.current = chart;

    // Auto-resize
    if (autoResize) {
      const resizeObserver = new ResizeObserver((entries) => {
        if (entries.length === 0 || !chartRef.current) return;
        const { width, height } = entries[0].contentRect;
        chartRef.current.resize(width, height);
      });
      resizeObserver.observe(container.current);
      addCleanup(() => resizeObserver.disconnect());
    }

    // Event handlers с автоматической очисткой
    const setupHandler = <T extends (...args: any[]) => void>(
      key: keyof ChartEventHandlers,
      subscribe: (handler: T) => void,
      unsubscribe: (handler: T) => void
    ) => {
      const wrapper = ((...args: any[]) => {
        const handler = handlersRef.current[key];
        if (handler) {
          (handler as Function)(...args);
        }
      }) as T;

      subscribe(wrapper);
      addCleanup(() => unsubscribe(wrapper));
    };

    setupHandler(
      'onClick',
      chart.subscribeClick.bind(chart),
      chart.unsubscribeClick.bind(chart)
    );

    setupHandler(
      'onCrosshairMove',
      chart.subscribeCrosshairMove.bind(chart),
      chart.unsubscribeCrosshairMove.bind(chart)
    );

    setupHandler(
      'onVisibleTimeRangeChange',
      chart.timeScale().subscribeVisibleTimeRangeChange.bind(chart.timeScale()),
      chart.timeScale().unsubscribeVisibleTimeRangeChange.bind(chart.timeScale())
    );

    // Master cleanup
    return () => {
      cleanupRef.current.forEach(fn => fn());
      cleanupRef.current = [];
      
      chart.remove();
      chartRef.current = null;
      seriesMapRef.current.clear();
    };
  }, [container, autoResize]); // options намеренно исключён

  // Public API
  const api = {
    get chart(): IChartApi | null {
      return chartRef.current;
    },

    addSeries<T extends SeriesType>(
      type: T,
      id: string,
      options?: any
    ): ISeriesApi<T> | null {
      if (!chartRef.current) return null;

      let series: ISeriesApi<T>;
      switch (type) {
        case 'Candlestick':
          series = chartRef.current.addCandlestickSeries(options) as ISeriesApi<T>;
          break;
        case 'Line':
          series = chartRef.current.addLineSeries(options) as ISeriesApi<T>;
          break;
        case 'Area':
          series = chartRef.current.addAreaSeries(options) as ISeriesApi<T>;
          break;
        case 'Bar':
          series = chartRef.current.addBarSeries(options) as ISeriesApi<T>;
          break;
        case 'Histogram':
          series = chartRef.current.addHistogramSeries(options) as ISeriesApi<T>;
          break;
        default:
          throw new Error(`Unknown series type: ${type}`);
      }

      seriesMapRef.current.set(id, series);
      return series;
    },

    getSeries(id: string): ISeriesApi<SeriesType> | undefined {
      return seriesMapRef.current.get(id);
    },

    removeSeries(id: string): boolean {
      const series = seriesMapRef.current.get(id);
      if (series && chartRef.current) {
        chartRef.current.removeSeries(series);
        seriesMapRef.current.delete(id);
        return true;
      }
      return false;
    },

    setHandlers(handlers: ChartEventHandlers) {
      handlersRef.current = { ...handlersRef.current, ...handlers };
    },
  };

  return api;
}

// ============== Пример использования ==============

function MyTradingChart({ data, symbol }: Props) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  const { chart, addSeries, getSeries, setHandlers } = useLightweightChart({
    container: containerRef,
    options: {
      layout: {
        background: { color: '#1a1a1a' },
        textColor: '#d1d1d1',
      },
      grid: {
        vertLines: { color: '#2a2a2a' },
        horzLines: { color: '#2a2a2a' },
      },
    },
  });

  // Добавляем серию при первом рендере
  useEffect(() => {
    if (chart) {
      const mainSeries = addSeries('Candlestick', 'main', {
        upColor: '#26a69a',
        downColor: '#ef5350',
      });
      
      if (mainSeries && data) {
        mainSeries.setData(data);
      }
    }
  }, [chart, addSeries]);

  // Обновляем данные
  useEffect(() => {
    const series = getSeries('main');
    if (series && data) {
      series.setData(data);
    }
  }, [data, getSeries]);

  // Устанавливаем обработчики (можно менять динамически)
  useEffect(() => {
    setHandlers({
      onClick: (param) => {
        console.log(`${symbol} clicked at:`, param.point);
      },
      onCrosshairMove: (param) => {
        // Обновить tooltip
      },
    });
  }, [symbol, setHandlers]);

  return <div ref={containerRef} style={{ width: '100%', height: 400 }} />;
}
```

**Преимущества:**
- ✅ Полная инкапсуляция lifecycle
- ✅ Невозможно допустить утечку
- ✅ Переиспользуемый код
- ✅ Type-safe API
- ✅ ResizeObserver вместо window.resize

**Недостатки:**
- ⚠️ Начальные усилия на создание hook

---

### Решение 5: Vue 3 Composition API
**Оценка: 9/10**

Для Vue 3 с аналогичными паттернами:

```typescript
// composables/useLightweightChart.ts
import { ref, onMounted, onUnmounted, watch, Ref } from 'vue';
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

export function useLightweightChart(
  containerRef: Ref<HTMLElement | null>,
  options: any = {}
) {
  const chart = ref<IChartApi | null>(null);
  const series = ref<Map<string, ISeriesApi<any>>>(new Map());
  const cleanupFns: Array<() => void> = [];

  const addCleanup = (fn: () => void) => cleanupFns.push(fn);

  onMounted(() => {
    if (!containerRef.value) return;

    const chartInstance = createChart(containerRef.value, {
      width: containerRef.value.clientWidth,
      height: containerRef.value.clientHeight,
      ...options,
    });
    chart.value = chartInstance;

    // ResizeObserver
    const resizeObserver = new ResizeObserver((entries) => {
      if (entries.length && chart.value) {
        const { width, height } = entries[0].contentRect;
        chart.value.resize(width, height);
      }
    });
    resizeObserver.observe(containerRef.value);
    addCleanup(() => resizeObserver.disconnect());
  });

  onUnmounted(() => {
    // Выполняем все cleanup функции
    cleanupFns.forEach(fn => fn());
    cleanupFns.length = 0;

    // Удаляем chart
    if (chart.value) {
      chart.value.remove();
      chart.value = null;
    }
    series.value.clear();
  });

  const subscribeClick = (handler: any) => {
    if (!chart.value) return;
    chart.value.subscribeClick(handler);
    addCleanup(() => chart.value?.unsubscribeClick(handler));
  };

  const subscribeCrosshairMove = (handler: any) => {
    if (!chart.value) return;
    chart.value.subscribeCrosshairMove(handler);
    addCleanup(() => chart.value?.unsubscribeCrosshairMove(handler));
  };

  return {
    chart,
    series,
    subscribeClick,
    subscribeCrosshairMove,
    addCleanup,
  };
}

// Использование в компоненте:
/*
<script setup>
import { ref } from 'vue';
import { useLightweightChart } from '@/composables/useLightweightChart';

const containerRef = ref(null);
const { chart, subscribeClick } = useLightweightChart(containerRef);

subscribeClick((param) => {
  console.log('Clicked:', param);
});
</script>

<template>
  <div ref="containerRef" class="chart-container" />
</template>
*/
```

---

## ✅ Рекомендуемое решение

### Для React проектов: Custom Hook (Решение 4)

**Почему это лучший выбор:**
1. **Инкапсуляция** — вся логика cleanup скрыта внутри hook
2. **Невозможность ошибки** — разработчик не может забыть cleanup
3. **Переиспользуемость** — один hook на весь проект
4. **Strict Mode ready** — корректно работает с React 18+
5. **Type safety** — полная поддержка TypeScript

### Checklist для любого решения

```markdown
## Обязательная очистка при unmount:

- [ ] `chart.unsubscribeClick(handler)`
- [ ] `chart.unsubscribeCrosshairMove(handler)`  
- [ ] `timeScale.unsubscribeVisibleTimeRangeChange(handler)`
- [ ] `timeScale.unsubscribeVisibleLogicalRangeChange(handler)`
- [ ] `priceScale.unsubscribeSizeChange(handler)`
- [ ] `window.removeEventListener('resize', handler)`
- [ ] `resizeObserver.disconnect()`
- [ ] `chart.remove()` — ПОСЛЕДНИМ! (удаляет DOM и внутренние ресурсы)
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Надёжность | Переиспользуемость |
|---------|--------|-----------|------------|---------------------|
| **Manual useEffect** | 9/10 | 🟢 Низкая | ⚠️ Зависит от dev | ❌ Нет |
| **SubscriptionRegistry** | 9/10 | 🟡 Средняя | ✅ Высокая | 🟡 Частичная |
| **AbortController** | 8/10 | 🟡 Средняя | ✅ Высокая | ❌ Ограничено |
| **Custom Hook** | 10/10 | 🟡 Средняя | ✅ Очень высокая | ✅ Полная |
| **Vue Composable** | 9/10 | 🟡 Средняя | ✅ Высокая | ✅ Полная |

---

## 🔬 Диагностика утечек

### Chrome DevTools

```javascript
// 1. Performance Monitor
// DevTools → More Tools → Performance Monitor
// Следить за "JS Heap size" и "DOM Nodes"

// 2. Memory Snapshots
// DevTools → Memory → Take Heap Snapshot
// Сравнить снимки до и после навигации

// 3. Event Listeners панель
// DevTools → Elements → Event Listeners
// Проверить, что listeners удаляются при unmount
```

### Программная проверка

```typescript
// Утилита для подсчёта event listeners
function countEventListeners(element: Element): number {
  const listeners = (window as any).getEventListeners?.(element);
  if (!listeners) {
    console.warn('getEventListeners not available (Chrome only)');
    return -1;
  }
  return Object.values(listeners).flat().length;
}

// Проверка перед unmount
console.log('Listeners before cleanup:', countEventListeners(chartContainer));
```

---

## 🔗 Источники

1. **React useEffect Cleanup**  
   https://react.dev/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed

2. **Lightweight Charts - Disposing Charts**  
   https://tradingview.github.io/lightweight-charts/docs/api#chartapiremove

3. **AbortController for event listeners**  
   https://developer.mozilla.org/en-US/docs/Web/API/AbortController

4. **Memory Leaks in JavaScript**  
   https://developer.chrome.com/docs/devtools/memory-problems/

5. **Vue 3 Lifecycle Hooks**  
   https://vuejs.org/guide/essentials/lifecycle.html

6. **React 18 Strict Mode**  
   https://react.dev/reference/react/StrictMode

---

## 📝 Дополнительные заметки

### Типичные ошибки

1. **Anonymous functions** — невозможно сделать unsubscribe:
```javascript
// ❌ НЕПРАВИЛЬНО
chart.subscribeClick(() => console.log('click'));
// Как сделать unsubscribe? Никак!

// ✅ ПРАВИЛЬНО
const handleClick = () => console.log('click');
chart.subscribeClick(handleClick);
// В cleanup: chart.unsubscribeClick(handleClick);
```

2. **Забытый chart.remove()**:
```javascript
// ❌ НЕПРАВИЛЬНО - chart DOM остаётся
return () => {
  chart.unsubscribeClick(handler);
};

// ✅ ПРАВИЛЬНО
return () => {
  chart.unsubscribeClick(handler);
  chart.remove(); // Удаляет DOM и все внутренние ресурсы
};
```

3. **useEffect с [data] dependency**:
```javascript
// ❌ НЕПРАВИЛЬНО - chart пересоздаётся при каждом изменении data
useEffect(() => {
  const chart = createChart(...);
  // ...
  return () => chart.remove();
}, [data]); // data меняется часто!

// ✅ ПРАВИЛЬНО - отдельные effects
useEffect(() => {
  const chart = createChart(...);
  return () => chart.remove();
}, []); // Создаём один раз

useEffect(() => {
  series.setData(data);
}, [data]); // Обновляем данные отдельно
```

### Когда ожидать улучшения в библиотеке?

Это не баг библиотеки, а **ответственность разработчика** за правильное управление lifecycle. Улучшения могут включать:
- Официальные React/Vue wrappers (маловероятно — community responsibility)
- Автоматическая очистка при удалении DOM (технически сложно)
