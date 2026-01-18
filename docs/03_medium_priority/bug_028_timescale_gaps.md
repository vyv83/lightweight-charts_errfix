# БАГ #28: Неверные значения шкалы времени при gaps

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#585](https://github.com/tradingview/lightweight-charts/issues/585)  
> **Связанные Issues:** [#2032](https://github.com/tradingview/lightweight-charts/issues/2032), [#699](https://github.com/tradingview/lightweight-charts/issues/699), [#700](https://github.com/tradingview/lightweight-charts/issues/700)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (По дизайну - библиотека не знает периодичность данных)  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы

Lightweight Charts не поддерживает **линейную временну́ю шкалу** по умолчанию. Библиотека размещает все точки данных с **равным интервалом** (bar spacing), независимо от реальной разницы во времени между ними. Это приводит к:

1. **Сжатию gaps**: Выходные дни, нерабочие часы и пропуски данных не отображаются как пустое пространство
2. **Искажению восприятия**: Движение цены за 1 минуту занимает столько же места, сколько движение за 1 день
3. **Некорректным меткам**: Подписи на временно́й оси могут показывать неверные интервалы

### Почему это происходит

Библиотека **не знает** ожидаемую периодичность ваших данных. Она просто отображает полученные точки с одинаковым расстоянием между ними:

```javascript
// Данные с пропуском (weekend)
const data = [
  { time: '2024-01-19', value: 100 },  // Пятница
  { time: '2024-01-22', value: 102 },  // Понедельник (3 дня спустя!)
];

// График покажет эти свечи РЯДОМ, без видимого gap
// На оси будет: "19 янв" -> "22 янв" (без промежутка)
```

### Сценарии возникновения

1. **Фондовый рынок**: Выходные и праздники создают gaps
2. **Отсутствие торгов**: Низколиквидные активы с пропусками
3. **Внутридневные данные**: Нерабочие часы биржи
4. **Нерегулярные данные**: API возвращает только точки с изменениями

### Визуальное сравнение

```
Реальная временна́я шкала:
|Mon|Tue|Wed|Thu|Fri|   |   |Mon|Tue|
                   ^^^ Выходные (пустое место)

Как показывает Lightweight Charts:
|Mon|Tue|Wed|Thu|Fri|Mon|Tue|
                ^^^ Выходных нет, сжато
```

---

## 🔍 Найденные решения

### Решение 1: Whitespace Data Points

**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

**Описание:** Добавление "пустых" точек данных (только с `time`, без `value`) для заполнения gaps.

**Преимущества:**
- Официально поддерживаемый подход
- Полный контроль над gaps
- Работает для всех типов серий
- Создаёт корректную временну́ю шкалу

**Недостатки:**
- Увеличивает размер данных
- Требует предварительной обработки
- Нужно знать ожидаемые интервалы

```javascript
/**
 * Генерирует whitespace данные для заполнения gaps
 * @param {number} startTime - Начальный timestamp (секунды)
 * @param {number} endTime - Конечный timestamp (секунды)
 * @param {number} intervalSeconds - Интервал между точками
 * @returns {Array} Массив whitespace точек
 */
function generateWhitespaceData(startTime, endTime, intervalSeconds) {
  const whitespace = [];
  for (let time = startTime; time <= endTime; time += intervalSeconds) {
    whitespace.push({ time });  // Только time, без value!
  }
  return whitespace;
}

// Использование
const whitespace = generateWhitespaceData(
  1705622400,  // Start: 2024-01-19 00:00:00 UTC
  1705881600,  // End: 2024-01-22 00:00:00 UTC
  86400        // Interval: 1 day
);

// Добавляем whitespace серию
const whitespaceSeries = chart.addLineSeries({
  lastValueVisible: false,
  priceLineVisible: false,
  crosshairMarkerVisible: false,
});
whitespaceSeries.setData(whitespace);

// Основные данные
const mainSeries = chart.addCandlestickSeries();
mainSeries.setData(actualData);
```

---

### Решение 2: Объединение данных с whitespace

**Оценка: 10/10** ⭐⭐ ЛУЧШЕЕ РЕШЕНИЕ

**Описание:** Interleve (чередование) реальных данных с whitespace точками в одном датасете.

**Преимущества:**
- Один датасет
- Оптимальная производительность
- Точный контроль над шкалой

**Недостатки:**
- Более сложная логика предобработки

```javascript
/**
 * Заполняет gaps в данных whitespace точками
 */
function fillDataGaps(data, intervalSeconds) {
  if (!data.length) return [];
  
  const result = [];
  
  for (let i = 0; i < data.length; i++) {
    const current = data[i];
    result.push(current);
    
    if (i < data.length - 1) {
      const next = data[i + 1];
      const gap = next.time - current.time;
      
      // Если gap больше ожидаемого интервала - заполняем
      if (gap > intervalSeconds * 1.5) {
        const numGaps = Math.floor(gap / intervalSeconds) - 1;
        for (let j = 1; j <= numGaps; j++) {
          result.push({
            time: current.time + (j * intervalSeconds),
            // Нет value - это whitespace!
          });
        }
      }
    }
  }
  
  return result;
}

// Использование
const interval = 60 * 60;  // 1 hour
const filledData = fillDataGaps(rawData, interval);
series.setData(filledData);
```

---

### Решение 3: Trading Hours Engine (исключение выходных)

**Оценка: 9/10**

**Описание:** Система для генерации whitespace с учётом торговых сессий.

**Преимущества:**
- Учитывает реальные торговые часы
- Поддержка праздников
- Переиспользуемый код

**Недостатки:**
- Требуется настройка под каждый рынок
- Сложнее в реализации

```javascript
class TradingHoursEngine {
  constructor(options = {}) {
    this.tradingDays = options.tradingDays || [1, 2, 3, 4, 5]; // Mon-Fri
    this.tradingHours = options.tradingHours || { start: 9.5, end: 16 }; // 9:30-16:00
    this.holidays = new Set(options.holidays || []);
    this.timezone = options.timezone || 'America/New_York';
  }

  /**
   * Проверяет, является ли timestamp торговым временем
   */
  isTradingTime(timestamp) {
    const date = new Date(timestamp * 1000);
    const dateStr = date.toISOString().split('T')[0];
    
    // Проверяем праздники
    if (this.holidays.has(dateStr)) return false;
    
    // Проверяем день недели
    const dayOfWeek = date.getDay();
    if (!this.tradingDays.includes(dayOfWeek)) return false;
    
    // Проверяем часы (упрощённо, без timezone)
    const hours = date.getHours() + date.getMinutes() / 60;
    if (hours < this.tradingHours.start || hours >= this.tradingHours.end) {
      return false;
    }
    
    return true;
  }

  /**
   * Генерирует полную временну́ю шкалу с whitespace
   */
  generateTimeline(startTime, endTime, intervalSeconds) {
    const timeline = [];
    
    for (let time = startTime; time <= endTime; time += intervalSeconds) {
      if (this.isTradingTime(time)) {
        timeline.push({ time, isTradingTime: true });
      } else {
        // Whitespace для нерабочего времени
        timeline.push({ time });
      }
    }
    
    return timeline;
  }

  /**
   * Объединяет торговую timeline с реальными данными
   */
  mergeWithData(timeline, data) {
    const dataMap = new Map(data.map(d => [d.time, d]));
    
    return timeline.map(point => {
      if (dataMap.has(point.time)) {
        return dataMap.get(point.time);
      }
      return { time: point.time };  // Whitespace
    });
  }
}

// Использование
const engine = new TradingHoursEngine({
  tradingDays: [1, 2, 3, 4, 5],
  tradingHours: { start: 9.5, end: 16 },
  holidays: ['2024-01-15', '2024-02-19'],
  timezone: 'America/New_York',
});

const timeline = engine.generateTimeline(startTime, endTime, 60 * 60);
const mergedData = engine.mergeWithData(timeline, rawData);
series.setData(mergedData);
```

---

### Решение 4: Uniform Distribution (отключение)

**Оценка: 4/10**

**Описание:** Использование опции `uniformDistribution: false` (если доступна в будущих версиях).

**Преимущества:**
- Простая конфигурация (когда будет доступна)

**Недостатки:**
- НЕ реализовано в текущей версии!
- Обсуждается как Feature Request (#2032)

```javascript
// ВНИМАНИЕ: Это НЕ работает в текущих версиях!
// Показано как потенциальное будущее решение

const chart = createChart(container, {
  timeScale: {
    // Потенциальная будущая опция
    // uniformDistribution: false,
    // fixedTimeScale: true,
  },
});
```

---

### Решение 5: Custom Time Scale (продвинутое)

**Оценка: 7/10**

**Описание:** Создание кастомной логики отображения времени через plugins/primitives.

**Преимущества:**
- Полный контроль
- Возможность любой логики

**Недостатки:**
- Сложная реализация
- Требует глубокого понимания библиотеки

---

## ✅ Рекомендуемое решение

### Полный код реализации

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// Утилиты для работы с gaps
// ============================================

/**
 * Определяет интервал данных автоматически
 */
function detectInterval(data) {
  if (data.length < 2) return 86400; // Default: 1 day
  
  const intervals = [];
  for (let i = 1; i < Math.min(data.length, 20); i++) {
    intervals.push(data[i].time - data[i - 1].time);
  }
  
  // Возвращаем минимальный интервал (наиболее частый)
  intervals.sort((a, b) => a - b);
  return intervals[0];
}

/**
 * Класс для управления временно́й шкалой с gaps
 */
class GapAwareTimeScale {
  constructor(options = {}) {
    this.interval = options.interval || null; // Auto-detect if null
    this.maxGapMultiplier = options.maxGapMultiplier || 2;
    this.showWeekends = options.showWeekends ?? true;
    this.tradingHours = options.tradingHours || null; // { start: 9.5, end: 16 }
  }

  /**
   * Обрабатывает данные для корректного отображения gaps
   */
  processData(data) {
    if (!data.length) return [];
    
    // Автоопределение интервала
    const interval = this.interval || detectInterval(data);
    
    const result = [];
    
    for (let i = 0; i < data.length; i++) {
      // Добавляем реальную точку
      result.push(data[i]);
      
      // Проверяем gap до следующей точки
      if (i < data.length - 1) {
        const current = data[i];
        const next = data[i + 1];
        const gap = next.time - current.time;
        
        // Если gap больше допустимого - заполняем whitespace
        if (gap > interval * this.maxGapMultiplier) {
          const numPoints = Math.floor(gap / interval) - 1;
          
          for (let j = 1; j <= numPoints; j++) {
            const whitespaceTime = current.time + (j * interval);
            
            // Опционально фильтруем выходные
            if (!this.showWeekends && this.isWeekend(whitespaceTime)) {
              continue;
            }
            
            // Опционально фильтруем нерабочие часы
            if (this.tradingHours && !this.isTradingHour(whitespaceTime)) {
              continue;
            }
            
            result.push({ time: whitespaceTime });
          }
        }
      }
    }
    
    // Сортируем по времени
    result.sort((a, b) => a.time - b.time);
    
    return result;
  }

  /**
   * Проверяет, является ли timestamp выходными
   */
  isWeekend(timestamp) {
    const date = new Date(timestamp * 1000);
    const day = date.getUTCDay();
    return day === 0 || day === 6; // Sunday = 0, Saturday = 6
  }

  /**
   * Проверяет торговые часы
   */
  isTradingHour(timestamp) {
    if (!this.tradingHours) return true;
    
    const date = new Date(timestamp * 1000);
    const hour = date.getUTCHours() + date.getUTCMinutes() / 60;
    
    return hour >= this.tradingHours.start && hour < this.tradingHours.end;
  }
}

// ============================================
// Создание графика с поддержкой gaps
// ============================================

function createChartWithGaps(container, options = {}) {
  const timeScaleManager = new GapAwareTimeScale({
    interval: options.interval,
    showWeekends: options.showWeekends ?? true,
    tradingHours: options.tradingHours,
  });

  const chart = createChart(container, {
    width: options.width || 800,
    height: options.height || 400,
    layout: {
      background: { type: 'solid', color: '#1E222D' },
      textColor: '#DDD',
    },
    grid: {
      vertLines: { color: '#2B2B43' },
      horzLines: { color: '#2B2B43' },
    },
    timeScale: {
      timeVisible: true,
      secondsVisible: false,
      borderColor: '#2B2B43',
    },
  });

  // Расширенный API
  return {
    chart,
    timeScaleManager,

    /**
     * Добавляет серию с автоматической обработкой gaps
     */
    addGapAwareSeries(type, seriesOptions = {}) {
      const addMethod = `add${type.charAt(0).toUpperCase() + type.slice(1)}Series`;
      const series = chart[addMethod](seriesOptions);
      
      // Переопределяем setData
      const originalSetData = series.setData.bind(series);
      series.setData = (data) => {
        const processedData = timeScaleManager.processData(data);
        originalSetData(processedData);
      };
      
      return series;
    },

    /**
     * Создаёт отдельную whitespace серию (альтернативный подход)
     */
    createWhitespaceSeries(startTime, endTime, interval) {
      const whitespaceSeries = chart.addLineSeries({
        lastValueVisible: false,
        priceLineVisible: false,
        crosshairMarkerVisible: false,
        lineWidth: 0,
        color: 'transparent',
      });

      const whitespace = [];
      for (let time = startTime; time <= endTime; time += interval) {
        whitespace.push({ time });
      }
      
      whitespaceSeries.setData(whitespace);
      return whitespaceSeries;
    },
  };
}

// ============================================
// Пример использования
// ============================================

// Создание графика
const chartWrapper = createChartWithGaps(document.getElementById('chart'), {
  width: 900,
  height: 500,
  showWeekends: true,  // Показывать gaps для выходных
  interval: 86400,     // 1 день (auto-detect если null)
});

// Добавление серии с поддержкой gaps
const candleSeries = chartWrapper.addGapAwareSeries('candlestick', {
  upColor: '#26a69a',
  downColor: '#ef5350',
});

// Данные с пропуском (пятница -> понедельник)
const data = [
  { time: 1705622400, open: 100, high: 102, low: 99, close: 101 },   // 2024-01-19 Fri
  { time: 1705881600, open: 101, high: 103, low: 100, close: 102 },  // 2024-01-22 Mon
];

// Данные автоматически дополнятся whitespace для выходных
candleSeries.setData(data);
```

### React компонент

```tsx
import React, { useEffect, useRef, useCallback } from 'react';
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

interface GapAwareChartProps {
  data: Array<{ time: number; open: number; high: number; low: number; close: number }>;
  interval?: number;  // Seconds between data points
  showWeekendGaps?: boolean;
  tradingHours?: { start: number; end: number };
}

export const GapAwareChart: React.FC<GapAwareChartProps> = ({
  data,
  interval = 86400,
  showWeekendGaps = true,
  tradingHours,
}) => {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Candlestick'> | null>(null);

  // Обработка данных с gaps
  const processDataWithGaps = useCallback((rawData: typeof data) => {
    if (!rawData.length) return [];
    
    const result: any[] = [];
    
    for (let i = 0; i < rawData.length; i++) {
      result.push(rawData[i]);
      
      if (i < rawData.length - 1) {
        const current = rawData[i];
        const next = rawData[i + 1];
        const gap = next.time - current.time;
        
        if (gap > interval * 2) {
          const numPoints = Math.floor(gap / interval) - 1;
          
          for (let j = 1; j <= numPoints; j++) {
            const whitespaceTime = current.time + (j * interval);
            
            // Фильтр выходных
            if (!showWeekendGaps) {
              const day = new Date(whitespaceTime * 1000).getUTCDay();
              if (day === 0 || day === 6) continue;
            }
            
            // Фильтр торговых часов
            if (tradingHours) {
              const hour = new Date(whitespaceTime * 1000).getUTCHours();
              if (hour < tradingHours.start || hour >= tradingHours.end) continue;
            }
            
            result.push({ time: whitespaceTime });
          }
        }
      }
    }
    
    return result.sort((a, b) => a.time - b.time);
  }, [interval, showWeekendGaps, tradingHours]);

  // Инициализация
  useEffect(() => {
    if (!containerRef.current) return;

    chartRef.current = createChart(containerRef.current, {
      width: containerRef.current.clientWidth,
      height: 400,
      layout: {
        background: { type: 'solid', color: '#1E222D' },
        textColor: '#DDD',
      },
      timeScale: {
        timeVisible: true,
      },
    });

    seriesRef.current = chartRef.current.addCandlestickSeries({
      upColor: '#26a69a',
      downColor: '#ef5350',
    });

    return () => {
      chartRef.current?.remove();
    };
  }, []);

  // Обновление данных
  useEffect(() => {
    if (!seriesRef.current) return;
    
    const processedData = processDataWithGaps(data);
    seriesRef.current.setData(processedData);
    chartRef.current?.timeScale().fitContent();
  }, [data, processDataWithGaps]);

  return <div ref={containerRef} style={{ width: '100%' }} />;
};
```

---

## 📊 Сравнительная таблица решений

| Критерий | Whitespace Series | Merged Data | Trading Hours | uniformDistribution | Custom Scale |
|----------|:-----------------:|:-----------:|:-------------:|:-------------------:|:------------:|
| **Эффективность** | ✅ | ✅ | ✅ | ❌ Не реализовано | ✅ |
| **Сложность** | ⚠️ Средняя | ⚠️ Средняя | 🔴 Высокая | ✅ Простая | 🔴 Высокая |
| **Производительность** | ⚠️ 2 серии | ✅ 1 серия | ✅ | - | ⚠️ |
| **Гибкость** | ✅ | ✅ | ✅ | ❌ | ✅ Полная |
| **Поддержка** | ✅ Официальная | ✅ Официальная | ✅ | ❌ | ⚠️ |
| **Оценка** | **9/10** | **10/10** | **9/10** | **4/10** | **7/10** |

---

## 🎯 Специфические сценарии

### 1. Внутридневные данные биржи NYSE

```javascript
const nyseEngine = new TradingHoursEngine({
  tradingDays: [1, 2, 3, 4, 5],  // Mon-Fri
  tradingHours: { start: 9.5, end: 16 },  // 9:30 AM - 4:00 PM ET
  holidays: [
    '2024-01-01', '2024-01-15', '2024-02-19',
    '2024-03-29', '2024-05-27', '2024-06-19',
    '2024-07-04', '2024-09-02', '2024-11-28',
    '2024-11-29', '2024-12-25',
  ],
});
```

### 2. 24/7 криптовалютные рынки

```javascript
// Для криптовалют gaps обычно не нужны
// Но если данные нерегулярные:
const cryptoProcessor = new GapAwareTimeScale({
  interval: 60,  // 1 минута
  showWeekends: true,  // Крипто торгуется 24/7
  maxGapMultiplier: 5,  // Заполняем только большие gaps
});
```

### 3. Форекс (круглосуточно в рабочие дни)

```javascript
const forexEngine = new TradingHoursEngine({
  tradingDays: [1, 2, 3, 4, 5],  // Без выходных
  tradingHours: null,  // 24 часа
  holidays: [],  // Форекс работает во все праздники
});
```

---

## 🔗 Источники

1. **GitHub Issue #585** - [Time Scale is not working properly](https://github.com/tradingview/lightweight-charts/issues/585)
2. **GitHub Issue #2032** - [Feature Request: fixedTimeScale option](https://github.com/tradingview/lightweight-charts/issues/2032)
3. **GitHub Issue #699** - [Whitespace data documentation](https://github.com/tradingview/lightweight-charts/issues/699)
4. **Lightweight Charts Documentation** - [Whitespace Data](https://tradingview.github.io/lightweight-charts/docs/api#whitespace-data)
5. **Stack Overflow** - [Time scale gaps in lightweight-charts](https://stackoverflow.com/questions/tagged/lightweight-charts+timescale)
6. **d3fc** - [Discontinuous Scale](https://d3fc.io/api/discontinuous-scale-d3fc.html)

---

## 📝 Примечания

- Whitespace данные — это точки, содержащие только `time` без `value`
- При больших объёмах данных whitespace может существенно увеличить датасет
- Для high-frequency данных рассмотрите агрегацию вместо заполнения gaps
- Feature Request #2032 предлагает нативную поддержку - следите за обновлениями

---

*Документ создан: 18 января 2026*  
*Последнее обновление: 18 января 2026*
