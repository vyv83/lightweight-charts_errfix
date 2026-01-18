# БАГ #6: React Strict Mode двойная инициализация (v18+)

> **Критичность:** 🔴 КРИТИЧЕСКАЯ (для React 18+ developers)  
> **GitHub Issue:** Часть общих React integration проблем  
> **Версии:** Все версии с React 18+  
> **Статус:** 🔴 Open (архитектурная проблема интеграции)

---

## 📋 Описание проблемы

### Симптомы
- Создаются **два экземпляра** графика одновременно
- Утечки памяти
- Визуальные баги (наложение графиков)
- `chart.remove()` в cleanup удаляет нужный график

### Причина
React 18 Strict Mode **намеренно** вызывает `useEffect` дважды в dev mode:
1. Mount → Effect runs
2. Cleanup runs
3. Re-mount → Effect runs снова

Это сделано для выявления багов в cleanup логике.

### Важно
- Происходит **ТОЛЬКО в development mode**
- Production build НЕ затронут
- Это **не баг React**, а фича для обнаружения проблем

---

## 🔍 Найденные решения

### Решение 1: Правильная структура useEffect с cleanup
**Оценка: 10/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Корректный cleanup, который правильно работает при double invocation.

```jsx
import { createChart } from 'lightweight-charts';
import { useEffect, useRef } from 'react';

function ChartComponent({ data }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    // Guard: проверяем что контейнер существует
    if (!containerRef.current) return;

    // Создаём график
    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
    });

    const series = chart.addCandlestickSeries();
    series.setData(data);

    // Сохраняем ссылку
    chartRef.current = chart;

    // КРИТИЧЕСКИ ВАЖНО: cleanup function
    return () => {
      chart.remove();
      chartRef.current = null;
    };
  }, []); // Пустой массив - только mount/unmount

  // Отдельный effect для обновления данных
  useEffect(() => {
    if (chartRef.current && data) {
      const series = chartRef.current.getSeries()[0];
      if (series) {
        series.setData(data);
      }
    }
  }, [data]);

  return <div ref={containerRef} style={{ width: '100%', height: '400px' }} />;
}
```

---

### Решение 2: useRef для предотвращения двойной инициализации
**Оценка: 9/10**

**Суть:** Использовать ref как флаг инициализации.

```jsx
import { createChart } from 'lightweight-charts';
import { useEffect, useRef } from 'react';

function ChartComponent({ data }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);
  const isInitializedRef = useRef(false);

  useEffect(() => {
    // Пропускаем если уже инициализировано
    if (isInitializedRef.current) return;
    if (!containerRef.current) return;

    isInitializedRef.current = true;

    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
    });

    chartRef.current = chart;
    chart.addCandlestickSeries().setData(data);

    return () => {
      if (chartRef.current) {
        chartRef.current.remove();
        chartRef.current = null;
      }
      isInitializedRef.current = false;
    };
  }, []);

  return <div ref={containerRef} />;
}
```

---

### Решение 3: Custom hook для chart management
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ для повторного использования

**Суть:** Создать переиспользуемый hook.

```jsx
// hooks/useLightweightChart.js
import { createChart } from 'lightweight-charts';
import { useEffect, useRef, useCallback } from 'react';

export function useLightweightChart(options = {}) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);
  const seriesRef = useRef(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: options.height || 400,
      ...options,
    });

    chartRef.current = chart;

    // Auto-resize
    const resizeObserver = new ResizeObserver((entries) => {
      for (const entry of entries) {
        if (chartRef.current) {
          chartRef.current.applyOptions({
            width: entry.contentRect.width,
          });
        }
      }
    });
    resizeObserver.observe(containerRef.current);

    return () => {
      resizeObserver.disconnect();
      if (chartRef.current) {
        chartRef.current.remove();
        chartRef.current = null;
        seriesRef.current = null;
      }
    };
  }, [options.height]);

  const addCandlestickSeries = useCallback((seriesOptions = {}) => {
    if (!chartRef.current) return null;
    const series = chartRef.current.addCandlestickSeries(seriesOptions);
    seriesRef.current = series;
    return series;
  }, []);

  const setData = useCallback((data) => {
    if (seriesRef.current) {
      seriesRef.current.setData(data);
    }
  }, []);

  return {
    containerRef,
    chartRef,
    seriesRef,
    addCandlestickSeries,
    setData,
  };
}

// Использование
function MyChart({ data }) {
  const { containerRef, addCandlestickSeries, setData } = useLightweightChart({
    height: 400,
  });

  useEffect(() => {
    addCandlestickSeries();
  }, [addCandlestickSeries]);

  useEffect(() => {
    setData(data);
  }, [data, setData]);

  return <div ref={containerRef} style={{ width: '100%' }} />;
}
```

