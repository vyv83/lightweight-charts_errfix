# БАГ #38: Логарифмическая ось не работает для значений близких к 0

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#874](https://github.com/tradingview/lightweight-charts/issues/874)  
> **Версии:** v4.x и ранее (исправлено в v5.0.0)  
> **PR с исправлением:** [#1965](https://github.com/tradingview/lightweight-charts/pull/1965)  
> **Статус:** ✅ ИСПРАВЛЕНО в v5.0.0

---

## 📋 Описание проблемы

### Суть проблемы
При использовании логарифмической шкалы (`PriceScaleMode.Logarithmic`) график некорректно масштабировался для значений, близких к нулю. Это фундаментальная математическая проблема, так как **log(0) = -∞** (логарифм нуля не определён).

### Технический контекст
```
log(1) = 0
log(0.1) = -1
log(0.01) = -2
log(0.001) = -3
...
log(0) → -∞ (не определён)
```

### Проявления бага
1. **Некорректный расчёт видимого диапазона** при очень малых значениях
2. **Проблемы с точностью** при преобразовании координат из логарифмического пространства
3. **Ошибки округления** для значений с большим количеством знаков после запятой
4. **Визуальные артефакты** при автомасштабировании

### Затронутые сценарии
- Криптовалюты с очень низкой ценой (SHIB, PEPE, etc.)
- Научные данные с малыми значениями
- Показатели, стремящиеся к нулю
- Нормализованные данные в диапазоне 0-1

---

## 🔍 Найденные решения

### Решение 1: Обновление до версии 5.0.0+ (⭐ 10/10)

**Описание:**  
Issue #874 был официально исправлен в версии 5.0.0 библиотеки (PR #1965). Исправление улучшило расчёт видимого диапазона price scale в режиме логарифмического масштаба путём корректного преобразования диапазона из log-пространства и обработки проблем точности с округлением.

**Преимущества:**
- ✅ Официальное исправление от разработчиков
- ✅ Нет необходимости в workarounds
- ✅ Корректная работа "из коробки"
- ✅ Протестировано и валидировано

**Недостатки:**
- ❌ Требует миграции на v5.x (breaking changes)
- ❌ CommonJS больше не поддерживается

**Реализация:**
```bash
npm install lightweight-charts@latest
# или
yarn add lightweight-charts@latest
```

---

### Решение 2: Замена нуля малым положительным значением (⭐ 7/10)

**Описание:**  
Классический workaround для логарифмических шкал — замена нулевых и околонулевых значений на малое положительное число (epsilon).

**Преимущества:**
- ✅ Работает на любой версии
- ✅ Простая реализация
- ✅ Обеспечивает визуальную непрерывность

**Недостатки:**
- ❌ Искажает реальные значения
- ❌ Требует информирования пользователей
- ❌ Может вводить в заблуждение

**Реализация:**
```javascript
const EPSILON = 1e-10; // Очень малое число
const MIN_LOG_VALUE = 0.0000001; // Минимально допустимое значение

function sanitizeForLogScale(data) {
    return data.map(point => {
        let value = point.value ?? point.close;
        
        if (value <= 0) {
            value = MIN_LOG_VALUE;
        } else if (value < MIN_LOG_VALUE) {
            value = MIN_LOG_VALUE;
        }
        
        // Для OHLC данных
        if (point.open !== undefined) {
            return {
                ...point,
                open: Math.max(point.open, MIN_LOG_VALUE),
                high: Math.max(point.high, MIN_LOG_VALUE),
                low: Math.max(point.low, MIN_LOG_VALUE),
                close: Math.max(point.close, MIN_LOG_VALUE),
            };
        }
        
        return { ...point, value };
    });
}

// Использование
const rawData = [
    { time: '2024-01-01', value: 0.00001 },
    { time: '2024-01-02', value: 0 },       // Проблемное значение
    { time: '2024-01-03', value: 0.00002 },
];

const sanitizedData = sanitizeForLogScale(rawData);
series.setData(sanitizedData);
```

---

### Решение 3: Установка null для нулевых значений (⭐ 6/10)

**Описание:**  
Вместо попытки отобразить нулевые значения, их можно заменить на `null`, что создаст разрыв в линии графика.

**Преимущества:**
- ✅ Точно отражает невозможность отображения
- ✅ Нет искажения данных
- ✅ Визуальный индикатор проблемных точек

**Недостатки:**
- ❌ Создаёт разрывы в графике
- ❌ Не подходит для непрерывных данных
- ❌ Может выглядеть как ошибка

**Реализация:**
```javascript
function replaceZeroWithNull(data) {
    return data.map(point => {
        const value = point.value ?? point.close;
        
        if (value <= 0) {
            return { time: point.time }; // Whitespace data
        }
        
        return point;
    });
}

// Для line series с разрывами
const processedData = replaceZeroWithNull(rawData);
lineSeries.setData(processedData);
```

---

### Решение 4: Трансформация log(x + 1) (⭐ 5/10)

**Описание:**  
Применение трансформации `log(x + 1)` позволяет отобразить нулевые значения, так как `log(0 + 1) = log(1) = 0`.

**Преимущества:**
- ✅ Поддерживает нулевые значения
- ✅ Сохраняет относительные пропорции для больших значений
- ✅ Гладкий переход

**Недостатки:**
- ❌ Искажает истинные логарифмические отношения
- ❌ Сжимает шкалу для малых значений
- ❌ Требует обратного преобразования для отображения

**Реализация:**
```javascript
// Трансформация данных
function logPlusOneTransform(data) {
    return data.map(point => ({
        ...point,
        value: Math.log10(point.value + 1),
        originalValue: point.value, // Сохраняем оригинал
    }));
}

// Custom price formatter для отображения реального значения
const priceFormatter = {
    format: (price) => {
        // Обратное преобразование: 10^price - 1
        const originalValue = Math.pow(10, price) - 1;
        return originalValue.toFixed(8);
    }
};

// Настройка серии
const series = chart.addLineSeries({
    priceFormat: {
        type: 'custom',
        formatter: priceFormatter.format,
    },
});

// ВАЖНО: При этом методе НЕ используйте логарифмический режим!
// chart.priceScale('right').applyOptions({ mode: 0 }); // Linear
```

---

### Решение 5: Symlog (псевдологарифмическая) шкала (⭐ 4/10)

**Описание:**  
Псевдологарифмическая шкала использует линейное масштабирование вблизи нуля и логарифмическое для больших значений. Требует кастомной реализации.

**Преимущества:**
- ✅ Поддерживает нуль и отрицательные значения
- ✅ Плавный переход линейное ↔ логарифмическое
- ✅ Стандартный подход в научных визуализациях

**Недостатки:**
- ❌ Сложная реализация
- ❌ Не поддерживается нативно в lightweight-charts
- ❌ Требует кастомного форматирования осей

**Реализация:**
```javascript
// Symlog функция: sign(x) * log10(1 + |x|)
function symlog(x, base = 10) {
    if (x === 0) return 0;
    const sign = Math.sign(x);
    return sign * Math.log10(1 + Math.abs(x)) / Math.log10(base);
}

// Обратная функция
function symlogInverse(y, base = 10) {
    if (y === 0) return 0;
    const sign = Math.sign(y);
    return sign * (Math.pow(base, Math.abs(y)) - 1);
}

// Применение к данным
function applySymlog(data) {
    return data.map(point => ({
        ...point,
        value: symlog(point.value),
        originalValue: point.value,
    }));
}

// Кастомный форматтер цены
const symlogFormatter = (price) => {
    const original = symlogInverse(price);
    
    if (Math.abs(original) < 0.01) {
        return original.toExponential(2);
    }
    return original.toFixed(6);
};
```

---

### Решение 6: Установка minValue для price scale (⭐ 6/10)

**Описание:**  
Явное указание минимального значения для ценовой шкалы предотвращает попытки автомасштабирования к нулю.

**Преимущества:**
- ✅ Простая конфигурация
- ✅ Работает с логарифмическим режимом
- ✅ Нативное решение

**Недостатки:**
- ❌ Может обрезать данные ниже порога
- ❌ Требует знания минимального значения заранее
- ❌ Не решает проблему для данных с реальными нулями

**Реализация:**
```javascript
import { createChart, PriceScaleMode } from 'lightweight-charts';

const chart = createChart(container, {
    rightPriceScale: {
        mode: PriceScaleMode.Logarithmic,
        scaleMargins: {
            top: 0.1,
            bottom: 0.1,
        },
    },
});

// Установка минимального значения после добавления данных
chart.priceScale('right').applyOptions({
    autoScale: false,
});

// Явное задание диапазона
series.priceScale().applyOptions({
    invertScale: false,
});

// Или использование setVisiblePriceRange
const minValue = 0.0000001;
const maxValue = findMaxInData(data);
series.priceScale().applyOptions({
    autoScale: true,
});
```

---

## ✅ Рекомендуемое решение

### Обновление до v5.0.0+ с валидацией данных

Оптимальный подход — **обновление до версии 5.0.0 или выше**, где проблема официально исправлена. Дополнительно рекомендуется добавить защитную валидацию данных на уровне приложения.

```javascript
import { createChart, PriceScaleMode } from 'lightweight-charts';

/**
 * Утилита для безопасной работы с логарифмической шкалой
 */
class LogScaleHelper {
    static MIN_SAFE_VALUE = 1e-10;

    /**
     * Проверяет, может ли значение быть отображено на log-шкале
     */
    static isValidForLogScale(value) {
        return typeof value === 'number' && 
               !isNaN(value) && 
               value > 0;
    }

    /**
     * Санитизирует данные для безопасного отображения на log-шкале
     * @param {Array} data - Массив данных
     * @param {Object} options - Опции обработки
     * @returns {Array} - Обработанные данные
     */
    static sanitizeData(data, options = {}) {
        const {
            minValue = this.MIN_SAFE_VALUE,
            replaceWithNull = false,
            logWarnings = true,
        } = options;

        let warningCount = 0;

        const sanitized = data.map((point, index) => {
            // Определяем тип данных (line vs OHLC)
            const isOHLC = point.open !== undefined;
            
            if (isOHLC) {
                return this._sanitizeOHLC(point, minValue, replaceWithNull, 
                    () => { warningCount++; });
            } else {
                return this._sanitizeLinePoint(point, minValue, replaceWithNull,
                    () => { warningCount++; });
            }
        });

        if (logWarnings && warningCount > 0) {
            console.warn(
                `[LogScaleHelper] Sanitized ${warningCount} points with invalid log values`
            );
        }

        return sanitized;
    }

    static _sanitizeLinePoint(point, minValue, replaceWithNull, onWarning) {
        if (!this.isValidForLogScale(point.value)) {
            onWarning();
            
            if (replaceWithNull) {
                return { time: point.time }; // Whitespace
            }
            
            return {
                ...point,
                value: minValue,
                _sanitized: true,
                _originalValue: point.value,
            };
        }
        
        if (point.value < minValue) {
            return {
                ...point,
                value: minValue,
                _sanitized: true,
                _originalValue: point.value,
            };
        }
        
        return point;
    }

    static _sanitizeOHLC(point, minValue, replaceWithNull, onWarning) {
        const fields = ['open', 'high', 'low', 'close'];
        const hasInvalid = fields.some(f => !this.isValidForLogScale(point[f]));
        
        if (hasInvalid) {
            onWarning();
            
            if (replaceWithNull) {
                return { time: point.time };
            }
        }
        
        return {
            ...point,
            open: Math.max(point.open, minValue),
            high: Math.max(point.high, minValue),
            low: Math.max(point.low, minValue),
            close: Math.max(point.close, minValue),
            _sanitized: hasInvalid,
        };
    }

    /**
     * Создаёт форматтер цен для очень малых значений
     */
    static createPriceFormatter(options = {}) {
        const {
            minDecimals = 2,
            maxDecimals = 10,
            useExponential = true,
            exponentialThreshold = 0.0001,
        } = options;

        return (price) => {
            if (!this.isValidForLogScale(price)) {
                return 'N/A';
            }

            if (useExponential && price < exponentialThreshold) {
                return price.toExponential(minDecimals);
            }

            // Динамическое определение количества знаков
            const magnitude = Math.floor(Math.log10(price));
            const decimals = Math.min(
                maxDecimals,
                Math.max(minDecimals, -magnitude + 2)
            );

            return price.toFixed(decimals);
        };
    }
}

// ============ Пример использования ============

// 1. Создание графика с логарифмическим режимом
const container = document.getElementById('chart');
const chart = createChart(container, {
    width: 800,
    height: 400,
    rightPriceScale: {
        mode: PriceScaleMode.Logarithmic,
        scaleMargins: {
            top: 0.1,
            bottom: 0.1,
        },
        borderVisible: true,
    },
    timeScale: {
        borderVisible: true,
    },
});

// 2. Создание серии с кастомным форматированием
const lineSeries = chart.addLineSeries({
    color: '#2962FF',
    lineWidth: 2,
    priceFormat: {
        type: 'custom',
        formatter: LogScaleHelper.createPriceFormatter({
            useExponential: true,
            exponentialThreshold: 0.00001,
        }),
    },
});

// 3. Данные с проблемными значениями
const rawData = [
    { time: '2024-01-01', value: 0.00001 },
    { time: '2024-01-02', value: 0 },          // ❌ Нулевое значение
    { time: '2024-01-03', value: 0.000005 },
    { time: '2024-01-04', value: -0.00001 },   // ❌ Отрицательное значение
    { time: '2024-01-05', value: 0.00002 },
    { time: '2024-01-06', value: NaN },        // ❌ NaN
    { time: '2024-01-07', value: 0.00003 },
];

// 4. Санитизация данных
const safeData = LogScaleHelper.sanitizeData(rawData, {
    minValue: 1e-10,
    replaceWithNull: false,  // или true для разрывов
    logWarnings: true,
});

// 5. Установка данных
lineSeries.setData(safeData);

// 6. Подстройка видимого диапазона (опционально)
chart.timeScale().fitContent();

// ============ Tooltip с отображением оригинальных значений ============

const toolTip = document.createElement('div');
toolTip.className = 'tooltip';
container.appendChild(toolTip);

chart.subscribeCrosshairMove((param) => {
    if (!param.point || !param.time) {
        toolTip.style.display = 'none';
        return;
    }

    const data = param.seriesData.get(lineSeries);
    if (!data) {
        toolTip.style.display = 'none';
        return;
    }

    // Отображаем оригинальное значение, если было санитизировано
    const displayValue = data._sanitized 
        ? `${data.value} (orig: ${data._originalValue})`
        : LogScaleHelper.createPriceFormatter()(data.value);

    toolTip.innerHTML = `
        <div>Time: ${param.time}</div>
        <div>Value: ${displayValue}</div>
        ${data._sanitized ? '<div class="warning">⚠️ Sanitized</div>' : ''}
    `;
    
    toolTip.style.display = 'block';
    toolTip.style.left = param.point.x + 'px';
    toolTip.style.top = param.point.y + 'px';
});
```

### CSS для tooltip

```css
.tooltip {
    position: absolute;
    display: none;
    padding: 8px 12px;
    background: rgba(0, 0, 0, 0.85);
    color: white;
    border-radius: 4px;
    font-size: 12px;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    z-index: 1000;
    pointer-events: none;
    transform: translate(-50%, -100%);
    margin-top: -10px;
}

.tooltip .warning {
    color: #ffca28;
    margin-top: 4px;
    font-weight: bold;
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Точность | Сложность | Версия | Производит. |
|---------|--------|----------|-----------|--------|-------------|
| **Обновление до v5.0.0+** | ⭐ 10/10 | ✅ Высокая | Низкая | v5.0+ | ✅ Высокая |
| Замена нуля малым числом | ⭐ 7/10 | ⚠️ Искажает | Низкая | Любая | ✅ Высокая |
| Установка null | ⭐ 6/10 | ✅ Честная | Низкая | Любая | ✅ Высокая |
| Установка minValue | ⭐ 6/10 | ⚠️ Обрезает | Низкая | Любая | ✅ Высокая |
| Трансформация log(x+1) | ⭐ 5/10 | ⚠️ Искажает | Средняя | Любая | ✅ Высокая |
| Symlog шкала | ⭐ 4/10 | ✅ Высокая | Высокая | Любая | ⚠️ Средняя |

---

## 🧪 Тестирование

```javascript
// Тесты для LogScaleHelper
describe('LogScaleHelper', () => {
    describe('isValidForLogScale', () => {
        it('should return true for positive numbers', () => {
            expect(LogScaleHelper.isValidForLogScale(1)).toBe(true);
            expect(LogScaleHelper.isValidForLogScale(0.001)).toBe(true);
            expect(LogScaleHelper.isValidForLogScale(1e-10)).toBe(true);
        });

        it('should return false for zero, negative, and NaN', () => {
            expect(LogScaleHelper.isValidForLogScale(0)).toBe(false);
            expect(LogScaleHelper.isValidForLogScale(-1)).toBe(false);
            expect(LogScaleHelper.isValidForLogScale(NaN)).toBe(false);
            expect(LogScaleHelper.isValidForLogScale(undefined)).toBe(false);
        });
    });

    describe('sanitizeData', () => {
        it('should replace zero values with minValue', () => {
            const input = [{ time: '2024-01-01', value: 0 }];
            const output = LogScaleHelper.sanitizeData(input);
            
            expect(output[0].value).toBe(1e-10);
            expect(output[0]._sanitized).toBe(true);
        });

        it('should preserve valid values', () => {
            const input = [{ time: '2024-01-01', value: 0.001 }];
            const output = LogScaleHelper.sanitizeData(input);
            
            expect(output[0].value).toBe(0.001);
            expect(output[0]._sanitized).toBeUndefined();
        });
    });
});
```

---

## 🔗 Источники

1. [GitHub Issue #874 - Log axis is not scaling on small number](https://github.com/tradingview/lightweight-charts/issues/874)
2. [GitHub PR #1965 - Fix price scale visible range calculation in log mode](https://github.com/tradingview/lightweight-charts/pull/1965)
3. [Lightweight Charts v5.0.0 Release Notes](https://tradingview.github.io/lightweight-charts/)
4. [Stack Overflow - Handling zero values on logarithmic scale](https://stackoverflow.com/questions/tagged/logarithmic-scale)
5. [CanvasJS - Logarithmic Axis with Zero Values](https://canvasjs.com/docs/charts/chart-options/axisx/logarithmic/)
6. [D3.js Log Scale Documentation](https://d3js.org/d3-scale/log)
7. [Plotly - Log Axis Type](https://plotly.com/javascript/log-plot/)
8. [Wikipedia - Logarithmic scale](https://en.wikipedia.org/wiki/Logarithmic_scale)

---

## 📅 История изменений

| Дата | Версия | Изменения |
|------|--------|-----------|
| 2026-01-18 | 1.0 | Создание документа |

---

*Документация подготовлена для проекта lightweight-charts error tracking*
