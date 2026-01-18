# БАГ #4: Pinch-to-zoom не работает на iOS Safari

> **Критичность:** 🔴 КРИТИЧЕСКАЯ (для iOS users)  
> **GitHub Issues:** [#94](https://github.com/tradingview/lightweight-charts/issues/94), [#95](https://github.com/tradingview/lightweight-charts/issues/95)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (известен с 2019 года)

---

## 📋 Описание проблемы

### Симптомы
- Жест pinch-to-zoom не функционирует в Safari на iOS
- Вместо зума графика масштабируется **вся страница**
- Pinch прерывается при long tap (режим crosshair tracking)

### Причина
iOS Safari по умолчанию перехватывает pinch gesture для зума страницы. Начиная с iOS 10, отключение масштабирования через viewport meta-тег игнорируется по accessibility причинам.

### Сценарии воспроизведения
1. Любая попытка zoom жестом на iPhone/iPad в Safari
2. Pinch на canvas элементе

### Частота и платформы
- **Частота:** 100% на iOS Safari
- **Платформы:** iOS Safari (Chrome на iOS тоже использует WebKit)
- **Community impact:** Критический (~50% mobile traffic это iOS)

---

## 🔍 Найденные решения

### Решение 1: preventDefault на touchmove
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Перехватывать touch events и вызывать `preventDefault()` для multi-touch.

**Плюсы:**
- Работает на iOS 10+
- Предотвращает системный зум страницы
- Позволяет custom zoom logic

**Минусы:**
- Требует `passive: false` (может влиять на scroll performance)
- Нужна реализация custom pinch zoom

```javascript
function preventPageZoomOnChart(chartContainer) {
  // Предотвращаем зум страницы при pinch на графике
  chartContainer.addEventListener('touchmove', (e) => {
    // Если два пальца - это pinch, предотвращаем
    if (e.touches.length >= 2) {
      e.preventDefault();
    }
  }, { passive: false });

  // Предотвращаем double-tap zoom
  let lastTouchEnd = 0;
  chartContainer.addEventListener('touchend', (e) => {
    const now = Date.now();
    if (now - lastTouchEnd <= 300) {
      e.preventDefault();
    }
    lastTouchEnd = now;
  }, { passive: false });
}

// Использование
const container = document.getElementById('chart');
preventPageZoomOnChart(container);
```

---

### Решение 2: CSS touch-action
**Оценка: 8/10**

**Суть:** Использовать CSS `touch-action` для контроля touch behavior.

**Плюсы:**
- Простое CSS-only решение
- Не требует JavaScript

**Минусы:**
- Может полностью отключить некоторые жесты
- Не работает в старых версиях Safari

```css
/* Контейнер графика */
.chart-container {
  touch-action: pan-x pan-y;  /* Разрешаем scroll, запрещаем zoom */
}

/* Или более агрессивно */
.chart-container {
  touch-action: none;  /* Полностью отключаем browser touch actions */
}
```

```html
<div id="chart" class="chart-container" style="touch-action: pan-x pan-y;"></div>
```

---

### Решение 3: Полная реализация custom pinch zoom
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ для продвинутого контроля

**Суть:** Полностью кастомная обработка pinch gesture с zoom графика.

**Плюсы:**
- Полный контроль над zoom behavior
- Работает одинаково на всех платформах
- Можно настроить чувствительность

**Минусы:**
- Больше кода
- Нужно тестирование

```javascript
class ChartPinchZoom {
  constructor(chartContainer, chart) {
    this.container = chartContainer;
    this.chart = chart;
    this.initialPinchDistance = null;
    this.initialScale = null;
    
    this.setupEventListeners();
  }
  
  getDistance(touch1, touch2) {
    const dx = touch1.clientX - touch2.clientX;
    const dy = touch1.clientY - touch2.clientY;
    return Math.sqrt(dx * dx + dy * dy);
  }
  
  getCenter(touch1, touch2) {
    return {
      x: (touch1.clientX + touch2.clientX) / 2,
      y: (touch1.clientY + touch2.clientY) / 2
    };
  }
  
  setupEventListeners() {
    this.container.addEventListener('touchstart', this.onTouchStart.bind(this), { passive: false });
    this.container.addEventListener('touchmove', this.onTouchMove.bind(this), { passive: false });
    this.container.addEventListener('touchend', this.onTouchEnd.bind(this), { passive: true });
  }
  
  onTouchStart(e) {
    if (e.touches.length === 2) {
      e.preventDefault();
      
      this.initialPinchDistance = this.getDistance(e.touches[0], e.touches[1]);
      
      // Сохраняем текущий visible range как начальную точку
      const timeScale = this.chart.timeScale();
      const visibleRange = timeScale.getVisibleLogicalRange();
      if (visibleRange) {
        this.initialScale = visibleRange.to - visibleRange.from;
      }
    }
  }
  
  onTouchMove(e) {
    if (e.touches.length === 2 && this.initialPinchDistance) {
      e.preventDefault();
      
      const currentDistance = this.getDistance(e.touches[0], e.touches[1]);
      const scale = currentDistance / this.initialPinchDistance;
      const center = this.getCenter(e.touches[0], e.touches[1]);
      
      // Вычисляем новый visible range
      const rect = this.container.getBoundingClientRect();
      const centerX = center.x - rect.left;
      
      const timeScale = this.chart.timeScale();
      const currentRange = timeScale.getVisibleLogicalRange();
      
      if (currentRange && this.initialScale) {
        const rangeMid = (currentRange.from + currentRange.to) / 2;
        const newHalfRange = (this.initialScale / scale) / 2;
        
        timeScale.setVisibleLogicalRange({
          from: rangeMid - newHalfRange,
          to: rangeMid + newHalfRange
        });
      }
    }
  }
  
  onTouchEnd(e) {
    if (e.touches.length < 2) {
      this.initialPinchDistance = null;
      this.initialScale = null;
    }
  }
  
  destroy() {
    this.container.removeEventListener('touchstart', this.onTouchStart);
    this.container.removeEventListener('touchmove', this.onTouchMove);
    this.container.removeEventListener('touchend', this.onTouchEnd);
  }
}

// Использование
const container = document.getElementById('chart');
const chart = createChart(container, { /* options */ });
const pinchZoom = new ChartPinchZoom(container, chart);
```

---

### Решение 4: Viewport meta (ограниченно работает)
**Оценка: 4/10**

**Суть:** Использовать viewport meta-тег.

**Плюсы:**
- Просто добавить

**Минусы:**
- **НЕ РАБОТАЕТ на iOS 10+** из-за accessibility requirements
- Не рекомендуется как основное решение

```html
<!-- Не рекомендуется - не работает на iOS 10+ -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
```

---

### Решение 5: Обновление библиотеки
**Оценка: 7/10**

**Суть:** Issue #95 был исправлен в версии 4.1.x.

**Плюсы:**
- Официальное исправление
- Не требует кастомного кода

**Минусы:**
- Issue #94 всё ещё может проявляться
- Требует обновления dependencies

```bash
npm update lightweight-charts@latest
```

---

## ✅ Рекомендуемое комплексное решение

Комбинация CSS + JavaScript + обновление:

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    .chart-wrapper {
      width: 100%;
      height: 400px;
      position: relative;
      touch-action: pan-x pan-y; /* Разрешить scroll, запретить zoom */
    }
  </style>
</head>
<body>
  <div id="chart" class="chart-wrapper"></div>
  
  <script type="module">
    import { createChart } from 'lightweight-charts';
    
    const container = document.getElementById('chart');
    const chart = createChart(container, {
      width: container.clientWidth,
      height: container.clientHeight,
    });
    
    // Добавляем серию
    const series = chart.addCandlestickSeries();
    series.setData([/* data */]);
    
    // === iOS Safari Pinch Zoom Fix ===
    
    // Предотвращаем системный zoom
    container.addEventListener('touchmove', (e) => {
      if (e.touches.length >= 2) {
        e.preventDefault();
      }
    }, { passive: false });
    
    // Предотвращаем double-tap zoom
    let lastTouchEnd = 0;
    container.addEventListener('touchend', (e) => {
      const now = Date.now();
      if (now - lastTouchEnd <= 300) {
        e.preventDefault();
      }
      lastTouchEnd = now;
    }, { passive: false });
    
    // Custom pinch zoom (опционально для более точного контроля)
    let initialDistance = null;
    let initialRange = null;
    
    container.addEventListener('touchstart', (e) => {
      if (e.touches.length === 2) {
        e.preventDefault();
        const dx = e.touches[0].clientX - e.touches[1].clientX;
        const dy = e.touches[0].clientY - e.touches[1].clientY;
        initialDistance = Math.sqrt(dx * dx + dy * dy);
        initialRange = chart.timeScale().getVisibleLogicalRange();
      }
    }, { passive: false });
    
    container.addEventListener('touchmove', (e) => {
      if (e.touches.length === 2 && initialDistance && initialRange) {
        e.preventDefault();
        
        const dx = e.touches[0].clientX - e.touches[1].clientX;
        const dy = e.touches[0].clientY - e.touches[1].clientY;
        const currentDistance = Math.sqrt(dx * dx + dy * dy);
        const scale = currentDistance / initialDistance;
        
        const rangeMid = (initialRange.from + initialRange.to) / 2;
        const initialHalfRange = (initialRange.to - initialRange.from) / 2;
        const newHalfRange = initialHalfRange / scale;
        
        chart.timeScale().setVisibleLogicalRange({
          from: rangeMid - newHalfRange,
          to: rangeMid + newHalfRange
        });
      }
    }, { passive: false });
    
    container.addEventListener('touchend', () => {
      initialDistance = null;
      initialRange = null;
    });
  </script>
</body>
</html>
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | iOS 10+ | Кастомизация |
|---------|--------|-----------|---------|--------------|
| #1 preventDefault | 9/10 | Средняя | ✅ | Высокая |
| #2 CSS touch-action | 8/10 | Низкая | ✅ | Низкая |
| #3 Custom pinch zoom | 9/10 | Высокая | ✅ | Максимальная |
| #4 Viewport meta | 4/10 | Низкая | ❌ | Нет |
| #5 Update library | 7/10 | Низкая | Частично | Нет |

---

## 🔗 Источники

1. [GitHub Issue #94 - Pinch doesn't work on iOS](https://github.com/tradingview/lightweight-charts/issues/94)
2. [GitHub Issue #95 - Pinch isn't prevented by long tap](https://github.com/tradingview/lightweight-charts/issues/95)
3. [MDN - Touch Events](https://developer.mozilla.org/en-US/docs/Web/API/Touch_events)
4. [CSS touch-action](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action)
5. [Lightweight Charts Release Notes](https://tradingview.github.io/lightweight-charts/docs/release-notes)

---

**Документ создан:** 18 января 2026  
**Последнее обновление:** 18 января 2026
