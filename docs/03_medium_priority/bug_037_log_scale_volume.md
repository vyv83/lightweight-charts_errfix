# БАГ #37: Логарифмический масштаб применяется к объёму

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#227](https://github.com/tradingview/lightweight-charts/issues/227)  
> **Версии:** v1.1.0 (исправлено в v1.2.1+)  
> **Статус:** ✅ Исправлено (но требует внимания при настройке)

---

## 📋 Описание проблемы

### Суть проблемы

При включении логарифмического масштаба (`mode: 'logarithmic'`) для основной ценовой шкалы, этот режим **также применяется к гистограмме объёма**, что приводит к некорректному отображению volume bars.

### Исторический контекст

**Баг был исправлен в версии 1.2.1** (и подтверждён в release notes v4.2.0). Однако проблема может возникать, если:

1. **Volume находится на той же price scale**, что и price data
2. **Используется устаревшая конфигурация** без разделения scales
3. **Нет понимания механизма overlay scales**

### Ожидаемое поведение

- Логарифмический масштаб применяется **только к ценовым данным**
- Volume histogram всегда отображается в **линейном масштабе**
- Каждая серия может иметь **собственную шкалу** с независимыми настройками

### Сценарии проблемы (в старых версиях)

```typescript
// Проблемный сценарий (pre-1.2.1)
const chart = createChart(container, {
  rightPriceScale: {
    mode: PriceScaleMode.Logarithmic, // Применялся ко ВСЕМ сериям на этой шкале
  },
});

const candlestickSeries = chart.addCandlestickSeries();
const volumeSeries = chart.addHistogramSeries(); // Тоже получал log scale!
```

### Влияние

- **Визуальное искажение** — объём отображается непропорционально
- **Аналитическая ошибка** — невозможно корректно оценить объёмы
- **Путаница пользователей** — особенно при больших различиях в volume

---

## 🔍 Найденные решения

### Решение 1: Использовать отдельную overlay scale для volume (Рекомендуемое)

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐⭐⭐ (10/10)

**Описание:**  
Назначить volume series на отдельную overlay price scale с линейным режимом.

**Преимущества:**
- Официальный и рекомендуемый подход
- Полная независимость масштабов
- Гибкое позиционирование volume

**Недостатки:**
- Требует правильной настройки scaleMargins

```typescript
import {
  createChart,
  IChartApi,
  ISeriesApi,
  PriceScaleMode,
  CandlestickData,
  HistogramData,
  Time,
} from 'lightweight-charts';

// Создаём график с логарифмическим масштабом для цен
const chart = createChart(container, {
  width: 800,
  height: 400,
  rightPriceScale: {
    mode: PriceScaleMode.Logarithmic, // Для ценовых данных
    visible: true,
  },
  // Настройка overlay scales (для volume)
  overlayPriceScales: {
    mode: PriceScaleMode.Normal, // ЛИНЕЙНЫЙ для overlays
  },
});

// Candlestick серия на правой шкале (с логарифмом)
const candlestickSeries = chart.addCandlestickSeries({
  priceScaleId: 'right', // Использует rightPriceScale
});

// Volume серия на ОТДЕЛЬНОЙ overlay шкале (линейной)
const volumeSeries = chart.addHistogramSeries({
  priceScaleId: 'volume', // Уникальный ID создаёт overlay scale
  color: '#26a69a',
  priceFormat: {
    type: 'volume',
  },
  priceLineVisible: false,
  lastValueVisible: false,
});

// Настройка позиции volume (внизу графика)
volumeSeries.priceScale().applyOptions({
  scaleMargins: {
    top: 0.85, // Volume занимает нижние 15%
    bottom: 0,
  },
  mode: PriceScaleMode.Normal, // Гарантированно линейный
});

// Данные
candlestickSeries.setData(priceData);
volumeSeries.setData(volumeData);
```

---

### Решение 2: Использовать левую шкалу для volume

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐⭐ (9/10)

**Описание:**  
Назначить volume на левую price scale, оставив правую для ценовых данных.

**Преимущества:**
- Чёткое визуальное разделение
- Метки volume видны на левой оси
- Простая конфигурация

**Недостатки:**
- Занимает место на левой стороне
- Может не подходить для всех layout

