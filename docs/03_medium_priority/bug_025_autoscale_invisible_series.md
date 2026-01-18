# БАГ #25: Автоскейл для невидимых серий

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#687](https://github.com/tradingview/lightweight-charts/issues/687)  
> **Версии:** Исправлено в современных версиях  
> **Платформы:** Все браузеры  
> **Статус:** 🟢 Закрыт (исправлен через PR #688)

---

## 📋 Описание проблемы

### Суть проблемы
В ранних версиях библиотеки невидимые серии (`visible: false`) продолжали влиять на автомасштабирование (autoscale), что приводило к неправильному отображению видимых серий — они выглядели сжатыми или непропорционально маленькими.

### Пример проблемы (в старых версиях)

```typescript
// Серия с большими значениями
const bigSeries = chart.addLineSeries();
bigSeries.setData([
  { time: '2024-01-01', value: 1000 },
  { time: '2024-01-02', value: 1500 },
]);

// Серия с маленькими значениями
const smallSeries = chart.addLineSeries();
smallSeries.setData([
  { time: '2024-01-01', value: 10 },
  { time: '2024-01-02', value: 15 },
]);

// Скрываем большую серию
bigSeries.applyOptions({ visible: false });

// ❌ В старых версиях: маленькая серия всё равно выглядит сжатой,
// потому что autoscale учитывал диапазон 10-1500
```

### Текущий статус

**Проблема решена** в современных версиях lightweight-charts. Невидимые серии больше не влияют на autoscale.

Однако остаются связанные проблемы:
- Не пересчитывается autoscale после ручного изменения scale
- `autoscaleInfoProvider` не всегда интуитивен
- Проблемы с очень малыми значениями (см. баг #20)

---

## 🔍 Найденные решения

### Решение 1: Обновление библиотеки (Рекомендуемое)

**Оценка: 10/10**

```bash
# Обновите до последней версии
npm update lightweight-charts
# или
npm install lightweight-charts@latest
```

Начиная с PR #688, невидимые серии автоматически исключаются из расчёта autoscale.

**Проверка версии:**
```typescript
import { version } from 'lightweight-charts';
console.log(`Lightweight Charts version: ${version}`);
// Рекомендуется v3.8.0+ или v4.x/v5.x
```

---

### Решение 2: Принудительный пересчёт autoscale

**Оценка: 8/10**

Если после изменения видимости серии autoscale не обновляется:

```typescript
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

/**
 * Переключает видимость серии с принудительным autoscale
 */
function toggleSeriesWithAutoscale(
  chart: IChartApi,
  series: ISeriesApi<any>,
  visible: boolean
): void {
  // Изменяем видимость
  series.applyOptions({ visible });

  // Принудительно включаем autoscale для price scale
  const priceScaleId = series.options().priceScaleId || 'right';
  
  chart.priceScale(priceScaleId).applyOptions({
    autoScale: true,
  });
}

// Использование
const chart = createChart(container);
const series1 = chart.addLineSeries({ priceScaleId: 'right' });
const series2 = chart.addLineSeries({ priceScaleId: 'right' });

series1.setData(largeValuesData);
series2.setData(smallValuesData);

// Toggle с автоматическим пересчётом
toggleSeriesWithAutoscale(chart, series1, false); // Скрыть
toggleSeriesWithAutoscale(chart, series1, true);  // Показать
```

---

### Решение 3: Использование autoscaleInfoProvider

**Оценка: 9/10**

Для полного контроля над autoscale используйте `autoscaleInfoProvider`:

```typescript
import { 
  createChart, 
  ISeriesApi,
  AutoscaleInfo,
  SeriesOptionsCommon
} from 'lightweight-charts';

const chart = createChart(container);

// Основная серия (определяет масштаб)
const mainSeries = chart.addCandlestickSeries({
  priceScaleId: 'main',
});

// Оверлей серия (SMA) - следует за основной
const smaSeries = chart.addLineSeries({
  color: '#2962FF',
  lineWidth: 2,
  priceScaleId: 'main',
  
  // Custom autoscale - игнорирует эту серию для autoscale
  autoscaleInfoProvider: () => {
    return null; // Не влияет на autoscale
  },
});

// Серия с custom autoscale logic
const volumeSeries = chart.addHistogramSeries({
  priceScaleId: 'volume',
  
  // Custom autoscale с margins
  autoscaleInfoProvider: (original) => {
    const res = original();
    if (res !== null) {
      return {
        priceRange: {
          minValue: res.priceRange.minValue,
          maxValue: res.priceRange.maxValue * 1.5, // Добавляем 50% сверху
        },
        margins: {
          above: 0.1,
          below: 0,
        },
      };
    }
    return null;
  },
});

mainSeries.setData(candlestickData);
smaSeries.setData(smaData);
volumeSeries.setData(volumeData);
```

---

### Решение 4: Раздельные price scales

**Оценка: 8/10**

Используйте разные price scale ID для серий с разным масштабом:

```typescript
import { createChart } from 'lightweight-charts';

const chart = createChart(container, {
  rightPriceScale: {
    visible: true,
    autoScale: true,
  },
  leftPriceScale: {
    visible: true,
    autoScale: true,
  },
});

// Основная серия на правой шкале
const priceSeries = chart.addCandlestickSeries({
  priceScaleId: 'right',
});

// Объём на левой шкале с отдельным масштабом
const volumeSeries = chart.addHistogramSeries({
  priceScaleId: 'left',
});

// RSI на отдельной шкале
const rsiSeries = chart.addLineSeries({
  priceScaleId: 'rsi',
  priceFormat: {
    type: 'custom',
    formatter: (price: number) => price.toFixed(0),
  },
});

// Настраиваем кастомную шкалу
chart.priceScale('rsi').applyOptions({
  scaleMargins: {
    top: 0.8,  // RSI занимает нижние 20%
    bottom: 0,
  },
  autoScale: true,
});
```

---

### Решение 5: Управление видимостью через data

**Оценка: 7/10**

Альтернативный подход — вместо `visible: false` очищаем данные:

```typescript
import { IChartApi, ISeriesApi, LineData } from 'lightweight-charts';

class ToggleableSeries {
  private series: ISeriesApi<'Line'>;
  private cachedData: LineData[] = [];
  private isVisible: boolean = true;

  constructor(series: ISeriesApi<'Line'>) {
    this.series = series;
  }

  setData(data: LineData[]): void {
    this.cachedData = data;
    if (this.isVisible) {
      this.series.setData(data);
    }
  }

  toggle(): void {
    this.isVisible = !this.isVisible;
    if (this.isVisible) {
      this.series.setData(this.cachedData);
    } else {
      this.series.setData([]); // Очищаем данные
    }
  }

  show(): void {
    if (!this.isVisible) {
      this.isVisible = true;
      this.series.setData(this.cachedData);
    }
  }

  hide(): void {
    if (this.isVisible) {
      this.isVisible = false;
      this.series.setData([]);
    }
  }

  getSeries(): ISeriesApi<'Line'> {
    return this.series;
  }
}

// Использование
const series = chart.addLineSeries();
const toggleable = new ToggleableSeries(series);

toggleable.setData(data);
toggleable.toggle(); // Скрыть (данные очищены)
toggleable.toggle(); // Показать (данные восстановлены)
```

---

## ✅ Рекомендуемое решение

### Комбинированный подход для современных версий

```typescript
import { 
  createChart, 
  IChartApi, 
  ISeriesApi,
  SeriesType,
  AutoscaleInfo
} from 'lightweight-charts';

interface ManagedSeries<T extends SeriesType = 'Line'> {
  series: ISeriesApi<T>;
  priceScaleId: string;
  affectsAutoscale: boolean;
}

/**
 * Менеджер серий с контролем autoscale
 */
class SeriesVisibilityManager {
  private chart: IChartApi;
  private managedSeries: Map<string, ManagedSeries> = new Map();

  constructor(chart: IChartApi) {
    this.chart = chart;
  }

  /**
   * Регистрирует серию с настройками autoscale
   */
  register<T extends SeriesType>(
    id: string,
    series: ISeriesApi<T>,
    options: {
      priceScaleId?: string;
      affectsAutoscale?: boolean;
    } = {}
  ): void {
    const { 
      priceScaleId = series.options().priceScaleId || 'right',
      affectsAutoscale = true 
    } = options;

    this.managedSeries.set(id, {
      series,
      priceScaleId,
      affectsAutoscale,
    });
  }

  /**
   * Переключает видимость серии с корректным autoscale
   */
  setVisible(id: string, visible: boolean): void {
    const managed = this.managedSeries.get(id);
    if (!managed) return;

    managed.series.applyOptions({ visible });

    // Принудительный пересчёт autoscale для price scale
    this.chart.priceScale(managed.priceScaleId).applyOptions({
      autoScale: true,
    });
  }

  /**
   * Показывает только указанные серии
   */
  showOnly(ids: string[]): void {
    for (const [id, managed] of this.managedSeries) {
      const visible = ids.includes(id);
      managed.series.applyOptions({ visible });
    }

    // Пересчитываем все затронутые price scales
    const affectedScales = new Set<string>();
    for (const [id, managed] of this.managedSeries) {
      if (managed.affectsAutoscale) {
        affectedScales.add(managed.priceScaleId);
      }
    }

    for (const scaleId of affectedScales) {
      this.chart.priceScale(scaleId).applyOptions({ autoScale: true });
    }
  }

  /**
   * Показывает все серии
   */
  showAll(): void {
    for (const [id, managed] of this.managedSeries) {
      managed.series.applyOptions({ visible: true });
    }
  }

  /**
   * Скрывает все серии
   */
  hideAll(): void {
    for (const [id, managed] of this.managedSeries) {
      managed.series.applyOptions({ visible: false });
    }
  }
}

// Пример использования
const chart = createChart(container);
const manager = new SeriesVisibilityManager(chart);

// Создаём серии
const priceSeries = chart.addCandlestickSeries();
const sma20 = chart.addLineSeries({ color: '#2962FF' });
const sma50 = chart.addLineSeries({ color: '#E91E63' });
const volume = chart.addHistogramSeries({ priceScaleId: 'volume' });

// Регистрируем в менеджере
manager.register('price', priceSeries);
manager.register('sma20', sma20, { affectsAutoscale: false });
manager.register('sma50', sma50, { affectsAutoscale: false });
manager.register('volume', volume, { priceScaleId: 'volume' });

// Управление видимостью
manager.setVisible('sma20', false);
manager.showOnly(['price', 'volume']);
manager.showAll();
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Современные версии | Legacy версии |
|---------|--------|-----------|-------------------|---------------|
| **Обновление библиотеки** | **10/10** | ⭐ | ✅ Работает | - |
| Принудительный autoscale | 8/10 | ⭐ | ✅ | ✅ |
| **autoscaleInfoProvider** | **9/10** | ⭐⭐ | ✅ | ⚠️ v3.8+ |
| Раздельные price scales | 8/10 | ⭐⭐ | ✅ | ✅ |
| Toggle через data | 7/10 | ⭐⭐ | ✅ | ✅ |

---

## 🔗 Источники

1. [GitHub Issue #687 - Disabling autoScale for non-visible series](https://github.com/tradingview/lightweight-charts/issues/687)
2. [GitHub PR #688 - Fix for invisible series autoscale](https://github.com/tradingview/lightweight-charts/pull/688)
3. [Lightweight Charts - Price Scale Documentation](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IPriceScaleApi)
4. [Lightweight Charts - autoscaleInfoProvider](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/SeriesOptionsCommon#autoscaleinfoprovider)
5. [Stack Overflow - Autoscale issues](https://stackoverflow.com/questions/tagged/lightweight-charts)

---

## 📝 Рекомендации

### Для новых проектов
1. Используйте последнюю версию библиотеки — проблема решена
2. Применяйте `autoscaleInfoProvider` для сложных сценариев
3. Используйте раздельные price scales для серий с разным масштабом

### Для legacy проектов
1. Обновитесь до v3.8.0+ или v4.x/v5.x
2. Если обновление невозможно — используйте workaround с очисткой данных
3. Принудительно вызывайте `priceScale().applyOptions({ autoScale: true })`

### Best practices
```typescript
// ✅ Рекомендуется: явно указывать priceScaleId
const series = chart.addLineSeries({
  priceScaleId: 'my-scale',
});

// ✅ Рекомендуется: использовать autoscaleInfoProvider для overlay серий
const overlay = chart.addLineSeries({
  autoscaleInfoProvider: () => null, // Не влияет на autoscale
});

// ✅ Рекомендуется: явно управлять autoscale после изменений
function updateVisibility(series: ISeriesApi<any>, visible: boolean) {
  series.applyOptions({ visible });
  chart.priceScale(series.options().priceScaleId).applyOptions({ autoScale: true });
}
```
