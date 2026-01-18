# БАГ #7: SSR crash в Next.js 13+ (App Router)

> **Критичность:** 🔴 КРИТИЧЕСКАЯ (для Next.js users)  
> **GitHub Issue:** Множество related issues  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (архитектурная проблема - client-only библиотека)

---

## 📋 Описание проблемы

### Симптомы
- Ошибки `"window is undefined"`, `"document is undefined"` при SSR
- Crash при build или runtime
- Hydration mismatch warnings

### Причина
- Lightweight Charts зависит от браузерных API (Canvas, window, document)
- Next.js 13+ App Router по умолчанию использует Server Components
- Даже `'use client'` компоненты pre-render на сервере

### Сценарии
1. Импорт в Server Component без `'use client'`
2. Попытка инициализации на сервере

---

## 🔍 Найденные решения

### Решение 1: Dynamic import с ssr: false
**Оценка: 10/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Использовать `next/dynamic` для client-only loading.

```jsx
// components/Chart.jsx - client компонент
'use client';

import { createChart } from 'lightweight-charts';
import { useEffect, useRef } from 'react';

export default function Chart({ data }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
    });

    chart.addCandlestickSeries().setData(data);
    chartRef.current = chart;

    return () => {
      chart.remove();
    };
  }, [data]);

  return <div ref={containerRef} style={{ width: '100%', height: '400px' }} />;
}
```

```jsx
// app/page.jsx - Server Component
import dynamic from 'next/dynamic';

// Dynamic import с отключением SSR
const Chart = dynamic(() => import('@/components/Chart'), {
  ssr: false,
  loading: () => <div>Loading chart...</div>,
});

export default function Page() {
  const data = [
    { time: '2024-01-01', open: 100, high: 110, low: 95, close: 105 },
    // ...
  ];

  return (
    <main>
      <h1>Trading Chart</h1>
      <Chart data={data} />
    </main>
  );
}
```

---

### Решение 2: Проверка typeof window
**Оценка: 7/10**

**Суть:** Условный рендеринг на клиенте.

```jsx
'use client';

import { createChart } from 'lightweight-charts';
import { useEffect, useRef, useState } from 'react';

export default function Chart({ data }) {
  const containerRef = useRef(null);
  const [isMounted, setIsMounted] = useState(false);

  useEffect(() => {
    setIsMounted(true);
  }, []);

  useEffect(() => {
    if (!isMounted || !containerRef.current) return;
    if (typeof window === 'undefined') return;

    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
    });

    chart.addCandlestickSeries().setData(data);

    return () => chart.remove();
  }, [isMounted, data]);

  // Не рендерим на сервере
  if (!isMounted) {
    return <div style={{ height: '400px' }}>Loading...</div>;
  }

  return <div ref={containerRef} style={{ width: '100%', height: '400px' }} />;
}
```

---

### Решение 3: Lazy import внутри useEffect
**Оценка: 8/10**

**Суть:** Динамический import библиотеки внутри useEffect.

```jsx
'use client';

import { useEffect, useRef, useState } from 'react';

export default function Chart({ data }) {
  const containerRef = useRef(null);
  const [LightweightCharts, setLightweightCharts] = useState(null);

  // Загружаем библиотеку только на клиенте
  useEffect(() => {
    import('lightweight-charts').then((module) => {
      setLightweightCharts(module);
    });
  }, []);

  // Создаём график после загрузки библиотеки
  useEffect(() => {
    if (!LightweightCharts || !containerRef.current) return;

    const chart = LightweightCharts.createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
    });

    chart.addCandlestickSeries().setData(data);

    return () => chart.remove();
  }, [LightweightCharts, data]);

  return (
    <div 
      ref={containerRef} 
      style={{ width: '100%', height: '400px' }}
    >
      {!LightweightCharts && <span>Loading chart...</span>}
    </div>
  );
}
```

---

## ✅ Полный пример для Next.js 13+ App Router

```
📁 app/
├── page.jsx          # Server Component
├── layout.jsx
📁 components/
├── ChartWrapper.jsx  # Client Component wrapper
├── ChartCore.jsx     # Chart logic
```

```jsx
// components/ChartCore.jsx
'use client';

import { createChart } from 'lightweight-charts';
import { useEffect, useRef } from 'react';

export default function ChartCore({ data, options = {} }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const chart = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: options.height || 400,
      layout: {
        background: { type: 'solid', color: '#1a1a2e' },
        textColor: '#d1d4dc',
      },
      ...options,
    });

    const series = chart.addCandlestickSeries();
    if (data?.length) {
      series.setData(data);
      chart.timeScale().fitContent();
    }

    chartRef.current = chart;

    const handleResize = () => {
      if (chart && containerRef.current) {
        chart.applyOptions({ width: containerRef.current.clientWidth });
      }
    };

    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      chart.remove();
    };
  }, [data, options]);

  return <div ref={containerRef} className="chart-container" />;
}
```

```jsx
// components/ChartWrapper.jsx
'use client';

import dynamic from 'next/dynamic';

const ChartCore = dynamic(() => import('./ChartCore'), {
  ssr: false,
  loading: () => (
    <div style={{ 
      height: '400px', 
      background: '#1a1a2e',
      display: 'flex',
      alignItems: 'center',
      justifyContent: 'center',
      color: '#d1d4dc'
    }}>
      Loading chart...
    </div>
  ),
});

export default function ChartWrapper(props) {
  return <ChartCore {...props} />;
}
```

```jsx
// app/page.jsx
import ChartWrapper from '@/components/ChartWrapper';

// Это Server Component - данные можно получить на сервере
async function getChartData() {
  // Fetch data from API
  return [
    { time: '2024-01-01', open: 100, high: 110, low: 95, close: 105 },
    { time: '2024-01-02', open: 105, high: 115, low: 100, close: 112 },
    // ...
  ];
}

export default async function Page() {
  const data = await getChartData();

  return (
    <main className="container">
      <h1>Trading Dashboard</h1>
      <ChartWrapper data={data} options={{ height: 500 }} />
    </main>
  );
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Загрузка данных |
|---------|--------|-----------|-----------------|
| #1 next/dynamic ssr:false | 10/10 | Низкая | Server + Client |
| #2 typeof window check | 7/10 | Низкая | Client only |
| #3 Lazy import in useEffect | 8/10 | Средняя | Client only |

---

## ⚠️ Важные замечания

1. **Данные можно получать на сервере** - только рендеринг графика должен быть на клиенте
2. **SEO не страдает** - график это декоративный элемент
3. **Loading state важен** - показывайте placeholder пока грузится

---

## 🔗 Источники

1. [Next.js Dynamic Imports](https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading)
2. [Next.js Client Components](https://nextjs.org/docs/app/building-your-application/rendering/client-components)
3. [Lightweight Charts SSR issues](https://github.com/tradingview/lightweight-charts/issues?q=ssr)

---

**Документ создан:** 18 января 2026
