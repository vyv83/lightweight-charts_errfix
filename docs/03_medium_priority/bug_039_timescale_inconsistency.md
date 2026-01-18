# БАГ #39: Несогласованность временной шкалы

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1809](https://github.com/tradingview/lightweight-charts/issues/1809)  
> **Версии:** Все версии  
> **Статус:** ⚠️ Working as Intended (архитектурная особенность)

---

## 📋 Описание проблемы

### Суть проблемы
При отображении данных на больших временных масштабах (1 месяц, 1 год) наблюдается **несогласованность ширины баров и интервалов тиков** на временной шкале. Даже при предоставлении данных с равномерными интервалами, график может отображать тики с переменными интервалами.

### Проявления проблемы

1. **Неравномерные интервалы тиков:**
   - Месячный график показывает тики с 4-дневными интервалами
   - Затем неожиданно переключается на 6-дневные интервалы
   - Возвращается обратно без видимой причины

2. **Переменная ширина баров:**
   - Бары на одном и том же таймфрейме имеют разную визуальную ширину
   - Особенно заметно на краях графика

3. **Проблемы с выравниванием:**
   - Метки времени не всегда соответствуют центру бара
   - При zoom визуальная ширина баров изменяется непропорционально

### Технический контекст

Lightweight Charts использует **логический индекс** для позиционирования данных, а не абсолютные временные значения. Это означает:

```
Логический индекс:  0    1    2    3    4    ...
Реальное время:     Jan  Feb  Mar  Apr  May  ...
                    ↑    ↑    ↑    ↑    ↑
                  Равномерное распределение независимо от реальных интервалов
```

При этом библиотека использует адаптивный алгоритм для отображения тиков, который оптимизирует читаемость, но может приводить к визуальной несогласованности.

### Затронутые сценарии

- Финансовые графики с большими таймфреймами (1M, 1Y)
- Данные с нерегулярными интервалами
- Множественные серии с разными временными диапазонами
- Синхронизация нескольких графиков

---

## 🔍 Найденные решения

### Решение 1: Ограничение количества точек данных (⭐ 7/10)

**Описание:**  
Ограничение количества отображаемых точек кратным значению (например, 7) помогает алгоритму генерации тиков работать более предсказуемо.

**Преимущества:**
- ✅ Простая реализация
- ✅ Улучшает равномерность тиков
- ✅ Не требует изменения логики отображения

**Недостатки:**
- ❌ Может обрезать данные
- ❌ Не всегда применимо
- ❌ "Магические числа" в коде

**Реализация:**
```javascript
function adjustDataPointsCount(data, multiple = 7) {
    const targetLength = Math.floor(data.length / multiple) * multiple;
    
    if (targetLength === 0) {
        return data;
    }
    
    // Оставляем последние targetLength точек
    return data.slice(-targetLength);
}

// Использование
const rawData = fetchMonthlyData();
const adjustedData = adjustDataPointsCount(rawData, 7);
series.setData(adjustedData);
```

---

### Решение 2: Добавление padding к ширине графика (⭐ 6/10)

**Описание:**  
Добавление padding к размерам контейнера графика для достижения ширины, кратной определённому значению.

**Преимущества:**
- ✅ Не затрагивает данные
- ✅ Может улучшить равномерность
- ✅ Легко реализуется через CSS

**Недостатки:**
- ❌ Влияет на layout страницы
- ❌ Не гарантирует результат
- ❌ Требует пересчёта при resize

**Реализация:**
```javascript
function calculateOptimalWidth(containerWidth, barCount, targetSpacing = 7) {
    const idealWidth = barCount * targetSpacing;
    const multiple = Math.ceil(containerWidth / targetSpacing) * targetSpacing;
    
    return Math.max(containerWidth, multiple);
}

// Создание графика с оптимизированной шириной
function createOptimizedChart(container, data) {
    const optimalWidth = calculateOptimalWidth(
        container.clientWidth,
        data.length,
        7
    );
    
    const chart = createChart(container, {
        width: optimalWidth,
        height: 400,
    });
    
    // Центрирование графика
    const padding = (optimalWidth - container.clientWidth) / 2;
    if (padding > 0) {
        container.style.marginLeft = `-${padding}px`;
        container.style.overflow = 'hidden';
    }
    
    return chart;
}
```

---

### Решение 3: Использование логического диапазона для управления тиками (⭐ 8/10)

**Описание:**  
Использование `barSpacing` и `logicalRange` для точного контроля отображения временной шкалы и количества меток.

**Преимущества:**
- ✅ Полный контроль над отображением
- ✅ Нативный API библиотеки
- ✅ Работает на всех версиях

**Недостатки:**
- ❌ Требует понимания internal mechanics
- ❌ Может потребовать подстройки для разных размеров экрана

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

class TimeScaleManager {
    constructor(chart) {
        this.chart = chart;
        this.timeScale = chart.timeScale();
    }

    /**
     * Настройка оптимального barSpacing для равномерного отображения
     */
    configureOptimalSpacing(dataLength, targetVisibleBars) {
        const chartWidth = this.chart.options().width;
        const optimalSpacing = chartWidth / targetVisibleBars;
        
        this.timeScale.applyOptions({
            barSpacing: Math.max(optimalSpacing, 1),
            rightOffset: 5,
        });
    }

    /**
     * Установка видимого логического диапазона
     */
    setVisibleRange(startIndex, endIndex) {
        this.timeScale.setVisibleLogicalRange({
            from: startIndex,
            to: endIndex,
        });
    }

    /**
     * Подписка на изменение видимого диапазона
     */
    subscribeRangeChange(callback) {
        return this.timeScale.subscribeVisibleLogicalRangeChange((range) => {
            if (range) {
                callback({
                    from: Math.round(range.from),
                    to: Math.round(range.to),
                    barsCount: Math.round(range.to - range.from),
                });
            }
        });
    }
}

// Использование
const chart = createChart(container, { width: 800, height: 400 });
const manager = new TimeScaleManager(chart);

// Настройка для отображения ~100 баров
manager.configureOptimalSpacing(data.length, 100);
```

---

### Решение 4: Исключение нерабочих дней (⭐ 8/10)

**Описание:**  
Исключение выходных дней и праздников из данных для предотвращения их влияния на расчёт интервалов тиков.

**Преимущества:**
- ✅ Решает проблему для финансовых данных
- ✅ Улучшает общий вид графика
- ✅ Стандартная практика в trading apps

**Недостатки:**
- ❌ Не применимо для 24/7 данных (криптовалюты)
- ❌ Требует знания календаря торговых дней
- ❌ Может запутать пользователей

**Реализация:**
```javascript
/**
 * Фильтрует данные, исключая нерабочие дни
 */
function filterTradingDays(data, options = {}) {
    const {
        excludeWeekends = true,
        holidays = [],  // Массив дат в формате 'YYYY-MM-DD'
    } = options;

    const holidaySet = new Set(holidays);

    return data.filter(point => {
        const date = new Date(point.time * 1000 || point.time);
        const dayOfWeek = date.getUTCDay();
        
        // Проверка выходных
        if (excludeWeekends && (dayOfWeek === 0 || dayOfWeek === 6)) {
            return false;
        }
        
        // Проверка праздников
        const dateStr = date.toISOString().split('T')[0];
        if (holidaySet.has(dateStr)) {
            return false;
        }
        
        return true;
    });
}

// Использование
const rawData = fetchStockData();
const tradingDaysOnly = filterTradingDays(rawData, {
    excludeWeekends: true,
    holidays: ['2024-01-01', '2024-12-25'], // Праздники
});

series.setData(tradingDaysOnly);
```

---

### Решение 5: Whitespace Data для заполнения пробелов (⭐ 9/10)

**Описание:**  
Создание "whitespace" данных (точки только с временем, без значения) для обеспечения равномерного распределения на временной шкале.

**Преимущества:**
- ✅ Официально рекомендованный подход
- ✅ Сохраняет реальные временные интервалы
- ✅ Визуально отображает пробелы в данных

**Недостатки:**
- ❌ Увеличивает объём данных
- ❌ Требует генерации временных интервалов
- ❌ Может повлиять на производительность для большого количества пробелов

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Генератор whitespace данных
 */
class WhitespaceGenerator {
    /**
     * Создаёт массив whitespace точек для заданного временного диапазона
     * @param {number} startTime - Начальное время (Unix timestamp)
     * @param {number} endTime - Конечное время (Unix timestamp)
     * @param {number} intervalSeconds - Интервал между точками в секундах
     */
    static generateTimeRange(startTime, endTime, intervalSeconds) {
        const points = [];
        let currentTime = startTime;

        while (currentTime <= endTime) {
            points.push({ time: currentTime });
            currentTime += intervalSeconds;
        }

        return points;
    }

    /**
     * Заполняет пробелы в данных whitespace точками
     * @param {Array} data - Исходные данные
     * @param {number} expectedInterval - Ожидаемый интервал между точками
     */
    static fillGaps(data, expectedInterval) {
        if (data.length < 2) return data;

        const result = [];
        
        for (let i = 0; i < data.length; i++) {
            result.push(data[i]);

            if (i < data.length - 1) {
                const currentTime = data[i].time;
                const nextTime = data[i + 1].time;
                const gap = nextTime - currentTime;

                // Если пробел больше ожидаемого интервала
                if (gap > expectedInterval * 1.5) {
                    // Генерируем whitespace точки
                    let fillTime = currentTime + expectedInterval;
                    while (fillTime < nextTime) {
                        result.push({ time: fillTime });
                        fillTime += expectedInterval;
                    }
                }
            }
        }

        return result;
    }

    /**
     * Объединяет whitespace серию с данными
     */
    static mergeWithWhitespace(data, whitespaceData) {
        // Создаём Map для быстрого доступа к данным
        const dataMap = new Map(data.map(d => [d.time, d]));
        
        // Объединяем
        return whitespaceData.map(ws => dataMap.get(ws.time) || ws);
    }
}

// ============ Пример использования ============

const chart = createChart(container, {
    width: 800,
    height: 400,
    timeScale: {
        timeVisible: true,
        secondsVisible: false,
    },
});

const lineSeries = chart.addLineSeries({
    color: '#2962FF',
    lineWidth: 2,
});

// Исходные данные с пробелами
const rawData = [
    { time: 1704067200, value: 100 }, // 2024-01-01
    { time: 1704153600, value: 102 }, // 2024-01-02
    // Пробел: 2024-01-03 - 2024-01-05
    { time: 1704499200, value: 105 }, // 2024-01-06
    { time: 1704585600, value: 103 }, // 2024-01-07
];

const SECONDS_IN_DAY = 86400;

// Заполняем пробелы
const filledData = WhitespaceGenerator.fillGaps(rawData, SECONDS_IN_DAY);
lineSeries.setData(filledData);

// Результат: равномерное распределение с видимыми пробелами
```

---

### Решение 6: Кастомный tickMarkFormatter (⭐ 7/10)

**Описание:**  
Использование кастомного форматтера для контроля отображения меток времени.

**Преимущества:**
- ✅ Полный контроль над форматом
- ✅ Можно скрывать "лишние" метки
- ✅ Нативный API

**Недостатки:**
- ❌ Не решает проблему неравномерных интервалов
- ❌ Только косметическое улучшение
- ❌ Сложность реализации "умного" форматтера

**Реализация:**
```javascript
import { createChart, TickMarkType } from 'lightweight-charts';

/**
 * Кастомный форматтер меток времени с контролем плотности
 */
function createSmartTickFormatter(options = {}) {
    const {
        yearFormat = 'YYYY',
        monthFormat = 'MMM',
        dayFormat = 'DD',
        minLabelSpacing = 50, // Минимальное расстояние между метками в px
    } = options;

    let lastDrawnTickTime = 0;
    let lastDrawnTickX = -Infinity;

    return (time, tickMarkType, locale) => {
        const date = new Date(time * 1000);
        
        // Определяем формат в зависимости от типа тика
        let format;
        
        switch (tickMarkType) {
            case TickMarkType.Year:
                format = date.getFullYear().toString();
                break;
            case TickMarkType.Month:
                format = date.toLocaleString(locale, { month: 'short' });
                break;
            case TickMarkType.DayOfMonth:
                format = date.getDate().toString();
                break;
            case TickMarkType.Time:
                format = date.toLocaleTimeString(locale, { 
                    hour: '2-digit', 
                    minute: '2-digit' 
                });
                break;
            case TickMarkType.TimeWithSeconds:
                format = date.toLocaleTimeString(locale);
                break;
            default:
                format = '';
        }
        
        return format;
    };
}

// Использование
const chart = createChart(container, {
    width: 800,
    height: 400,
    timeScale: {
        tickMarkFormatter: createSmartTickFormatter(),
        timeVisible: true,
    },
});
```

---

### Решение 7: Синхронизация нескольких графиков (⭐ 8/10)

**Описание:**  
Для multi-chart layouts важно синхронизировать временные шкалы между графиками.

**Преимущества:**
- ✅ Обеспечивает согласованный UX
- ✅ Стандартный паттерн для trading apps
- ✅ Работает с обновлениями в реальном времени

**Недостатки:**
- ❌ Дополнительная сложность
- ❌ Возможны бесконечные циклы при неправильной реализации

**Реализация:**
```javascript
import { createChart } from 'lightweight-charts';

/**
 * Менеджер синхронизации временных шкал
 */
class TimeScaleSyncManager {
    constructor() {
        this.charts = new Map();
        this.isSyncing = false;
    }

    /**
     * Добавляет график в группу синхронизации
     */
    addChart(id, chart) {
        this.charts.set(id, chart);
        this._setupListener(id, chart);
    }

    /**
     * Удаляет график из группы
     */
    removeChart(id) {
        this.charts.delete(id);
    }

    _setupListener(sourceId, sourceChart) {
        sourceChart.timeScale().subscribeVisibleLogicalRangeChange((range) => {
            if (this.isSyncing || !range) return;

            this.isSyncing = true;

            try {
                this.charts.forEach((chart, id) => {
                    if (id !== sourceId) {
                        chart.timeScale().setVisibleLogicalRange(range);
                    }
                });
            } finally {
                // Используем setTimeout для сброса флага после всех обновлений
                setTimeout(() => {
                    this.isSyncing = false;
                }, 0);
            }
        });
    }

    /**
     * Синхронизирует все графики к указанному диапазону
     */
    syncToRange(range) {
        this.isSyncing = true;
        
        try {
            this.charts.forEach(chart => {
                chart.timeScale().setVisibleLogicalRange(range);
            });
        } finally {
            setTimeout(() => {
                this.isSyncing = false;
            }, 0);
        }
    }

    /**
     * Подстраивает контент всех графиков
     */
    fitAllContent() {
        this.charts.forEach(chart => {
            chart.timeScale().fitContent();
        });
    }
}

// ============ Пример использования ============

const syncManager = new TimeScaleSyncManager();

// Создание нескольких графиков
const chart1 = createChart(document.getElementById('chart1'), {
    width: 800, height: 300
});
const chart2 = createChart(document.getElementById('chart2'), {
    width: 800, height: 200
});
const chart3 = createChart(document.getElementById('chart3'), {
    width: 800, height: 200
});

// Добавление в группу синхронизации
syncManager.addChart('main', chart1);
syncManager.addChart('volume', chart2);
syncManager.addChart('indicator', chart3);

// Добавление данных
chart1.addCandlestickSeries().setData(priceData);
chart2.addHistogramSeries().setData(volumeData);
chart3.addLineSeries().setData(rsiData);

// Синхронное отображение всего контента
syncManager.fitAllContent();
```

---

## ✅ Рекомендуемое решение

### Комплексный подход: Whitespace Data + Настройка TimeScale

Для надёжного решения проблемы несогласованности временной шкалы рекомендуется комбинация подходов:

```javascript
import { createChart } from 'lightweight-charts';

/**
 * Комплексный менеджер временной шкалы
 * Решает проблему несогласованности через:
 * 1. Заполнение пробелов whitespace данными
 * 2. Оптимизацию настроек timeScale
 * 3. Контроль видимого диапазона
 */
class ConsistentTimeScaleManager {
    constructor(chart, options = {}) {
        this.chart = chart;
        this.timeScale = chart.timeScale();
        
        this.options = {
            expectedIntervalSeconds: options.expectedIntervalSeconds || 86400, // 1 день
            excludeWeekends: options.excludeWeekends ?? true,
            holidays: options.holidays || [],
            minBarSpacing: options.minBarSpacing || 6,
            targetVisibleBars: options.targetVisibleBars || 100,
        };
    }

    /**
     * Подготавливает данные для согласованной временной шкалы
     */
    prepareData(rawData) {
        if (!rawData || rawData.length === 0) return [];

        // Шаг 1: Фильтруем нерабочие дни (опционально)
        let processedData = this.options.excludeWeekends
            ? this._filterNonTradingDays(rawData)
            : rawData;

        // Шаг 2: Заполняем пробелы whitespace данными
        processedData = this._fillGapsWithWhitespace(processedData);

        return processedData;
    }

    /**
     * Настраивает оптимальные параметры временной шкалы
     */
    configureTimeScale() {
        const chartWidth = this.chart.options().width;
        const optimalSpacing = Math.max(
            chartWidth / this.options.targetVisibleBars,
            this.options.minBarSpacing
        );

        this.timeScale.applyOptions({
            barSpacing: optimalSpacing,
            minBarSpacing: this.options.minBarSpacing,
            rightOffset: 5,
            lockVisibleTimeRangeOnResize: true,
            timeVisible: true,
            secondsVisible: false,
            tickMarkFormatter: this._createTickFormatter(),
        });
    }

    /**
     * Фильтрует выходные и праздники
     */
    _filterNonTradingDays(data) {
        const holidaySet = new Set(this.options.holidays);

        return data.filter(point => {
            const timestamp = typeof point.time === 'number' 
                ? point.time 
                : new Date(point.time).getTime() / 1000;
            
            const date = new Date(timestamp * 1000);
            const dayOfWeek = date.getUTCDay();
            
            if (dayOfWeek === 0 || dayOfWeek === 6) {
                return false;
            }
            
            const dateStr = date.toISOString().split('T')[0];
            return !holidaySet.has(dateStr);
        });
    }

    /**
     * Заполняет пробелы whitespace данными
     */
    _fillGapsWithWhitespace(data) {
        if (data.length < 2) return data;

        const result = [];
        const interval = this.options.expectedIntervalSeconds;
        const threshold = interval * 1.5;

        for (let i = 0; i < data.length; i++) {
            result.push(data[i]);

            if (i < data.length - 1) {
                const currentTime = this._normalizeTime(data[i].time);
                const nextTime = this._normalizeTime(data[i + 1].time);
                const gap = nextTime - currentTime;

                if (gap > threshold) {
                    let fillTime = currentTime + interval;
                    
                    while (fillTime < nextTime) {
                        // Пропускаем выходные если нужно
                        if (!this.options.excludeWeekends || 
                            !this._isWeekend(fillTime)) {
                            result.push({ time: fillTime });
                        }
                        fillTime += interval;
                    }
                }
            }
        }

        return result;
    }

    _normalizeTime(time) {
        if (typeof time === 'number') return time;
        return Math.floor(new Date(time).getTime() / 1000);
    }

    _isWeekend(timestamp) {
        const day = new Date(timestamp * 1000).getUTCDay();
        return day === 0 || day === 6;
    }

    _createTickFormatter() {
        return (time, tickMarkType, locale) => {
            const date = new Date(time * 1000);
            
            // Упрощённый формат для читаемости
            const formatters = {
                0: () => date.getFullYear().toString(), // Year
                1: () => date.toLocaleString(locale, { month: 'short' }), // Month
                2: () => date.getDate().toString(), // Day
                3: () => date.toLocaleTimeString(locale, { 
                    hour: '2-digit', 
                    minute: '2-digit' 
                }),
            };
            
            return (formatters[tickMarkType] || (() => ''))();
        };
    }

    /**
     * Применяет данные к серии с оптимизацией
     */
    applyToSeries(series, rawData) {
        const preparedData = this.prepareData(rawData);
        series.setData(preparedData);
        
        // Настраиваем шкалу после применения данных
        this.configureTimeScale();
        
        // Подстраиваем видимый диапазон
        this.timeScale.fitContent();
        
        return preparedData;
    }
}

// ============ Полный пример использования ============

const container = document.getElementById('chart');

const chart = createChart(container, {
    width: 1000,
    height: 500,
    layout: {
        background: { type: 'solid', color: '#1e222d' },
        textColor: '#d1d4dc',
    },
    grid: {
        vertLines: { color: 'rgba(42, 46, 57, 0.5)' },
        horzLines: { color: 'rgba(42, 46, 57, 0.5)' },
    },
});

const candleSeries = chart.addCandlestickSeries({
    upColor: '#26a69a',
    downColor: '#ef5350',
    borderVisible: false,
    wickUpColor: '#26a69a',
    wickDownColor: '#ef5350',
});

// Создаём менеджер
const tsManager = new ConsistentTimeScaleManager(chart, {
    expectedIntervalSeconds: 86400, // Дневные данные
    excludeWeekends: true,
    holidays: ['2024-01-01', '2024-07-04', '2024-12-25'],
    targetVisibleBars: 80,
    minBarSpacing: 8,
});

// Исходные данные с потенциальными пробелами
const rawOHLCData = [
    { time: '2024-01-02', open: 100, high: 105, low: 98, close: 103 },
    { time: '2024-01-03', open: 103, high: 107, low: 102, close: 106 },
    // Пробел: 04-05 января пропущены
    { time: '2024-01-08', open: 106, high: 110, low: 104, close: 108 },
    { time: '2024-01-09', open: 108, high: 112, low: 107, close: 111 },
    // ... больше данных
];

// Применяем данные с автоматической подготовкой
tsManager.applyToSeries(candleSeries, rawOHLCData);

// Обработка resize
window.addEventListener('resize', () => {
    const newWidth = container.clientWidth;
    chart.resize(newWidth, 500);
    tsManager.configureTimeScale();
});
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Эффективность | Применимость |
|---------|--------|-----------|---------------|--------------|
| **Whitespace Data** | ⭐ 9/10 | Средняя | Высокая | Универсальная |
| Логический диапазон | ⭐ 8/10 | Средняя | Высокая | Программный контроль |
| Синхронизация графиков | ⭐ 8/10 | Высокая | Высокая | Multi-chart |
| Исключение нерабочих дней | ⭐ 8/10 | Низкая | Средняя | Финансовые данные |
| Ограничение точек данных | ⭐ 7/10 | Низкая | Средняя | Статические данные |
| Кастомный tickFormatter | ⭐ 7/10 | Средняя | Низкая | Косметическая |
| Padding ширины | ⭐ 6/10 | Низкая | Низкая | Edge cases |

---

## ⚠️ Важные замечания

1. **Issue #1809 помечен как "Working as Intended"** — это архитектурная особенность библиотеки, а не баг, который будет исправлен.

2. **Lightweight Charts использует логическую индексацию**, а не реальное время для позиционирования данных.

3. **Для криптовалют** (24/7 торги) решение с исключением выходных неприменимо.

4. **При синхронизации графиков** следует предотвращать бесконечные циклы обновлений.

---

## 🔗 Источники

1. [GitHub Issue #1809 - Inconsistent TimeScale bar width and tick interval](https://github.com/tradingview/lightweight-charts/issues/1809)
2. [GitHub Issue #1426 - Irregular time scale when data has gaps](https://github.com/tradingview/lightweight-charts/issues/1426)
3. [Lightweight Charts - Time Scale API](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ITimeScaleApi)
4. [Lightweight Charts - Whitespace Data](https://tradingview.github.io/lightweight-charts/docs/series-types)
5. [GitHub - Time Scale Synchronization Examples](https://github.com/tradingview/lightweight-charts/issues?q=synchronize)

---

## 📅 История изменений

| Дата | Версия | Изменения |
|------|--------|-----------|
| 2026-01-18 | 1.0 | Создание документа |

---

*Документация подготовлена для проекта lightweight-charts error tracking*
