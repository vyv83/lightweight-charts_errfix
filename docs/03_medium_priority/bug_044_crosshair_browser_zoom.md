# БАГ #44: Некорректная работа crosshair при browser zoom

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issues:** Множественные reports ([StackOverflow](https://stackoverflow.com/questions/75557556/lightweight-charts-issue-with-crosshair-when-page-is-zoomed)), [#982](https://github.com/tradingview/lightweight-charts/issues/982)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2023+

## 📋 Описание проблемы

### Суть проблемы

При изменении масштаба браузера (browser zoom) или использовании CSS `zoom`/`transform: scale()` на родительских элементах, **crosshair (перекрестие)** отображается **со смещением** относительно фактической позиции курсора мыши.

### Детали

1. **Browser zoom (Ctrl+/-):**
   - При zoom уровне ≠ 100% crosshair не совпадает с курсором
   - Смещение пропорционально уровню zoom

2. **CSS zoom/scale:**
   - При `document.body.style.zoom = '80%'` или `transform: scale(0.8)` crosshair сдвигается
   - Библиотека рассчитывает позицию мыши без учёта трансформации

3. **devicePixelRatio:**
   - На Retina/HiDPI дисплеях при browser zoom бары могут исчезать
   - Связано с sub-pixel рендерингом

### Визуальный эффект

```
Ожидаемое:                  Фактическое (при zoom 80%):
                            
   │                           │
───┼───  ← Курсор тут      ───┼───  ← Crosshair тут
   │                           │
   ▲                               ▲ ← Курсор смещён сюда
   Crosshair точно под курсором
```

### Сценарии возникновения

1. **Browser zoom:**
   - Пользователь нажал Ctrl+/-
   - Или изменил масштаб в настройках браузера

2. **CSS zoom для адаптации UI:**
   ```javascript
   // Уменьшение интерфейса для компактности
   document.body.style.zoom = '80%';
   ```

3. **Responsive design с transform:**
   ```css
   .dashboard {
     transform: scale(0.9);
     transform-origin: top left;
   }
   ```

4. **Embed в iframe со скейлом:**
   ```html
   <iframe style="transform: scale(0.75);">
   ```

## 🔍 Найденные решения

### Решение 1: Контр-zoom на контейнере графика (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Самое простое и эффективное

```typescript
/**
 * Применяет контр-zoom к контейнеру графика
 * чтобы компенсировать zoom родительского элемента
 */
function applyCounterZoom(
  chartContainer: HTMLElement,
  parentZoom: number // Например, 0.8 для 80%
): void {
  const counterZoom = 1 / parentZoom;
  
  // Метод 1: CSS zoom (предпочтительно)
  chartContainer.style.zoom = String(counterZoom);
  
  // ИЛИ Метод 2: transform scale
  // chartContainer.style.transform = `scale(${counterZoom})`;
  // chartContainer.style.transformOrigin = 'top left';
  
  // Корректируем размеры для сохранения видимой области
  chartContainer.style.width = `${100 * parentZoom}%`;
  chartContainer.style.height = `${100 * parentZoom}%`;
}

// Использование
const parentZoom = 0.8; // Родитель имеет zoom 80%
document.body.style.zoom = String(parentZoom);

const chartContainer = document.getElementById('chart')!;
applyCounterZoom(chartContainer, parentZoom);

const chart = createChart(chartContainer, {
  width: chartContainer.clientWidth,
  height: chartContainer.clientHeight,
});
```

**Плюсы:**
- Простое решение
- Полная совместимость с библиотекой
- Crosshair работает корректно

**Минусы:**
- Требует знания текущего zoom уровня
- Нужна корректировка размеров контейнера

---

### Решение 2: Wrapper с изоляцией от zoom

**Оценка:** ⭐⭐⭐⭐ (4/5) - Изолированный контейнер

```typescript
/**
 * Создаёт изолированный контейнер для графика,
 * защищённый от внешних CSS трансформаций
 */
function createIsolatedChartContainer(
  parentElement: HTMLElement,
  options: {
    width: number;
    height: number;
  }
): HTMLElement {
  // Создаём wrapper с фиксированными размерами
  const wrapper = document.createElement('div');
  wrapper.style.cssText = `
    position: relative;
    width: ${options.width}px;
    height: ${options.height}px;
    overflow: hidden;
  `;
  
  // Контейнер графика с reset всех трансформаций
  const chartContainer = document.createElement('div');
  chartContainer.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    zoom: 1 !important;
    transform: none !important;
    -webkit-transform: none !important;
  `;
  
  wrapper.appendChild(chartContainer);
  parentElement.appendChild(wrapper);
  
  return chartContainer;
}

// Использование
const container = createIsolatedChartContainer(document.body, {
  width: 800,
  height: 400,
});

const chart = createChart(container);
```

**Плюсы:**
- Изоляция от внешних стилей
- Предсказуемое поведение

**Минусы:**
- Фиксированные размеры
- Не адаптивный

---

### Решение 3: Отслеживание и коррекция zoom

**Оценка:** ⭐⭐⭐⭐ (4/5) - Динамическое отслеживание

```typescript
import { IChartApi, createChart } from 'lightweight-charts';

interface ZoomAwareChartOptions {
  container: HTMLElement;
  chartOptions?: Parameters<typeof createChart>[1];
}

/**
 * Создаёт график с автоматической коррекцией zoom
 */
function createZoomAwareChart(options: ZoomAwareChartOptions): {
  chart: IChartApi;
  destroy: () => void;
} {
  const { container, chartOptions } = options;
  
  // Определяем текущий effectiveZoom
  const getEffectiveZoom = (): number => {
    let zoom = 1;
    let element: HTMLElement | null = container;
    
    while (element) {
      const computedStyle = window.getComputedStyle(element);
      const elementZoom = parseFloat(computedStyle.zoom) || 1;
      zoom *= elementZoom;
      
      // Также учитываем transform: scale
      const transform = computedStyle.transform;
      if (transform && transform !== 'none') {
        const match = transform.match(/matrix\(([^,]+)/);
        if (match) {
          const scale = parseFloat(match[1]);
          zoom *= scale;
        }
      }
      
      element = element.parentElement;
    }
    
    // Учитываем browser zoom через devicePixelRatio
    // (приблизительно, не всегда точно)
    const deviceZoom = window.devicePixelRatio / (window.screen.deviceXDPICoef || 1);
    
    return zoom;
  };
  
  // Создаём внутренний контейнер с коррекцией
  const innerContainer = document.createElement('div');
  innerContainer.style.cssText = `
    width: 100%;
    height: 100%;
    position: relative;
  `;
  container.appendChild(innerContainer);
  
  // Применяем начальную коррекцию
  const applyZoomCorrection = () => {
    const effectiveZoom = getEffectiveZoom();
    if (effectiveZoom !== 1) {
      const counter = 1 / effectiveZoom;
      innerContainer.style.zoom = String(counter);
      innerContainer.style.width = `${100 * effectiveZoom}%`;
      innerContainer.style.height = `${100 * effectiveZoom}%`;
    }
  };
  
  applyZoomCorrection();
  
  // Создаём график
  const chart = createChart(innerContainer, chartOptions);
  
  // Отслеживаем изменения zoom (MutationObserver для CSS изменений)
  const observer = new MutationObserver(() => {
    applyZoomCorrection();
    chart.resize(innerContainer.clientWidth, innerContainer.clientHeight);
  });
  
  observer.observe(document.body, {
    attributes: true,
    attributeFilter: ['style'],
    subtree: true,
  });
  
  // Отслеживаем resize (который может означать browser zoom)
  const resizeHandler = () => {
    applyZoomCorrection();
    chart.resize(innerContainer.clientWidth, innerContainer.clientHeight);
  };
  window.addEventListener('resize', resizeHandler);
  
  return {
    chart,
    destroy: () => {
      observer.disconnect();
      window.removeEventListener('resize', resizeHandler);
      chart.remove();
      container.removeChild(innerContainer);
    },
  };
}

// Использование
const { chart, destroy } = createZoomAwareChart({
  container: document.getElementById('chart')!,
  chartOptions: {
    width: 800,
    height: 400,
  },
});

// При unmount
// destroy();
```

**Плюсы:**
- Автоматическое отслеживание
- Работает с динамическим zoom
- Комплексное решение

**Минусы:**
- Сложная реализация
- MutationObserver может быть ресурсоёмким
- Не 100% точное определение browser zoom

---

### Решение 4: Использование transform: scale() вместо zoom

**Оценка:** ⭐⭐⭐ (3/5) - Альтернатива CSS zoom

```typescript
/**
 * Используем transform вместо zoom для более предсказуемого поведения
 */
function setupScaledLayout(scaleFactor: number): {
  chartContainer: HTMLElement;
  cleanup: () => void;
} {
  // Создаём структуру:
  // .outer-wrapper (видимый размер)
  //   .scale-wrapper (transform: scale)
  //     #chart (реальный размер 1:1)
  
  const outerWrapper = document.createElement('div');
  outerWrapper.style.cssText = `
    width: 100%;
    height: 400px;
    overflow: hidden;
    position: relative;
  `;
  
  const scaleWrapper = document.createElement('div');
  scaleWrapper.style.cssText = `
    transform: scale(${scaleFactor});
    transform-origin: top left;
    width: ${100 / scaleFactor}%;
    height: ${100 / scaleFactor}%;
    position: absolute;
    top: 0;
    left: 0;
  `;
  
  const chartContainer = document.createElement('div');
  chartContainer.style.cssText = `
    width: 100%;
    height: 100%;
  `;
  
  scaleWrapper.appendChild(chartContainer);
  outerWrapper.appendChild(scaleWrapper);
  document.body.appendChild(outerWrapper);
  
  // Создаём график в НЕ-масштабированном контейнере
  // Размеры графика соответствуют реальным пикселям
  const realWidth = outerWrapper.clientWidth / scaleFactor;
  const realHeight = outerWrapper.clientHeight / scaleFactor;
  
  return {
    chartContainer,
    cleanup: () => {
      document.body.removeChild(outerWrapper);
    },
  };
}

// Использование
const { chartContainer, cleanup } = setupScaledLayout(0.8);

const chart = createChart(chartContainer, {
  // Размеры в "реальных" пикселях
  width: 1000,  // Будет отображаться как 800px при scale 0.8
  height: 500,  // Будет отображаться как 400px при scale 0.8
});
```

**Плюсы:**
- Transform более предсказуем чем zoom
- Crosshair рассчитывается в "реальных" координатах

**Минусы:**
- Сложная структура DOM
- Размеры нужно пересчитывать
- Может быть проблема с hit testing

---

### Решение 5: Custom crosshair через overlay

**Оценка:** ⭐⭐⭐ (3/5) - Полностью кастомный crosshair

```typescript
import { IChartApi, createChart, CrosshairMode } from 'lightweight-charts';

/**
 * Создаёт кастомный crosshair, корректно работающий при zoom
 */
function createCustomCrosshair(
  chart: IChartApi,
  container: HTMLElement,
  zoomFactor: number = 1
): {
  show: (x: number, y: number) => void;
  hide: () => void;
  destroy: () => void;
} {
  // Отключаем встроенный crosshair
  chart.applyOptions({
    crosshair: {
      mode: CrosshairMode.Hidden,
    },
  });
  
  // Создаём overlay для кастомного crosshair
  const overlay = document.createElement('div');
  overlay.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 100;
  `;
  
  const vertLine = document.createElement('div');
  vertLine.style.cssText = `
    position: absolute;
    width: 1px;
    height: 100%;
    background: rgba(150, 150, 150, 0.5);
    display: none;
  `;
  
  const horzLine = document.createElement('div');
  horzLine.style.cssText = `
    position: absolute;
    width: 100%;
    height: 1px;
    background: rgba(150, 150, 150, 0.5);
    display: none;
  `;
  
  overlay.appendChild(vertLine);
  overlay.appendChild(horzLine);
  container.style.position = 'relative';
  container.appendChild(overlay);
  
  // Обработчик движения мыши с коррекцией zoom
  const handleMouseMove = (event: MouseEvent) => {
    const rect = container.getBoundingClientRect();
    
    // Корректируем координаты с учётом zoom
    const x = (event.clientX - rect.left) / zoomFactor;
    const y = (event.clientY - rect.top) / zoomFactor;
    
    vertLine.style.left = `${x}px`;
    vertLine.style.display = 'block';
    
    horzLine.style.top = `${y}px`;
    horzLine.style.display = 'block';
  };
  
  const handleMouseLeave = () => {
    vertLine.style.display = 'none';
    horzLine.style.display = 'none';
  };
  
  container.addEventListener('mousemove', handleMouseMove);
  container.addEventListener('mouseleave', handleMouseLeave);
  
  return {
    show: (x: number, y: number) => {
      vertLine.style.left = `${x}px`;
      vertLine.style.display = 'block';
      horzLine.style.top = `${y}px`;
      horzLine.style.display = 'block';
    },
    hide: () => {
      vertLine.style.display = 'none';
      horzLine.style.display = 'none';
    },
    destroy: () => {
      container.removeEventListener('mousemove', handleMouseMove);
      container.removeEventListener('mouseleave', handleMouseLeave);
      container.removeChild(overlay);
    },
  };
}
```

**Плюсы:**
- Полный контроль над crosshair
- Работает при любом zoom

**Минусы:**
- Потеря функционала магнитного режима
- Нужно синхронизировать с данными графика
- Дополнительная сложность

## ✅ Рекомендуемое решение

Для большинства случаев рекомендуется **Решение 1** с дополнительным утилитарным классом:

```typescript
import { createChart, IChartApi, ChartOptions, DeepPartial } from 'lightweight-charts';

interface ZoomSafeChartConfig {
  container: HTMLElement;
  parentZoom?: number;
  chartOptions?: DeepPartial<ChartOptions>;
}

/**
 * Создаёт график, защищённый от проблем с browser/CSS zoom
 */
export function createZoomSafeChart(config: ZoomSafeChartConfig): {
  chart: IChartApi;
  container: HTMLElement;
  updateZoom: (newZoom: number) => void;
  destroy: () => void;
} {
  const { container, parentZoom = 1, chartOptions = {} } = config;
  
  // Создаём внутренний контейнер
  const innerContainer = document.createElement('div');
  innerContainer.className = 'lightweight-chart-zoom-safe';
  
  // Функция применения коррекции zoom
  const applyZoomCorrection = (zoom: number) => {
    if (zoom === 1) {
      innerContainer.style.cssText = `
        width: 100%;
        height: 100%;
      `;
    } else {
      const counterZoom = 1 / zoom;
      innerContainer.style.cssText = `
        zoom: ${counterZoom};
        width: ${zoom * 100}%;
        height: ${zoom * 100}%;
      `;
    }
  };
  
  applyZoomCorrection(parentZoom);
  container.appendChild(innerContainer);
  
  // Создаём график
  const chart = createChart(innerContainer, {
    width: container.clientWidth,
    height: container.clientHeight,
    ...chartOptions,
  });
  
  // Обработчик resize
  const handleResize = () => {
    const width = innerContainer.clientWidth;
    const height = innerContainer.clientHeight;
    chart.resize(width, height);
  };
  
  const resizeObserver = new ResizeObserver(handleResize);
  resizeObserver.observe(innerContainer);
  
  return {
    chart,
    container: innerContainer,
    updateZoom: (newZoom: number) => {
      applyZoomCorrection(newZoom);
      handleResize();
    },
    destroy: () => {
      resizeObserver.disconnect();
      chart.remove();
      container.removeChild(innerContainer);
    },
  };
}

// ==================== ПРИМЕР ИСПОЛЬЗОВАНИЯ ====================

// Сценарий 1: Известный CSS zoom на body
document.body.style.zoom = '80%';

const { chart, updateZoom, destroy } = createZoomSafeChart({
  container: document.getElementById('chart')!,
  parentZoom: 0.8,
  chartOptions: {
    layout: {
      background: { type: 'solid', color: '#1e1e1e' },
      textColor: '#d1d4dc',
    },
  },
});

const series = chart.addCandlestickSeries();
series.setData(myData);

// Сценарий 2: Динамическое изменение zoom
function setAppZoom(newZoom: number) {
  document.body.style.zoom = String(newZoom);
  updateZoom(newZoom);
}

// При размонтировании
// destroy();
```

## 📊 Сравнительная таблица решений

| Решение | Простота | Надёжность | Совместимость | Рекомендация |
|---------|----------|------------|---------------|--------------|
| **#1 Контр-zoom** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Рекомендуется |
| **#2 Изолированный контейнер** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Для фиксированных размеров |
| **#3 Динамическое отслеживание** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Для сложных случаев |
| **#4 Transform вместо zoom** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Альтернатива |
| **#5 Custom crosshair** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Когда нужен полный контроль |

## 🔧 Дополнительные рекомендации

### Определение текущего browser zoom

```typescript
/**
 * Приблизительное определение browser zoom
 * Точного способа не существует!
 */
function estimateBrowserZoom(): number {
  // Метод 1: Через devicePixelRatio (неточно)
  // На Retina devicePixelRatio = 2 при 100% zoom
  const baseRatio = window.matchMedia('(min-resolution: 2dppx)').matches ? 2 : 1;
  const estimated = window.devicePixelRatio / baseRatio;
  
  // Метод 2: Через размер экрана (ещё менее точно)
  // const zoomLevel = window.outerWidth / window.innerWidth;
  
  return Math.round(estimated * 100) / 100;
}
```

### CSS для предотвращения browser zoom

```css
/* Предотвращение случайного zoom на mobile */
html {
  touch-action: manipulation;
}

/* Фиксированный viewport */
@viewport {
  zoom: 1.0;
  width: device-width;
}
```

### Предупреждение пользователя о zoom

```typescript
function warnIfZoomed(): void {
  const estimatedZoom = estimateBrowserZoom();
  
  if (Math.abs(estimatedZoom - 1) > 0.05) {
    console.warn(
      `Browser zoom detected (~${Math.round(estimatedZoom * 100)}%). ` +
      `Chart crosshair may not work correctly. ` +
      `Please reset zoom to 100% (Ctrl+0) for best experience.`
    );
  }
}
```

## 🔗 Источники

- [StackOverflow: Lightweight charts issue with crosshair when page is zoomed](https://stackoverflow.com/questions/75557556/lightweight-charts-issue-with-crosshair-when-page-is-zoomed) - Описание проблемы
- [GitHub Issue #982](https://github.com/tradingview/lightweight-charts/issues/982) - Bars disappear with devicePixelRatio < 1
- [Recharts Issue #2829](https://github.com/recharts/recharts/issues/2829) - Аналогичная проблема в другой библиотеке
- [MDN: CSS zoom](https://developer.mozilla.org/en-US/docs/Web/CSS/zoom) - Документация CSS zoom
- [MDN: currentCSSZoom](https://developer.mozilla.org/en-US/docs/Web/API/Element/currentCSSZoom) - API для определения zoom

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
