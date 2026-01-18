# БАГ #21: Медленный pinch-zoom на Android

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#1300](https://github.com/tradingview/lightweight-charts/issues/1300)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Последнее обновление:** январь 2026

---

## 📋 Описание проблемы

### Суть бага
Pinch-to-zoom работает **значительно медленнее при zoom OUT** на Android устройствах по сравнению с zoom IN. Скорость zoom определяется начальным направлением жеста и остаётся асимметричной на протяжении всего взаимодействия.

### Симптомы
- ❌ **Zoom out** (пальцы сводятся) — очень медленный, "вязкий"
- ❌ **Zoom in** (пальцы разводятся) — нормальная скорость
- ❌ Ощущение "неотзывчивости" при навигации
- ❌ Плохой UX на мобильных trading apps

### Технические причины
1. **Внутренний алгоритм расчёта zoom** — неравномерная обработка delta для zoom in vs zoom out
2. **Canvas в WebView** — дополнительные задержки rendering
3. **Touch events processing** — особенности обработки жестов на Android
4. **Data conflation** — при zoom out требуется перерисовка большего количества данных

### Сценарии воспроизведения
- Любое использование pinch gesture на Android устройствах
- Особенно заметно при работе с большими датасетами
- Воспроизводится на всех браузерах Android (Chrome, Firefox, Samsung Browser)

### Частота
**100%** на Android devices

### Платформы
- ✅ Воспроизводится: Android (все браузеры)
- ⚠️ Частично: iOS (меньше выражено)
- ❌ Не воспроизводится: Desktop

---

## 🔍 Найденные решения

### Решение 1: Нормализация скорости zoom через Wrapper
**Оценка: 8/10** ⭐ РЕКОМЕНДУЕТСЯ

Перехват touch events и нормализация delta для симметричной скорости:

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

/**
 * Wrapper для нормализации скорости pinch-zoom на Android
 */
class NormalizedZoomChart {
  private chart: IChartApi;
  private container: HTMLElement;
  
  // Touch tracking
  private initialPinchDistance: number = 0;
  private lastPinchDistance: number = 0;
  private isPinching: boolean = false;
  
  // Zoom normalization factor (увеличиваем для zoom out)
  private zoomOutMultiplier: number = 2.5;
  
  constructor(container: HTMLElement, options: any = {}) {
    this.container = container;
    
    // Создаём chart
    this.chart = createChart(container, {
      ...options,
      handleScale: {
        axisPressedMouseMove: true,
        mouseWheel: true,
        pinch: false, // Отключаем встроенный pinch
      },
    });
    
    this.setupTouchHandlers();
  }

  private setupTouchHandlers() {
    let touchStartHandler: (e: TouchEvent) => void;
    let touchMoveHandler: (e: TouchEvent) => void;
    let touchEndHandler: (e: TouchEvent) => void;

    touchStartHandler = (e: TouchEvent) => {
      if (e.touches.length === 2) {
        this.isPinching = true;
        this.initialPinchDistance = this.getPinchDistance(e);
        this.lastPinchDistance = this.initialPinchDistance;
        e.preventDefault();
      }
    };

    touchMoveHandler = (e: TouchEvent) => {
      if (!this.isPinching || e.touches.length !== 2) return;
      
      e.preventDefault();
      
      const currentDistance = this.getPinchDistance(e);
      const delta = currentDistance - this.lastPinchDistance;
      
      // Нормализуем скорость zoom
      let normalizedDelta = delta;
      
      if (delta < 0) {
        // Zoom OUT — применяем multiplier для компенсации
        normalizedDelta = delta * this.zoomOutMultiplier;
      }
      
      // Применяем zoom
      this.applyZoom(normalizedDelta, e);
      
      this.lastPinchDistance = currentDistance;
    };

    touchEndHandler = (e: TouchEvent) => {
      if (e.touches.length < 2) {
        this.isPinching = false;
        this.initialPinchDistance = 0;
        this.lastPinchDistance = 0;
      }
    };

    // Добавляем listeners с passive: false для preventDefault
    this.container.addEventListener('touchstart', touchStartHandler, { passive: false });
    this.container.addEventListener('touchmove', touchMoveHandler, { passive: false });
    this.container.addEventListener('touchend', touchEndHandler);
    this.container.addEventListener('touchcancel', touchEndHandler);
  }

