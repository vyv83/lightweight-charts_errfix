# БАГ #49: Разные scale margins для price и volume panes скрывают метки

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#2013](https://github.com/tradingview/lightweight-charts/issues/2013)  
> **Версии:** v5.0+  
> **Статус:** 🔴 Open  
> **Последнее обновление:** Ноябрь 2025

## 📋 Описание проблемы

### Суть проблемы

При настройке **разных `scaleMargins`** для ценовой панели (candlesticks) и панели объёмов (volume histogram), **метки оси volume исчезают**. Попытка установить отдельные margins для каждой панели приводит к скрытию price scale labels на одной из панелей.

### Детали

1. **Симптомы:**
   - Volume axis labels полностью исчезают
   - Price scale для volume pane становится невидимой
   - При наведении crosshair значения volume не отображаются

2. **Когда возникает:**
   - Использование multi-pane layout (несколько панелей)
   - Назначение разных `scaleMargins` для разных серий
   - Использование отдельного `priceScaleId` для volume

3. **Техническая причина:**
   - Конфликт между `scaleMargins` разных серий на одном price scale
   - Неправильный расчёт доступного пространства для labels
   - Проблема с pane ordering и visibility

### Сценарии возникновения

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  width: window.innerWidth,
  height: window.innerHeight,
});

// Серия свечей на pane 0
const candleSeries = chart.addSeries(
  LightweightCharts.CandlestickSeries,
  {}
);

// Серия объёмов на pane 1
const volumeSeries = chart.addSeries(
  LightweightCharts.HistogramSeries,
  {
    priceScaleId: 'volume-scale', // Отдельный price scale
    priceFormat: {
      type: 'volume',
    },
  },
  1 // pane index
);

candleSeries.setData([
  { time: '2018-10-19', open: 75, high: 82.8, low: 70, close: 80 },
  { time: '2018-10-22', open: 77, high: 88, low: 73, close: 86 },
  // ...
]);

volumeSeries.setData([
  { time: '2018-10-19', value: 522000 },
  { time: '2018-10-22', value: 1500000 },
  // ...
]);

// ❌ ПРОБЛЕМА: При установке разных scaleMargins
candleSeries.priceScale().applyOptions({ 
  scaleMargins: { top: 0.2, bottom: 0.1 } 
});

// Volume axis исчезает!
chart.priceScale('volume-scale').applyOptions({
  visible: true,
  scaleMargins: { top: 0.0, bottom: 0.0 }, // Другие margins
  borderVisible: true,
});
```

### Визуализация проблемы

```
Ожидаемое поведение:          Реальное поведение:
┌─────────────────┬──────┐    ┌─────────────────┬──────┐
│                 │ 88   │    │                 │ 88   │
│    Candlestick  │ 80   │    │    Candlestick  │ 80   │
│      Chart      │ 72   │    │      Chart      │ 72   │
├─────────────────┼──────┤    ├─────────────────┼──────┤
│                 │ 2.1M │    │                 │      │← Пусто!
│     Volume      │ 1.5M │    │     Volume      │      │← Labels
│    Histogram    │ 0.5M │    │    Histogram    │      │  исчезли!
└─────────────────┴──────┘    └─────────────────┴──────┘
```

### Реальные сценарии

1. **Trading dashboard:**
   - Цена занимает 70% высоты
   - Объём занимает 30% высоты
   - Разные margins для визуального разделения

2. **Multi-indicator layout:**
   - Основной график (top: 0.1, bottom: 0.3)
   - RSI индикатор (top: 0.1, bottom: 0.1)
   - Volume (top: 0, bottom: 0)

3. **Custom pane layouts:**
   - Профессиональные терминалы
   - Аналитические дашборды

## 🔍 Найденные решения

### Решение 1: Единый priceScaleId для всех серий на pane (⭐ Простое)

**Оценка:** ⭐⭐⭐ (3/5) - Workaround с ограничениями

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  height: 500,
});

// Используем стандартный right price scale для обеих серий
const candleSeries = chart.addCandlestickSeries({
  priceScaleId: 'right', // Стандартный
});

// Volume тоже на right, но на отдельном pane
const volumeSeries = chart.addHistogramSeries(
  {
    priceScaleId: 'right', // Тот же scale!
    priceFormat: { type: 'volume' },
  },
  1 // pane index
);

// Применяем scaleMargins к price scale, а не к серии
chart.priceScale('right').applyOptions({
  scaleMargins: { 
    top: 0.1, 
    bottom: 0.3, // Оставляем место для volume pane
  },
});

candleSeries.setData(candleData);
volumeSeries.setData(volumeData);
```