---

### Решение 4: Disable Strict Mode (НЕ РЕКОМЕНДУЕТСЯ)
**Оценка: 3/10**

**Суть:** Убрать `<React.StrictMode>`.

```jsx
// index.js - НЕ РЕКОМЕНДУЕТСЯ
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  // <React.StrictMode> - закомментировано
    <App />
  // </React.StrictMode>
);
```

**Почему не рекомендуется:**
- Скрывает потенциальные баги
- Нарушает best practices
- Проблемы проявятся в будущих версиях React

---

## ✅ Полный пример компонента

```jsx
// components/TradingChart.jsx
import { createChart } from 'lightweight-charts';
import { useEffect, useRef, memo } from 'react';

const TradingChart = memo(function TradingChart({ 
  data, 
  width = '100%', 
  height = 400,
  options = {} 
}) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);
  const seriesRef = useRef(null);

  // Инициализация графика
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    // Создаём график
    const chart = createChart(container, {
      width: container.clientWidth,
      height: height,
      layout: {
        background: { type: 'solid', color: '#1a1a2e' },
        textColor: '#d1d4dc',
      },
      grid: {
        vertLines: { color: '#2a2a4e' },
        horzLines: { color: '#2a2a4e' },
      },
      ...options,
    });

    chartRef.current = chart;
    seriesRef.current = chart.addCandlestickSeries({
      upColor: '#26a69a',
      downColor: '#ef5350',
      borderVisible: false,
      wickUpColor: '#26a69a',
      wickDownColor: '#ef5350',
    });

    // Auto-resize
    const handleResize = () => {
      if (chartRef.current && container) {
        chartRef.current.applyOptions({ width: container.clientWidth });
      }
    };
    const resizeObserver = new ResizeObserver(handleResize);
    resizeObserver.observe(container);

    // Cleanup - КРИТИЧЕСКИ ВАЖНО для React 18
    return () => {
      resizeObserver.disconnect();
      if (chartRef.current) {
        chartRef.current.remove();
        chartRef.current = null;
        seriesRef.current = null;
      }
    };
  }, [height, options]);

  // Обновление данных
  useEffect(() => {
    if (seriesRef.current && data?.length > 0) {
      seriesRef.current.setData(data);
      chartRef.current?.timeScale().fitContent();
    }
  }, [data]);

  return (
    <div 
      ref={containerRef} 
      style={{ 
        width, 
        height,
        minHeight: '100px'
      }} 
    />
  );
});

export default TradingChart;
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Production-ready |
|---------|--------|-----------|------------------|
| #1 Proper cleanup | 10/10 | Низкая | ✅ Да |
| #2 isInitialized ref | 9/10 | Низкая | ✅ Да |
| #3 Custom hook | 9/10 | Средняя | ✅ Да |
| #4 Disable StrictMode | 3/10 | Низкая | ❌ Нет |

---

## 🔗 Источники

1. [React 18 Strict Mode](https://react.dev/reference/react/StrictMode)
2. [useEffect cleanup](https://react.dev/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development)
3. [Lightweight Charts React Tutorial](https://tradingview.github.io/lightweight-charts/tutorials/react/simple)

---

**Документ создан:** 18 января 2026