  private getPinchDistance(e: TouchEvent): number {
    const touch1 = e.touches[0];
    const touch2 = e.touches[1];
    
    return Math.sqrt(
      Math.pow(touch2.clientX - touch1.clientX, 2) +
      Math.pow(touch2.clientY - touch1.clientY, 2)
    );
  }

  private getPinchCenter(e: TouchEvent): { x: number; y: number } {
    const touch1 = e.touches[0];
    const touch2 = e.touches[1];
    const rect = this.container.getBoundingClientRect();
    
    return {
      x: ((touch1.clientX + touch2.clientX) / 2) - rect.left,
      y: ((touch1.clientY + touch2.clientY) / 2) - rect.top,
    };
  }

  private applyZoom(delta: number, e: TouchEvent) {
    const timeScale = this.chart.timeScale();
    const center = this.getPinchCenter(e);
    
    // Конвертируем delta в zoom factor
    // Положительный delta = zoom in, отрицательный = zoom out
    const zoomFactor = 1 + (delta / 500);
    
    // Получаем текущий visible range
    const visibleRange = timeScale.getVisibleLogicalRange();
    if (!visibleRange) return;
    
    const rangeLength = visibleRange.to - visibleRange.from;
    const newRangeLength = rangeLength / zoomFactor;
    
    // Вычисляем новый range с центром в точке pinch
    const containerWidth = this.container.clientWidth;
    const centerRatio = center.x / containerWidth;
    
    const fromDelta = (rangeLength - newRangeLength) * centerRatio;
    const toDelta = (rangeLength - newRangeLength) * (1 - centerRatio);
    
    timeScale.setVisibleLogicalRange({
      from: visibleRange.from + fromDelta,
      to: visibleRange.to - toDelta,
    });
  }

  public getChart(): IChartApi {
    return this.chart;
  }

  public setZoomOutMultiplier(multiplier: number) {
    this.zoomOutMultiplier = multiplier;
  }

  public destroy() {
    this.chart.remove();
  }
}

// Использование
const container = document.getElementById('chart')!;
const chartWrapper = new NormalizedZoomChart(container, {
  layout: {
    background: { color: '#1a1a2e' },
    textColor: '#e0e0e0',
  },
});

const chart = chartWrapper.getChart();
const series = chart.addCandlestickSeries();
series.setData(data);
```

**Преимущества:**
- ✅ Полный контроль над скоростью zoom
- ✅ Симметричное поведение zoom in/out
- ✅ Настраиваемый multiplier

**Недостатки:**
- ⚠️ Отключает встроенную обработку pinch
- ⚠️ Требует тщательной настройки коэффициентов

---

### Решение 2: CSS touch-action оптимизация
**Оценка: 6/10**

Настройка CSS для предотвращения конфликтов с browser gestures:

```css
/* Стили для chart container */
.chart-container {
  /* Разрешаем только pinch-zoom, блокируем pan */
  touch-action: pinch-zoom;
  
  /* Или полный контроль через JS */
  /* touch-action: none; */
  
  /* Предотвращаем выделение текста при жестах */
  user-select: none;
  -webkit-user-select: none;
  
  /* Отключаем highlight при tap на Android */
  -webkit-tap-highlight-color: transparent;
  
  /* Предотвращаем context menu */
  -webkit-touch-callout: none;
}

/* Для улучшения отзывчивости */
.chart-container canvas {
  /* GPU acceleration */
  transform: translateZ(0);
  will-change: transform;
  
  /* Предотвращаем blur при transform */
  image-rendering: crisp-edges;
  image-rendering: -webkit-optimize-contrast;
}
```

```typescript
// JavaScript дополнение
function setupChartContainer(container: HTMLElement) {
  // Предотвращаем стандартные browser gestures
  container.addEventListener('gesturestart', (e) => e.preventDefault());
  container.addEventListener('gesturechange', (e) => e.preventDefault());
  container.addEventListener('gestureend', (e) => e.preventDefault());
  
  // Для Android - предотвращаем долгий тап
  container.addEventListener('contextmenu', (e) => e.preventDefault());
  
  // Оптимизация - passive: false только где нужен preventDefault
  container.addEventListener('touchmove', (e) => {
    if (e.touches.length > 1) {
      e.preventDefault();
    }
  }, { passive: false });
}
```

**Преимущества:**
- ✅ Простое решение
- ✅ Улучшает общую отзывчивость
- ✅ Не требует изменений в JS логике chart

**Недостатки:**
- ⚠️ Не решает проблему полностью
- ⚠️ Может конфликтовать с другими touch interactions

---

### Решение 3: Data Conflation (v5.1.0+)
**Оценка: 7/10**

Использование встроенной оптимизации для больших датасетов:

```typescript
import { createChart } from 'lightweight-charts';

