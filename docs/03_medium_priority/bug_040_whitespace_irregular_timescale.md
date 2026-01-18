# БАГ #40: Whitespace и нерегулярность временной шкалы

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issues:** [#1426](https://github.com/tradingview/lightweight-charts/issues/1426), [#699](https://github.com/tradingview/lightweight-charts/issues/699), [#700](https://github.com/tradingview/lightweight-charts/issues/700)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (частично решено через whitespace API)

---

## 📋 Описание проблемы

### Суть проблемы
Lightweight Charts **не создаёт автоматические пробелы** (gaps) на временной шкале для отсутствующих данных. Библиотека равномерно распределяет все предоставленные точки данных, независимо от реального временного интервала между ними.

### Проявления проблемы

1. **Сжатие временных пробелов:**
   - Выходные дни отсутствуют визуально
   - Большие временные интервалы выглядят как маленькие
   - Линия графика соединяется "через ночь" без видимого пробела

2. **Нерегулярный вид шкалы времени:**
   - Метки времени распределены по логическим индексам, а не по реальному времени
   - Визуальное несоответствие ожиданиям пользователей

3. **Проблемы с line series:**
   - Whitespace данные не всегда создают видимые разрывы (Issue #700)
   - Линия может продолжать соединяться через whitespace точки

### Технический контекст

```javascript
// Исходные данные с пробелом (выходные)
const data = [
    { time: '2024-01-05', value: 100 }, // Пятница
    // Суббота и воскресенье пропущены
    { time: '2024-01-08', value: 102 }, // Понедельник
];

// Результат: точки будут отображены рядом друг с другом,
// без визуального указания на 3-дневный пробел
```

### Дизайн-решение библиотеки

Lightweight Charts использует **логическую индексацию** вместо временной:

```
Логический индекс: 0     1     2     3     4
                   ↓     ↓     ↓     ↓     ↓
Данные:           Mon   Tue   Wed   Thu   Fri
                   
Не будет:         Mon   Tue   Wed   Thu   Fri   [gap]   [gap]   Mon
                   ↓     ↓     ↓     ↓     ↓     ↓       ↓       ↓
Логический индекс: 0     1     2     3     4     5       6       7
```

### Связь между Issues

- **Issue #699** - "Ability to display gaps" (основной feature request)
- **Issue #700** - "Whitespace on line series isn't showing the gaps" (баг)
- **Issue #1426** - "TimeScale is not regular when data has gaps" (описание проблемы)

---

## 🔍 Найденные решения

### Решение 1: Whitespace Data Points (⭐ 9/10)

**Описание:**  
Официально рекомендованный подход — добавление whitespace данных (точки только с `time`, без `value`) для явного определения пробелов.

**Преимущества:**
- ✅ Официально поддерживается
- ✅ Создаёт визуальные пробелы
- ✅ Гибкий контроль над временной шкалой

**Недостатки:**
- ❌ Требует предварительной обработки данных
- ❌ Увеличивает объём данных
- ❌ Для line series может потребоваться дополнительная настройка

**Реализация:**
```javascript
/**
 * Генерирует whitespace данные для заполнения пробелов
 */
function createWhitespaceForGaps(data, expectedIntervalMs) {
    if (data.length < 2) return data;

    const result = [];
    
    for (let i = 0; i < data.length; i++) {
        result.push(data[i]);
        
        if (i < data.length - 1) {
            const currentTime = getTimeInMs(data[i].time);
            const nextTime = getTimeInMs(data[i + 1].time);
            const gap = nextTime - currentTime;
            
            // Если пробел больше ожидаемого интервала
            if (gap > expectedIntervalMs * 1.5) {
                // Заполняем whitespace точками
                let fillTime = currentTime + expectedIntervalMs;
                
                while (fillTime < nextTime) {
                    result.push({ 
                        time: Math.floor(fillTime / 1000) // Unix timestamp в секундах
                    });
                    fillTime += expectedIntervalMs;
                }
            }
        }
    }
    
    return result;
}

function getTimeInMs(time) {
    if (typeof time === 'number') {
        return time * 1000; // Unix timestamp в секундах → ms
    }
    return new Date(time).getTime();
}

// Использование
const HOUR_MS = 60 * 60 * 1000;
const processedData = createWhitespaceForGaps(rawData, HOUR_MS);
series.setData(processedData);
```

---

### Решение 2: Отдельная Whitespace серия (⭐ 8/10)

**Описание:**  
Создание отдельной невидимой серии, которая определяет полную временную шкалу, включая все ожидаемые интервалы.

**Преимущества:**
- ✅ Чёткое разделение данных и шкалы
- ✅ Временная шкала определяется независимо
- ✅ Легче управлять для сложных сценариев

**Недостатки:**
- ❌ Дополнительная серия
- ❌ Требует генерации полного временного диапазона
- ❌ Может влиять на производительность

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Менеджер whitespace серий
 */
class WhitespaceSeriesManager {
    constructor(chart) {
        this.chart = chart;
        this.whitespaceSeries = null;
    }

    /**
     * Создаёт whitespace серию для определения временной шкалы
     */
    createTimeScaleFrame(startTime, endTime, intervalSeconds) {
        // Создаём невидимую линейную серию
        this.whitespaceSeries = this.chart.addLineSeries({
            visible: false, // Невидимая серия
            priceScaleId: '', // Без привязки к price scale
        });

        // Генерируем whitespace данные для всего диапазона
        const whitespaceData = this._generateTimeRange(
            startTime, 
            endTime, 
            intervalSeconds
        );
        
        this.whitespaceSeries.setData(whitespaceData);
        
        return whitespaceData;
    }

    _generateTimeRange(startTime, endTime, intervalSeconds) {
        const points = [];
        let currentTime = startTime;

        while (currentTime <= endTime) {
            points.push({ time: currentTime });
            currentTime += intervalSeconds;
        }

        return points;
    }

    /**
     * Объединяет данные с whitespace фреймом
     */
    mergeDataWithFrame(data, whitespaceTimes) {
        // Создаём set для быстрого поиска
        const dataMap = new Map(data.map(d => [d.time, d]));
        
        // Заменяем whitespace на реальные данные где они есть
        return whitespaceTimes.map(ws => dataMap.get(ws.time) || ws);
    }
}

// ============ Использование ============

const chart = createChart(container, { width: 800, height: 400 });

const wsManager = new WhitespaceSeriesManager(chart);

// Определяем временную рамку (1 неделя с шагом 1 час)
const startTime = Math.floor(new Date('2024-01-01').getTime() / 1000);
const endTime = Math.floor(new Date('2024-01-07').getTime() / 1000);
const HOUR = 3600;

const timeFrame = wsManager.createTimeScaleFrame(startTime, endTime, HOUR);

// Добавляем реальные данные
const mainSeries = chart.addLineSeries({ color: '#2962FF' });
const mergedData = wsManager.mergeDataWithFrame(realData, timeFrame);
mainSeries.setData(mergedData);
```

---

### Решение 3: Фильтрация выходных с whitespace (⭐ 8/10)

**Описание:**  
Для финансовых данных — удаление выходных дней и добавление whitespace для праздников и нерабочих периодов.

**Преимущества:**
- ✅ Чистый график без "мёртвых" зон
- ✅ Стандарт для финансовых приложений
- ✅ Уменьшает объём данных

**Недостатки:**
- ❌ Не применимо для 24/7 рынков
- ❌ Требует знания торгового календаря

**Реализация:**
```javascript
/**
 * Процессор данных для финансовых графиков
 */
class FinancialDataProcessor {
    constructor(options = {}) {
        this.options = {
            excludeWeekends: options.excludeWeekends ?? true,
            tradingStartHour: options.tradingStartHour ?? 9,
            tradingEndHour: options.tradingEndHour ?? 17,
            holidays: new Set(options.holidays || []),
            timezone: options.timezone || 'UTC',
        };
    }

    /**
     * Проверяет, является ли время торговым
     */
    isTradingTime(timestamp) {
        const date = new Date(timestamp * 1000);
        
        // Проверяем выходные
        if (this.options.excludeWeekends) {
            const day = date.getUTCDay();
            if (day === 0 || day === 6) return false;
        }
        
        // Проверяем праздники
        const dateStr = date.toISOString().split('T')[0];
        if (this.options.holidays.has(dateStr)) return false;
        
        // Проверяем торговые часы (для intraday данных)
        const hour = date.getUTCHours();
        if (hour < this.options.tradingStartHour || 
            hour >= this.options.tradingEndHour) {
            return false;
        }
        
        return true;
    }

    /**
     * Обрабатывает данные, добавляя whitespace для нерабочих периодов
     */
    processData(data, intervalSeconds) {
        if (data.length === 0) return [];

        const result = [];
        
        for (let i = 0; i < data.length; i++) {
            const point = data[i];
            const time = typeof point.time === 'number' 
                ? point.time 
                : Math.floor(new Date(point.time).getTime() / 1000);
            
            // Добавляем точку данных
            result.push({ ...point, time });
            
            // Если есть следующая точка, проверяем пробел
            if (i < data.length - 1) {
                const nextTime = typeof data[i + 1].time === 'number'
                    ? data[i + 1].time
                    : Math.floor(new Date(data[i + 1].time).getTime() / 1000);
                
                // Заполняем пробелы только в торговое время
                this._fillTradingGaps(result, time, nextTime, intervalSeconds);
            }
        }
        
        return result;
    }

    _fillTradingGaps(result, startTime, endTime, intervalSeconds) {
        let current = startTime + intervalSeconds;
        
        while (current < endTime) {
            if (this.isTradingTime(current)) {
                // Добавляем whitespace только для торгового времени
                result.push({ time: current });
            }
            current += intervalSeconds;
        }
    }
}

// ============ Использование ============

const processor = new FinancialDataProcessor({
    excludeWeekends: true,
    tradingStartHour: 9,
    tradingEndHour: 17,
    holidays: ['2024-01-01', '2024-07-04', '2024-12-25'],
});

const MINUTE = 60;
const processedData = processor.processData(rawData, MINUTE);
series.setData(processedData);
```

---

### Решение 4: TimeScale Options для управления whitespace (⭐ 7/10)

**Описание:**  
Использование встроенных опций `TimeScaleOptions` для контроля поведения whitespace.

**Преимущества:**
- ✅ Нативный API
- ✅ Простая конфигурация
- ✅ Влияет на отображение сетки и кроссхейра

**Недостатки:**
- ❌ Не создаёт визуальные пробелы автоматически
- ❌ Только влияет на поведение, не на данные

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
    width: 800,
    height: 400,
    timeScale: {
        /**
         * Игнорировать whitespace индексы при:
         * - Рисовании линий сетки
         * - Рисовании тиков
         * - Привязке кроссхейра
         */
        ignoreWhitespaceIndices: true,
        
        /**
         * Разрешить сдвиг видимого диапазона при замене whitespace
         * новыми данными (если shiftVisibleRangeOnNewBar = true)
         */
        allowShiftVisibleRangeOnWhitespaceReplacement: true,
        
        /**
         * Автоматический сдвиг при появлении нового бара
         */
        shiftVisibleRangeOnNewBar: true,
        
        /**
         * Отступ справа для whitespace
         */
        rightOffset: 10,
        
        /**
         * Минимальный отступ между барами
         */
        minBarSpacing: 2,
    },
});
```

---

### Решение 5: Real-time Whitespace Updates (⭐ 7/10)

**Описание:**  
Для real-time данных — динамическое добавление whitespace при поступлении новых данных.

**Преимущества:**
- ✅ Работает с live данными
- ✅ Автоматическое заполнение пробелов
- ✅ Не требует предварительной обработки

**Недостатки:**
- ❌ Может быть ресурсоёмким
- ❌ Сложнее в реализации
- ❌ Требует отслеживания последнего времени

**Реализация:**
```javascript
/**
 * Менеджер real-time данных с поддержкой whitespace
 */
class RealTimeWhitespaceManager {
    constructor(series, expectedIntervalSeconds) {
        this.series = series;
        this.expectedInterval = expectedIntervalSeconds;
        this.lastDataTime = null;
    }

    /**
     * Обновляет серию с автоматическим добавлением whitespace
     */
    update(newData) {
        const newTime = typeof newData.time === 'number'
            ? newData.time
            : Math.floor(new Date(newData.time).getTime() / 1000);

        // Если это первая точка
        if (this.lastDataTime === null) {
            this.series.update(newData);
            this.lastDataTime = newTime;
            return;
        }

        // Вычисляем пробел
        const gap = newTime - this.lastDataTime;
        
        // Если пробел больше ожидаемого интервала
        if (gap > this.expectedInterval * 1.5) {
            // Добавляем whitespace точки
            let fillTime = this.lastDataTime + this.expectedInterval;
            
            while (fillTime < newTime) {
                this.series.update({ time: fillTime });
                fillTime += this.expectedInterval;
            }
        }
        
        // Добавляем актуальные данные
        this.series.update(newData);
        this.lastDataTime = newTime;
    }

    /**
     * Принудительно добавляет whitespace до текущего времени
     */
    fillToNow() {
        if (this.lastDataTime === null) return;
        
        const now = Math.floor(Date.now() / 1000);
        let fillTime = this.lastDataTime + this.expectedInterval;
        
        while (fillTime <= now) {
            this.series.update({ time: fillTime });
            fillTime += this.expectedInterval;
        }
    }
}

// ============ Использование с WebSocket ============

const chart = createChart(container, { width: 800, height: 400 });
const lineSeries = chart.addLineSeries({ color: '#2962FF' });

const MINUTE = 60;
const rtManager = new RealTimeWhitespaceManager(lineSeries, MINUTE);

// WebSocket connection
const ws = new WebSocket('wss://example.com/stream');

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    rtManager.update({
        time: Math.floor(Date.now() / 1000),
        value: data.price,
    });
};

// Периодически заполняем пробелы если нет данных
setInterval(() => {
    rtManager.fillToNow();
}, MINUTE * 1000);
```

---

### Решение 6: Custom Series для работы с gaps (⭐ 6/10)

**Описание:**  
Для line series, где whitespace не создаёт визуальных разрывов, можно использовать обходной путь с area series или custom rendering.

**Преимущества:**
- ✅ Решает проблему Issue #700
- ✅ Визуальные разрывы в линии

**Недостатки:**
- ❌ Требует дополнительной логики
- ❌ Может не подходить для всех случаев

**Реализация:**
```javascript
/**
 * Создаёт серию с поддержкой визуальных разрывов
 */
function createGapSupportedSeries(chart, data, options = {}) {
    const {
        color = '#2962FF',
        lineWidth = 2,
        gapThreshold = 2, // Множитель интервала для определения gap
    } = options;

    // Разбиваем данные на сегменты по gaps
    const segments = splitDataByGaps(data, gapThreshold);
    
    // Создаём серию для каждого сегмента
    const series = [];
    
    segments.forEach((segment, index) => {
        const s = chart.addLineSeries({
            color,
            lineWidth,
            // Первый сегмент на основной шкале, остальные скрывают метку
            lastValueVisible: index === segments.length - 1,
            priceLineVisible: index === segments.length - 1,
        });
        
        s.setData(segment);
        series.push(s);
    });
    
    return series;
}

function splitDataByGaps(data, gapMultiplier) {
    if (data.length < 2) return [data];
    
    // Вычисляем средний интервал
    let totalInterval = 0;
    for (let i = 1; i < data.length; i++) {
        totalInterval += data[i].time - data[i - 1].time;
    }
    const avgInterval = totalInterval / (data.length - 1);
    const gapThreshold = avgInterval * gapMultiplier;
    
    const segments = [];
    let currentSegment = [data[0]];
    
    for (let i = 1; i < data.length; i++) {
        const gap = data[i].time - data[i - 1].time;
        
        if (gap > gapThreshold) {
            // Начинаем новый сегмент
            segments.push(currentSegment);
            currentSegment = [data[i]];
        } else {
            currentSegment.push(data[i]);
        }
    }
    
    if (currentSegment.length > 0) {
        segments.push(currentSegment);
    }
    
    return segments;
}

// Использование
const allSeries = createGapSupportedSeries(chart, data, {
    color: '#2962FF',
    gapThreshold: 2,
});
```

---

## ✅ Рекомендуемое решение

### Комплексный Whitespace Manager

Объединяет все подходы в единый менеджер для работы с нерегулярными данными:

```javascript
import { createChart } from 'lightweight-charts';

/**
 * Комплексный менеджер whitespace для временных рядов
 */
class WhitespaceManager {
    constructor(options = {}) {
        this.options = {
            expectedIntervalSeconds: options.expectedIntervalSeconds || 3600,
            gapMultiplier: options.gapMultiplier || 1.5,
            excludeWeekends: options.excludeWeekends ?? false,
            excludedHours: options.excludedHours || [], // [0-6] = ночь
            holidays: new Set(options.holidays || []),
            autoFillGaps: options.autoFillGaps ?? true,
        };
    }

    /**
     * Проверяет, должно ли время отображаться
     */
    shouldDisplayTime(timestamp) {
        const date = new Date(timestamp * 1000);
        
        // Проверка выходных
        if (this.options.excludeWeekends) {
            const day = date.getUTCDay();
            if (day === 0 || day === 6) return false;
        }
        
        // Проверка праздников
        const dateStr = date.toISOString().split('T')[0];
        if (this.options.holidays.has(dateStr)) return false;
        
        // Проверка часов
        const hour = date.getUTCHours();
        if (this.options.excludedHours.includes(hour)) return false;
        
        return true;
    }

    /**
     * Обрабатывает данные с учётом whitespace
     */
    processData(data) {
        if (!data || data.length === 0) return [];
        
        // Нормализуем timestamps
        const normalized = data.map(point => ({
            ...point,
            time: this._normalizeTime(point.time),
        }));

        // Сортируем по времени
        normalized.sort((a, b) => a.time - b.time);

        // Заполняем gaps если нужно
        if (this.options.autoFillGaps) {
            return this._fillGaps(normalized);
        }

        return normalized;
    }

    _normalizeTime(time) {
        if (typeof time === 'number') return time;
        return Math.floor(new Date(time).getTime() / 1000);
    }

    _fillGaps(data) {
        const result = [];
        const interval = this.options.expectedIntervalSeconds;
        const threshold = interval * this.options.gapMultiplier;

        for (let i = 0; i < data.length; i++) {
            result.push(data[i]);

            if (i < data.length - 1) {
                const currentTime = data[i].time;
                const nextTime = data[i + 1].time;
                const gap = nextTime - currentTime;

                if (gap > threshold) {
                    let fillTime = currentTime + interval;
                    
                    while (fillTime < nextTime) {
                        if (this.shouldDisplayTime(fillTime)) {
                            result.push({ time: fillTime });
                        }
                        fillTime += interval;
                    }
                }
            }
        }

        return result;
    }

    /**
     * Генерирует полный временной фрейм
     */
    generateTimeFrame(startTimestamp, endTimestamp) {
        const points = [];
        const interval = this.options.expectedIntervalSeconds;
        let current = startTimestamp;

        while (current <= endTimestamp) {
            if (this.shouldDisplayTime(current)) {
                points.push({ time: current });
            }
            current += interval;
        }

        return points;
    }

    /**
     * Объединяет данные с временным фреймом
     */
    mergeWithTimeFrame(data, timeFrame) {
        const dataMap = new Map(
            data.map(d => [this._normalizeTime(d.time), d])
        );
        
        return timeFrame.map(tf => dataMap.get(tf.time) || tf);
    }

    /**
     * Создаёт real-time updater
     */
    createRealtimeUpdater(series) {
        return new RealtimeWhitespaceUpdater(series, this);
    }
}

/**
 * Real-time обновления с whitespace
 */
class RealtimeWhitespaceUpdater {
    constructor(series, manager) {
        this.series = series;
        this.manager = manager;
        this.lastTime = null;
    }

    update(data) {
        const time = this.manager._normalizeTime(data.time);

        if (this.lastTime !== null) {
            const gap = time - this.lastTime;
            const threshold = this.manager.options.expectedIntervalSeconds * 
                              this.manager.options.gapMultiplier;

            if (gap > threshold) {
                let fillTime = this.lastTime + 
                               this.manager.options.expectedIntervalSeconds;
                
                while (fillTime < time) {
                    if (this.manager.shouldDisplayTime(fillTime)) {
                        this.series.update({ time: fillTime });
                    }
                    fillTime += this.manager.options.expectedIntervalSeconds;
                }
            }
        }

        this.series.update({ ...data, time });
        this.lastTime = time;
    }
}

// ============ Полный пример использования ============

const container = document.getElementById('chart');

const chart = createChart(container, {
    width: 1000,
    height: 500,
    layout: {
        background: { type: 'solid', color: '#131722' },
        textColor: '#d1d4dc',
    },
    grid: {
        vertLines: { color: 'rgba(42, 46, 57, 0.6)' },
        horzLines: { color: 'rgba(42, 46, 57, 0.6)' },
    },
    timeScale: {
        timeVisible: true,
        secondsVisible: false,
        borderColor: '#485c7b',
        ignoreWhitespaceIndices: true,
    },
    rightPriceScale: {
        borderColor: '#485c7b',
    },
});

// Создаём менеджер для часовых данных с исключением выходных
const wsManager = new WhitespaceManager({
    expectedIntervalSeconds: 3600, // 1 час
    excludeWeekends: true,
    holidays: ['2024-01-01', '2024-07-04', '2024-12-25'],
    autoFillGaps: true,
});

// Серия свечей
const candleSeries = chart.addCandlestickSeries({
    upColor: '#26a69a',
    downColor: '#ef5350',
    borderVisible: false,
    wickUpColor: '#26a69a',
    wickDownColor: '#ef5350',
});

// Исходные данные с пробелами
const rawData = [
    { time: '2024-01-02T10:00:00Z', open: 100, high: 102, low: 99, close: 101 },
    { time: '2024-01-02T11:00:00Z', open: 101, high: 103, low: 100, close: 102 },
    // Пробел: 12:00-14:00
    { time: '2024-01-02T15:00:00Z', open: 102, high: 104, low: 101, close: 103 },
    { time: '2024-01-02T16:00:00Z', open: 103, high: 105, low: 102, close: 104 },
];

// Обработка данных
const processedData = wsManager.processData(rawData);
candleSeries.setData(processedData);

// Для real-time обновлений
const updater = wsManager.createRealtimeUpdater(candleSeries);

// Симуляция WebSocket
function simulateWebSocket() {
    const ws = new WebSocket('wss://example.com/stream');
    
    ws.onmessage = (event) => {
        const tick = JSON.parse(event.data);
        updater.update({
            time: tick.timestamp,
            open: tick.open,
            high: tick.high,
            low: tick.low,
            close: tick.close,
        });
    };
}

chart.timeScale().fitContent();
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Визуальные gaps | Real-time |
|---------|--------|-----------|-----------------|-----------|
| **Whitespace Data Points** | ⭐ 9/10 | Низкая | ✅ Да | ⚠️ Частично |
| Отдельная Whitespace серия | ⭐ 8/10 | Средняя | ✅ Да | ❌ Нет |
| Фильтрация выходных | ⭐ 8/10 | Средняя | ✅ Да | ⚠️ Частично |
| TimeScale Options | ⭐ 7/10 | Низкая | ❌ Нет | ✅ Да |
| Real-time Whitespace | ⭐ 7/10 | Средняя | ✅ Да | ✅ Да |
| Custom Series segments | ⭐ 6/10 | Высокая | ✅ Да | ❌ Нет |

---

## ⚠️ Важные замечания

1. **Issue #700 (Line series gaps)** — whitespace может не создавать визуальные разрывы в line series. Используйте сегментирование или area series как альтернативу.

2. **Производительность** — большое количество whitespace точек может влиять на производительность. Используйте разумные интервалы.

3. **Data Conflation (v5.1.0)** — новая функция агрегации данных может влиять на отображение gaps при сильном zoom out.

4. **Криптовалюты** — для 24/7 рынков не используйте исключение выходных.

---

## 🔗 Источники

1. [GitHub Issue #1426 - TimeScale is not regular when data has gaps](https://github.com/tradingview/lightweight-charts/issues/1426)
2. [GitHub Issue #699 - Ability to display gaps](https://github.com/tradingview/lightweight-charts/issues/699)
3. [GitHub Issue #700 - Whitespace on line series isn't showing the gaps](https://github.com/tradingview/lightweight-charts/issues/700)
4. [Lightweight Charts - TimeScaleOptions](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions)
5. [Lightweight Charts - Series Types](https://tradingview.github.io/lightweight-charts/docs/series-types)
6. [Lightweight Charts - v5.1.0 Data Conflation](https://tradingview.github.io/lightweight-charts/)

---

## 📅 История изменений

| Дата | Версия | Изменения |
|------|--------|-----------|
| 2026-01-18 | 1.0 | Создание документа |

---

*Документация подготовлена для проекта lightweight-charts error tracking*
