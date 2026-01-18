# БАГ #27: Проблемы с таймзонами и DST transitions

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1189](https://github.com/tradingview/lightweight-charts/issues/1189)  
> **Связанные Issues:** [#1426](https://github.com/tradingview/lightweight-charts/issues/1426), [#699](https://github.com/tradingview/lightweight-charts/issues/699)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (Архитектурное ограничение - "By Design")  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы

Lightweight Charts **не поддерживает таймзоны нативно** - все временны́е значения обрабатываются внутри библиотеки как UTC. Это приводит к следующим проблемам:

1. **DST переходы (Daylight Saving Time)**: При конвертации timestamps для отображения локального времени могут возникать:
   - **Дублирующиеся метки** - когда часы переводятся назад (осенью), один час повторяется
   - **Пропущенные часы** - когда часы переводятся вперёд (весной), один час "исчезает"
   - **Ошибки о дубликатах** - библиотека требует уникальные временные метки

2. **Некорректное отображение времени**: Метки на временно́й оси показывают UTC вместо локального времени

3. **Смещение данных**: Данные отображаются с неправильным смещением относительно ожидаемого времени

### Почему это проблема

1. **Финансовые рынки**: Трейдеры работают в локальных таймзонах (NYSE - EST, LSE - GMT, и т.д.)
2. **Международные пользователи**: Приложение должно показывать время в таймзоне пользователя
3. **Исторические данные**: DST правила менялись со временем, что усложняет обработку
4. **Ошибки при переходах**: График может "ломаться" два раза в год при DST переходах

### Сценарии возникновения

```javascript
// Проблема: данные в UTC, пользователь в Europe/Moscow (UTC+3)
// Свеча за 2024-03-31T01:00:00Z (переход на летнее время)

const data = [
  { time: 1711843200, open: 100, high: 101, low: 99, close: 100.5 }, // 01:00 UTC
  { time: 1711846800, open: 100.5, high: 102, low: 100, close: 101 }, // 02:00 UTC
  // При конвертации в локальное время:
  // 01:00 UTC -> 04:00 MSK
  // 02:00 UTC -> 05:00 MSK
  // Но в день перехода на летнее время часы прыгают с 02:00 на 03:00!
];

// При попытке "подкрутить" время для локального отображения:
const localizedData = data.map(bar => ({
  ...bar,
  time: bar.time + (3 * 3600), // Добавляем 3 часа для MSK
}));
// Это работает для большинства дней, но НЕ работает при DST переходах
```

### Частота проблемы

- **Постоянная** для пользователей не в UTC таймзоне
- **Критическая** два раза в год при DST переходах
- Затрагивает ~70% пользователей (большинство не в UTC)

---

## 🔍 Найденные решения

### Решение 1: Модификация timestamps с учётом таймзоны через date-fns-tz

**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

**Описание:** Конвертация каждого timestamp в целевую таймзону с помощью библиотеки `date-fns-tz`, которая корректно обрабатывает DST.

**Преимущества:**
- Корректная обработка DST переходов
- Использует IANA Time Zone Database
- Маленький размер библиотеки
- Современный API

**Недостатки:**
- Дополнительная зависимость (~5KB gzipped)
- Требует обработки данных перед установкой в график

```javascript
import { toZonedTime, getTimezoneOffset } from 'date-fns-tz';

/**
 * Конвертирует UTC timestamp в "локальный" timestamp для отображения
 * в указанной таймзоне. Note: результат всё ещё Unix timestamp,
 * но его значение смещено для корректного отображения в lightweight-charts
 */
function convertToDisplayTimestamp(utcTimestamp, timeZone) {
  // Получаем offset для данной таймзоны в этот конкретный момент времени
  const date = new Date(utcTimestamp * 1000);
  const offset = getTimezoneOffset(timeZone, date);
  
  // Добавляем offset (в миллисекундах -> секунды)
  return utcTimestamp + (offset / 1000);
}

// Использование
const timeZone = 'America/New_York';
const convertedData = originalData.map(bar => ({
  ...bar,
  time: convertToDisplayTimestamp(bar.time, timeZone),
}));
```

---

### Решение 2: Pure JavaScript с Intl.DateTimeFormat

**Оценка: 8/10**

**Описание:** Нативное решение без внешних зависимостей, использующее Intl API.

**Преимущества:**
- Нет внешних зависимостей
- Работает во всех современных браузерах
- Нативная поддержка IANA таймзон

**Недостатки:**
- Более сложная реализация
- Нужно кэширование для производительности
- Ограниченная точность в edge cases

```javascript
/**
 * Получает offset для таймзоны в конкретный момент времени
 */
function getTimezoneOffsetFor(date, timeZone) {
  // Создаём форматтеры для UTC и целевой таймзоны
  const utcFormatter = new Intl.DateTimeFormat('en-US', {
    timeZone: 'UTC',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  });
  
  const tzFormatter = new Intl.DateTimeFormat('en-US', {
    timeZone: timeZone,
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  });
  
  // Парсим компоненты
  const utcParts = parseFormattedDate(utcFormatter.format(date));
  const tzParts = parseFormattedDate(tzFormatter.format(date));
  
  // Создаём Date объекты и получаем разницу
  const utcDate = new Date(Date.UTC(
    utcParts.year, utcParts.month - 1, utcParts.day,
    utcParts.hour, utcParts.minute, utcParts.second
  ));
  const tzDate = new Date(Date.UTC(
    tzParts.year, tzParts.month - 1, tzParts.day,
    tzParts.hour, tzParts.minute, tzParts.second
  ));
  
  return tzDate.getTime() - utcDate.getTime();
}

function parseFormattedDate(formatted) {
  const [datePart, timePart] = formatted.split(', ');
  const [month, day, year] = datePart.split('/').map(Number);
  const [hour, minute, second] = timePart.split(':').map(Number);
  return { year, month, day, hour, minute, second };
}

// Оптимизированная версия с кэшированием
const offsetCache = new Map();

function getCachedOffset(timestamp, timeZone) {
  // Кэшируем по дням (offset меняется редко)
  const dayKey = `${timeZone}-${Math.floor(timestamp / 86400)}`;
  
  if (!offsetCache.has(dayKey)) {
    const date = new Date(timestamp * 1000);
    const offset = getTimezoneOffsetFor(date, timeZone);
    offsetCache.set(dayKey, offset);
    
    // Ограничиваем размер кэша
    if (offsetCache.size > 1000) {
      const firstKey = offsetCache.keys().next().value;
      offsetCache.delete(firstKey);
    }
  }
  
  return offsetCache.get(dayKey);
}
```

---

### Решение 3: Luxon для комплексной работы с датами

**Оценка: 8/10**

**Описание:** Использование библиотеки Luxon (современная замена moment.js) для полноценной работы с таймзонами.

**Преимущества:**
- Мощный API для работы с датами
- Отлично обрабатывает DST
- Хорошая документация

**Недостатки:**
- Больший размер (~23KB gzipped)
- Может быть излишним только для этой задачи

```javascript
import { DateTime } from 'luxon';

/**
 * Конвертирует данные для отображения в указанной таймзоне
 */
function convertDataToTimezone(data, targetZone = 'local') {
  return data.map(bar => {
    // Создаём DateTime из UTC timestamp
    const dt = DateTime.fromSeconds(bar.time, { zone: 'UTC' });
    
    // Конвертируем в целевую таймзону
    const localDt = targetZone === 'local' 
      ? dt.toLocal() 
      : dt.setZone(targetZone);
    
    // Получаем "искусственный" UTC timestamp для отображения
    // Это timestamp, который при отображении как UTC покажет локальное время
    const displayTime = localDt.toUTC().toSeconds() + localDt.offset * 60;
    
    return {
      ...bar,
      time: displayTime,
    };
  });
}

// Использование
const localizedData = convertDataToTimezone(data, 'Europe/Moscow');
```

---

### Решение 4: Custom timeFormatter для отображения

**Оценка: 6/10**

**Описание:** Использование `timeFormatter` и `tickMarkFormatter` для форматирования меток в нужной таймзоне, не меняя сами данные.

**Преимущества:**
- Не модифицирует исходные данные
- Простая реализация
- Работает для отображения crosshair и меток

**Недостатки:**
- НЕ влияет на генерацию tick marks (метки могут быть в "некрасивых" логических временах)
- Данные всё ещё в UTC внутри
- Проблемы с tick weight algorithm

```javascript
const chart = createChart(container, {
  localization: {
    // Форматирование времени в crosshair
    timeFormatter: (timestamp) => {
      const date = new Date(timestamp * 1000);
      return date.toLocaleTimeString('ru-RU', {
        timeZone: 'Europe/Moscow',
        hour: '2-digit',
        minute: '2-digit',
      });
    },
    // Форматирование даты
    dateFormat: 'dd.MM.yyyy',
  },
  timeScale: {
    // Форматирование меток на оси
    tickMarkFormatter: (timestamp, tickMarkType, locale) => {
      const date = new Date(timestamp * 1000);
      if (tickMarkType === 0) { // Year
        return date.toLocaleDateString('ru-RU', { 
          year: 'numeric',
          timeZone: 'Europe/Moscow',
        });
      }
      if (tickMarkType === 1) { // Month
        return date.toLocaleDateString('ru-RU', { 
          month: 'short',
          timeZone: 'Europe/Moscow',
        });
      }
      // ... continue for other tick types
      return date.toLocaleString('ru-RU', { timeZone: 'Europe/Moscow' });
    },
  },
});
```

---

### Решение 5: Комбинированный подход (модификация данных + форматирование)

**Оценка: 10/10** ⭐⭐ ЛУЧШЕЕ РЕШЕНИЕ

**Описание:** Модификация timestamps для корректного отображения + custom formatters для согласованности.

**Преимущества:**
- Tick marks генерируются в "красивых" локальных временах
- Полная согласованность отображения
- Корректная обработка DST
- Production-ready решение

---

## ✅ Рекомендуемое решение

### Полный код реализации

```javascript
import { createChart } from 'lightweight-charts';
import { getTimezoneOffset } from 'date-fns-tz';

// ============================================
// Модуль работы с таймзонами
// ============================================

/**
 * Класс для работы с таймзонами в Lightweight Charts
 */
class ChartTimezoneManager {
  constructor(targetTimezone = 'UTC') {
    this.timezone = targetTimezone;
    this.offsetCache = new Map();
  }

  /**
   * Получает offset для конкретного timestamp
   * @param {number} utcTimestamp - Unix timestamp в секундах
   * @returns {number} Offset в секундах
   */
  getOffset(utcTimestamp) {
    // Кэшируем по дате (offset обычно не меняется в течение дня)
    const dayKey = Math.floor(utcTimestamp / 86400);
    
    if (!this.offsetCache.has(dayKey)) {
      const date = new Date(utcTimestamp * 1000);
      // date-fns-tz возвращает offset в миллисекундах
      const offsetMs = getTimezoneOffset(this.timezone, date);
      this.offsetCache.set(dayKey, offsetMs / 1000);
    }
    
    return this.offsetCache.get(dayKey);
  }

  /**
   * Конвертирует UTC timestamp в "display" timestamp
   */
  toDisplayTime(utcTimestamp) {
    return utcTimestamp + this.getOffset(utcTimestamp);
  }

  /**
   * Конвертирует "display" timestamp обратно в UTC
   */
  toUtcTime(displayTimestamp) {
    // Приблизительное решение (для большинства случаев)
    return displayTimestamp - this.getOffset(displayTimestamp);
  }

  /**
   * Конвертирует массив данных
   */
  convertData(data) {
    return data.map(bar => ({
      ...bar,
      time: this.toDisplayTime(bar.time),
    }));
  }

  /**
   * Проверяет наличие DST перехода в диапазоне данных
   */
  hasDstTransition(data) {
    if (data.length < 2) return false;
    
    const offsets = data.map(bar => this.getOffset(bar.time));
    return new Set(offsets).size > 1;
  }

  /**
   * Обновляет таймзону
   */
  setTimezone(newTimezone) {
    this.timezone = newTimezone;
    this.offsetCache.clear();
  }

  /**
   * Создаёт timeFormatter для использования в chart options
   */
  createTimeFormatter() {
    return (timestamp) => {
      // timestamp уже "сконвертирован", нужно отформатировать как UTC
      const date = new Date(timestamp * 1000);
      return date.toLocaleTimeString('en-US', {
        timeZone: 'UTC', // Потому что мы уже модифицировали время
        hour: '2-digit',
        minute: '2-digit',
        hour12: false,
      });
    };
  }

  /**
   * Создаёт tickMarkFormatter
   */
  createTickMarkFormatter() {
    return (timestamp, tickMarkType, locale) => {
      const date = new Date(timestamp * 1000);
      
      const formatOptions = {
        timeZone: 'UTC',
      };

      switch (tickMarkType) {
        case 0: // Year
          return date.toLocaleDateString(locale, { 
            ...formatOptions, 
            year: 'numeric' 
          });
        case 1: // Month
          return date.toLocaleDateString(locale, { 
            ...formatOptions, 
            month: 'short' 
          });
        case 2: // Day of Month
          return date.toLocaleDateString(locale, { 
            ...formatOptions, 
            day: 'numeric' 
          });
        case 3: // Time
          return date.toLocaleTimeString(locale, {
            ...formatOptions,
            hour: '2-digit',
            minute: '2-digit',
            hour12: false,
          });
        default:
          return date.toLocaleDateString(locale, formatOptions);
      }
    };
  }
}

// ============================================
// Создание графика с поддержкой таймзон
// ============================================

function createTimezoneAwareChart(container, timezone = 'America/New_York') {
  const tzManager = new ChartTimezoneManager(timezone);

  const chart = createChart(container, {
    width: 800,
    height: 400,
    layout: {
      background: { type: 'solid', color: '#1E222D' },
      textColor: '#DDD',
    },
    grid: {
      vertLines: { color: '#2B2B43' },
      horzLines: { color: '#2B2B43' },
    },
    localization: {
      locale: 'en-US',
      timeFormatter: tzManager.createTimeFormatter(),
    },
    timeScale: {
      tickMarkFormatter: tzManager.createTickMarkFormatter(),
      timeVisible: true,
      secondsVisible: false,
    },
  });

  // Расширяем API графика
  return {
    chart,
    tzManager,
    
    /**
     * Добавляет серию с автоматической конвертацией данных
     */
    addTimezoneSeries(type, options = {}) {
      const addMethod = `add${type.charAt(0).toUpperCase() + type.slice(1)}Series`;
      const series = chart[addMethod](options);
      
      // Переопределяем setData
      const originalSetData = series.setData.bind(series);
      series.setData = (data) => {
        const convertedData = tzManager.convertData(data);
        originalSetData(convertedData);
      };
      
      // Переопределяем update
      const originalUpdate = series.update.bind(series);
      series.update = (bar) => {
        const convertedBar = {
          ...bar,
          time: tzManager.toDisplayTime(bar.time),
        };
        originalUpdate(convertedBar);
      };
      
      return series;
    },

    /**
     * Изменяет таймзону и обновляет все данные
     */
    setTimezone(newTimezone, seriesDataMap) {
      tzManager.setTimezone(newTimezone);
      
      // Перезагружаем данные для каждой серии
      for (const [series, originalData] of seriesDataMap) {
        const convertedData = tzManager.convertData(originalData);
        series.setData(convertedData);
      }
    },
  };
}

// ============================================
// Пример использования
// ============================================

// Создание графика
const chartWrapper = createTimezoneAwareChart(
  document.getElementById('chart'),
  'Europe/Moscow'
);

// Добавление серии
const candleSeries = chartWrapper.addTimezoneSeries('candlestick', {
  upColor: '#26a69a',
  downColor: '#ef5350',
});

// Данные в UTC
const data = [
  { time: 1704067200, open: 100, high: 102, low: 99, close: 101 },
  { time: 1704153600, open: 101, high: 104, low: 100, close: 103 },
  // ... более данных
];

// Данные автоматически конвертируются при установке
candleSeries.setData(data);

// Переключение таймзоны
document.getElementById('tz-select').addEventListener('change', (e) => {
  chartWrapper.setTimezone(e.target.value, new Map([[candleSeries, data]]));
});
```

### React компонент с поддержкой таймзон

```tsx
import React, { useEffect, useRef, useState, useCallback } from 'react';
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';
import { getTimezoneOffset } from 'date-fns-tz';

interface TimezoneChartProps {
  data: Array<{ time: number; value: number }>;
  timezone?: string;
  width?: number;
  height?: number;
}

export const TimezoneChart: React.FC<TimezoneChartProps> = ({
  data,
  timezone = 'UTC',
  width = 600,
  height = 400,
}) => {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Line'> | null>(null);

  // Мемоизированная функция конвертации
  const convertData = useCallback((rawData: typeof data, tz: string) => {
    return rawData.map(bar => {
      const date = new Date(bar.time * 1000);
      const offset = getTimezoneOffset(tz, date);
      return {
        ...bar,
        time: bar.time + (offset / 1000),
      };
    });
  }, []);

  // Инициализация графика
  useEffect(() => {
    if (!containerRef.current) return;

    chartRef.current = createChart(containerRef.current, {
      width,
      height,
      layout: {
        background: { type: 'solid', color: '#1E222D' },
        textColor: '#DDD',
      },
      timeScale: {
        timeVisible: true,
      },
    });

    seriesRef.current = chartRef.current.addLineSeries({
      color: '#2962FF',
    });

    return () => {
      chartRef.current?.remove();
    };
  }, [width, height]);

  // Обновление данных при изменении data или timezone
  useEffect(() => {
    if (!seriesRef.current || !data) return;

    const convertedData = convertData(data, timezone);
    seriesRef.current.setData(convertedData);
    chartRef.current?.timeScale().fitContent();
  }, [data, timezone, convertData]);

  return <div ref={containerRef} />;
};

// Пример использования с селектором таймзоны
export const TimezoneChartWithSelector: React.FC<{ data: any[] }> = ({ data }) => {
  const [timezone, setTimezone] = useState('America/New_York');

  const timezones = [
    { value: 'UTC', label: 'UTC' },
    { value: 'America/New_York', label: 'New York (EST/EDT)' },
    { value: 'Europe/London', label: 'London (GMT/BST)' },
    { value: 'Europe/Moscow', label: 'Moscow (MSK)' },
    { value: 'Asia/Tokyo', label: 'Tokyo (JST)' },
    { value: 'Asia/Hong_Kong', label: 'Hong Kong (HKT)' },
  ];

  return (
    <div>
      <select value={timezone} onChange={(e) => setTimezone(e.target.value)}>
        {timezones.map(tz => (
          <option key={tz.value} value={tz.value}>{tz.label}</option>
        ))}
      </select>
      <TimezoneChart data={data} timezone={timezone} />
    </div>
  );
};
```

---

## 📊 Сравнительная таблица решений

| Критерий | date-fns-tz | Pure JS | Luxon | Formatters Only | Комбинированный |
|----------|:-----------:|:-------:|:-----:|:---------------:|:---------------:|
| **Корректность DST** | ✅ | ✅ | ✅ | ⚠️ | ✅ |
| **Tick marks позиции** | ✅ | ✅ | ✅ | ❌ | ✅ |
| **Размер зависимости** | 5KB | 0KB | 23KB | 0KB | 5KB |
| **Простота реализации** | ✅ | ⚠️ | ✅ | ✅ | ⚠️ |
| **Производительность** | ✅ | ⚠️ | ✅ | ✅ | ✅ |
| **Надёжность** | ✅ | ⚠️ | ✅ | ⚠️ | ✅ |
| **Оценка** | **9/10** | **8/10** | **8/10** | **6/10** | **10/10** |

---

## 🎯 Обработка DST переходов

### Проблема дублирующихся timestamps

При переводе часов назад (осенью) один час повторяется, что может привести к дублирующимся timestamps:

```javascript
// Пример: 27 октября 2024, переход с EEST на EET в Europe/Kyiv
// 03:00 -> 02:00 (час повторяется)

// Решение: использовать уникальные UTC timestamps и НЕ модифицировать их
// вместо этого модифицировать отображение

function handleDstAmbiguity(data, timezone) {
  const seen = new Set();
  
  return data.filter(bar => {
    const displayTime = convertToDisplayTime(bar.time, timezone);
    
    // Если такое "отображаемое" время уже есть - это DST дубликат
    if (seen.has(displayTime)) {
      console.warn(`DST ambiguity detected at ${displayTime}`);
      // Опции:
      // 1. Пропустить (filter out)
      // 2. Сместить на 1 секунду
      // 3. Агрегировать (для OHLC - объединить свечи)
      return false;
    }
    
    seen.add(displayTime);
    return true;
  });
}

// Или: агрегация дублирующихся свечей
function aggregateDstDuplicates(data, timezone) {
  const aggregated = new Map();
  
  for (const bar of data) {
    const displayTime = convertToDisplayTime(bar.time, timezone);
    
    if (aggregated.has(displayTime)) {
      const existing = aggregated.get(displayTime);
      // Объединяем OHLC данные
      aggregated.set(displayTime, {
        time: displayTime,
        open: existing.open,
        high: Math.max(existing.high, bar.high),
        low: Math.min(existing.low, bar.low),
        close: bar.close,
        volume: (existing.volume || 0) + (bar.volume || 0),
      });
    } else {
      aggregated.set(displayTime, { ...bar, time: displayTime });
    }
  }
  
  return Array.from(aggregated.values());
}
```

---

## 🔗 Источники

1. **GitHub Issue #1189** - [how to handle timezone and dst?](https://github.com/tradingview/lightweight-charts/issues/1189)
2. **Lightweight Charts Documentation** - [Time Zones Guide](https://tradingview.github.io/lightweight-charts/docs/time-zones)
3. **date-fns-tz** - [npm package](https://www.npmjs.com/package/date-fns-tz)
4. **Luxon** - [Documentation](https://moment.github.io/luxon/)
5. **Stack Overflow** - [Lightweight Charts timezone support](https://stackoverflow.com/questions/tagged/lightweight-charts+timezone)
6. **IANA Time Zone Database** - [tzdata](https://www.iana.org/time-zones)
7. **MDN** - [Intl.DateTimeFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat)

---

## 📝 Примечания

- Библиотека **не планирует** добавлять нативную поддержку таймзон (by design)
- `date-fns-tz` использует `Intl` API и не требует bundled timezone data
- При работе с историческими данными учитывайте изменения правил DST
- Для серверной генерации данных рекомендуется конвертировать на сервере

---

*Документ создан: 18 января 2026*  
*Последнее обновление: 18 января 2026*