// Data conflation автоматически включён в v5.1.0+
const chart = createChart(container, {
  // Настройки по умолчанию включают data conflation
  layout: {
    background: { color: '#1a1a2e' },
    textColor: '#e0e0e0',
  },
});

// Для оптимальной производительности при больших данных:
const series = chart.addCandlestickSeries({
  // Минимизируем визуальные эффекты для performance
  lastValueVisible: false,
  priceLineVisible: false,
});

// Оптимизированная загрузка данных
async function loadOptimizedData() {
  // Загружаем только видимый диапазон + буфер
  const visibleRange = chart.timeScale().getVisibleLogicalRange();
  
  // При zoom out грузим агрегированные данные
  const logicalWidth = visibleRange 
    ? visibleRange.to - visibleRange.from 
    : 1000;
  
  if (logicalWidth > 500) {
    // Zoom out - используем агрегированные данные (1h вместо 1m)
    const aggregatedData = await fetchAggregatedData('1h');
    series.setData(aggregatedData);
  } else {
    // Zoom in - детальные данные
    const detailedData = await fetchDetailedData('1m');
    series.setData(detailedData);
  }
}

// Подписываемся на изменения visible range
chart.timeScale().subscribeVisibleLogicalRangeChange(loadOptimizedData);
```

**Преимущества:**
- ✅ Автоматическая оптимизация в новых версиях
- ✅ Значительное улучшение для больших датасетов
- ✅ Не требует custom touch handling

**Недостатки:**
- ⚠️ Требует v5.1.0+
- ⚠️ Не решает проблему асимметрии напрямую

---

### Решение 4: Hammer.js для Advanced Gesture Control
**Оценка: 8/10**

Использование библиотеки для точного контроля жестов:

```typescript
import Hammer from 'hammerjs';
import { createChart, IChartApi } from 'lightweight-charts';

class HammerZoomChart {
  private chart: IChartApi;
  private hammer: HammerManager;
  private container: HTMLElement;
  
  private initialZoomLevel: number = 1;
  private currentZoomLevel: number = 1;
  private lastScale: number = 1;

  constructor(container: HTMLElement, options: any = {}) {
    this.container = container;
    
    // Создаём chart без встроенного pinch
    this.chart = createChart(container, {
      ...options,
      handleScale: {
        pinch: false,
        mouseWheel: true,
        axisPressedMouseMove: true,
      },
    });

    this.setupHammer();
  }

  private setupHammer() {
    this.hammer = new Hammer.Manager(this.container, {
      recognizers: [
        [Hammer.Pinch, { enable: true }],
        [Hammer.Pan, { enable: true, direction: Hammer.DIRECTION_ALL }],
      ],
    });

    // Настраиваем pinch recognizer
    const pinch = this.hammer.get('pinch');
    pinch.set({
      threshold: 0.1, // Чувствительность
    });

    let lastScale = 1;
    let initialRange: { from: number; to: number } | null = null;

    this.hammer.on('pinchstart', (e) => {
      lastScale = 1;
      initialRange = this.chart.timeScale().getVisibleLogicalRange();
    });

    this.hammer.on('pinchmove', (e) => {
      if (!initialRange) return;

      // e.scale: 1 = начальное положение, <1 = zoom out, >1 = zoom in
      let scaleChange = e.scale / lastScale;
      
      // Нормализация: ускоряем zoom out
      if (scaleChange < 1) {
        // Zoom out - применяем коррекцию
        const correction = 1 + (1 - scaleChange) * 1.5; // 1.5x усиление
        scaleChange = 1 / correction;
      }

      const rangeLength = initialRange.to - initialRange.from;
      const newRangeLength = rangeLength / scaleChange;
      
      // Центр pinch
      const rect = this.container.getBoundingClientRect();
      const centerX = e.center.x - rect.left;
      const centerRatio = centerX / this.container.clientWidth;
      
      // Применяем zoom с сохранением центра
      const fromDelta = (rangeLength - newRangeLength) * centerRatio;
      const toDelta = (rangeLength - newRangeLength) * (1 - centerRatio);
      
      this.chart.timeScale().setVisibleLogicalRange({
        from: initialRange.from + fromDelta,
        to: initialRange.to - toDelta,
      });

      lastScale = e.scale;
    });

    this.hammer.on('pinchend', () => {
      lastScale = 1;
      initialRange = null;
    });

    // Pan для прокрутки
    this.hammer.on('panmove', (e) => {
      if (e.maxPointers > 1) return; // Игнорируем во время pinch
      
      const timeScale = this.chart.timeScale();
      const range = timeScale.getVisibleLogicalRange();
      if (!range) return;

      const rangeLength = range.to - range.from;
      const pixelsPerBar = this.container.clientWidth / rangeLength;
      const barsDelta = -e.deltaX / pixelsPerBar;

      // Применяем pan с throttling
      requestAnimationFrame(() => {
        timeScale.setVisibleLogicalRange({
          from: range.from + barsDelta * 0.1,
          to: range.to + barsDelta * 0.1,
        });
      });
    });
  }