```typescript
const chart = createChart(container, {
  width: 800,
  height: 500,
  rightPriceScale: {
    mode: PriceScaleMode.Logarithmic, // Логарифм для цен
    visible: true,
  },
  leftPriceScale: {
    mode: PriceScaleMode.Normal, // Линейный для volume
    visible: true,
    scaleMargins: {
      top: 0.7,
      bottom: 0,
    },
  },
});

// Candlestick на правой шкале
const candlestickSeries = chart.addCandlestickSeries({
  priceScaleId: 'right',
});

// Volume на левой шкале
const volumeSeries = chart.addHistogramSeries({
  priceScaleId: 'left',
  color: '#26a69a',
  priceFormat: {
    type: 'volume',
  },
});

candlestickSeries.setData(priceData);
volumeSeries.setData(volumeData);
```

---

### Решение 3: Создать volume в отдельной pane (v5.0+)

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐⭐ (9/10)

**Описание:**  
В версии 5.0+ использовать multi-pane для отображения volume в отдельной панели.

**Преимущества:**
- Полная изоляция price и volume
- Ясная визуальная структура
- Независимые настройки для каждой pane

**Недостатки:**
- Требует v5.0+
- Увеличивает высоту графика

```typescript
import { createChart, PriceScaleMode } from 'lightweight-charts';

const chart = createChart(container, {
  width: 800,
  height: 600, // Больше высоты для двух pane
});

// Pane 0: Candlestick с логарифмическим масштабом
const candlestickSeries = chart.addCandlestickSeries({
  paneIndex: 0,
  priceScaleId: 'right',
});

// Настройка логарифмического масштаба для pane 0
chart.priceScale('right').applyOptions({
  mode: PriceScaleMode.Logarithmic,
});

// Pane 1: Volume с линейным масштабом
const volumeSeries = chart.addHistogramSeries({
  paneIndex: 1, // Отдельная pane
  priceScaleId: 'volume-scale',
  color: '#26a69a',
  priceFormat: {
    type: 'volume',
  },
});

// Volume pane с линейным масштабом
chart.priceScale('volume-scale').applyOptions({
  mode: PriceScaleMode.Normal,
});

// Настройка высоты panes
chart.panes()[0].setHeight(450); // Цена
chart.panes()[1].setHeight(150); // Volume

candlestickSeries.setData(priceData);
volumeSeries.setData(volumeData);
```

---

### Решение 4: Программное переключение масштаба

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐ (7/10)

**Описание:**  
Создать UI для динамического переключения режима масштаба, гарантируя линейный режим для volume.

**Преимущества:**
- Гибкость для пользователя
- Возможность сравнить log и linear

**Недостатки:**
- Требует дополнительного UI
- Больше кода

```typescript
import { createChart, PriceScaleMode, IChartApi } from 'lightweight-charts';

interface ChartScaleConfig {
  priceMode: PriceScaleMode;
  volumeAlwaysLinear: boolean;
}

class ScalableChart {
  private chart: IChartApi;
  private candlestickSeries: ISeriesApi<'Candlestick'>;
  private volumeSeries: ISeriesApi<'Histogram'>;
  private config: ChartScaleConfig;

  constructor(container: HTMLElement, config: ChartScaleConfig) {
    this.config = config;

    this.chart = createChart(container, {
      width: 800,
      height: 500,
    });

    this.candlestickSeries = this.chart.addCandlestickSeries({
      priceScaleId: 'right',
    });

    this.volumeSeries = this.chart.addHistogramSeries({
      priceScaleId: 'volume',
      color: '#26a69a',
      priceFormat: { type: 'volume' },
    });

    // Настройка volume scale
    this.volumeSeries.priceScale().applyOptions({
      scaleMargins: { top: 0.8, bottom: 0 },
      mode: PriceScaleMode.Normal, // Всегда линейный
    });

    this.applyScaleConfig();
  }

  /**
   * Переключить режим масштаба для цен
   */
  setPriceScaleMode(mode: PriceScaleMode): void {
    this.config.priceMode = mode;
    this.applyScaleConfig();
  }

  /**
   * Переключить между log и linear
   */
  toggleLogarithmic(): void {
    const newMode =
      this.config.priceMode === PriceScaleMode.Logarithmic
        ? PriceScaleMode.Normal
        : PriceScaleMode.Logarithmic;
    this.setPriceScaleMode(newMode);
  }

  /**
   * Получить текущий режим
   */
  isLogarithmic(): boolean {
    return this.config.priceMode === PriceScaleMode.Logarithmic;
  }

  private applyScaleConfig(): void {
    // Применяем режим к ценовой шкале
    this.chart.priceScale('right').applyOptions({
      mode: this.config.priceMode,
    });

    // Volume ВСЕГДА остаётся линейным
    if (this.config.volumeAlwaysLinear) {
      this.volumeSeries.priceScale().applyOptions({
        mode: PriceScaleMode.Normal,
      });
    }
  }

  setData(
    priceData: CandlestickData<Time>[],
    volumeData: HistogramData<Time>[]
  ): void {
    this.candlestickSeries.setData(priceData);
    this.volumeSeries.setData(volumeData);
  }
}

// Использование
const chart = new ScalableChart(container, {
  priceMode: PriceScaleMode.Logarithmic,
  volumeAlwaysLinear: true,
});

// UI toggle button
document.getElementById('toggleLog')?.addEventListener('click', () => {
  chart.toggleLogarithmic();
  updateButtonLabel();
});
```

