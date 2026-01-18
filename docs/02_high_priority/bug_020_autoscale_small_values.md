# БАГ #20: AutoScale некорректен для малых значений

> **Критичность:** 🟠 ВЫСОКАЯ  
> **Issue:** [#1495](https://github.com/tradingview/lightweight-charts/issues/1495)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (needs investigation)  
> **Последнее обновление:** январь 2026

---

## 📋 Описание проблемы

### Суть бага
Автоматическое масштабирование (`autoScale: true`) **не работает корректно** для очень малых значений (< 0.001). Особенно критично для:
- Криптовалют с низкой ценой (SHIB, PEPE, BONK, FLOKI и т.д.)
- Научных данных с малыми величинами
- Технических индикаторов с нормализованными значениями

### Технические причины
1. **Floating-point precision** — JavaScript использует IEEE 754 double precision, что приводит к ошибкам при работе с очень малыми числами
2. **Внутренние расчёты scale** — алгоритм масштабирования не учитывает специфику extremely small values
3. **Visibility toggle** — при переключении видимости серий autoscale не пересчитывается корректно

### Симптомы
- ❌ При `autoScale: true` scale "делает ничего" для малых значений
- ❌ После скрытия серии с большими значениями, scale для малых не пересчитывается
- ❌ Данные могут отображаться как горизонтальная линия (неразличимые колебания)
- ❌ Price axis показывает некорректные значения или формат

### Сценарии воспроизведения
```typescript
// Проблемный сценарий
const chart = createChart(container, {
  leftPriceScale: { autoScale: true, visible: true },
  rightPriceScale: { visible: false }
});

const shibSeries = chart.addLineSeries({
  priceScaleId: 'left',
});

// Цена SHIB: ~0.00001234
shibSeries.setData([
  { time: '2024-01-01', value: 0.00001234 },
  { time: '2024-01-02', value: 0.00001256 },
  { time: '2024-01-03', value: 0.00001198 },
  // ...
]);
// AutoScale может не сработать корректно!
```

### Частота
**100%** для values < 0.001 в определённых сценариях

---

## 🔍 Найденные решения

### Решение 1: Правильная настройка priceFormat
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

Явное указание `minMove` и `precision` для корректного отображения малых значений:

```typescript
import { createChart, PriceFormat } from 'lightweight-charts';

// Функция для определения оптимального precision
function calculatePrecision(minValue: number): number {
  if (minValue === 0) return 8;
  
  const absValue = Math.abs(minValue);
  if (absValue >= 1) return 2;
  
  // Подсчитываем количество нулей после точки
  const str = absValue.toExponential();
  const exp = parseInt(str.split('e')[1]);
  
  // Precision = количество знаков после запятой + 2 для точности
  return Math.max(0, -exp + 2);
}

// Функция для определения minMove
function calculateMinMove(precision: number): number {
  return Math.pow(10, -precision);
}

// Пример для SHIB (цена ~0.00001)
const priceRange = data.reduce(
  (acc, d) => ({
    min: Math.min(acc.min, d.value || d.close || 0),
    max: Math.max(acc.max, d.value || d.close || 0)
  }),
  { min: Infinity, max: -Infinity }
);

const precision = calculatePrecision(priceRange.min);
const minMove = calculateMinMove(precision);

const series = chart.addLineSeries({
  priceFormat: {
    type: 'price',
    precision: precision, // например, 8 для SHIB
    minMove: minMove,     // например, 0.00000001
  },
  priceScaleId: 'left',
});

console.log(`Price format: precision=${precision}, minMove=${minMove}`);
// Для SHIB: precision=8, minMove=0.00000001
```

**Преимущества:**
- ✅ Правильное отображение значений на оси
- ✅ Корректные расчёты scale
- ✅ Нативная поддержка библиотеки

**Недостатки:**
- ⚠️ Требует предварительного анализа данных
- ⚠️ Не решает проблему полностью в edge cases

---

### Решение 2: Нормализация данных (Scale Factor)
**Оценка: 8/10**

Умножение данных на scale factor для работы в "нормальном" диапазоне:

```typescript
interface NormalizedData {
  original: number[];
  normalized: number[];
  scaleFactor: number;
  format: (value: number) => string;
}

class PriceNormalizer {
  private scaleFactor: number = 1;
  private precision: number = 2;

  /**
   * Нормализует массив цен для корректного отображения
   */
  normalize(prices: number[]): NormalizedData {
    if (prices.length === 0) {
      return {
        original: [],
        normalized: [],
        scaleFactor: 1,
        format: (v) => v.toFixed(2)
      };
    }

    const minPrice = Math.min(...prices.filter(p => p > 0));
    
    // Определяем scale factor
    if (minPrice < 0.000001) {
      this.scaleFactor = 100000000; // 1e8 для очень малых
      this.precision = 8;
    } else if (minPrice < 0.0001) {
      this.scaleFactor = 10000; // 1e4
      this.precision = 4;
    } else if (minPrice < 0.01) {
      this.scaleFactor = 100;
      this.precision = 6;
    } else {
      this.scaleFactor = 1;
      this.precision = 2;
    }

    const normalized = prices.map(p => p * this.scaleFactor);

    return {
      original: prices,
      normalized: normalized,
      scaleFactor: this.scaleFactor,
      format: (value: number) => this.formatPrice(value)
    };
  }

  /**
   * Форматирует нормализованное значение обратно в оригинальное
   */
  formatPrice(normalizedValue: number): string {
    const original = normalizedValue / this.scaleFactor;
    return original.toFixed(this.precision);
  }

  /**
   * Денормализует значение
   */
  denormalize(normalizedValue: number): number {
    return normalizedValue / this.scaleFactor;
  }
}

// Использование
const normalizer = new PriceNormalizer();
const { normalized, format, scaleFactor } = normalizer.normalize(
  data.map(d => d.value)
);

const series = chart.addLineSeries({
  // Работаем с нормализованными данными
  priceFormat: {
    type: 'custom',
    formatter: (price: number) => format(price),
    minMove: 0.01,
  },
});

// Конвертируем данные
const normalizedData = data.map((d, i) => ({
  time: d.time,
  value: normalized[i]
}));

series.setData(normalizedData);

console.log(`Applied scale factor: ${scaleFactor}x`);
```

**Преимущества:**
- ✅ AutoScale работает корректно с "нормальными" числами
- ✅ Визуально данные отображаются правильно
- ✅ Совместимо с любой версией библиотеки

**Недостатки:**
- ⚠️ Усложняет логику (нужно помнить о scale factor)
- ⚠️ Crosshair values требуют обратной конвертации

---

### Решение 3: Custom Price Formatter + Manual Scale
**Оценка: 7/10**

Ручное управление scale в сочетании с custom formatter:

```typescript
class CryptoPriceScale {
  private series: ISeriesApi<'Line'>;
  private chart: IChartApi;
  private data: LineData[];

  constructor(chart: IChartApi) {
    this.chart = chart;
    this.data = [];
    
    this.series = chart.addLineSeries({
      priceFormat: {
        type: 'custom',
        formatter: this.formatPrice.bind(this),
      },
    });
  }

  setData(data: LineData[]) {
    this.data = data;
    this.series.setData(data);
    this.applyOptimalScale();
  }

  private formatPrice(price: number): string {
    if (price === 0) return '0';
    
    const absPrice = Math.abs(price);
    
    if (absPrice >= 1000) {
      return price.toLocaleString('en-US', { 
        maximumFractionDigits: 2 
      });
    } else if (absPrice >= 1) {
      return price.toFixed(2);
    } else if (absPrice >= 0.01) {
      return price.toFixed(4);
    } else if (absPrice >= 0.0001) {
      return price.toFixed(6);
    } else {
      // Для очень малых используем экспоненциальную нотацию
      // или фиксированное количество значащих цифр
      return price.toExponential(4);
      // Или: return price.toPrecision(4);
    }
  }

  private applyOptimalScale() {
    if (this.data.length === 0) return;

    const values = this.data.map(d => d.value);
    const minVal = Math.min(...values);
    const maxVal = Math.max(...values);
    
    // Вычисляем оптимальные margins
    const range = maxVal - minVal;
    const margin = range * 0.1; // 10% margin

    // Применяем ручной scale
    this.series.priceScale().applyOptions({
      autoScale: false, // Отключаем автоскейл
      scaleMargins: {
        top: 0.1,
        bottom: 0.1,
      },
    });

    // Устанавливаем visible range вручную
    // Note: setVisiblePriceRange не существует напрямую,
    // используем workaround через данные
    this.chart.timeScale().fitContent();
  }

  /**
   * Пересчитать scale после изменения видимости
   */
  recalculateScale() {
    const priceScale = this.series.priceScale();
    
    // Toggle autoScale для принудительного пересчёта
    priceScale.applyOptions({ autoScale: false });
    
    setTimeout(() => {
      priceScale.applyOptions({ autoScale: true });
    }, 0);
  }
}
```

**Преимущества:**
- ✅ Полный контроль над форматированием
- ✅ Адаптивное отображение для любых значений
- ✅ Поддержка экспоненциальной нотации

**Недостатки:**
- ⚠️ Сложная реализация
- ⚠️ recalculateScale() — хрупкий workaround

---

### Решение 4: Visibility Toggle Workaround
**Оценка: 6/10**

Обход проблемы при переключении видимости серий:

```typescript
class MultiSeriesChart {
  private chart: IChartApi;
  private seriesMap: Map<string, ISeriesApi<SeriesType>>;
  private seriesData: Map<string, any[]>;

  constructor(container: HTMLElement) {
    this.chart = createChart(container, {
      leftPriceScale: { 
        autoScale: true, 
        visible: true 
      },
      rightPriceScale: { 
        visible: false 
      }
    });
    
    this.seriesMap = new Map();
    this.seriesData = new Map();
  }

  addSeries(id: string, data: any[], options: any = {}) {
    const series = this.chart.addLineSeries({
      priceScaleId: 'left',
      ...options
    });
    series.setData(data);
    
    this.seriesMap.set(id, series);
    this.seriesData.set(id, data);
  }

  /**
   * Скрытие серии через удаление (workaround)
   */
  hideSeries(id: string): boolean {
    const series = this.seriesMap.get(id);
    if (!series) return false;

    // Удаляем серию вместо скрытия
    this.chart.removeSeries(series);
    this.seriesMap.delete(id);

    // Принудительно пересчитываем scale
    this.forceAutoScaleRecalculation();

    return true;
  }

  /**
   * Показ серии через пересоздание
   */
  showSeries(id: string, options: any = {}): boolean {
    const data = this.seriesData.get(id);
    if (!data) return false;

    // Пересоздаём серию
    const series = this.chart.addLineSeries({
      priceScaleId: 'left',
      ...options
    });
    series.setData(data);
    
    this.seriesMap.set(id, series);
    
    // Пересчитываем scale
    this.forceAutoScaleRecalculation();

    return true;
  }

  /**
   * WORKAROUND: Принудительный пересчёт autoScale
   * Из GitHub issue #1495
   */
  private forceAutoScaleRecalculation() {
    const leftScale = this.chart.priceScale('left');
    const rightScale = this.chart.priceScale('right');

    // Шаг 1: Делаем обе шкалы видимыми
    leftScale.applyOptions({ visible: true });
    rightScale.applyOptions({ visible: true });

    // Шаг 2: Скрываем правую шкалу
    setTimeout(() => {
      rightScale.applyOptions({ visible: false });
      
      // Шаг 3: Применяем fitContent для пересчёта
      this.chart.timeScale().fitContent();
    }, 10);
  }

  /**
   * Альтернативный workaround: Toggle autoScale
   */
  private forceAutoScaleRecalculationV2() {
    const scale = this.chart.priceScale('left');
    
    // Disable then re-enable autoScale
    scale.applyOptions({ autoScale: false });
    
    requestAnimationFrame(() => {
      scale.applyOptions({ autoScale: true });
    });
  }
}
```

**Преимущества:**
- ✅ Решает проблему visibility toggle
- ✅ Проверенный workaround из GitHub

**Недостатки:**
- ⚠️ Хрупкое решение с таймаутами
- ⚠️ Может вызывать визуальные glitches
- ⚠️ Полное пересоздание серий — не оптимально

---

### Решение 5: Logarithmic Scale для экстремальных диапазонов
**Оценка: 5/10**

Использование логарифмической шкалы (с ограничениями):

```typescript
// ВНИМАНИЕ: Логарифмическая шкала имеет ограничения
// Не работает для значений <= 0

const chart = createChart(container, {
  leftPriceScale: {
    mode: 1, // Logarithmic mode
    autoScale: true,
    visible: true,
  },
});

const series = chart.addLineSeries({
  priceScaleId: 'left',
  priceFormat: {
    type: 'custom',
    formatter: (price: number) => {
      // Custom formatter для логарифмической шкалы
      if (price >= 1) {
        return price.toFixed(2);
      } else if (price >= 0.0001) {
        return price.toFixed(6);
      } else {
        return price.toExponential(4);
      }
    },
  },
});

// ВАЖНО: Данные должны быть > 0 для log scale!
const validData = data.filter(d => d.value > 0);
series.setData(validData);
```

**Преимущества:**
- ✅ Хорошо показывает относительные изменения
- ✅ Компактное отображение широких диапазонов

**Недостатки:**
- ⚠️ Не работает для значений <= 0
- ⚠️ Есть известные баги с log scale (#874)
- ⚠️ Может быть неинтуитивно для пользователей

---

## ✅ Рекомендуемое решение

### Комплексный подход: Price Format + Normalizer

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi, 
  LineData,
  PriceFormatCustom 
} from 'lightweight-charts';

/**
 * Утилита для работы с очень малыми ценами (SHIB, PEPE и т.д.)
 */
class SmallValueChartManager {
  private chart: IChartApi;
  private series: ISeriesApi<'Line'> | null = null;
  private scaleFactor: number = 1;
  private originalPrecision: number = 8;

  constructor(container: HTMLElement) {
    this.chart = createChart(container, {
      layout: {
        background: { color: '#1a1a2e' },
        textColor: '#e0e0e0',
      },
      grid: {
        vertLines: { color: '#2a2a4a' },
        horzLines: { color: '#2a2a4a' },
      },
      leftPriceScale: {
        autoScale: true,
        visible: true,
        borderColor: '#2a2a4a',
      },
      rightPriceScale: {
        visible: false,
      },
    });
  }

  /**
   * Анализирует данные и возвращает оптимальные настройки
   */
  private analyzeData(data: LineData[]): {
    scaleFactor: number;
    precision: number;
    minMove: number;
  } {
    const values = data.map(d => Math.abs(d.value)).filter(v => v > 0);
    if (values.length === 0) {
      return { scaleFactor: 1, precision: 2, minMove: 0.01 };
    }

    const minValue = Math.min(...values);
    
    // Определяем количество ведущих нулей
    let precision = 0;
    let testValue = minValue;
    
    while (testValue < 1 && precision < 18) {
      testValue *= 10;
      precision++;
    }
    
    // Добавляем 2 знака для точности
    precision = Math.min(precision + 2, 18);
    
    // Scale factor для нормализации
    let scaleFactor = 1;
    if (minValue < 0.000001) {
      scaleFactor = 1e9;
    } else if (minValue < 0.0001) {
      scaleFactor = 1e6;
    } else if (minValue < 0.01) {
      scaleFactor = 1e4;
    } else if (minValue < 1) {
      scaleFactor = 1e2;
    }

    const minMove = Math.pow(10, -precision);

    return { scaleFactor, precision, minMove };
  }

  /**
   * Форматирует цену с учётом scale factor
   */
  private createPriceFormatter(scaleFactor: number, precision: number): 
    (price: number) => string {
    return (price: number) => {
      const original = price / scaleFactor;
      
      if (original === 0) return '0';
      
      const absOriginal = Math.abs(original);
      
      // Для очень малых значений используем компактный формат
      if (absOriginal < 0.0001) {
        // Формат: $0.0₆1234 (subscript для количества нулей)
        const expMatch = original.toExponential().match(/e([+-]\d+)/);
        const exp = expMatch ? parseInt(expMatch[1]) : 0;
        
        if (exp < -4) {
          // Показываем как 0.0{n}xxxxx
          const zeros = Math.abs(exp) - 1;
          const mantissa = original * Math.pow(10, Math.abs(exp));
          const mantissaStr = mantissa.toFixed(4).replace('0.', '');
          return `0.0{${zeros}}${mantissaStr}`;
        }
      }
      
      // Стандартное форматирование
      if (absOriginal >= 1000) {
        return original.toLocaleString('en-US', { 
          maximumFractionDigits: 2 
        });
      } else if (absOriginal >= 1) {
        return original.toFixed(2);
      } else if (absOriginal >= 0.01) {
        return original.toFixed(4);
      } else if (absOriginal >= 0.0001) {
        return original.toFixed(6);
      } else {
        return original.toFixed(precision);
      }
    };
  }

  /**
   * Устанавливает данные с автоматической оптимизацией для малых значений
   */
  setData(data: LineData[]) {
    // Анализируем данные
    const { scaleFactor, precision, minMove } = this.analyzeData(data);
    this.scaleFactor = scaleFactor;
    this.originalPrecision = precision;

    console.log(`SmallValueChart: scaleFactor=${scaleFactor}, precision=${precision}`);

    // Удаляем старую серию если есть
    if (this.series) {
      this.chart.removeSeries(this.series);
    }

    // Создаём серию с оптимальным price format
    this.series = this.chart.addLineSeries({
      priceScaleId: 'left',
      color: '#26a69a',
      lineWidth: 2,
      priceFormat: {
        type: 'custom',
        formatter: this.createPriceFormatter(scaleFactor, precision),
        minMove: minMove / scaleFactor, // Adjust for scale
      } as PriceFormatCustom,
    });

    // Нормализуем данные
    const normalizedData = data.map(d => ({
      time: d.time,
      value: d.value * scaleFactor,
    }));

    this.series.setData(normalizedData);
    
    // Fit content после установки данных
    this.chart.timeScale().fitContent();
  }

  /**
   * Обновляет одну точку данных
   */
  update(dataPoint: LineData) {
    if (!this.series) return;
    
    this.series.update({
      time: dataPoint.time,
      value: dataPoint.value * this.scaleFactor,
    });
  }

  /**
   * Получает оригинальное значение из нормализованного
   */
  getOriginalValue(normalizedValue: number): number {
    return normalizedValue / this.scaleFactor;
  }

  /**
   * Cleanup
   */
  destroy() {
    this.chart.remove();
  }
}

// ============== ПРИМЕР ИСПОЛЬЗОВАНИЯ ==============

async function displayShibChart() {
  const container = document.getElementById('chart')!;
  const chartManager = new SmallValueChartManager(container);

  // Пример данных SHIB
  const shibData: LineData[] = [
    { time: '2024-01-01', value: 0.00001234 },
    { time: '2024-01-02', value: 0.00001256 },
    { time: '2024-01-03', value: 0.00001198 },
    { time: '2024-01-04', value: 0.00001345 },
    { time: '2024-01-05', value: 0.00001289 },
    { time: '2024-01-06', value: 0.00001156 },
    { time: '2024-01-07', value: 0.00001423 },
    // ... больше данных
  ];

  chartManager.setData(shibData);

  // Real-time update
  // chartManager.update({ time: '2024-01-08', value: 0.00001489 });

  // Cleanup при необходимости
  // chartManager.destroy();
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Универсальность | Performance |
|---------|--------|-----------|-----------------|-------------|
| **priceFormat config** | 9/10 | 🟢 Низкая | ⚠️ Частичная | ✅ Отличный |
| **Нормализация данных** | 8/10 | 🟡 Средняя | ✅ Высокая | ✅ Отличный |
| **Custom Formatter** | 7/10 | 🟡 Средняя | ✅ Высокая | ✅ Хороший |
| **Visibility Workaround** | 6/10 | 🟡 Средняя | ⚠️ Частичная | ⚠️ Средний |
| **Logarithmic Scale** | 5/10 | 🟢 Низкая | ❌ Ограничено | ✅ Хороший |

### Рекомендации по выбору:

| Сценарий | Рекомендуемое решение |
|----------|----------------------|
| Один тип данных (только crypto) | Решение 1: priceFormat |
| Динамические данные разных масштабов | Решение 2: Нормализация |
| Переключение серий с разными масштабами | Решение 4: Visibility Workaround |
| Экстремально широкий диапазон | Решение 5: Log Scale (с осторожностью) |

---

## 🔗 Источники

1. **GitHub Issue #1495** - Price Scale AutoScale does not work for small values  
   https://github.com/tradingview/lightweight-charts/issues/1495

2. **Price Format Documentation**  
   https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceFormat

3. **JavaScript Floating Point Precision**  
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number

4. **BigNumber.js** - Arbitrary-precision decimal arithmetic  
   https://github.com/MikeMcl/bignumber.js/

5. **Price Scale Options**  
   https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceScaleOptions

---

## 📝 Дополнительные заметки

### Форматы отображения цены crypto

| Цена | Рекомендуемый формат | Пример |
|------|---------------------|---------|
| >= 1000 | Локализованный | 1,234.56 |
| >= 1 | 2 знака | 12.34 |
| >= 0.01 | 4 знака | 0.1234 |
| >= 0.0001 | 6 знаков | 0.001234 |
| < 0.0001 | Subscript notation | 0.0₅1234 |

### JavaScript precision limits

```javascript
// Минимальное положительное число в JS
Number.MIN_VALUE // 5e-324

// Но! Потеря точности начинается гораздо раньше
0.1 + 0.2 // 0.30000000000000004

// Для финансовых расчётов используйте:
// - BigNumber.js
// - Decimal.js
// - Или работайте в целых числах (сатоши вместо BTC)
```

### Когда ожидать официальное исправление?

Issue #1495 помечен как "needs investigation". Рекомендуется использовать workaround с нормализацией данных или явным указанием priceFormat.