  public getChart(): IChartApi {
    return this.chart;
  }

  public destroy() {
    this.hammer.destroy();
    this.chart.remove();
  }
}

// Использование
const chartWrapper = new HammerZoomChart(container, {
  layout: { background: { color: '#1a1a2e' } }
});
```

**Преимущества:**
- ✅ Профессиональная обработка жестов
- ✅ Точная настройка чувствительности
- ✅ Поддержка сложных gesture patterns

**Недостатки:**
- ⚠️ Дополнительная зависимость (~7KB gzip)
- ⚠️ Требует настройки

---

### Решение 5: Zoom Controls UI (кнопки)
**Оценка: 6/10**

Альтернативный интерфейс для zoom без touch gestures:

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

class ChartWithZoomControls {
  private chart: IChartApi;
  private container: HTMLElement;
  private controlsContainer: HTMLElement;

  constructor(container: HTMLElement, options: any = {}) {
    this.container = container;
    this.chart = createChart(container, options);
    this.createZoomControls();
  }

  private createZoomControls() {
    this.controlsContainer = document.createElement('div');
    this.controlsContainer.className = 'chart-zoom-controls';
    this.controlsContainer.innerHTML = `
      <button class="zoom-btn zoom-in" aria-label="Zoom In">+</button>
      <button class="zoom-btn zoom-out" aria-label="Zoom Out">−</button>
      <button class="zoom-btn zoom-reset" aria-label="Reset Zoom">⟲</button>
    `;

    this.container.style.position = 'relative';
    this.container.appendChild(this.controlsContainer);

    // Стили
    const style = document.createElement('style');
    style.textContent = `
      .chart-zoom-controls {
        position: absolute;
        top: 10px;
        right: 10px;
        display: flex;
        flex-direction: column;
        gap: 5px;
        z-index: 100;
      }
      .zoom-btn {
        width: 36px;
        height: 36px;
        border: none;
        border-radius: 6px;
        background: rgba(255, 255, 255, 0.1);
        color: #e0e0e0;
        font-size: 18px;
        cursor: pointer;
        touch-action: manipulation;
        transition: background 0.2s;
      }
      .zoom-btn:hover, .zoom-btn:active {
        background: rgba(255, 255, 255, 0.2);
      }
      .zoom-btn:active {
        transform: scale(0.95);
      }
    `;
    document.head.appendChild(style);

    // Event handlers
    this.controlsContainer.querySelector('.zoom-in')!
      .addEventListener('click', () => this.zoom(1.5));
    
    this.controlsContainer.querySelector('.zoom-out')!
      .addEventListener('click', () => this.zoom(0.67)); // Симметричный к 1.5
    
    this.controlsContainer.querySelector('.zoom-reset')!
      .addEventListener('click', () => this.resetZoom());
  }

  private zoom(factor: number) {
    const timeScale = this.chart.timeScale();
    const range = timeScale.getVisibleLogicalRange();
    if (!range) return;

    const rangeLength = range.to - range.from;
    const newRangeLength = rangeLength / factor;
    const center = (range.from + range.to) / 2;

    timeScale.setVisibleLogicalRange({
      from: center - newRangeLength / 2,
      to: center + newRangeLength / 2,
    });
  }

  private resetZoom() {
    this.chart.timeScale().fitContent();
  }

  public getChart(): IChartApi {
    return this.chart;
  }

  public destroy() {
    this.chart.remove();
    this.controlsContainer.remove();
  }
}
```