---

### Решение 5: Нормализация volume данных для log-совместимости

**Рейтинг:** ⭐⭐⭐⭐⭐ (5/10)

**Описание:**  
Если по какой-то причине нужно использовать одну шкалу, можно нормализовать volume для корректного отображения в log scale.

**Преимущества:**
- Работает без разделения scales
- Может быть полезно для специфических use cases

**Недостатки:**
- Искажает реальные значения volume
- Требует обратного преобразования для отображения
- Не рекомендуется для production

```typescript
/**
 * Нормализация volume для log-совместимого отображения
 * НЕ РЕКОМЕНДУЕТСЯ - используйте отдельные scales
 */
function normalizeVolumeForLogScale(
  volumeData: HistogramData<Time>[],
  priceData: CandlestickData<Time>[]
): HistogramData<Time>[] {
  // Вычисляем диапазоны
  const prices = priceData.flatMap((d) => [d.open, d.high, d.low, d.close]);
  const minPrice = Math.min(...prices);
  const maxPrice = Math.max(...prices);
  const priceRange = maxPrice - minPrice;

  const volumes = volumeData.map((d) => d.value ?? 0);
  const maxVolume = Math.max(...volumes);

  // Нормализуем volume к нижней части price range
  const volumeScaleFactor = (priceRange * 0.15) / maxVolume;
  const volumeOffset = minPrice;

  return volumeData.map((d) => ({
    ...d,
    value: d.value ? d.value * volumeScaleFactor + volumeOffset : undefined,
  }));
}

// ВНИМАНИЕ: Это хак, используйте отдельные scales!
```

---

## ✅ Рекомендуемое решение

### Решение 1: Отдельная overlay scale для volume

Официальный и самый надёжный подход — использовать **отдельную overlay price scale** для volume series:

```typescript
import {
  createChart,
  IChartApi,
  ISeriesApi,
  PriceScaleMode,
  CandlestickData,
  HistogramData,
  Time,
  ColorType,
} from 'lightweight-charts';

// ============================================
// 1. Типы конфигурации
// ============================================

interface VolumeChartConfig {
  container: HTMLElement;
  width?: number;
  height?: number;
  volumeHeightPercent?: number; // 0.1 = 10%
  useLogarithmicPrice?: boolean;
  volumeColors?: {
    up: string;
    down: string;
  };
}

// ============================================
// 2. Класс графика с раздельными масштабами
// ============================================

class PriceVolumeChart {
  private chart: IChartApi;
  private priceSeries: ISeriesApi<'Candlestick'>;
  private volumeSeries: ISeriesApi<'Histogram'>;
  private config: Required<VolumeChartConfig>;

  constructor(config: VolumeChartConfig) {
    this.config = {
      container: config.container,
      width: config.width ?? 800,
      height: config.height ?? 500,
      volumeHeightPercent: config.volumeHeightPercent ?? 0.15,
      useLogarithmicPrice: config.useLogarithmicPrice ?? false,
      volumeColors: config.volumeColors ?? {
        up: '#26a69a',
        down: '#ef5350',
      },
    };

    this.chart = this.createChart();
    this.priceSeries = this.createPriceSeries();
    this.volumeSeries = this.createVolumeSeries();
  }

  private createChart(): IChartApi {
    return createChart(this.config.container, {
      width: this.config.width,
      height: this.config.height,
      layout: {
        background: { type: ColorType.Solid, color: '#1e222d' },
        textColor: '#d1d4dc',
      },
      grid: {
        vertLines: { color: '#2B2B43' },
        horzLines: { color: '#2B2B43' },
      },
      rightPriceScale: {
        mode: this.config.useLogarithmicPrice
          ? PriceScaleMode.Logarithmic
          : PriceScaleMode.Normal,
        visible: true,
        borderColor: '#2B2B43',
      },
      // Overlay scales (для volume) - ВСЕГДА линейный
      overlayPriceScales: {
        mode: PriceScaleMode.Normal,
      },
      timeScale: {
        borderColor: '#2B2B43',
        timeVisible: true,
        secondsVisible: false,
      },
      crosshair: {
        mode: 1, // CrosshairMode.Normal
      },
    });
  }

  private createPriceSeries(): ISeriesApi<'Candlestick'> {
    return this.chart.addCandlestickSeries({
      priceScaleId: 'right',
      upColor: '#26a69a',
      downColor: '#ef5350',
      borderDownColor: '#ef5350',
      borderUpColor: '#26a69a',
      wickDownColor: '#ef5350',
      wickUpColor: '#26a69a',
    });
  }

  private createVolumeSeries(): ISeriesApi<'Histogram'> {
    const volumeMarginTop = 1 - this.config.volumeHeightPercent;

    const series = this.chart.addHistogramSeries({
      priceScaleId: 'volume', // Уникальный ID = отдельная overlay scale
      color: this.config.volumeColors.up,
      priceFormat: {
        type: 'volume',
      },
      priceLineVisible: false,
      lastValueVisible: false,
    });

    // Критично: Гарантируем линейный масштаб для volume
    series.priceScale().applyOptions({
      scaleMargins: {
        top: volumeMarginTop,
        bottom: 0,
      },
      mode: PriceScaleMode.Normal, // ВСЕГДА линейный!
    });

    return series;
  }

  /**
   * Установить данные с автоматической colorization volume
   */
  setData(
    priceData: CandlestickData<Time>[],
    volumeData?: HistogramData<Time>[]
  ): void {
    this.priceSeries.setData(priceData);

    if (volumeData) {
      // Colorize volume based on price direction
      const colorizedVolume = this.colorizeVolume(priceData, volumeData);
      this.volumeSeries.setData(colorizedVolume);
    }
  }

  /**
   * Обновить последний бар
   */
  update(
    priceBar: CandlestickData<Time>,
    volumeBar?: HistogramData<Time>
  ): void {
    this.priceSeries.update(priceBar);

    if (volumeBar) {
      const color =
        priceBar.close >= priceBar.open
          ? this.config.volumeColors.up
          : this.config.volumeColors.down;
      this.volumeSeries.update({ ...volumeBar, color });
    }
  }

  /**
   * Переключить логарифмический режим для цен
   */
  setLogarithmic(enabled: boolean): void {
    this.config.useLogarithmicPrice = enabled;

    // Применяем только к ценовой шкале
    this.chart.priceScale('right').applyOptions({
      mode: enabled ? PriceScaleMode.Logarithmic : PriceScaleMode.Normal,
    });

    // Volume остаётся линейным
    this.volumeSeries.priceScale().applyOptions({
      mode: PriceScaleMode.Normal,
    });
  }

  /**
   * Проверить текущий режим
   */
  isLogarithmic(): boolean {
    return this.config.useLogarithmicPrice;
  }

  /**
   * Получить API графика
   */
  getChart(): IChartApi {
    return this.chart;
  }

  /**
   * Удалить график
   */
  remove(): void {
    this.chart.remove();
  }

  private colorizeVolume(
    priceData: CandlestickData<Time>[],
    volumeData: HistogramData<Time>[]
  ): HistogramData<Time>[] {
    const priceMap = new Map(
      priceData.map((p) => [String(p.time), p])
    );

    return volumeData.map((v) => {
      const price = priceMap.get(String(v.time));
      const color =
        price && price.close >= price.open
          ? this.config.volumeColors.up
          : this.config.volumeColors.down;
      return { ...v, color };
    });
  }
}

// ============================================
// 3. Пример использования
// ============================================

function createPriceVolumeChart(): void {
  const container = document.getElementById('chart')!;

  const chart = new PriceVolumeChart({
    container,
    width: 900,
    height: 500,
    volumeHeightPercent: 0.2, // Volume занимает 20% высоты
    useLogarithmicPrice: true, // Цены в log scale
    volumeColors: {
      up: '#26a69a',
      down: '#ef5350',
    },
  });

  // Демо-данные
  const priceData: CandlestickData<Time>[] = [
    { time: '2024-01-01', open: 100, high: 105, low: 98, close: 103 },
    { time: '2024-01-02', open: 103, high: 110, low: 102, close: 108 },
    { time: '2024-01-03', open: 108, high: 112, low: 106, close: 107 },
    // ... more data
  ];

  const volumeData: HistogramData<Time>[] = [
    { time: '2024-01-01', value: 1000000 },
    { time: '2024-01-02', value: 1500000 },
    { time: '2024-01-03', value: 800000 },
    // ... more data
  ];

  chart.setData(priceData, volumeData);

  // Toggle button
  document.getElementById('toggleLog')?.addEventListener('click', () => {
    chart.setLogarithmic(!chart.isLogarithmic());
  });
}

export { PriceVolumeChart, VolumeChartConfig };
```

