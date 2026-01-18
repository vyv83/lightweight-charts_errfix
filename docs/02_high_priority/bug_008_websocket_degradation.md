# БАГ #8: Деградация производительности при длительной работе с WebSocket

> **Критичность:** 🟠 ВЫСОКАЯ  
> **GitHub Issues:** [#338](https://github.com/tradingview/lightweight-charts/issues/338), [#946](https://github.com/tradingview/lightweight-charts/issues/946)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (частично улучшено, но не решено полностью)

---

## 📋 Описание проблемы

### Симптомы
- После нескольких часов непрерывной работы график становится "медленным" и неотзывчивым
- Постепенное увеличение потребления CPU (может достигать 100%)
- Заметные задержки при взаимодействии с графиком (scroll, zoom)
- Накопительный эффект - чем дольше работает приложение, тем хуже производительность
- Особенно заметно при high-frequency updates (>10 updates/sec)

### Причина
1. **Накопление данных в памяти:** При частых вызовах `series.update()` данные накапливаются в внутренних буферах библиотеки
2. **CPU overhead в TimeAxisWidget:** Issue #946 выявил, что процесс отрисовки временной оси (painting) постепенно потребляет всё больше ресурсов
3. **Отсутствие автоматического data pruning:** Библиотека не удаляет старые данные автоматически
4. **requestAnimationFrame накопление:** При очень частых updates может накапливаться очередь перерисовок

### Сценарии воспроизведения
```javascript
// Типичный проблемный сценарий
const ws = new WebSocket('wss://your-data-feed.com');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // Вызывается 10-100 раз в секунду
  series.update({
    time: data.timestamp,
    open: data.open,
    high: data.high,
    low: data.low,
    close: data.close
  });
};

// После 4-6 часов непрерывной работы производительность критически падает
```

### Частота и платформы
- **Частота:** 100% после 4-6 часов непрерывной работы при частых updates
- **Платформы:** Все браузеры
- **Критично для:** Real-time trading applications, live crypto price feeds, long-running dashboards

---

## 🔍 Найденные решения

### Решение 1: Throttling/Debouncing обновлений
**Оценка: 8/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Ограничить частоту обновлений графика до разумного уровня (например, 1-4 раза в секунду), накапливая промежуточные данные.

**Плюсы:**
- Радикально снижает нагрузку на CPU
- Простая реализация
- Не требует изменений в библиотеке
- Человеческий глаз всё равно не способен воспринять >30 FPS

**Минусы:**
- Незначительная задержка отображения (50-250ms)
- Требует буферизации входящих данных

```javascript
// ===== ПОЛНОЕ РЕШЕНИЕ С THROTTLING =====

class ThrottledChartUpdater {
  constructor(series, options = {}) {
    this.series = series;
    this.intervalMs = options.intervalMs || 100; // 10 updates/sec max
    this.pendingUpdate = null;
    this.lastUpdateTime = 0;
    this.timeoutId = null;
  }

  /**
   * Обновляет график с throttling
   * @param {Object} data - Данные для обновления {time, open, high, low, close, value}
   */
  update(data) {
    const now = Date.now();
    const timeSinceLastUpdate = now - this.lastUpdateTime;

    // Если оригинальная свеча с тем же timestamp - объединяем OHLC
    if (this.pendingUpdate && this.pendingUpdate.time === data.time) {
      this.pendingUpdate = {
        time: data.time,
        open: this.pendingUpdate.open, // Сохраняем оригинальный open
        high: Math.max(this.pendingUpdate.high, data.high),
        low: Math.min(this.pendingUpdate.low, data.low),
        close: data.close, // Последний close
        // Для volume - суммируем
        ...(data.volume !== undefined && { 
          volume: (this.pendingUpdate.volume || 0) + data.volume 
        })
      };
    } else {
      // Новый timestamp - заменяем pending
      this.pendingUpdate = { ...data };
    }

    // Если прошло достаточно времени - обновляем сразу
    if (timeSinceLastUpdate >= this.intervalMs) {
      this._flushUpdate();
    } else if (!this.timeoutId) {
      // Иначе планируем отложенное обновление
      this.timeoutId = setTimeout(() => {
        this._flushUpdate();
      }, this.intervalMs - timeSinceLastUpdate);
    }
  }

  _flushUpdate() {
    if (this.pendingUpdate) {
      this.series.update(this.pendingUpdate);
      this.pendingUpdate = null;
      this.lastUpdateTime = Date.now();
    }
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
      this.timeoutId = null;
    }
  }

  // Немедленная отправка (при закрытии, смене инструмента и т.д.)
  flush() {
    this._flushUpdate();
  }

  destroy() {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }
  }
}

// ===== ИСПОЛЬЗОВАНИЕ =====
const chart = createChart(container);
const candlestickSeries = chart.addCandlestickSeries();

// Создаём throttled updater (обновления не чаще 4 раз в секунду)
const updater = new ThrottledChartUpdater(candlestickSeries, {
  intervalMs: 250 // 4 updates/sec
});

// WebSocket handler
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updater.update({
    time: data.timestamp,
    open: data.open,
    high: data.high,
    low: data.low,
    close: data.close
  });
};

// При закрытии
ws.onclose = () => {
  updater.flush();
  updater.destroy();
};
```

---

### Решение 2: Data Pruning (обрезка старых данных)
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Периодически удалять старые данные, оставляя только видимую область + буфер.

**Плюсы:**
- Предотвращает рост потребления памяти
- Стабильная производительность независимо от времени работы
- Можно хранить старые данные отдельно для ленивой загрузки

**Минусы:**
- Использует `setData()` который перерисовывает весь график
- Требует отдельного хранения исторических данных
- Сложнее реализовать корректно

```javascript
// ===== ПОЛНОЕ РЕШЕНИЕ С DATA PRUNING =====

class ChartDataManager {
  constructor(series, options = {}) {
    this.series = series;
    this.maxDataPoints = options.maxDataPoints || 10000; // Максимум точек
    this.pruneThreshold = options.pruneThreshold || 12000; // Когда запускать обрезку
    this.pruneInterval = options.pruneInterval || 60000; // Проверять каждую минуту
    this.data = [];
    this.historicalData = []; // Для хранения удалённых данных (опционально)
    this.keepHistorical = options.keepHistorical || false;
    
    this._startPruningInterval();
  }

  /**
   * Добавляет или обновляет последнюю точку
   */
  update(point) {
    const lastPoint = this.data[this.data.length - 1];
    
    if (lastPoint && lastPoint.time === point.time) {
      // Обновляем существующую точку
      this.data[this.data.length - 1] = point;
    } else {
      // Добавляем новую точку
      this.data.push(point);
    }
    
    this.series.update(point);
  }

  /**
   * Устанавливает начальные данные
   */
  setData(data) {
    this.data = [...data];
    this.series.setData(data);
  }

  /**
   * Обрезает старые данные если необходимо
   */
  prune(force = false) {
    if (!force && this.data.length < this.pruneThreshold) {
      return false;
    }

    const excessPoints = this.data.length - this.maxDataPoints;
    if (excessPoints <= 0) return false;

    // Сохраняем исторические данные если нужно
    if (this.keepHistorical) {
      const removedData = this.data.slice(0, excessPoints);
      this.historicalData = [...this.historicalData, ...removedData];
    }

    // Обрезаем данные
    this.data = this.data.slice(excessPoints);
    
    // Используем setData для применения изменений
    // ВАЖНО: Это вызовет полный redraw
    this.series.setData(this.data);
    
    console.log(`[ChartDataManager] Pruned ${excessPoints} points. Current: ${this.data.length}`);
    
    return true;
  }

  _startPruningInterval() {
    this.pruneIntervalId = setInterval(() => {
      this.prune();
    }, this.pruneInterval);
  }

  /**
   * Загрузка исторических данных (для scroll влево)
   */
  async loadHistoricalData(fromTime, toTime) {
    // Сначала проверяем локальный кэш
    const cachedData = this.historicalData.filter(
      p => p.time >= fromTime && p.time <= toTime
    );
    
    if (cachedData.length > 0) {
      this.data = [...cachedData, ...this.data];
      this.series.setData(this.data);
      return cachedData;
    }
    
    // Иначе загружаем с сервера (реализация зависит от вашего API)
    // const serverData = await this.fetchHistoricalData(fromTime, toTime);
    // this.data = [...serverData, ...this.data];
    // this.series.setData(this.data);
    
    return [];
  }

  destroy() {
    if (this.pruneIntervalId) {
      clearInterval(this.pruneIntervalId);
    }
  }

  // Статистика
  getStats() {
    return {
      currentPoints: this.data.length,
      historicalPoints: this.historicalData.length,
      memoryEstimate: `${((this.data.length + this.historicalData.length) * 50 / 1024).toFixed(2)} KB`
    };
  }
}

// ===== ИСПОЛЬЗОВАНИЕ =====
const chart = createChart(container);
const candlestickSeries = chart.addCandlestickSeries();

const dataManager = new ChartDataManager(candlestickSeries, {
  maxDataPoints: 5000,         // Максимум 5000 точек на графике
  pruneThreshold: 6000,        // Обрезать когда больше 6000
  pruneInterval: 30000,        // Проверять каждые 30 секунд
  keepHistorical: true         // Сохранять удалённые данные для lazy load
});

// Установка начальных данных
dataManager.setData(initialData);

// WebSocket updates
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  dataManager.update({
    time: data.timestamp,
    open: data.open,
    high: data.high,
    low: data.low,
    close: data.close
  });
};

// Логирование статистики каждые 5 минут
setInterval(() => {
  console.log('[Stats]', dataManager.getStats());
}, 300000);
```

---

### Решение 3: Data Conflation (версия 5.1.0+)
**Оценка: 7/10**

**Суть:** Использование встроенной функции conflation, которая автоматически объединяет точки данных при zoom out.

**Плюсы:**
- Официальное решение от разработчиков
- Автоматическая оптимизация при zoom out
- Не требует ручного управления данными

**Минусы:**
- Доступно только в версии 5.1.0+
- Не решает проблему накопления данных в памяти
- Помогает только при больших datasets, не при частых updates

```javascript
// ===== ВКЛЮЧЕНИЕ DATA CONFLATION =====

const chart = createChart(container, {
  // Включаем conflation для больших datasets
  enableConflation: true,
  
  // Фактор порога: при каком уровне zoom включается conflation
  // Чем меньше значение - тем раньше срабатывает
  conflationThresholdFactor: 2,
  
  // Предварительный расчёт conflation при инициализации
  // Улучшает скорость первого zoom, но замедляет загрузку
  precomputeConflationOnInit: true,
  
  // Приоритет предварительного расчёта
  // 'immediate' | 'idle' (использует requestIdleCallback)
  precomputeConflationPriority: 'idle',
});

const candlestickSeries = chart.addCandlestickSeries();

// Conflation работает автоматически для больших datasets (10k+ точек)
candlestickSeries.setData(largeDataset);
```

---

### Решение 4: Batch Updates с requestAnimationFrame
**Оценка: 7/10**

**Суть:** Группировать все обновления в один frame, используя requestAnimationFrame.

**Плюсы:**
- Синхронизация с браузерным rendering cycle
- Предотвращает избыточные repaints
- Стандартный паттерн для анимаций

**Минусы:**
- Сложнее реализовать для нескольких серий
- Может вызвать задержку до 16ms
- Не решает проблему накопления данных

```javascript
// ===== BATCH UPDATES С RAF =====

class BatchedChartUpdater {
  constructor() {
    this.pendingUpdates = new Map(); // Map<series, data>
    this.rafId = null;
    this.isScheduled = false;
  }

  /**
   * Планирует обновление серии
   * @param {ISeriesApi} series - Серия для обновления
   * @param {Object} data - Данные обновления
   */
  scheduleUpdate(series, data) {
    // Сохраняем последнее обновление для каждой серии
    const existing = this.pendingUpdates.get(series);
    
    if (existing && existing.time === data.time) {
      // Объединяем OHLC для того же timestamp
      this.pendingUpdates.set(series, {
        time: data.time,
        open: existing.open,
        high: Math.max(existing.high, data.high),
        low: Math.min(existing.low, data.low),
        close: data.close
      });
    } else {
      this.pendingUpdates.set(series, data);
    }

    // Планируем отрисовку если ещё не запланирована
    if (!this.isScheduled) {
      this.isScheduled = true;
      this.rafId = requestAnimationFrame(() => this._flush());
    }
  }

  _flush() {
    // Применяем все обновления
    for (const [series, data] of this.pendingUpdates) {
      series.update(data);
    }
    
    // Очищаем
    this.pendingUpdates.clear();
    this.isScheduled = false;
  }

  destroy() {
    if (this.rafId) {
      cancelAnimationFrame(this.rafId);
    }
    this.pendingUpdates.clear();
  }
}

// ===== ИСПОЛЬЗОВАНИЕ =====
const batcher = new BatchedChartUpdater();

// Множественные серии
const priceSeries = chart.addCandlestickSeries();
const volumeSeries = chart.addHistogramSeries();

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  
  // Все обновления группируются в один frame
  batcher.scheduleUpdate(priceSeries, {
    time: data.timestamp,
    open: data.open,
    high: data.high,
    low: data.low,
    close: data.close
  });
  
  batcher.scheduleUpdate(volumeSeries, {
    time: data.timestamp,
    value: data.volume,
    color: data.close >= data.open ? '#26a69a' : '#ef5350'
  });
};
```

---

### Решение 5: Периодический restart графика
**Оценка: 4/10**

**Суть:** Полностью пересоздавать график через определённые интервалы времени.

**Плюсы:**
- Гарантированно решает проблему утечек
- Простая реализация
- Не требует глубокого понимания библиотеки

**Минусы:**
- Видимое "мерцание" при пересоздании
- Потеря состояния (zoom level, scroll position)
- "Костыльное" решение

```javascript
// ===== ПЕРИОДИЧЕСКИЙ RESTART (НЕ РЕКОМЕНДУЕТСЯ для production) =====

class ChartWithAutoRestart {
  constructor(container, options = {}) {
    this.container = container;
    this.chartOptions = options.chartOptions || {};
    this.seriesOptions = options.seriesOptions || {};
    this.restartInterval = options.restartInterval || 4 * 60 * 60 * 1000; // 4 часа
    
    this.chart = null;
    this.series = null;
    this.data = [];
    
    this._createChart();
    this._startRestartTimer();
  }

  _createChart() {
    // Сохраняем текущее состояние если есть
    let visibleRange = null;
    if (this.chart) {
      try {
        visibleRange = this.chart.timeScale().getVisibleRange();
      } catch (e) {}
      this.chart.remove();
    }

    // Создаём новый график
    this.chart = createChart(this.container, this.chartOptions);
    this.series = this.chart.addCandlestickSeries(this.seriesOptions);
    
    // Восстанавливаем данные
    if (this.data.length > 0) {
      this.series.setData(this.data);
    }
    
    // Восстанавливаем видимый диапазон
    if (visibleRange) {
      this.chart.timeScale().setVisibleRange(visibleRange);
    }
    
    console.log('[ChartWithAutoRestart] Chart recreated');
  }

  _startRestartTimer() {
    setInterval(() => {
      console.log('[ChartWithAutoRestart] Scheduled restart...');
      this._createChart();
    }, this.restartInterval);
  }

  update(point) {
    const lastPoint = this.data[this.data.length - 1];
    if (lastPoint && lastPoint.time === point.time) {
      this.data[this.data.length - 1] = point;
    } else {
      this.data.push(point);
    }
    this.series.update(point);
  }

  setData(data) {
    this.data = [...data];
    this.series.setData(data);
  }

  destroy() {
    if (this.chart) {
      this.chart.remove();
    }
  }
}

// ===== ИСПОЛЬЗОВАНИЕ =====
const chartManager = new ChartWithAutoRestart(container, {
  chartOptions: { /* ... */ },
  seriesOptions: { /* ... */ },
  restartInterval: 2 * 60 * 60 * 1000 // Рестарт каждые 2 часа
});
```

---

## ✅ Рекомендуемое решение

### Комбинированный подход: Throttling + Data Pruning

Для production-ready решения рекомендуется комбинировать **Решение 1 (Throttling)** и **Решение 2 (Data Pruning)**:

```javascript
// ===== PRODUCTION-READY РЕШЕНИЕ =====

class OptimizedChartManager {
  constructor(series, options = {}) {
    this.series = series;
    
    // Throttling настройки
    this.throttleMs = options.throttleMs || 100; // 10 updates/sec max
    this.pendingUpdate = null;
    this.lastUpdateTime = 0;
    this.throttleTimeoutId = null;
    
    // Data pruning настройки
    this.maxDataPoints = options.maxDataPoints || 10000;
    this.pruneThreshold = options.pruneThreshold || 12000;
    this.pruneIntervalMs = options.pruneIntervalMs || 60000;
    
    // Данные
    this.data = [];
    
    this._startPruningInterval();
  }

  // ===== THROTTLED UPDATE =====
  update(point) {
    const now = Date.now();
    const timeSinceLastUpdate = now - this.lastUpdateTime;

    // Объединяем данные если тот же timestamp
    if (this.pendingUpdate && this.pendingUpdate.time === point.time) {
      this.pendingUpdate = {
        time: point.time,
        open: this.pendingUpdate.open,
        high: Math.max(this.pendingUpdate.high || point.high, point.high),
        low: Math.min(this.pendingUpdate.low || point.low, point.low),
        close: point.close,
        ...(point.volume !== undefined && { 
          volume: (this.pendingUpdate.volume || 0) + (point.volume || 0)
        })
      };
    } else {
      this.pendingUpdate = { ...point };
    }

    if (timeSinceLastUpdate >= this.throttleMs) {
      this._flushUpdate();
    } else if (!this.throttleTimeoutId) {
      this.throttleTimeoutId = setTimeout(
        () => this._flushUpdate(),
        this.throttleMs - timeSinceLastUpdate
      );
    }
  }

  _flushUpdate() {
    if (this.pendingUpdate) {
      // Обновляем внутренний массив
      const lastPoint = this.data[this.data.length - 1];
      if (lastPoint && lastPoint.time === this.pendingUpdate.time) {
        this.data[this.data.length - 1] = this.pendingUpdate;
      } else {
        this.data.push(this.pendingUpdate);
      }
      
      // Обновляем график
      this.series.update(this.pendingUpdate);
      
      this.pendingUpdate = null;
      this.lastUpdateTime = Date.now();
    }
    
    if (this.throttleTimeoutId) {
      clearTimeout(this.throttleTimeoutId);
      this.throttleTimeoutId = null;
    }
  }

  // ===== DATA PRUNING =====
  _startPruningInterval() {
    this.pruneIntervalId = setInterval(() => {
      this._prune();
    }, this.pruneIntervalMs);
  }

  _prune() {
    if (this.data.length < this.pruneThreshold) return;
    
    const excessPoints = this.data.length - this.maxDataPoints;
    if (excessPoints <= 0) return;
    
    this.data = this.data.slice(excessPoints);
    this.series.setData(this.data);
    
    console.log(`[OptimizedChartManager] Pruned ${excessPoints} points`);
  }

  // ===== PUBLIC API =====
  setData(data) {
    this.data = [...data];
    this.series.setData(data);
  }

  flush() {
    this._flushUpdate();
  }

  getStats() {
    return {
      dataPoints: this.data.length,
      maxDataPoints: this.maxDataPoints,
      throttleMs: this.throttleMs
    };
  }

  destroy() {
    if (this.throttleTimeoutId) clearTimeout(this.throttleTimeoutId);
    if (this.pruneIntervalId) clearInterval(this.pruneIntervalId);
  }
}

// ===== ПОЛНЫЙ ПРИМЕР ИСПОЛЬЗОВАНИЯ =====

import { createChart } from 'lightweight-charts';

// Создание графика
const container = document.getElementById('chart');
const chart = createChart(container, {
  width: 800,
  height: 400,
  layout: {
    background: { color: '#1e1e1e' },
    textColor: '#d1d4dc',
  },
  grid: {
    vertLines: { color: '#2B2B43' },
    horzLines: { color: '#2B2B43' },
  },
});

const candlestickSeries = chart.addCandlestickSeries({
  upColor: '#26a69a',
  downColor: '#ef5350',
  borderVisible: false,
  wickUpColor: '#26a69a',
  wickDownColor: '#ef5350',
});

// Создаём оптимизированный менеджер
const chartManager = new OptimizedChartManager(candlestickSeries, {
  throttleMs: 100,        // Не чаще 10 updates/sec
  maxDataPoints: 5000,    // Хранить максимум 5000 точек
  pruneThreshold: 6000,   // Обрезать при 6000+ точках
  pruneIntervalMs: 30000  // Проверять каждые 30 сек
});

// Загрузка исторических данных
async function loadInitialData() {
  const response = await fetch('/api/historical-data');
  const data = await response.json();
  chartManager.setData(data);
}

// WebSocket обработчик
function connectWebSocket() {
  const ws = new WebSocket('wss://your-data-feed.com');
  
  ws.onmessage = (event) => {
    const tick = JSON.parse(event.data);
    chartManager.update({
      time: tick.timestamp,
      open: tick.open,
      high: tick.high,
      low: tick.low,
      close: tick.close
    });
  };
  
  ws.onclose = () => {
    chartManager.flush();
    // Реконнект логика
    setTimeout(connectWebSocket, 1000);
  };
  
  return ws;
}

// Запуск
loadInitialData().then(() => {
  const ws = connectWebSocket();
  
  // Логирование статистики
  setInterval(() => {
    console.log('[Chart Stats]', chartManager.getStats());
  }, 60000);
  
  // Cleanup при выходе
  window.addEventListener('beforeunload', () => {
    chartManager.destroy();
    ws.close();
  });
});
```

### Почему именно эта комбинация?

1. **Throttling** решает проблему избыточных обновлений, снижая нагрузку на CPU
2. **Data Pruning** решает проблему накопления данных в памяти
3. **Вместе** они обеспечивают стабильную работу на протяжении дней/недель
4. **Версия 5.1.0+** добавляет Data Conflation как бонус для zoom out

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Надёжность | Рекомендация |
|---------|--------|-----------|------------|--------------|
| #1 Throttling | 8/10 | Низкая | Высокая | ⭐ Обязательно |
| #2 Data Pruning | 9/10 | Средняя | Высокая | ⭐ Обязательно |
| #3 Data Conflation | 7/10 | Низкая | Средняя | Дополнительно |
| #4 Batch Updates (RAF) | 7/10 | Средняя | Средняя | Альтернатива #1 |
| #5 Periodic Restart | 4/10 | Низкая | Низкая | Не рекомендуется |

---

## 🔗 Источники

1. [GitHub Issue #338: Real-time updating grows sluggish over time](https://github.com/tradingview/lightweight-charts/issues/338)
2. [GitHub Issue #946: Painting slows down after a while with tons of updates](https://github.com/tradingview/lightweight-charts/issues/946)
3. [Lightweight Charts Series API - update() documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#update)
4. [Mozilla MDN: Canvas Optimization](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)
5. [Throttle vs Debounce - JavaScript Performance Patterns](https://css-tricks.com/debouncing-throttling-explained-examples/)
6. [Lightweight Charts v5.1.0 Release Notes - Data Conflation](https://github.com/tradingview/lightweight-charts/releases/tag/v5.1.0)

---

**Документ создан:** 2026-01-18  
**Автор:** AI Assistant  
**Версия:** 1.0
