# БАГ #12: Автоскейл игнорирует price lines

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#1587](https://github.com/tradingview/lightweight-charts/issues/1587)  
> **Версии:** v5.0+  
> **Статус:** 🔴 Open  
> **Платформы:** Все браузеры

---

## 📋 Описание проблемы

При использовании `priceScale().applyOptions({ autoScale: true })` горизонтальные линии (price lines), добавленные через `createPriceLine()`, **не учитываются** при автоматическом масштабировании ценовой шкалы.

### Симптомы:
- Price lines могут выходить за пределы видимой области графика
- Масштабирование учитывает только данные серий (свечи, линии)
- Stop-loss, take-profit, и другие критические уровни становятся невидимыми

### Сценарии воспроизведения:
1. **Trading apps:** Stop-loss/take-profit линии далеко от текущей цены
2. **Liquidation levels:** "Soft liquidation" и "hard liquidation" линии
3. **Support/Resistance:** Важные уровни вне видимого диапазона данных

### Пример проблемного кода:
```javascript
const series = chart.addCandlestickSeries();
series.setData(candleData);

// Добавляем price line далеко от текущей цены
series.createPriceLine({
    price: 50000,  // Допустим, текущая цена 45000
    color: 'red',
    lineWidth: 2,
    title: 'Stop Loss'
});

// autoScale НЕ включит эту линию в расчёт видимого диапазона!
chart.priceScale('right').applyOptions({ autoScale: true });
```

---

## 🔍 Найденные решения

### Решение 1: autoscaleInfoProvider (Расширение диапазона)
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

Использование `autoscaleInfoProvider` для явного включения price lines в расчёт диапазона.

**Преимущества:**
- ✅ Официальный и документированный подход
- ✅ Полный контроль над логикой масштабирования
- ✅ Работает с любым количеством price lines
- ✅ Динамически обновляется при изменении линий

**Недостатки:**
- ⚠️ Требует ручного трекинга всех price lines
- ⚠️ Немного увеличивает сложность кода

```javascript
// Хранилище для price lines
const priceLines = [];

const series = chart.addCandlestickSeries({
    autoscaleInfoProvider: (original) => {
        const res = original();
        
        if (res !== null && priceLines.length > 0) {
            // Получаем все значения price lines
            const linePrices = priceLines.map(line => line.options().price);
            const minLinePrice = Math.min(...linePrices);
            const maxLinePrice = Math.max(...linePrices);
            
            // Расширяем диапазон, включая price lines
            res.priceRange.minValue = Math.min(res.priceRange.minValue, minLinePrice);
            res.priceRange.maxValue = Math.max(res.priceRange.maxValue, maxLinePrice);
        }
        
        return res;
    }
});

// Создаём price lines и сохраняем ссылки
const stopLossLine = series.createPriceLine({
    price: 42000,
    color: '#ef5350',
    lineWidth: 2,
    title: 'Stop Loss',
    axisLabelVisible: true
});
priceLines.push(stopLossLine);

const takeProfitLine = series.createPriceLine({
    price: 52000,
    color: '#26a69a',
    lineWidth: 2,
    title: 'Take Profit',
    axisLabelVisible: true
});
priceLines.push(takeProfitLine);
```

---

### Решение 2: Класс-обёртка для автоматического трекинга
**Оценка: 8/10**

Создание класса, который автоматически управляет price lines и autoscale.

**Преимущества:**
- ✅ Инкапсулирует всю логику
- ✅ Простое API для добавления/удаления линий
- ✅ Переиспользуемый код

**Недостатки:**
- ⚠️ Дополнительная абстракция
- ⚠️ Требует изменения архитектуры существующего кода

```javascript
class PriceLineManager {
    constructor(series) {
        this.series = series;
        this.priceLines = new Map();
        this._setupAutoscale();
    }
    
    _setupAutoscale() {
        this.series.applyOptions({
            autoscaleInfoProvider: (original) => {
                const res = original();
                if (res !== null && this.priceLines.size > 0) {
                    const prices = Array.from(this.priceLines.values())
                        .map(line => line.options().price);
                    res.priceRange.minValue = Math.min(res.priceRange.minValue, ...prices);
                    res.priceRange.maxValue = Math.max(res.priceRange.maxValue, ...prices);
                }
                return res;
            }
        });
    }
    
    addPriceLine(id, options) {
        const line = this.series.createPriceLine(options);
        this.priceLines.set(id, line);
        return line;
    }
    
    removePriceLine(id) {
        const line = this.priceLines.get(id);
        if (line) {
            this.series.removePriceLine(line);
            this.priceLines.delete(id);
        }
    }
    
    updatePriceLine(id, options) {
        const line = this.priceLines.get(id);
        if (line) {
            line.applyOptions(options);
        }
    }
    
    clear() {
        this.priceLines.forEach(line => this.series.removePriceLine(line));
        this.priceLines.clear();
    }
}

// Использование:
const manager = new PriceLineManager(candleSeries);
manager.addPriceLine('stop-loss', { price: 42000, color: 'red', title: 'SL' });
manager.addPriceLine('take-profit', { price: 52000, color: 'green', title: 'TP' });
```