### Почему это решение оптимально

| Критерий | Оценка |
|----------|--------|
| **Корректность** | ✅ Volume всегда в линейном масштабе |
| **Гибкость** | ✅ Независимое управление каждой шкалой |
| **Совместимость** | ✅ Работает с v4.0+ |
| **Простота использования** | ✅ Чистый API |
| **Производительность** | ✅ Нативная поддержка библиотеки |

---

## 📊 Сравнительная таблица решений

| Решение | Эффективность | Сложность | Совместимость | Рейтинг |
|---------|--------------|-----------|---------------|---------|
| #1: Overlay scale | ⭐⭐⭐⭐⭐ | Низкая | v3.0+ | 10/10 |
| #2: Левая шкала | ⭐⭐⭐⭐⭐ | Низкая | Все версии | 9/10 |
| #3: Multi-pane | ⭐⭐⭐⭐⭐ | Средняя | v5.0+ | 9/10 |
| #4: Программное переключение | ⭐⭐⭐⭐ | Средняя | Все версии | 7/10 |
| #5: Нормализация данных | ⭐⭐ | Высокая | Все версии | 5/10 |

### Рекомендации по выбору

- **Стандартный сценарий** → Решение #1 (Overlay scale)
- **Нужны видимые метки volume** → Решение #2 (Левая шкала)
- **Современный проект (v5.0+)** → Решение #3 (Multi-pane)
- **Динамическое переключение** → Решение #4

---

## 🔗 Источники

1. **GitHub Issue #227** — [Logarithmic scaling is applied to volume](https://github.com/tradingview/lightweight-charts/issues/227)

2. **Release Notes v4.2.0** — [Fixed: Logarithmic scale for volume](https://tradingview.github.io/lightweight-charts/docs/release-notes)

3. **Официальная документация: Price Scales** — [Multiple Price Scales](https://tradingview.github.io/lightweight-charts/docs/price-scale)

4. **Официальная документация: PriceScaleMode** — [API Reference](https://tradingview.github.io/lightweight-charts/docs/api/enums/PriceScaleMode)

5. **How-To: Volume on Chart** — [Adding Volume](https://tradingview.github.io/lightweight-charts/tutorials/how_to/volume-on-chart)

6. **Официальная документация: Multi-Pane (v5.0+)** — [Panes](https://tradingview.github.io/lightweight-charts/docs/panes)

---

**Документ создан:** 2026-01-18  
**Версия документа:** 1.0  
**Примечание:** Баг был исправлен в v1.2.1, но остаётся актуальным как руководство по правильной настройке volume с логарифмическими ценами.