**Плюсы:**
- Простая реализация
- Labels видны

**Минусы:**
- Ограниченный контроль над layout
- Shared scale между разными данными

---

### Решение 2: Overlay pane для volume (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Правильный подход

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

interface MultiPaneChartConfig {
  container: HTMLElement;
  height: number;
  paneRatios: number[]; // [0.7, 0.3] = 70% price, 30% volume
}

/**
 * Создаёт multi-pane chart с правильными margins
 */
function createMultiPaneChart(config: MultiPaneChartConfig): {
  chart: IChartApi;
  panes: { pricePane: number; volumePane: number };
} {
  const { container, height, paneRatios } = config;
  
  // Валидация ratios
  const totalRatio = paneRatios.reduce((a, b) => a + b, 0);
  if (Math.abs(totalRatio - 1) > 0.001) {
    throw new Error('Pane ratios must sum to 1');
  }
  
  const chart = createChart(container, {
    height,
  });
  
  // Рассчитываем margins для каждого pane
  const priceTop = 0.05; // 5% отступ сверху
  const priceBottom = 1 - paneRatios[0] + 0.02; // Место для volume + небольшой gap
  
  const volumeTop = paneRatios[0] + 0.02; // После price pane + gap
  const volumeBottom = 0.05; // 5% отступ снизу
  
  return {
    chart,
    panes: {
      pricePane: 0,
      volumePane: 1,
    },
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const { chart } = createMultiPaneChart({
  container: document.getElementById('chart')!,
  height: 600,
  paneRatios: [0.7, 0.3],
});

// Добавляем серии
const candleSeries = chart.addCandlestickSeries();

const volumeSeries = chart.addHistogramSeries(
  {
    priceFormat: { type: 'volume' },
    priceScaleId: 'volume',
  },
  1 // Второй pane
);

// Настраиваем price scales раздельно
candleSeries.priceScale().applyOptions({
  scaleMargins: { top: 0.05, bottom: 0.35 }, // 35% снизу для volume
});

chart.priceScale('volume').applyOptions({
  scaleMargins: { top: 0.72, bottom: 0.03 }, // 72% сверху
  visible: true,
});

candleSeries.setData(candleData);
volumeSeries.setData(volumeData);
```

**Плюсы:**
- Правильное разделение панелей
- Labels видны на обеих панелях
- Настраиваемые пропорции

**Минусы:**
- Требует тщательного расчёта margins
- Overlap вместо настоящих panes

---

### Решение 3: Визуальное скрытие volume scale с кастомным overlay

**Оценка:** ⭐⭐⭐⭐ (4/5) - Полный контроль

```typescript
import { createChart, IChartApi, MouseEventParams } from 'lightweight-charts';

interface VolumeOverlayConfig {
  container: HTMLElement;
  formatter?: (volume: number) => string;
  position?: 'left' | 'right';
}

/**
 * Создаёт кастомный volume axis overlay
 */
function createVolumeAxisOverlay(
  chart: IChartApi,
  volumeSeries: ISeriesApi<'Histogram'>,
  config: VolumeOverlayConfig
): { destroy: () => void; update: () => void } {
  const { 
    container, 
    formatter = formatVolume,
    position = 'right',
  } = config;
  
  // Создаём overlay контейнер
  const overlay = document.createElement('div');
  overlay.style.cssText = `
    position: absolute;
    ${position}: 0;
    bottom: 0;
    height: 30%;
    width: 60px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    padding: 5px;
    background: transparent;
    pointer-events: none;
    z-index: 10;
  `;
  
  container.style.position = 'relative';
  container.appendChild(overlay);
  
  // Создаём элементы для min/max/current
  const labels = ['max', 'mid', 'min'].map(type => {
    const label = document.createElement('div');
    label.className = `volume-label volume-${type}`;
    label.style.cssText = `
      color: #787b86;
      font-size: 10px;
      text-align: ${position};
      font-family: -apple-system, BlinkMacSystemFont, sans-serif;
    `;
    overlay.appendChild(label);
    return label;
  });
  
  function formatVolume(volume: number): string {
    if (volume >= 1e9) return `${(volume / 1e9).toFixed(1)}B`;
    if (volume >= 1e6) return `${(volume / 1e6).toFixed(1)}M`;
    if (volume >= 1e3) return `${(volume / 1e3).toFixed(1)}K`;
    return volume.toFixed(0);
  }
  
  function update(): void {
    const data = volumeSeries.data() as { value: number }[];
    if (data.length === 0) return;
    
    const values = data.map(d => d.value).filter(v => v !== undefined);
    const max = Math.max(...values);
    const min = Math.min(...values);
    const mid = (max + min) / 2;
    
    labels[0].textContent = formatter(max);
    labels[1].textContent = formatter(mid);
    labels[2].textContent = formatter(min);
  }
  
  // Initial update
  update();
  
  return {
    destroy: () => overlay.remove(),
    update,
  };
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const chart = createChart(container, { height: 500 });

const candleSeries = chart.addCandlestickSeries();

// Volume с скрытой осью
const volumeSeries = chart.addHistogramSeries(
  {
    priceFormat: { type: 'volume' },
    priceScaleId: 'volume',
  },
  1
);

// Скрываем встроенную ось volume
chart.priceScale('volume').applyOptions({
  visible: false, // Скрываем!
  scaleMargins: { top: 0.7, bottom: 0.02 },
});

// Создаём кастомный overlay
const volumeOverlay = createVolumeAxisOverlay(chart, volumeSeries, {
  container,
  position: 'right',
});

candleSeries.setData(candleData);
volumeSeries.setData(volumeData);
volumeOverlay.update();
```

**Плюсы:**
- Полный контроль над отображением
- Нет конфликтов с margins
- Настраиваемый дизайн

**Минусы:**
- Дополнительный код
- Overlay не связан с autoscale
- Нужно синхронизировать при изменении данных

---

### Решение 4: Фиксированные пропорции через CSS

**Оценка:** ⭐⭐⭐ (3/5) - Быстрый workaround

```typescript
import { createChart } from 'lightweight-charts';

// Создаём контейнеры для раздельных графиков
const priceContainer = document.createElement('div');
priceContainer.style.cssText = 'height: 70%; width: 100%;';

const volumeContainer = document.createElement('div');
volumeContainer.style.cssText = 'height: 30%; width: 100%;';

container.appendChild(priceContainer);
container.appendChild(volumeContainer);

// Создаём ОТДЕЛЬНЫЕ графики
const priceChart = createChart(priceContainer, {
  height: container.clientHeight * 0.7,
  timeScale: {
    visible: false, // Скрываем на price chart
  },
});

const volumeChart = createChart(volumeContainer, {
  height: container.clientHeight * 0.3,
});

// Синхронизируем time scales
priceChart.timeScale().subscribeVisibleTimeRangeChange(() => {
  const range = priceChart.timeScale().getVisibleRange();
  if (range) {
    volumeChart.timeScale().setVisibleRange(range);
  }
});

volumeChart.timeScale().subscribeVisibleTimeRangeChange(() => {
  const range = volumeChart.timeScale().getVisibleRange();
  if (range) {
    priceChart.timeScale().setVisibleRange(range);
  }
});

// Добавляем серии в свои контейнеры
const candleSeries = priceChart.addCandlestickSeries();
const volumeSeries = volumeChart.addHistogramSeries({
  priceFormat: { type: 'volume' },
});

// Scale margins теперь независимы!
candleSeries.priceScale().applyOptions({
  scaleMargins: { top: 0.1, bottom: 0.1 },
});

volumeSeries.priceScale().applyOptions({
  scaleMargins: { top: 0.1, bottom: 0.1 },
});
```

**Плюсы:**
- Полная независимость panels
- Нет конфликтов margins
- Простая синхронизация

**Минусы:**
- Два отдельных chart instance
- Больше памяти
- Нужна синхронизация crosshair

---

### Решение 5: Правильная настройка pane heights

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Нативное решение (v5.0+)

```typescript
import { createChart, PriceScaleMode } from 'lightweight-charts';

const chart = createChart(container, {
  height: 600,
  // Включаем multi-pane layout
});

// Добавляем candlestick на pane 0
const candleSeries = chart.addCandlestickSeries({
  priceScaleId: 'price',
});

// Добавляем volume на pane 1
const volumeSeries = chart.addHistogramSeries(
  {
    priceScaleId: 'volume',
    priceFormat: { type: 'volume' },
    color: '#26a69a',
  },
  1 // Pane index
);

// КЛЮЧЕВОЙ МОМЕНТ: Настраиваем price scales для каждого pane отдельно
// и используем непересекающиеся scaleMargins

// Price scale для свечей (занимает верхние 70%)
chart.priceScale('price').applyOptions({
  scaleMargins: {
    top: 0.05,    // 5% отступ сверху
    bottom: 0.32, // 32% снизу (для volume + gap)
  },
  visible: true,
  borderVisible: true,
});

// Price scale для volume (занимает нижние 25%)
chart.priceScale('volume').applyOptions({
  scaleMargins: {
    top: 0.73,    // 73% сверху (после price pane)
    bottom: 0.02, // 2% отступ снизу
  },
  visible: true,
  borderVisible: true,
  mode: PriceScaleMode.Normal,
});

// Загружаем данные
candleSeries.setData(candleData);
volumeSeries.setData(volumeData);

chart.timeScale().fitContent();
```

**Плюсы:**
- Нативное решение
- Правильная работа labels
- Использует API библиотеки

**Минусы:**
- Требует точного расчёта margins
- Overlap, а не настоящие panes

## ✅ Рекомендуемое решение

Используйте **Решение 5** с правильно рассчитанными margins:

```typescript
// Расчёт margins для multi-pane layout
function calculatePaneMargins(
  paneHeights: number[], // [70, 30] = 70% и 30%
  gaps: number = 3       // gap между panes в %
): { top: number; bottom: number }[] {
  const total = paneHeights.reduce((a, b) => a + b, 0);
  const normalized = paneHeights.map(h => h / total);
  
  const margins: { top: number; bottom: number }[] = [];
  let currentTop = 0;
  
  for (let i = 0; i < normalized.length; i++) {
    const height = normalized[i];
    const gapSize = gaps / 100;
    
    margins.push({
      top: currentTop + (i > 0 ? gapSize : 0.02),
      bottom: 1 - currentTop - height + (i < normalized.length - 1 ? gapSize : 0.02),
    });
    
    currentTop += height;
  }
  
  return margins;
}

// Использование
const [priceMargins, volumeMargins] = calculatePaneMargins([70, 30]);

chart.priceScale('price').applyOptions({ scaleMargins: priceMargins });
chart.priceScale('volume').applyOptions({ scaleMargins: volumeMargins });
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Надёжность | Гибкость | Рекомендация |
|---------|----------|------------|----------|--------------|
| **#1 Единый priceScaleId** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | Простые случаи |
| **#2 Overlay pane** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Рекомендуется |
| **#3 Кастомный overlay** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Full control |
| **#4 Отдельные charts** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Максимальная независимость |
| **#5 Правильные margins** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Native решение |

## 🔗 Источники

- [GitHub Issue #2013](https://github.com/tradingview/lightweight-charts/issues/2013) - Different scale margins hide labels
- [Price Scale Docs](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceScaleOptions) - Документация
- [Multi-pane Examples](https://tradingview.github.io/lightweight-charts/tutorials/how_to/price-and-volume) - Примеры

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
