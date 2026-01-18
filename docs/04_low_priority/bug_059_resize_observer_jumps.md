# БАГ #59: Проблемы с ResizeObserver — скачки при resize

> **Критичность:** 🟢 НИЗКАЯ  
> **Issues:** [#71](https://github.com/tradingview/lightweight-charts/issues/71), [#1746](https://github.com/tradingview/lightweight-charts/issues/1746)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** ⚠️ Частично исправлено (есть autoSize опция)  
> **Последнее обновление:** 2024

## 📋 Описание проблемы

### Суть проблемы

При изменении размера контейнера графика могут возникать **визуальные "скачки" или мерцание**. Это связано с работой ResizeObserver и тем, как библиотека обрабатывает изменения размеров. Особенно заметно при плавном resize (drag resize) или при быстрых изменениях размера.

### Детали

1. **Симптомы:**
   - Мерцание графика при resize
   - "Прыжки" содержимого
   - Временное исчезновение элементов
   - Некорректный размер после resize

2. **Когда возникает:**
   - Изменение размера окна браузера
   - Drag resize сплиттеров/панелей
   - Анимированное изменение размера
   - Toggle sidebar с transition

3. **Причины:**
   - ResizeObserver может срабатывать несколько раз
   - Асинхронность обновления размеров
   - Конфликт между requestAnimationFrame и resize

### Сценарий воспроизведения

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;

// Без autoSize — нужно вручную обрабатывать resize
const chart = createChart(container, {
  width: container.clientWidth,
  height: 400,
});

// При быстром resize могут быть проблемы
window.addEventListener('resize', () => {
  chart.resize(container.clientWidth, 400);
});

// ❌ Множественные вызовы resize могут вызвать скачки
// ❌ Нет debounce
// ❌ Размеры могут не совпадать во время анимации
```

## 🔍 Найденные решения

### Решение 1: Использование autoSize (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Встроенное решение

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;

// autoSize автоматически обрабатывает resize
const chart = createChart(container, {
  autoSize: true, // ✅ Автоматическое управление размером
});

// Контейнер должен иметь определённые размеры
container.style.cssText = `
  width: 100%;
  height: 400px;
`;

const series = chart.addLineSeries();
series.setData(data);

// Не нужен ручной resize listener!
```

**Плюсы:**
- Встроенное решение
- Автоматический debounce
- Минимум кода

**Минусы:**
- Контейнер должен иметь явные размеры
- Меньше контроля

---

### Решение 2: Debounced resize handler

**Оценка:** ⭐⭐⭐⭐ (4/5) - Контролируемое решение

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

/**
 * Создаёт debounced функцию
 */
function debounce<T extends (...args: any[]) => void>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  
  return (...args: Parameters<T>) => {
    if (timeout) {
      clearTimeout(timeout);
    }
    timeout = setTimeout(() => func(...args), wait);
  };
}

/**
 * Создаёт throttled функцию
 */