**Преимущества:**
- ✅ Избегает проблем с touch gestures полностью
- ✅ Доступный UI (a11y)
- ✅ Предсказуемое поведение

**Недостатки:**
- ⚠️ Не такой intuitive как pinch
- ⚠️ Требует дополнительного UI пространства

---

## ✅ Рекомендуемое решение

### Комбинированный подход для Production

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

/**
 * Production-ready chart with normalized Android pinch zoom
 */
class MobileOptimizedChart {
  private chart: IChartApi;
  private container: HTMLElement;
  private isAndroid: boolean;
  private zoomOutCorrection: number = 2.0;
  
  // Touch state
  private touches: Map<number, { x: number; y: number }> = new Map();
  private initialPinchDistance: number = 0;
  private lastPinchDistance: number = 0;
  private pinchCenter: { x: number; y: number } = { x: 0, y: 0 };
  private isPinching: boolean = false;

  constructor(container: HTMLElement, options: any = {}) {
    this.container = container;
    this.isAndroid = /android/i.test(navigator.userAgent);

    // Применяем CSS оптимизации
    this.applyContainerStyles();

    // Создаём chart
    this.chart = createChart(container, {
      ...options,
      handleScale: {
        axisPressedMouseMove: true,
        mouseWheel: true,
        pinch: !this.isAndroid, // Отключаем на Android
      },
    });

    // Custom pinch только для Android
    if (this.isAndroid) {
      this.setupNormalizedPinch();
    }

    console.log(`MobileOptimizedChart: Android=${this.isAndroid}`);
  }

  private applyContainerStyles() {
    Object.assign(this.container.style, {
      touchAction: 'none',
      userSelect: 'none',
      WebkitUserSelect: 'none',
      WebkitTapHighlightColor: 'transparent',
    });
  }

  private setupNormalizedPinch() {
    const handleTouchStart = (e: TouchEvent) => {
      for (let i = 0; i < e.changedTouches.length; i++) {
        const touch = e.changedTouches[i];
        this.touches.set(touch.identifier, { 
          x: touch.clientX, 
          y: touch.clientY 
        });
      }

      if (this.touches.size === 2) {
        this.isPinching = true;
        const points = Array.from(this.touches.values());
        this.initialPinchDistance = this.getDistance(points[0], points[1]);
        this.lastPinchDistance = this.initialPinchDistance;
        this.pinchCenter = this.getMidpoint(points[0], points[1]);
        e.preventDefault();
      }
    };

    const handleTouchMove = (e: TouchEvent) => {
      // Update touch positions
      for (let i = 0; i < e.changedTouches.length; i++) {
        const touch = e.changedTouches[i];
        this.touches.set(touch.identifier, { 
          x: touch.clientX, 
          y: touch.clientY 
        });
      }

      if (!this.isPinching || this.touches.size !== 2) return;
      e.preventDefault();

      const points = Array.from(this.touches.values());
      const currentDistance = this.getDistance(points[0], points[1]);
      const delta = currentDistance - this.lastPinchDistance;

      // Нормализация скорости
      let normalizedDelta = delta;
      if (delta < 0) {
        // Zoom out - усиливаем
        normalizedDelta = delta * this.zoomOutCorrection;
      }

      // Применяем zoom
      this.applyNormalizedZoom(normalizedDelta);

      this.lastPinchDistance = currentDistance;
      this.pinchCenter = this.getMidpoint(points[0], points[1]);
    };

    const handleTouchEnd = (e: TouchEvent) => {
      for (let i = 0; i < e.changedTouches.length; i++) {
        this.touches.delete(e.changedTouches[i].identifier);
      }

      if (this.touches.size < 2) {
        this.isPinching = false;
        this.initialPinchDistance = 0;
        this.lastPinchDistance = 0;
      }
    };

    this.container.addEventListener('touchstart', handleTouchStart, { passive: false });
    this.container.addEventListener('touchmove', handleTouchMove, { passive: false });
    this.container.addEventListener('touchend', handleTouchEnd);
    this.container.addEventListener('touchcancel', handleTouchEnd);
  }

  private getDistance(p1: { x: number; y: number }, p2: { x: number; y: number }): number {
    return Math.sqrt(Math.pow(p2.x - p1.x, 2) + Math.pow(p2.y - p1.y, 2));
  }

  private getMidpoint(p1: { x: number; y: number }, p2: { x: number; y: number }): { x: number; y: number } {
    return {
      x: (p1.x + p2.x) / 2,
      y: (p1.y + p2.y) / 2,
    };
  }