---

### Решение 3: Отключение autoscale + scaleMargins
**Оценка: 5/10**

Ручное управление масштабом с фиксированными отступами.

**Преимущества:**
- ✅ Простая реализация
- ✅ Предсказуемое поведение

**Недостатки:**
- ❌ Не динамический - не адаптируется к данным
- ❌ Требует ручного пересчёта при изменении данных
- ❌ Может приводить к слишком сжатому или растянутому графику

```javascript
// Отключаем autoscale
chart.priceScale('right').applyOptions({
    autoScale: false,
    scaleMargins: {
        top: 0.1,  // 10% отступ сверху
        bottom: 0.1 // 10% отступ снизу
    }
});

// Ручной расчёт диапазона
function calculateVisibleRange(data, priceLines) {
    const dataMin = Math.min(...data.map(d => d.low || d.value));
    const dataMax = Math.max(...data.map(d => d.high || d.value));
    const lineMin = Math.min(...priceLines.map(l => l.options().price));
    const lineMax = Math.max(...priceLines.map(l => l.options().price));
    
    return {
        min: Math.min(dataMin, lineMin),
        max: Math.max(dataMax, lineMax)
    };
}
```

---

### Решение 4: Добавление padding через margins
**Оценка: 4/10**

Увеличение отступов, чтобы вместить возможные price lines.

**Преимущества:**
- ✅ Минимальные изменения кода
- ✅ Работает "из коробки"

**Недостатки:**
- ❌ Не гарантирует видимость конкретных линий
- ❌ Может создать слишком большие отступы
- ❌ Не масштабируется

```javascript
// Увеличиваем margins для "запаса"
const series = chart.addCandlestickSeries({
    priceScaleId: 'right',
});

chart.priceScale('right').applyOptions({
    autoScale: true,
    scaleMargins: {
        top: 0.3,    // 30% сверху
        bottom: 0.3  // 30% снизу
    }
});
```

---

## ✅ Рекомендуемое решение

### Полный рабочий пример с autoscaleInfoProvider

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// РЕШЕНИЕ: Price Lines с корректным autoscale
// ============================================

class ChartWithPriceLines {
    constructor(container, options = {}) {
        this.priceLines = [];
        this.chart = createChart(container, {
            width: options.width || 800,
            height: options.height || 400,
            layout: {
                background: { type: 'solid', color: '#1e222d' },
                textColor: '#d1d4dc',
            },
            grid: {
                vertLines: { color: '#2B2B43' },
                horzLines: { color: '#2B2B43' },
            },
            rightPriceScale: {
                autoScale: true,
                borderColor: '#2B2B43',
            },
            timeScale: {
                borderColor: '#2B2B43',
            },
        });
        
        this._initSeries();
    }
    
    _initSeries() {
        // Создаём серию с кастомным autoscaleInfoProvider
        this.series = this.chart.addCandlestickSeries({
            upColor: '#26a69a',
            downColor: '#ef5350',
            borderVisible: false,
            wickUpColor: '#26a69a',
            wickDownColor: '#ef5350',
            
            // 🔑 КЛЮЧЕВОЕ: включаем price lines в autoscale
            autoscaleInfoProvider: (original) => {
                const res = original();
                
                if (res !== null && this.priceLines.length > 0) {
                    const linePrices = this.priceLines
                        .filter(line => line !== null)
                        .map(line => {
                            try {
                                return line.options().price;
                            } catch {
                                return null;
                            }
                        })
                        .filter(price => price !== null);
                    
                    if (linePrices.length > 0) {
                        const minLinePrice = Math.min(...linePrices);
                        const maxLinePrice = Math.max(...linePrices);
                        
                        // Добавляем небольшой padding (2%) для красоты
                        const padding = (maxLinePrice - minLinePrice) * 0.02 || 
                                        res.priceRange.maxValue * 0.02;
                        
                        res.priceRange.minValue = Math.min(
                            res.priceRange.minValue, 
                            minLinePrice - padding
                        );
                        res.priceRange.maxValue = Math.max(
                            res.priceRange.maxValue, 
                            maxLinePrice + padding
                        );
                    }
                }
                
                return res;
            }
        });
    }
    