function throttle<T extends (...args: any[]) => void>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;
  
  return (...args: Parameters<T>) => {
    if (!inThrottle) {
      func(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

class SmoothedResizeChart {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _resizeObserver: ResizeObserver;
  private _lastWidth = 0;
  private _lastHeight = 0;
  
  constructor(container: HTMLElement, height: number) {
    this._container = container;
    this._lastWidth = container.clientWidth;
    this._lastHeight = height;
    
    this._chart = createChart(container, {
      width: this._lastWidth,
      height: this._lastHeight,
      autoSize: false, // Мы управляем вручную
    });
    
    // Debounced resize для плавности
    const debouncedResize = debounce((width: number, height: number) => {
      this._performResize(width, height);
    }, 100);
    
    // Throttled resize для немедленного отклика
    const throttledResize = throttle((width: number, height: number) => {
      this._performResize(width, height);
    }, 16); // ~60fps
    
    // ResizeObserver с оптимизацией
    this._resizeObserver = new ResizeObserver((entries) => {
      for (const entry of entries) {
        const { width, height } = entry.contentRect;
        
        // Проверяем значительное изменение
        if (Math.abs(width - this._lastWidth) > 1 || 
            Math.abs(height - this._lastHeight) > 1) {
          
          // Используем throttle для плавности
          throttledResize(width, height);
          
          // Используем debounce для финального размера
          debouncedResize(width, height);
        }
      }
    });
    
    this._resizeObserver.observe(container);
  }
  
  private _performResize(width: number, height: number): void {
    // Используем requestAnimationFrame для синхронизации с отрисовкой
    requestAnimationFrame(() => {
      this._chart.resize(width, height);
      this._lastWidth = width;
      this._lastHeight = height;
    });
  }
  
  get chart(): IChartApi {
    return this._chart;
  }
  
  destroy(): void {
    this._resizeObserver.disconnect();
    this._chart.remove();
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
const smoothChart = new SmoothedResizeChart(container, 400);

const series = smoothChart.chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Плавный resize
- Контролируемый debounce/throttle
- Избегает множественных вызовов

**Минусы:**
- Больше кода
- Небольшая задержка

---

### Решение 3: CSS contain для оптимизации

**Оценка:** ⭐⭐⭐⭐ (4/5) - CSS оптимизация

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;

// CSS containment для оптимизации рендеринга
container.style.cssText = `
  width: 100%;
  height: 400px;
  contain: layout size; /* Изолирует layout от остальной страницы */
  will-change: width, height; /* Hint для браузера */
`;

const chart = createChart(container, {
  autoSize: true,
});

// Дополнительные стили для плавности
const style = document.createElement('style');
style.textContent = `
  .tv-lightweight-charts {
    will-change: transform;
  }
  
  /* Предотвращает мерцание при resize */
  .tv-lightweight-charts canvas {
    image-rendering: -webkit-optimize-contrast;
  }
`;
document.head.appendChild(style);
```

**Плюсы:**
- CSS-уровень оптимизации
- Минимальный overhead
- Улучшает производительность

**Минусы:**
- Не все браузеры поддерживают contain
- Может не решить все проблемы

---

### Решение 4: Transition-aware resize

**Оценка:** ⭐⭐⭐⭐ (4/5) - Для анимированных изменений

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

class TransitionAwareChart {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _isTransitioning = false;
  private _animationFrame: number | null = null;
  
  constructor(container: HTMLElement) {
    this._container = container;
    
    this._chart = createChart(container, {
      autoSize: false,
      width: container.clientWidth,
      height: container.clientHeight,
    });
    
    // Слушаем transition events
    container.addEventListener('transitionstart', () => {
      this._isTransitioning = true;
      this._startContinuousResize();
    });
    
    container.addEventListener('transitionend', () => {
      this._isTransitioning = false;
      this._stopContinuousResize();
      // Финальный resize
      this._resize();
    });
    
    // Также слушаем обычный resize
    const resizeObserver = new ResizeObserver(() => {
      if (!this._isTransitioning) {
        this._resize();
      }
    });
    resizeObserver.observe(container);
  }
  
  private _startContinuousResize(): void {
    const loop = () => {
      if (this._isTransitioning) {
        this._resize();
        this._animationFrame = requestAnimationFrame(loop);
      }
    };
    loop();
  }
  
  private _stopContinuousResize(): void {
    if (this._animationFrame !== null) {
      cancelAnimationFrame(this._animationFrame);
      this._animationFrame = null;
    }
  }
  
  private _resize(): void {
    const width = this._container.clientWidth;
    const height = this._container.clientHeight;
    this._chart.resize(width, height);
  }
  
  get chart(): IChartApi {
    return this._chart;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

// CSS с transition
const container = document.getElementById('chart')!;
container.style.cssText = `
  width: 100%;
  height: 400px;
  transition: width 0.3s ease;
`;

const transitionChart = new TransitionAwareChart(container);
const series = transitionChart.chart.addLineSeries();
series.setData(data);

// При изменении ширины контейнера график плавно адаптируется
document.getElementById('toggle-sidebar')?.addEventListener('click', () => {
  container.style.width = container.style.width === '100%' ? '70%' : '100%';
});
```

**Плюсы:**
- Плавный resize при CSS transitions
- Continuous update во время анимации

**Минусы:**
- Зависит от transition events
- Дополнительная нагрузка во время анимации

---

### Решение 5: Double buffering эффект

**Оценка:** ⭐⭐⭐ (3/5) - Для критичных случаев

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

class DoubleBufferedChart {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _overlay: HTMLElement;
  private _isResizing = false;
  private _resizeTimeout: ReturnType<typeof setTimeout> | null = null;
  
  constructor(container: HTMLElement) {
    this._container = container;
    
    // Создаём overlay для скрытия мерцания
    this._overlay = document.createElement('div');
    this._overlay.style.cssText = `
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: inherit;
      z-index: 100;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.1s ease;
    `;
    container.style.position = 'relative';
    container.appendChild(this._overlay);
    
    this._chart = createChart(container, {
      autoSize: true,
    });
    
    // Отслеживаем resize
    const resizeObserver = new ResizeObserver(() => {
      this._onResizeStart();
    });
    resizeObserver.observe(container);
  }
  
  private _onResizeStart(): void {
    if (!this._isResizing) {
      this._isResizing = true;
      // Показываем overlay
      this._overlay.style.opacity = '0.5';
    }
    
    // Сбрасываем таймер
    if (this._resizeTimeout) {
      clearTimeout(this._resizeTimeout);
    }
    
    // Скрываем overlay после стабилизации
    this._resizeTimeout = setTimeout(() => {
      this._isResizing = false;
      this._overlay.style.opacity = '0';
    }, 150);
  }
  
  get chart(): IChartApi {
    return this._chart;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
const bufferedChart = new DoubleBufferedChart(container);

const series = bufferedChart.chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Скрывает визуальные артефакты
- Простая концепция

**Минусы:**
- Overlay может быть заметен
- Не решает проблему, а маскирует

---

### Решение 6: requestAnimationFrame synchronization

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Оптимальная синхронизация

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

class SyncedResizeChart {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _pendingResize = false;
  private _targetWidth = 0;
  private _targetHeight = 0;
  
  constructor(container: HTMLElement) {
    this._container = container;
    this._targetWidth = container.clientWidth;
    this._targetHeight = container.clientHeight;
    
    this._chart = createChart(container, {
      width: this._targetWidth,
      height: this._targetHeight,
      autoSize: false,
    });
    
    // ResizeObserver только собирает размеры
    const resizeObserver = new ResizeObserver((entries) => {
      for (const entry of entries) {
        this._targetWidth = entry.contentRect.width;
        this._targetHeight = entry.contentRect.height;
        this._scheduleResize();
      }
    });
    
    resizeObserver.observe(container);
  }
  
  private _scheduleResize(): void {
    if (this._pendingResize) return;
    
    this._pendingResize = true;
    
    requestAnimationFrame(() => {
      this._pendingResize = false;
      
      // Проверяем что размеры действительно изменились
      const currentWidth = this._chart.options().width;
      const currentHeight = this._chart.options().height;
      
      if (this._targetWidth !== currentWidth || this._targetHeight !== currentHeight) {
        this._chart.resize(this._targetWidth, this._targetHeight);
      }
    });
  }
  
  get chart(): IChartApi {
    return this._chart;
  }
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
container.style.cssText = 'width: 100%; height: 400px;';

const syncedChart = new SyncedResizeChart(container);
const series = syncedChart.chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Синхронизация с render loop
- Избегает множественных resize
- Оптимальная производительность

**Минусы:**
- Небольшая задержка (1 frame)

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 1** (autoSize):

```typescript
// Простейшее решение
const chart = createChart(container, {
  autoSize: true,
});
```

Если нужен больший контроль, используйте **Решение 6** (requestAnimationFrame synchronization).

## 📊 Сравнительная таблица решений

| Решение | Простота | Плавность | Производительность | Рекомендация |
|---------|----------|-----------|-------------------|--------------|
| **#1 autoSize** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ Стандартное |
| **#2 Debounced** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Плавный resize |
| **#3 CSS contain** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Оптимизация |
| **#4 Transition-aware** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Анимации |
| **#5 Double buffer** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Маскировка |
| **#6 RAF sync** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ Оптимальное |

## 🔗 Источники

- [GitHub Issue #71](https://github.com/tradingview/lightweight-charts/issues/71) - Initial resize issues
- [GitHub Issue #1746](https://github.com/tradingview/lightweight-charts/issues/1746) - ResizeObserver improvements
- [autoSize Documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ChartOptions#autosize)
- [Chart Sizing Guide](https://tradingview.github.io/lightweight-charts/tutorials/how_to/set-chart-size)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