  private applyNormalizedZoom(delta: number) {
    const timeScale = this.chart.timeScale();
    const visibleRange = timeScale.getVisibleLogicalRange();
    if (!visibleRange) return;

    const zoomFactor = 1 + (delta / 400);
    const rangeLength = visibleRange.to - visibleRange.from;
    const newRangeLength = rangeLength / zoomFactor;

    // Центрируем zoom на точке pinch
    const rect = this.container.getBoundingClientRect();
    const centerX = this.pinchCenter.x - rect.left;
    const centerRatio = centerX / this.container.clientWidth;

    const fromDelta = (rangeLength - newRangeLength) * centerRatio;
    const toDelta = (rangeLength - newRangeLength) * (1 - centerRatio);

    timeScale.setVisibleLogicalRange({
      from: visibleRange.from + fromDelta,
      to: visibleRange.to - toDelta,
    });
  }

  /**
   * Настроить коэффициент коррекции zoom out
   * По умолчанию: 2.0 (zoom out в 2 раза быстрее)
   */
  public setZoomOutCorrection(factor: number) {
    this.zoomOutCorrection = factor;
  }

  public getChart(): IChartApi {
    return this.chart;
  }

  public destroy() {
    this.chart.remove();
  }
}

// ============== ИСПОЛЬЗОВАНИЕ ==============

function createMobileChart(containerId: string) {
  const container = document.getElementById(containerId)!;
  
  const chartManager = new MobileOptimizedChart(container, {
    layout: {
      background: { color: '#1a1a2e' },
      textColor: '#e0e0e0',
    },
    grid: {
      vertLines: { color: '#2a2a4a' },
      horzLines: { color: '#2a2a4a' },
    },
    timeScale: {
      borderColor: '#2a2a4a',
    },
    rightPriceScale: {
      borderColor: '#2a2a4a',
    },
  });

  // Опционально: настроить коррекцию
  // chartManager.setZoomOutCorrection(2.5);

  const chart = chartManager.getChart();
  const series = chart.addCandlestickSeries({
    upColor: '#26a69a',
    downColor: '#ef5350',
    borderVisible: false,
    wickUpColor: '#26a69a',
    wickDownColor: '#ef5350',
  });

  series.setData(yourData);

  return chartManager;
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Универсальность | Без зависимостей |
|---------|--------|-----------|-----------------|------------------|
| **Normalized Wrapper** | 8/10 | 🟡 Средняя | ✅ Да | ✅ Да |
| **CSS touch-action** | 6/10 | 🟢 Низкая | ⚠️ Частичная | ✅ Да |
| **Data Conflation** | 7/10 | 🟢 Низкая | ⚠️ v5.1.0+ | ✅ Да |
| **Hammer.js** | 8/10 | 🟡 Средняя | ✅ Да | ❌ Нет |
| **Zoom Controls UI** | 6/10 | 🟢 Низкая | ✅ Да | ✅ Да |

### Рекомендации:

| Сценарий | Решение |
|----------|---------|
| Production mobile app | Решение 1 или 4 |
| Быстрое исправление | Решение 2 (CSS) |
| Большие датасеты | Решение 3 + 1 |
| Accessibility focus | Решение 5 |

---

## 🔗 Источники

1. **GitHub Issue #1300** - Pinch zoom out slow on mobile  
   https://github.com/tradingview/lightweight-charts/issues/1300

2. **Data Conflation (v5.1.0)**  
   https://tradingview.github.io/lightweight-charts/docs/next/whats-new

3. **Hammer.js** - Touch gestures library  
   https://hammerjs.github.io/

4. **Touch Events MDN**  
   https://developer.mozilla.org/en-US/docs/Web/API/Touch_events

5. **CSS touch-action**  
   https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action

---

## 📝 Дополнительные заметки

### Тестирование на Android

```bash
# Chrome DevTools → Remote Debugging
adb devices
# Откройте chrome://inspect в Chrome desktop
```

### Отладка touch events

```javascript
// Добавьте для отладки
document.addEventListener('touchmove', (e) => {
  console.log('Touch count:', e.touches.length);
  console.log('Touch positions:', Array.from(e.touches).map(t => ({
    x: t.clientX,
    y: t.clientY
  })));
}, { passive: true });
```

### Когда ожидать официальное исправление?

Issue #1300 остаётся открытым. Рекомендуется использовать custom touch handling для critical mobile applications.