    setData(data) {
        this.series.setData(data);
        return this;
    }
    
    addPriceLine(options) {
        const defaultOptions = {
            lineWidth: 2,
            lineStyle: 0, // Solid
            axisLabelVisible: true,
        };
        
        const line = this.series.createPriceLine({
            ...defaultOptions,
            ...options
        });
        
        this.priceLines.push(line);
        
        // Принудительно обновляем autoscale
        this.chart.timeScale().fitContent();
        
        return line;
    }
    
    removePriceLine(line) {
        const index = this.priceLines.indexOf(line);
        if (index > -1) {
            this.series.removePriceLine(line);
            this.priceLines.splice(index, 1);
        }
    }
    
    addStopLoss(price, title = 'Stop Loss') {
        return this.addPriceLine({
            price,
            color: '#ef5350',
            title,
            lineStyle: 2, // Dashed
        });
    }
    
    addTakeProfit(price, title = 'Take Profit') {
        return this.addPriceLine({
            price,
            color: '#26a69a',
            title,
            lineStyle: 2, // Dashed
        });
    }
    
    addEntryLevel(price, title = 'Entry') {
        return this.addPriceLine({
            price,
            color: '#2196f3',
            title,
            lineStyle: 0, // Solid
        });
    }
    
    destroy() {
        this.priceLines = [];
        this.chart.remove();
    }
}

// ============================================
// ПРИМЕР ИСПОЛЬЗОВАНИЯ
// ============================================

const container = document.getElementById('chart');
const chart = new ChartWithPriceLines(container, {
    width: 1000,
    height: 500
});

// Генерируем тестовые данные
const data = generateCandleData(100, 45000, 47000);
chart.setData(data);

// Добавляем trading levels
chart.addEntryLevel(46000, 'Entry @46000');
chart.addStopLoss(42000, 'SL @42000');      // Далеко от данных!
chart.addTakeProfit(52000, 'TP @52000');    // Тоже далеко!

// Все линии будут видны благодаря autoscaleInfoProvider!

// Вспомогательная функция генерации данных
function generateCandleData(count, minPrice, maxPrice) {
    const data = [];
    let time = Math.floor(Date.now() / 1000) - count * 3600;
    let price = (minPrice + maxPrice) / 2;
    
    for (let i = 0; i < count; i++) {
        const volatility = (Math.random() - 0.5) * 500;
        const open = price;
        const close = price + volatility;
        const high = Math.max(open, close) + Math.random() * 200;
        const low = Math.min(open, close) - Math.random() * 200;
        
        data.push({
            time: time + i * 3600,
            open: Math.max(minPrice, Math.min(maxPrice, open)),
            high: Math.max(minPrice, Math.min(maxPrice, high)),
            low: Math.max(minPrice, Math.min(maxPrice, low)),
            close: Math.max(minPrice, Math.min(maxPrice, close))
        });
        
        price = close;
    }
    
    return data;
}
```

---

## 📊 Сравнительная таблица

| Решение | Оценка | Сложность | Динамичность | Production Ready |
|---------|--------|-----------|--------------|------------------|
| **autoscaleInfoProvider** | 9/10 ⭐ | Средняя | ✅ Да | ✅ Да |
| Класс-обёртка | 8/10 | Высокая | ✅ Да | ✅ Да |
| Отключение autoscale | 5/10 | Низкая | ❌ Нет | ⚠️ Частично |
| Увеличение margins | 4/10 | Минимальная | ❌ Нет | ❌ Нет |

---

## 🔗 Источники

1. **GitHub Issue #1587** - [autoScale: true ignores horizontal lines](https://github.com/tradingview/lightweight-charts/issues/1587)
2. **Документация autoscaleInfoProvider** - [Price Scale Options](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/SeriesOptionsCommon#autoscaleinfoprovider)
3. **API Reference: createPriceLine** - [Price Lines Documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#createpriceline)
4. **Stack Overflow** - Примеры использования autoscaleInfoProvider
5. **GitHub Discussions** - Community workarounds для autoscale issues

---

## 💡 Дополнительные советы

1. **Производительность:** При большом количестве price lines кэшируйте значения вместо вызова `options()` каждый раз
2. **React/Vue:** Используйте `useMemo`/`computed` для массива priceLines
3. **Real-time updates:** При изменении цены линии вызывайте `chart.timeScale().fitContent()` для обновления масштаба
4. **TypeScript:** Типизируйте price lines как `IPriceLine[]` из `lightweight-charts`
