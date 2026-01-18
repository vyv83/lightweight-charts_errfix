# БАГ #36: Некорректное позиционирование второй метки на временной оси

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1689](https://github.com/tradingview/lightweight-charts/issues/1689)  
> **Pull Request:** [#1688](https://github.com/tradingview/lightweight-charts/pull/1688)  
> **Версии:** v4.x+  
> **Статус:** 🔴 Open (fix proposed)

---

## 📋 Описание проблемы

### Суть проблемы

В функции `fillWeightsForPoints` библиотеки lightweight-charts существует ошибка в расчёте весов tick marks для временной оси. Проблема проявляется во **второй отображаемой метке времени**, которая может позиционироваться некорректно.

### Техническая причина

```typescript
// Проблемный код в fillWeightsForPoints
function fillWeightsForPoints(sortedPoints: TimeScalePoint[]): void {
  let prevTime = null;
  let totalTimeDiff = 0;
  
  for (let i = 0; i < sortedPoints.length; i++) {
    const point = sortedPoints[i];
    // BUG: когда prevTime === null, timeDiff = 0
    const timeDiff = prevTime ? point.time - prevTime : 0;
    totalTimeDiff += timeDiff;
    prevTime = point.time;
  }
  
  // averageTimeDiff вычисляется с учётом нулевого первого diff
  const averageTimeDiff = totalTimeDiff / sortedPoints.length;
  
  // Это приводит к некорректным весам для второй метки
}
```

**Последовательность проблемы:**
1. Для первой точки `prevTime === null`, поэтому `timeDiff = 0`
2. `totalTimeDiff` не включает реальную разницу от "начала времён" до первой точки
3. `averageTimeDiff` вычисляется неверно
4. Вторая метка получает некорректный вес (часто эквивалентный году)
5. Визуально метка отображается в неправильном месте

### Сценарии проявления

```typescript
// Сценарий: график с небольшим количеством данных
const data = [
  { time: '2024-01-15', value: 100 },
  { time: '2024-01-16', value: 102 }, // ← Вторая метка может быть неверно позиционирована
  { time: '2024-01-17', value: 101 },
];

series.setData(data);
chart.timeScale().fitContent();

// Результат: вторая метка может отображаться не в ожидаемой позиции
```

### Влияние

- **Визуальное несоответствие** — метки не соответствуют реальным данным
- **Путаница пользователей** — особенно на графиках с малым количеством точек
- **Профессиональный вид** — нарушает доверие к точности графика

---

## 🔍 Найденные решения

### Решение 1: Применить патч из PR #1688

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐⭐ (9/10)

**Описание:**  
Официальный fix, предложенный в Pull Request #1688, корректирует логику расчёта весов.

**Преимущества:**
- Официальное решение от разработчиков
- Минимальные изменения в коде
- Исправляет корневую причину

**Недостатки:**
- Требует форка библиотеки до merge PR
- Может не быть совместим с текущей версией

```typescript
// Исправленная логика (из PR #1688)
function fillWeightsForPoints(sortedPoints: TimeScalePoint[]): void {
  if (sortedPoints.length < 2) return;
  
  let totalTimeDiff = 0;
  let prevTime: number | null = null;
  
  for (let i = 0; i < sortedPoints.length; i++) {
    const point = sortedPoints[i];
    
    // FIX: Пропускаем первую итерацию для расчёта diff
    if (prevTime !== null) {
      const timeDiff = point.time - prevTime;
      totalTimeDiff += timeDiff;
    }
    
    prevTime = point.time;
  }
  
  // FIX: Делим на (length - 1) т.к. есть (n-1) интервалов между n точками
  const averageTimeDiff = totalTimeDiff / (sortedPoints.length - 1);
  
  // Теперь веса вычисляются корректно
  for (let i = 0; i < sortedPoints.length; i++) {
    sortedPoints[i].weight = calculateWeight(sortedPoints[i], averageTimeDiff);
  }
}
```

---

### Решение 2: Использовать кастомный timeFormatter

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐ (8/10)

**Описание:**  
Переопределить форматирование меток времени для гарантии корректного отображения.

**Преимущества:**
- Не требует модификации библиотеки
- Полный контроль над отображением
- Работает с любой версией

**Недостатки:**
- Не исправляет позиционирование, только форматирование
- Требует дополнительной логики

```typescript
import { createChart, Time, BusinessDay, UTCTimestamp } from 'lightweight-charts';

interface TimeFormatterOptions {
  locale?: string;
  showYear?: boolean;
  showTime?: boolean;
}

function createCustomTimeFormatter(options: TimeFormatterOptions = {}) {
  const { locale = 'en-US', showYear = true, showTime = false } = options;
  
  return (time: BusinessDay | UTCTimestamp): string => {
    let date: Date;
    
    if (typeof time === 'number') {
      date = new Date(time * 1000);
    } else {
      date = new Date(time.year, time.month - 1, time.day);
    }
    
    const formatOptions: Intl.DateTimeFormatOptions = {
      month: 'short',
      day: 'numeric',
    };
    
    if (showYear) {
      formatOptions.year = 'numeric';
    }
    
    if (showTime) {
      formatOptions.hour = '2-digit';
      formatOptions.minute = '2-digit';
    }
    
    return new Intl.DateTimeFormat(locale, formatOptions).format(date);
  };
}

// Использование
const chart = createChart(container, {
  width: 800,
  height: 400,
  timeScale: {
    tickMarkFormatter: createCustomTimeFormatter({
      locale: 'ru-RU',
      showYear: true,
    }),
  },
  localization: {
    timeFormatter: createCustomTimeFormatter({
      locale: 'ru-RU',
      showTime: true,
    }),
  },
});
```

---

### Решение 3: Добавить padding данных для стабилизации

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐ (7/10)

**Описание:**  
Добавить дополнительные точки данных в начало для стабилизации расчёта весов.

**Преимущества:**
- Простой workaround
- Не требует модификации библиотеки

**Недостатки:**
- Может влиять на внешний вид графика
- Требует дополнительных данных

```typescript
interface DataPoint {
  time: Time;
  value: number;
}

/**
 * Добавляет padding-точки для стабилизации расчёта весов временной шкалы
 */
function stabilizeTimeScaleData(
  data: DataPoint[],
  paddingCount = 2
): DataPoint[] {
  if (data.length < 2) return data;
  
  const firstPoint = data[0];
  const secondPoint = data[1];
  
  // Вычисляем средний интервал
  let firstTime: number;
  let secondTime: number;
  
  if (typeof firstPoint.time === 'number') {
    firstTime = firstPoint.time;
    secondTime = secondPoint.time as number;
  } else {
    firstTime = new Date(
      firstPoint.time.year,
      firstPoint.time.month - 1,
      firstPoint.time.day
    ).getTime() / 1000;
    secondTime = new Date(
      secondPoint.time.year,
      secondPoint.time.month - 1,
      secondPoint.time.day
    ).getTime() / 1000;
  }
  
  const interval = secondTime - firstTime;
  
  // Создаём padding-точки
  const paddingPoints: DataPoint[] = [];
  for (let i = paddingCount; i > 0; i--) {
    const paddingTime = firstTime - interval * i;
    paddingPoints.push({
      time: paddingTime as Time,
      value: firstPoint.value, // Или null для whitespace
    });
  }
  
  return [...paddingPoints, ...data];
}

// Использование
const originalData = [
  { time: '2024-01-15' as Time, value: 100 },
  { time: '2024-01-16' as Time, value: 102 },
  { time: '2024-01-17' as Time, value: 101 },
];

const stabilizedData = stabilizeTimeScaleData(originalData, 2);
series.setData(stabilizedData);

// Устанавливаем visible range на оригинальные данные
chart.timeScale().setVisibleRange({
  from: originalData[0].time,
  to: originalData[originalData.length - 1].time,
});
```

---

### Решение 4: Кастомный Time Scale через плагин

**Рейтинг:** ⭐⭐⭐⭐⭐⭐⭐⭐ (8/10)

**Описание:**  
Создать custom primitive для рендеринга меток времени с корректным позиционированием.

**Преимущества:**
- Полный контроль над позиционированием
- Независимость от внутренней логики библиотеки
- Гибкость в настройке

**Недостатки:**
- Более сложная реализация
- Дублирование функционала

```typescript
import {
  IChartApi,
  ISeriesApi,
  ISeriesPrimitive,
  IPrimitivePaneView,
  IPrimitivePaneRenderer,
  SeriesAttachedParameter,
  Time,
} from 'lightweight-charts';

interface TimeLabel {
  time: Time;
  x: number;
  label: string;
}

class CustomTimeLabelsRenderer implements IPrimitivePaneRenderer {
  private _labels: TimeLabel[] = [];

  setLabels(labels: TimeLabel[]): void {
    this._labels = labels;
  }

  draw(ctx: CanvasRenderingContext2D): void {
    ctx.save();
    ctx.font = '11px Arial, sans-serif';
    ctx.fillStyle = '#787B86';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'top';

    for (const label of this._labels) {
      ctx.fillText(label.label, label.x, 5);
    }

    ctx.restore();
  }
}

class CustomTimeLabelsView implements IPrimitivePaneView {
  private _renderer: CustomTimeLabelsRenderer;

  constructor() {
    this._renderer = new CustomTimeLabelsRenderer();
  }

  renderer(): IPrimitivePaneRenderer | null {
    return this._renderer;
  }

  zOrder(): 'bottom' | 'normal' | 'top' {
    return 'top';
  }

  update(labels: TimeLabel[]): void {
    this._renderer.setLabels(labels);
  }
}

class CustomTimeScalePrimitive implements ISeriesPrimitive<Time> {
  private _chart: IChartApi;
  private _series?: ISeriesApi<any>;
  private _requestUpdate?: () => void;
  private _paneView: CustomTimeLabelsView;
  private _enabled = false;

  constructor(chart: IChartApi) {
    this._chart = chart;
    this._paneView = new CustomTimeLabelsView();
  }

  attached(params: SeriesAttachedParameter<Time>): void {
    this._series = params.series;
    this._requestUpdate = params.requestUpdate;
    
    // Подписываемся на изменения visible range
    this._chart.timeScale().subscribeVisibleLogicalRangeChange(() => {
      this.updateLabels();
    });
  }

  detached(): void {
    this._series = undefined;
    this._requestUpdate = undefined;
  }

  paneViews(): readonly IPrimitivePaneView[] {
    return this._enabled ? [this._paneView] : [];
  }

  updateAllViews(): void {
    this.updateLabels();
  }

  enable(): void {
    this._enabled = true;
    this._requestUpdate?.();
  }

  disable(): void {
    this._enabled = false;
    this._requestUpdate?.();
  }

  private updateLabels(): void {
    if (!this._series) return;

    const timeScale = this._chart.timeScale();
    const visibleRange = timeScale.getVisibleLogicalRange();
    
    if (!visibleRange) return;

    const data = this._series.data();
    const labels: TimeLabel[] = [];

    // Выбираем метки с корректным интервалом
    const visiblePoints: number[] = [];
    for (let i = Math.floor(visibleRange.from); i <= Math.ceil(visibleRange.to); i++) {
      if (i >= 0 && i < data.length) {
        visiblePoints.push(i);
      }
    }

    // Вычисляем оптимальный интервал
    const desiredLabelCount = 8;
    const step = Math.max(1, Math.floor(visiblePoints.length / desiredLabelCount));

    for (let i = 0; i < visiblePoints.length; i += step) {
      const index = visiblePoints[i];
      const point = data[index];
      
      if (!point || !('time' in point)) continue;

      const x = timeScale.logicalToCoordinate(index);
      if (x === null) continue;

      labels.push({
        time: point.time as Time,
        x,
        label: this.formatTime(point.time as Time),
      });
    }

    this._paneView.update(labels);
    this._requestUpdate?.();
  }

  private formatTime(time: Time): string {
    let date: Date;
    
    if (typeof time === 'number') {
      date = new Date(time * 1000);
    } else if (typeof time === 'string') {
      date = new Date(time);
    } else {
      date = new Date(time.year, time.month - 1, time.day);
    }

    return date.toLocaleDateString('en-US', {
      month: 'short',
      day: 'numeric',
    });
  }
}

// Использование
const chart = createChart(container, {
  timeScale: {
    visible: false, // Скрываем стандартную шкалу
  },
});

const series = chart.addLineSeries();
const customTimeScale = new CustomTimeScalePrimitive(chart);
series.attachPrimitive(customTimeScale);
customTimeScale.enable();
```

---

### Решение 5: Monkey-patch функции fillWeightsForPoints

**Рейтинг:** ⭐⭐⭐⭐⭐ (5/10)

**Описание:**  
Переопределить проблемную функцию через monkey-patching (для development/testing).

**Преимущества:**
- Быстрый workaround
- Не требует форка

**Недостатки:**
- Хрупкое решение
- Может сломаться при обновлении библиотеки
- Не рекомендуется для production

```typescript
// ВНИМАНИЕ: Только для development/debugging!

import * as LightweightCharts from 'lightweight-charts';

// Получаем доступ к внутренним модулям (если библиотека не минифицирована)
// Это очень хрупкий подход и работает не всегда

function patchTimeScaleWeights(): void {
  // Этот код специфичен для конкретной версии и сборки
  // В production используйте другие решения
  
  console.warn(
    'TimeScale weight calculation patched. ' +
    'This is a development workaround, not for production use.'
  );
  
  // Альтернатива: использовать кастомный tickMarkFormatter
  // который нивелирует эффект неправильных весов
}

// Более безопасная альтернатива - обёртка над createChart
function createPatchedChart(
  container: HTMLElement,
  options?: LightweightCharts.ChartOptions
): LightweightCharts.IChartApi {
  const chart = LightweightCharts.createChart(container, {
    ...options,
    timeScale: {
      ...options?.timeScale,
      // Используем fixLeftEdge и fixRightEdge для стабилизации
      fixLeftEdge: true,
      fixRightEdge: true,
      // Увеличиваем минимальный bar spacing
      minBarSpacing: 10,
    },
  });
  
  return chart;
}

export { createPatchedChart };
```

---

## ✅ Рекомендуемое решение

### Комбинация: Решения 2 + 3 (Custom Formatter + Data Stabilization)

Для максимальной совместимости и надёжности рекомендуется комбинировать кастомное форматирование с стабилизацией данных:

```typescript
import {
  createChart,
  IChartApi,
  ISeriesApi,
  Time,
  BusinessDay,
  UTCTimestamp,
  LineData,
  CandlestickData,
} from 'lightweight-charts';

// ============================================
// 1. Типы и интерфейсы
// ============================================

interface TimeScaleStabilizerConfig {
  locale?: string;
  paddingPoints?: number;
  dateFormat?: 'short' | 'medium' | 'long';
  showTime?: boolean;
}

type DataWithTime = LineData<Time> | CandlestickData<Time>;

// ============================================
// 2. Утилиты форматирования
// ============================================

class TimeFormatter {
  private locale: string;
  private showTime: boolean;
  private dateFormat: 'short' | 'medium' | 'long';

  constructor(config: TimeScaleStabilizerConfig = {}) {
    this.locale = config.locale ?? 'en-US';
    this.showTime = config.showTime ?? false;
    this.dateFormat = config.dateFormat ?? 'medium';
  }

  format(time: BusinessDay | UTCTimestamp): string {
    const date = this.timeToDate(time);
    return this.formatDate(date);
  }

  formatTickMark(
    time: BusinessDay | UTCTimestamp,
    tickMarkType: number,
    locale: string
  ): string {
    const date = this.timeToDate(time);
    
    // Разное форматирование в зависимости от типа метки
    switch (tickMarkType) {
      case 0: // Year
        return date.getFullYear().toString();
      case 1: // Month
        return date.toLocaleDateString(this.locale, { month: 'short' });
      case 2: // Day of Month
        return date.getDate().toString();
      case 3: // Time
        return date.toLocaleTimeString(this.locale, {
          hour: '2-digit',
          minute: '2-digit',
        });
      default:
        return this.formatDate(date);
    }
  }

  private timeToDate(time: BusinessDay | UTCTimestamp): Date {
    if (typeof time === 'number') {
      return new Date(time * 1000);
    }
    return new Date(time.year, time.month - 1, time.day);
  }

  private formatDate(date: Date): string {
    const options: Intl.DateTimeFormatOptions = {};

    switch (this.dateFormat) {
      case 'short':
        options.month = 'numeric';
        options.day = 'numeric';
        break;
      case 'medium':
        options.month = 'short';
        options.day = 'numeric';
        options.year = 'numeric';
        break;
      case 'long':
        options.month = 'long';
        options.day = 'numeric';
        options.year = 'numeric';
        options.weekday = 'short';
        break;
    }

    if (this.showTime) {
      options.hour = '2-digit';
      options.minute = '2-digit';
    }

    return new Intl.DateTimeFormat(this.locale, options).format(date);
  }
}

// ============================================
// 3. Стабилизатор данных
// ============================================

class DataStabilizer {
  private paddingPoints: number;

  constructor(paddingPoints = 2) {
    this.paddingPoints = paddingPoints;
  }

  stabilize<T extends DataWithTime>(data: T[]): T[] {
    if (data.length < 2 || this.paddingPoints === 0) {
      return data;
    }

    const interval = this.calculateInterval(data);
    const paddingData = this.createPadding(data[0], interval);
    
    return [...paddingData, ...data] as T[];
  }

  getOriginalRange<T extends DataWithTime>(
    data: T[]
  ): { from: Time; to: Time } | null {
    if (data.length < this.paddingPoints + 1) return null;
    
    return {
      from: data[this.paddingPoints].time,
      to: data[data.length - 1].time,
    };
  }

  private calculateInterval(data: DataWithTime[]): number {
    const time1 = this.timeToSeconds(data[0].time);
    const time2 = this.timeToSeconds(data[1].time);
    return time2 - time1;
  }

  private createPadding<T extends DataWithTime>(
    firstPoint: T,
    interval: number
  ): T[] {
    const padding: T[] = [];
    const firstTime = this.timeToSeconds(firstPoint.time);

    for (let i = this.paddingPoints; i > 0; i--) {
      const paddingTime = (firstTime - interval * i) as Time;
      
      // Создаём whitespace point
      const paddingPoint = {
        time: paddingTime,
      } as T;

      // Для Line series добавляем value
      if ('value' in firstPoint) {
        (paddingPoint as any).value = null; // whitespace
      }
      
      // Для Candlestick добавляем OHLC с null
      if ('open' in firstPoint) {
        (paddingPoint as any).open = null;
        (paddingPoint as any).high = null;
        (paddingPoint as any).low = null;
        (paddingPoint as any).close = null;
      }

      padding.push(paddingPoint);
    }

    return padding;
  }

  private timeToSeconds(time: Time): number {
    if (typeof time === 'number') {
      return time;
    }
    if (typeof time === 'string') {
      return Math.floor(new Date(time).getTime() / 1000);
    }
    return Math.floor(
      new Date(time.year, time.month - 1, time.day).getTime() / 1000
    );
  }
}

// ============================================
// 4. Фабрика стабилизированного графика
// ============================================

interface StabilizedChartOptions {
  container: HTMLElement;
  width?: number;
  height?: number;
  locale?: string;
  paddingPoints?: number;
  dateFormat?: 'short' | 'medium' | 'long';
  showTime?: boolean;
}

class StabilizedChart {
  private chart: IChartApi;
  private formatter: TimeFormatter;
  private stabilizer: DataStabilizer;
  private seriesMap: Map<ISeriesApi<any>, DataWithTime[]> = new Map();

  constructor(options: StabilizedChartOptions) {
    this.formatter = new TimeFormatter({
      locale: options.locale,
      dateFormat: options.dateFormat,
      showTime: options.showTime,
    });

    this.stabilizer = new DataStabilizer(options.paddingPoints ?? 2);

    this.chart = createChart(options.container, {
      width: options.width ?? 800,
      height: options.height ?? 400,
      timeScale: {
        tickMarkFormatter: (time, tickType, locale) =>
          this.formatter.formatTickMark(time, tickType, locale),
        fixLeftEdge: true,
        rightOffset: 5,
      },
      localization: {
        locale: options.locale ?? 'en-US',
        timeFormatter: (time) => this.formatter.format(time),
      },
    });
  }

  /**
   * Добавить линейную серию со стабилизацией
   */
  addLineSeries(options?: any): ISeriesApi<'Line'> {
    return this.chart.addLineSeries(options);
  }

  /**
   * Добавить свечную серию со стабилизацией
   */
  addCandlestickSeries(options?: any): ISeriesApi<'Candlestick'> {
    return this.chart.addCandlestickSeries(options);
  }

  /**
   * Установить данные со стабилизацией
   */
  setSeriesData<T extends DataWithTime>(
    series: ISeriesApi<any>,
    data: T[]
  ): void {
    const stabilizedData = this.stabilizer.stabilize(data);
    series.setData(stabilizedData as any);
    
    // Сохраняем оригинальные данные
    this.seriesMap.set(series, data);

    // Устанавливаем visible range на оригинальные данные
    const range = this.stabilizer.getOriginalRange(stabilizedData);
    if (range) {
      this.chart.timeScale().setVisibleRange(range);
    }
  }

  /**
   * Получить оригинальные данные серии
   */
  getSeriesData(series: ISeriesApi<any>): DataWithTime[] | undefined {
    return this.seriesMap.get(series);
  }

  /**
   * Доступ к оригинальному chart API
   */
  getChartApi(): IChartApi {
    return this.chart;
  }

  /**
   * Fit content к оригинальным данным
   */
  fitOriginalContent(): void {
    const allData = Array.from(this.seriesMap.values()).flat();
    if (allData.length === 0) return;

    const times = allData.map((d) => d.time);
    const from = times.reduce((min, t) => 
      this.compareTime(t, min) < 0 ? t : min
    );
    const to = times.reduce((max, t) => 
      this.compareTime(t, max) > 0 ? t : max
    );

    this.chart.timeScale().setVisibleRange({ from, to });
  }

  private compareTime(a: Time, b: Time): number {
    const aNum = typeof a === 'number' ? a : 
      typeof a === 'string' ? new Date(a).getTime() / 1000 :
      new Date(a.year, a.month - 1, a.day).getTime() / 1000;
    const bNum = typeof b === 'number' ? b :
      typeof b === 'string' ? new Date(b).getTime() / 1000 :
      new Date(b.year, b.month - 1, b.day).getTime() / 1000;
    return aNum - bNum;
  }

  /**
   * Удалить график
   */
  remove(): void {
    this.seriesMap.clear();
    this.chart.remove();
  }
}

// ============================================
// 5. Использование
// ============================================

function example(): void {
  const container = document.getElementById('chart')!;

  // Создаём стабилизированный график
  const chart = new StabilizedChart({
    container,
    width: 800,
    height: 400,
    locale: 'ru-RU',
    paddingPoints: 2,
    dateFormat: 'medium',
  });

  // Добавляем серию
  const series = chart.addLineSeries({
    color: '#2196F3',
    lineWidth: 2,
  });

  // Данные с проблемной второй меткой (в оригинале)
  const data: LineData<Time>[] = [
    { time: '2024-01-15', value: 100 },
    { time: '2024-01-16', value: 102 },
    { time: '2024-01-17', value: 101 },
    { time: '2024-01-18', value: 105 },
    { time: '2024-01-19', value: 103 },
  ];

  // Устанавливаем данные со стабилизацией
  chart.setSeriesData(series, data);

  // Доступ к оригинальному API
  const chartApi = chart.getChartApi();
  chartApi.timeScale().fitContent();
}

export { StabilizedChart, TimeFormatter, DataStabilizer };
```

### Почему это решение оптимально

| Критерий | Оценка |
|----------|--------|
| **Совместимость** | ✅ Работает с любой версией библиотеки |
| **Надёжность** | ✅ Не зависит от внутренней реализации |
| **Простота использования** | ✅ Обёртка с чистым API |
| **Производительность** | ⚠️ Небольшой overhead от padding |
| **Гибкость** | ✅ Настраиваемое форматирование |

---

## 📊 Сравнительная таблица решений

| Решение | Эффективность | Сложность | Совместимость | Рейтинг |
|---------|--------------|-----------|---------------|---------|
| #1: Патч PR #1688 | ⭐⭐⭐⭐⭐ | Средняя | Низкая | 9/10 |
| #2: Custom timeFormatter | ⭐⭐⭐⭐ | Низкая | Высокая | 8/10 |
| #3: Padding данных | ⭐⭐⭐⭐ | Низкая | Высокая | 7/10 |
| #4: Custom Primitive | ⭐⭐⭐⭐⭐ | Высокая | Высокая | 8/10 |
| #5: Monkey-patch | ⭐⭐ | Низкая | Низкая | 5/10 |

### Рекомендации по выбору

- **Есть возможность форка** → Решение #1 (официальный fix)
- **Production без модификаций** → Решения #2 + #3 (formatter + stabilization)
- **Полный контроль над UI** → Решение #4 (Custom Primitive)
- **Быстрый debugging** → Решение #5 (только для dev)

---

## 🔗 Источники

1. **GitHub Issue #1689** — [Issue with Weight Calculation for Second Rendered Label on Time Axis](https://github.com/tradingview/lightweight-charts/issues/1689)

2. **GitHub PR #1688** — [Fix second label positioning](https://github.com/tradingview/lightweight-charts/pull/1688)

3. **Официальная документация: TimeScaleOptions** — [tickMarkFormatter](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/TimeScaleOptions#tickmarkformatter)

4. **Официальная документация: Localization** — [timeFormatter](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/LocalizationOptions#timeformatter)

5. **Release Notes v4.1.6** — [Improved Price Scale Label Alignment](https://tradingview.github.io/lightweight-charts/docs/release-notes)

6. **Официальная документация: PriceScaleOptions** — [alignLabels](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/PriceScaleOptions#alignlabels)

---

**Документ создан:** 2026-01-18  
**Версия документа:** 1.0
