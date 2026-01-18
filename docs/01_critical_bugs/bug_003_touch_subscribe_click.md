# БАГ #3: subscribeClick не работает на touch устройствах

> **Критичность:** 🔴 КРИТИЧЕСКАЯ (для mobile apps)  
> **GitHub Issue:** [#1417](https://github.com/tradingview/lightweight-charts/issues/1417)  
> **Версии:** v4.0+  
> **Статус:** 🔴 Open

---

## 📋 Описание проблемы

### Симптомы
- События клика не срабатывают на touch-устройствах
- Touch events не конвертируются в click events
- `subscribeClick` callback никогда не вызывается при tap
- Полная потеря интерактивности на мобильных

### Причина
Библиотека использует "Tracking mode" на мобильных устройствах - кроссхэр активируется по **long press**, а не по tap. Простой tap интерпретируется как scroll, а не click.

### Сценарии воспроизведения
1. Любое использование `chart.subscribeClick()` на мобильных
2. Попытка обработки кликов по маркерам, price lines, примитивам
3. Tap и hold для crosshair

### Частота и платформы
- **Частота:** 100% на touch devices
- **Платформы:** iOS, Android (все браузеры)
- **Community impact:** Критический для mobile-first приложений

---

## 🔍 Найденные решения

### Решение 1: Ручная обработка touch events на контейнере
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Добавить собственные touch event listeners на контейнер графика.

**Плюсы:**
- Полный контроль над touch поведением
- Работает на всех мобильных устройствах
- Не зависит от внутренней реализации библиотеки

**Минусы:**
- Требует дополнительного кода
- Нужно вычислять координаты вручную

```javascript
function setupTouchClickHandler(chartContainer, chart, callback) {
  let touchStartTime = 0;
  let touchStartX = 0;
  let touchStartY = 0;
  const TAP_THRESHOLD = 300; // ms
  const MOVE_THRESHOLD = 10; // px

  chartContainer.addEventListener('touchstart', (e) => {
    if (e.touches.length === 1) {
      touchStartTime = Date.now();
      touchStartX = e.touches[0].clientX;
      touchStartY = e.touches[0].clientY;
    }
  }, { passive: true });

  chartContainer.addEventListener('touchend', (e) => {
    const touchDuration = Date.now() - touchStartTime;
    
    if (touchDuration < TAP_THRESHOLD && e.changedTouches.length === 1) {
      const touch = e.changedTouches[0];
      const moveX = Math.abs(touch.clientX - touchStartX);
      const moveY = Math.abs(touch.clientY - touchStartY);
      
      // Если палец почти не двигался - это tap
      if (moveX < MOVE_THRESHOLD && moveY < MOVE_THRESHOLD) {
        const rect = chartContainer.getBoundingClientRect();
        const x = touch.clientX - rect.left;
        const y = touch.clientY - rect.top;
        
        // Конвертируем координаты в данные графика
        const timeScale = chart.timeScale();
        const time = timeScale.coordinateToTime(x);
        
        // Получаем серию и её координаты
        const series = chart.getSeries()[0]; // первая серия
        if (series && time) {
          const price = series.coordinateToPrice(y);
          
          callback({
            time,
            point: { x, y },
            seriesData: new Map([[series, { time, value: price }]])
          });
        }
      }
    }
  }, { passive: true });
}

// Использование
const container = document.getElementById('chart');
const chart = createChart(container, { /* options */ });

setupTouchClickHandler(container, chart, (params) => {
  console.log('Touch click at:', params.time, params.seriesData);
});
```

---

### Решение 2: Использование subscribeCrosshairMove + long press
**Оценка: 7/10**

**Суть:** Использовать `subscribeCrosshairMove` для отслеживания позиции.

**Плюсы:**
- Использует родной API библиотеки
- Работает стабильно

**Минусы:**
- Требует long press для активации
- Не подходит для простых tap interactions
- Другой UX паттерн

```javascript
// Настройка для track mode
const chart = createChart(container, {
  handleScroll: {
    vertTouchDrag: false,  // Отключает вертикальный drag scroll
  },
  kineticScroll: {
    touch: false,  // Отключает инерционный скролл
  },
});

// Подписка на движение кроссхэра
chart.subscribeCrosshairMove((params) => {
  if (params.time && params.point) {
    console.log('Crosshair at:', params.time);
    // Показываем tooltip или другую UI
  }
});
```

---

### Решение 3: Hammer.js для gesture recognition
**Оценка: 8/10**

**Суть:** Использовать стороннюю библиотеку для gesture recognition.

**Плюсы:**
- Профессиональная обработка жестов
- Поддержка tap, double tap, swipe и др.
- Хорошо документировано

**Минусы:**
- Дополнительная зависимость (~7KB gzipped)
- Может конфликтовать с внутренним touch handling

```bash
npm install hammerjs
npm install @types/hammerjs  # для TypeScript
```

```javascript
import Hammer from 'hammerjs';

function setupHammerTouchHandler(chartContainer, chart, callback) {
  const hammer = new Hammer(chartContainer);
  
  // Настройка recognizers
  hammer.get('tap').set({ time: 300 });
  
  hammer.on('tap', (event) => {
    const rect = chartContainer.getBoundingClientRect();
    const x = event.center.x - rect.left;
    const y = event.center.y - rect.top;
    
    const timeScale = chart.timeScale();
    const time = timeScale.coordinateToTime(x);
    
    if (time) {
      callback({
        time,
        point: { x, y },
        originalEvent: event
      });
    }
  });
  
  return hammer; // Сохранить для cleanup
}

// Использование
const hammer = setupHammerTouchHandler(container, chart, (params) => {
  console.log('Hammer tap at:', params.time);
});

// Cleanup
// hammer.destroy();
```

---

### Решение 4: Pointer Events API (универсальный подход)
**Оценка: 8/10**

**Суть:** Использовать Pointer Events, которые объединяют mouse и touch.

**Плюсы:**
- Работает на mouse, touch и pen
- Современный стандарт W3C
- Один код для всех устройств

**Минусы:**
- Требует полифилл для старых браузеров
- Нужна аккуратная обработка

```javascript
function setupUniversalClickHandler(chartContainer, chart, callback) {
  let pointerDownTime = 0;
  let pointerDownPos = null;
  
  chartContainer.addEventListener('pointerdown', (e) => {
    if (e.isPrimary) {  // Только primary pointer
      pointerDownTime = Date.now();
      pointerDownPos = { x: e.clientX, y: e.clientY };
    }
  });

  chartContainer.addEventListener('pointerup', (e) => {
    if (!e.isPrimary || !pointerDownPos) return;
    
    const duration = Date.now() - pointerDownTime;
    const moveX = Math.abs(e.clientX - pointerDownPos.x);
    const moveY = Math.abs(e.clientY - pointerDownPos.y);
    
    // Tap detection: короткое нажатие без движения
    if (duration < 300 && moveX < 10 && moveY < 10) {
      const rect = chartContainer.getBoundingClientRect();
      const x = e.clientX - rect.left;
      const y = e.clientY - rect.top;
      
      const time = chart.timeScale().coordinateToTime(x);
      if (time) {
        callback({ time, point: { x, y }, pointerType: e.pointerType });
      }
    }
    
    pointerDownPos = null;
  });
}
```

---

## ✅ Рекомендуемое комплексное решение

Комбинация решения #1 + #4 с fallback на subscribeClick для desktop:

```javascript
class UniversalChartClickHandler {
  constructor(chartContainer, chart) {
    this.container = chartContainer;
    this.chart = chart;
    this.callbacks = [];
    this.pointerState = null;
    
    this.TAP_DURATION = 300;
    this.MOVE_THRESHOLD = 10;
    
    this.setupEventListeners();
  }
  
  setupEventListeners() {
    // Pointer Events для универсальности
    this.container.addEventListener('pointerdown', this.onPointerDown.bind(this));
    this.container.addEventListener('pointerup', this.onPointerUp.bind(this));
    this.container.addEventListener('pointercancel', this.onPointerCancel.bind(this));
    
    // Fallback через subscribeClick для desktop
    this.chart.subscribeClick(this.onChartClick.bind(this));
  }
  
  onPointerDown(e) {
    if (e.isPrimary && e.pointerType === 'touch') {
      this.pointerState = {
        time: Date.now(),
        x: e.clientX,
        y: e.clientY
      };
    }
  }
  
  onPointerUp(e) {
    if (!e.isPrimary || e.pointerType !== 'touch' || !this.pointerState) return;
    
    const duration = Date.now() - this.pointerState.time;
    const moveX = Math.abs(e.clientX - this.pointerState.x);
    const moveY = Math.abs(e.clientY - this.pointerState.y);
    
    if (duration < this.TAP_DURATION && 
        moveX < this.MOVE_THRESHOLD && 
        moveY < this.MOVE_THRESHOLD) {
      this.handleTap(e.clientX, e.clientY, 'touch');
    }
    
    this.pointerState = null;
  }
  
  onPointerCancel() {
    this.pointerState = null;
  }
  
  onChartClick(params) {
    // Это сработает для mouse clicks
    this.callbacks.forEach(cb => cb(params));
  }
  
  handleTap(clientX, clientY, pointerType) {
    const rect = this.container.getBoundingClientRect();
    const x = clientX - rect.left;
    const y = clientY - rect.top;
    
    const timeScale = this.chart.timeScale();
    const time = timeScale.coordinateToTime(x);
    
    if (!time) return;
    
    const params = {
      time,
      point: { x, y },
      pointerType,
      logical: timeScale.coordinateToLogical(x)
    };
    
    this.callbacks.forEach(cb => cb(params));
  }
  
  subscribe(callback) {
    this.callbacks.push(callback);
    return () => {
      this.callbacks = this.callbacks.filter(cb => cb !== callback);
    };
  }
  
  destroy() {
    this.container.removeEventListener('pointerdown', this.onPointerDown);
    this.container.removeEventListener('pointerup', this.onPointerUp);
    this.container.removeEventListener('pointercancel', this.onPointerCancel);
    this.callbacks = [];
  }
}

// === Использование ===
const container = document.getElementById('chart');
const chart = createChart(container, { /* ... */ });

const clickHandler = new UniversalChartClickHandler(container, chart);

const unsubscribe = clickHandler.subscribe((params) => {
  console.log(`Click at time: ${params.time}, type: ${params.pointerType}`);
});

// При unmount
// clickHandler.destroy();
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Зависимости | Touch | Mouse | Сложность |
|---------|--------|-------------|-------|-------|-----------|
| #1 Touch events | 9/10 | Нет | ✅ | ❌ | Средняя |
| #2 subscribeCrosshairMove | 7/10 | Нет | ✅* | ✅ | Низкая |
| #3 Hammer.js | 8/10 | hammerjs | ✅ | ✅ | Низкая |
| #4 Pointer Events | 8/10 | Нет | ✅ | ✅ | Средняя |

*требует long press

---

## 🔗 Источники

1. [GitHub Issue #1417 - subscribeClick not working on mobile](https://github.com/tradingview/lightweight-charts/issues/1417)
2. [GitHub Issue #138 - subscribeTouch request](https://github.com/tradingview/lightweight-charts/issues/138)
3. [MDN - Touch Events](https://developer.mozilla.org/en-US/docs/Web/API/Touch_events)
4. [MDN - Pointer Events](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events)
5. [Hammer.js Documentation](https://hammerjs.github.io/)

---

**Документ создан:** 18 января 2026  
**Последнее обновление:** 18 января 2026
