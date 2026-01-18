# БАГ #56: Временная шкала не полностью видна

> **Критичность:** 🟢 НИЗКАЯ  
> **Issue:** [#978](https://github.com/tradingview/lightweight-charts/issues/978)  
> **Версии:** Все версии, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** 2022

## 📋 Описание проблемы

### Суть проблемы

В некоторых конфигурациях **временная шкала (time scale) может быть частично скрыта или обрезана**. Это происходит при определённых комбинациях размеров контейнера, настроек шкал и CSS стилей.

### Детали

1. **Симптомы:**
   - Нижняя часть временной шкалы обрезана
   - Метки времени частично видны
   - Scroll/overflow скрывает шкалу

2. **Когда возникает:**
   - Контейнер с фиксированной высотой без учёта шкалы
   - CSS overflow: hidden на родителе
   - Неправильный расчёт высоты

3. **Ожидаемое поведение:**
   - Временная шкала полностью видна
   - Высота графика автоматически учитывает шкалу

### Визуализация проблемы

```
Ожидаемое:                        Реальное:
┌─────────────────────────────┐   ┌─────────────────────────────┐
│                             │   │                             │
│       Chart Area            │   │       Chart Area            │
│                             │   │                             │
├─────────────────────────────┤   ├─────────────────────────────┤
│  Jan │ Feb │ Mar │ Apr │May │   │  Jan │ Feb │ Mar │ Apr │May │
└─────────────────────────────┘   └───────────────────────────┬─┘
   ↑ Полностью видна                 ↑ Обрезана             ──┘
                                                          (hidden)
```

### Сценарий воспроизведения

```typescript
// Проблемный CSS
const container = document.getElementById('chart')!;
container.style.cssText = `
  height: 300px;
  overflow: hidden; /* ❌ Может обрезать time scale */
`;

const chart = createChart(container, {
  height: 300, // Высота включает time scale
  timeScale: {
    visible: true,
    borderVisible: true,
  },
});

// Time scale занимает ~30px, но container не учитывает это
// Результат: нижние 30px обрезаны
```

## 🔍 Найденные решения

### Решение 1: Правильный расчёт высоты (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Корректный подход

```typescript
import { createChart } from 'lightweight-charts';

const TIME_SCALE_HEIGHT = 30; // Приблизительная высота time scale
const PRICE_SCALE_RESERVED = 0; // Если нужно учитывать

/**
 * Создаёт график с правильным расчётом высоты
 */
function createChartWithCorrectHeight(
  container: HTMLElement,
  desiredChartAreaHeight: number
): IChartApi {
  const totalHeight = desiredChartAreaHeight + TIME_SCALE_HEIGHT;
  
  // Устанавливаем высоту контейнера
  container.style.height = `${totalHeight}px`;
  container.style.overflow = 'visible'; // Важно!
  
  return createChart(container, {
    height: totalHeight,
    timeScale: {
      visible: true,
    },
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;

// Хотим chart area 270px, time scale добавит ~30px
const chart = createChartWithCorrectHeight(container, 270);

const series = chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Правильный расчёт
- Нет обрезания
- Понятная логика

**Минусы:**
- Нужно знать примерную высоту time scale

---

### Решение 2: autoSize с правильным контейнером

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Автоматическое решение

```typescript
import { createChart } from 'lightweight-charts';

const container = document.getElementById('chart')!;

// Убеждаемся что контейнер не обрезает содержимое
container.style.cssText = `
  width: 100%;
  height: 400px;
  overflow: visible; /* Критически важно! */
`;

const chart = createChart(container, {
  autoSize: true, // Автоматически подстраивается под контейнер
  timeScale: {
    visible: true,
    borderVisible: true,
  },
});

// autoSize учитывает все элементы включая time scale
```

**Плюсы:**
- Автоматический размер
- Responsive
- Нет ручных расчётов

**Минусы:**
- Контейнер должен иметь правильные стили

---

### Решение 3: CSS flex layout

**Оценка:** ⭐⭐⭐⭐ (4/5) - Современный подход

```html
<div id="chart-wrapper" style="display: flex; flex-direction: column; height: 400px;">
  <div id="chart" style="flex: 1; min-height: 0;"></div>
</div>
```

```typescript
import { createChart } from 'lightweight-charts';

const chartWrapper = document.getElementById('chart-wrapper')!;
const chartContainer = document.getElementById('chart')!;

const chart = createChart(chartContainer, {
  autoSize: true,
  timeScale: {
    visible: true,
  },
});

// Flex layout правильно распределяет пространство
// Time scale гарантированно видна
```

**Плюсы:**
- Гибкий layout
- Работает с разными размерами
- Современный CSS

**Минусы:**
- Требует flex контейнер

---

### Решение 4: ResizeObserver с корректировкой

**Оценка:** ⭐⭐⭐⭐ (4/5) - Динамическое решение

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

class TimeScaleVisibilityManager {
  private _chart: IChartApi;
  private _container: HTMLElement;
  private _timeScaleHeight = 30;
  private _resizeObserver: ResizeObserver;
  
  constructor(container: HTMLElement) {
    this._container = container;
    
    // Создаём chart
    this._chart = createChart(container, {
      timeScale: {
        visible: true,
      },
    });
    
    // Следим за размерами
    this._resizeObserver = new ResizeObserver((entries) => {
      for (const entry of entries) {
        this._adjustSize(entry.contentRect);
      }
    });
    
    this._resizeObserver.observe(container);
    this._adjustSize(container.getBoundingClientRect());
  }
  
  private _adjustSize(rect: DOMRectReadOnly): void {
    const containerHeight = rect.height;
    
    // Проверяем достаточно ли места для time scale
    if (containerHeight < this._timeScaleHeight + 100) {
      console.warn('Container too small for time scale');
      // Можно скрыть time scale или уменьшить chart
      this._chart.timeScale().applyOptions({ visible: false });
    } else {
      this._chart.timeScale().applyOptions({ visible: true });
      this._chart.resize(rect.width, containerHeight);
    }
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
const manager = new TimeScaleVisibilityManager(container);

const series = manager.chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Динамическая адаптация
- Обрабатывает edge cases
- Предупреждения о проблемах

**Минусы:**
- Дополнительная логика
- Может скрывать time scale

---

### Решение 5: CSS Grid layout

**Оценка:** ⭐⭐⭐⭐ (4/5) - Точный контроль

```html
<div id="chart-grid" style="
  display: grid;
  grid-template-rows: 1fr 30px;
  height: 400px;
">
  <div id="chart-area"></div>
  <div id="time-scale-placeholder"></div>
</div>
```

```typescript
import { createChart } from 'lightweight-charts';

const gridContainer = document.getElementById('chart-grid')!;
const chartArea = document.getElementById('chart-area')!;

// Рассчитываем высоту
const totalHeight = gridContainer.clientHeight;
const chartHeight = totalHeight; // Grid уже учёл структуру

const chart = createChart(chartArea, {
  width: chartArea.clientWidth,
  height: chartHeight,
  timeScale: {
    visible: true,
  },
});

// Grid гарантирует место для time scale
```

**Плюсы:**
- Точный контроль размеров
- Grid layout
- Явное резервирование места

**Минусы:**
- Более сложная HTML структура

---

### Решение 6: Проверка и исправление overflow

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Диагностика и фикс

```typescript
import { createChart, IChartApi } from 'lightweight-charts';

/**
 * Проверяет и исправляет проблемы с видимостью time scale
 */
function ensureTimeScaleVisible(container: HTMLElement): void {
  // Проверяем overflow у всех родителей
  let element: HTMLElement | null = container;
  
  while (element) {
    const style = getComputedStyle(element);
    
    if (style.overflow === 'hidden' || style.overflowY === 'hidden') {
      console.warn(`Element with overflow:hidden found:`, element);
      
      // Вариант 1: Изменить на visible
      // element.style.overflow = 'visible';
      
      // Вариант 2: Увеличить высоту
      const currentHeight = element.clientHeight;
      const neededHeight = currentHeight + 30; // +time scale
      element.style.minHeight = `${neededHeight}px`;
    }
    
    element = element.parentElement;
  }
}

/**
 * Создаёт chart с гарантированно видимой time scale
 */
function createSafeChart(container: HTMLElement, height: number): IChartApi {
  // Исправляем потенциальные проблемы
  ensureTimeScaleVisible(container);
  
  // Устанавливаем безопасные стили
  container.style.cssText = `
    position: relative;
    width: 100%;
    height: ${height}px;
    overflow: visible;
  `;
  
  return createChart(container, {
    height,
    autoSize: false,
    timeScale: {
      visible: true,
      borderVisible: true,
    },
  });
}

// ==================== ИСПОЛЬЗОВАНИЕ ====================

const container = document.getElementById('chart')!;
const chart = createSafeChart(container, 400);

const series = chart.addLineSeries();
series.setData(data);
```

**Плюсы:**
- Автоматическая диагностика
- Исправление проблем
- Логирование warnings

**Минусы:**
- Изменяет стили родителей
- Может повлиять на layout

## ✅ Рекомендуемое решение

Для большинства случаев используйте **Решение 2** (autoSize с правильным контейнером):

```typescript
// Минимальный рабочий пример
const container = document.getElementById('chart')!;
container.style.cssText = `
  width: 100%;
  height: 400px;
  overflow: visible;
`;

const chart = createChart(container, {
  autoSize: true,
  timeScale: { visible: true },
});
```

## 📊 Чеклист для отладки

1. ✅ Проверьте `overflow` на контейнере и родителях
2. ✅ Убедитесь что высота контейнера достаточна (chart + 30px)
3. ✅ Используйте `autoSize: true` для автоматического размера
4. ✅ Проверьте CSS не обрезает график
5. ✅ Используйте `overflow: visible` на контейнере

## 📊 Сравнительная таблица решений

| Решение | Простота | Надёжность | Автоматизация | Рекомендация |
|---------|----------|------------|---------------|--------------|
| **#1 Расчёт высоты** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Ручной контроль |
| **#2 autoSize** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ Универсальное |
| **#3 Flex layout** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Современный CSS |
| **#4 ResizeObserver** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Динамический |
| **#5 Grid layout** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Точный контроль |
| **#6 Диагностика** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Debug |

## 🔗 Источники

- [GitHub Issue #978](https://github.com/tradingview/lightweight-charts/issues/978) - Time scale visibility
- [autoSize Option](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ChartOptions#autosize)
- [Chart Sizing](https://tradingview.github.io/lightweight-charts/tutorials/how_to/set-chart-size)

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
