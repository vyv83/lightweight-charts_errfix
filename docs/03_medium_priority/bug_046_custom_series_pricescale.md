# БАГ #46: Некорректное поведение кастомной серии влияет на price scale

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1498](https://github.com/tradingview/lightweight-charts/issues/1498)  
> **Версии:** v4.1+, включая v5.x  
> **Статус:** 🔴 Open  
> **Последнее обновление:** Январь 2024

## 📋 Описание проблемы

### Суть проблемы

При добавлении **кастомной серии (custom series)** на график, **price scale растягивается до очень большого значения**, игнорируя реальный диапазон данных. Настройка `autoScale: true` не решает проблему.

### Детали

1. **Симптомы:**
   - Price scale показывает диапазон от 0 до огромного значения (например, 1e+308)
   - Реальные данные становятся невидимыми или сжимаются в тонкую линию
   - AutoScale не корректирует масштаб

2. **Причина:**
   - Custom series возвращает некорректные значения для price range calculation
   - Внутренние функции `minMaxOnRangeCached()` получают неверные boundaries
   - Проблема в интерфейсе `ICustomSeriesPaneView`

3. **Когда возникает:**
   - При создании custom series с методом `priceValueBuilder`
   - При возврате `null`/`undefined` значений из custom series
   - При неправильной реализации `autoscaleInfo()`

### Сценарии возникновения

```typescript
// Пример проблемного custom series
import { 
  createChart, 
  CustomSeriesOptions,
  ICustomSeriesPaneView,
  PaneRendererCustomData,
  WhitespaceData,
  Time
} from 'lightweight-charts';

interface MySeriesData {
  time: Time;
  value: number;
}

// ❌ ПРОБЛЕМА: Неправильная реализация custom series
class MyCustomSeries implements ICustomSeriesPaneView<Time, MySeriesData> {
  priceValueBuilder(plotRow: MySeriesData): number[] {
    // Возвращаем пустой массив или некорректные значения
    return []; // ЭТО ВЫЗЫВАЕТ ПРОБЛЕМУ!
  }
  
  isWhitespace(data: MySeriesData | WhitespaceData): data is WhitespaceData {
    return (data as WhitespaceData).time !== undefined && 
           !('value' in data);
  }
  
  renderer(): any {
    // ... renderer implementation
  }
  
  // ❌ Отсутствует autoscaleInfo() - критическая ошибка!
}
```

### Реальные сценарии

1. **Кастомные типы графиков:**
   - Heikin-Ashi свечи
   - Renko блоки
   - Point & Figure

2. **Специализированные визуализации:**
   - Heatmaps
   - Profile distributions
   - Custom indicators

3. **Интеграция внешних библиотек:**
   - D3.js overlays
   - Custom drawing libraries

## 🔍 Найденные решения

### Решение 1: Корректная реализация priceValueBuilder (⭐ Рекомендуемое)

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Правильный подход

```typescript
import { 
  createChart, 
  ICustomSeriesPaneView,
  CustomData,
  PriceToCoordinateConverter,
  Time,
  ISeriesPrimitivePaneRenderer,
  ISeriesPrimitivePaneView
} from 'lightweight-charts';

interface MyBarData extends CustomData<Time> {
  time: Time;
  open: number;
  high: number;
  low: number;
  close: number;
}

class CorrectCustomSeries implements ICustomSeriesPaneView<Time, MyBarData> {
  
  // ✅ ПРАВИЛЬНО: Возвращаем ВСЕ значения, используемые для price scale
  priceValueBuilder(plotRow: MyBarData): number[] {
    // Для OHLC данных возвращаем все 4 значения
    return [plotRow.open, plotRow.high, plotRow.low, plotRow.close];
  }
  
  // ✅ ПРАВИЛЬНО: Корректная проверка whitespace
  isWhitespace(data: MyBarData): boolean {
    return !('open' in data) || data.open === undefined;
  }
  
  renderer(): ISeriesPrimitivePaneRenderer {
    return new MyCustomRenderer();
  }
  
  // ✅ ВАЖНО: Опциональный, но рекомендуемый метод
  defaultOptions(): Partial<CustomSeriesOptions> {
    return {
      priceScaleId: 'right',
    };
  }
}

// Для простых line-подобных данных
interface SimpleLineData extends CustomData<Time> {
  time: Time;
  value: number;
}

class SimpleCustomSeries implements ICustomSeriesPaneView<Time, SimpleLineData> {
  
  priceValueBuilder(plotRow: SimpleLineData): number[] {
    // ✅ Возвращаем одно значение для line series
    if (plotRow.value === undefined || plotRow.value === null) {
      return []; // Whitespace - пустой массив корректен
    }
    return [plotRow.value];
  }
  
  isWhitespace(data: SimpleLineData): boolean {
    return data.value === undefined || data.value === null;
  }
  
  renderer(): ISeriesPrimitivePaneRenderer {
    return new SimpleLineRenderer();
  }
}
```

**Плюсы:**
- Решает проблему в корне
- Следует документации библиотеки
- Надёжная работа autoscale

**Минусы:**
- Требует понимания внутренней работы библиотеки

---

### Решение 2: Реализация autoscaleInfo() для точного контроля

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Полный контроль над масштабированием

```typescript
import { 
  ICustomSeriesPaneView,
  AutoscaleInfo,
  CustomData,
  Time,
  Logical
} from 'lightweight-charts';

interface MyData extends CustomData<Time> {
  time: Time;
  value: number;
  upperBand?: number;
  lowerBand?: number;
}

class CustomSeriesWithAutoscale implements ICustomSeriesPaneView<Time, MyData> {
  private _data: MyData[] = [];
  
  // Сохраняем данные для расчёта autoscale
  update(data: MyData[]): void {
    this._data = data;
  }
  
  priceValueBuilder(plotRow: MyData): number[] {
    const values: number[] = [];
    
    if (plotRow.value !== undefined) {
      values.push(plotRow.value);
    }
    if (plotRow.upperBand !== undefined) {
      values.push(plotRow.upperBand);
    }
    if (plotRow.lowerBand !== undefined) {
      values.push(plotRow.lowerBand);
    }
    
    return values;
  }
  
  isWhitespace(data: MyData): boolean {
    return data.value === undefined;
  }
  
  // ✅ КЛЮЧЕВОЙ МЕТОД: Явно указываем диапазон для autoscale
  autoscaleInfo(
    startTimePoint: Logical,
    endTimePoint: Logical
  ): AutoscaleInfo | null {
    // Фильтруем данные в видимом диапазоне
    const visibleData = this._data.filter((_, index) => {
      return index >= startTimePoint && index <= endTimePoint;
    });
    
    if (visibleData.length === 0) {
      return null;
    }
    
    // Вычисляем min/max для видимых данных
    let min = Infinity;
    let max = -Infinity;
    
    for (const item of visibleData) {
      if (item.value !== undefined) {
        min = Math.min(min, item.value);
        max = Math.max(max, item.value);
      }
      if (item.upperBand !== undefined) {
        max = Math.max(max, item.upperBand);
      }
      if (item.lowerBand !== undefined) {
        min = Math.min(min, item.lowerBand);
      }
    }
    
    if (!Number.isFinite(min) || !Number.isFinite(max)) {
      return null;
    }
    
    // Добавляем margin для лучшего отображения
    const range = max - min;
    const margin = range * 0.1;
    
    return {
      priceRange: {
        minValue: min - margin,
        maxValue: max + margin,
      },
      // Опционально: указываем margins для price scale
      margins: {
        above: 10, // пиксели
        below: 10,
      },
    };
  }
  
  renderer() {
    return new MyRenderer(this._data);
  }
}

// Использование
const chart = createChart(container);

const customSeriesView = new CustomSeriesWithAutoscale();
const series = chart.addCustomSeries(customSeriesView);

const data: MyData[] = [
  { time: '2024-01-01', value: 100, upperBand: 110, lowerBand: 90 },
  { time: '2024-01-02', value: 105, upperBand: 115, lowerBand: 95 },
  // ...
];

customSeriesView.update(data);
series.setData(data);
```

**Плюсы:**
- Полный контроль над autoscale
- Можно учитывать дополнительные элементы (bands, markers)
- Настраиваемые margins

**Минусы:**
- Больше кода для реализации
- Нужно вручную отслеживать данные

---

### Решение 3: Принудительная установка price scale range

**Оценка:** ⭐⭐⭐ (3/5) - Workaround для простых случаев

```typescript
import { createChart, IChartApi, ISeriesApi, CustomSeriesOptions } from 'lightweight-charts';

class ChartWithCustomSeries {
  private chart: IChartApi;
  private customSeries: ISeriesApi<'Custom'>;
  
  constructor(container: HTMLElement) {
    this.chart = createChart(container);
    this.customSeries = this.chart.addCustomSeries(new MyCustomSeries());
  }
  
  setData(data: MyData[]): void {
    this.customSeries.setData(data);
    
    // Принудительно устанавливаем диапазон price scale
    this.forceUpdatePriceScale(data);
  }
  
  private forceUpdatePriceScale(data: MyData[]): void {
    const values = data
      .filter(d => d.value !== undefined)
      .map(d => d.value);
    
    if (values.length === 0) return;
    
    const min = Math.min(...values);
    const max = Math.max(...values);
    const range = max - min || 1;
    const margin = range * 0.1;
    
    // Отключаем autoscale и устанавливаем диапазон вручную
    this.customSeries.priceScale().applyOptions({
      autoScale: false,
    });
    
    // Используем setVisiblePriceRange если доступен
    // Или добавляем невидимую серию с нужным диапазоном
    this.addInvisibleRangeSeries(min - margin, max + margin);
  }
  
  private addInvisibleRangeSeries(min: number, max: number): void {
    // Создаём невидимую линейную серию для установки диапазона
    const invisibleSeries = this.chart.addLineSeries({
      visible: false,
      priceScaleId: this.customSeries.options().priceScaleId || 'right',
    });
    
    const data = this.customSeries.data();
    if (data.length < 2) return;
    
    const firstTime = data[0].time;
    const lastTime = data[data.length - 1].time;
    
    invisibleSeries.setData([
      { time: firstTime, value: min },
      { time: lastTime, value: max },
    ]);
  }
}
```

**Плюсы:**
- Быстрый workaround
- Не требует изменения custom series

**Минусы:**
- Hack, не идиоматичное решение
- Требует дополнительной невидимой серии
- Может влиять на производительность

---

### Решение 4: Валидация данных перед передачей

**Оценка:** ⭐⭐⭐⭐ (4/5) - Защитная мера

```typescript
import { CustomData, Time } from 'lightweight-charts';

interface ValidatedData<T extends CustomData<Time>> {
  data: T[];
  priceRange: { min: number; max: number } | null;
  warnings: string[];
}

/**
 * Валидирует данные для custom series
 */
function validateCustomSeriesData<T extends CustomData<Time>>(
  data: T[],
  valueExtractor: (item: T) => number[]
): ValidatedData<T> {
  const warnings: string[] = [];
  const validData: T[] = [];
  let min = Infinity;
  let max = -Infinity;
  
  for (let i = 0; i < data.length; i++) {
    const item = data[i];
    const values = valueExtractor(item);
    
    // Проверяем каждое значение
    let isValid = true;
    for (const value of values) {
      if (value === undefined || value === null) {
        continue; // whitespace допустим
      }
      
      if (!Number.isFinite(value)) {
        warnings.push(`Invalid value at index ${i}: ${value}`);
        isValid = false;
        break;
      }
      
      // Проверяем на экстремальные значения
      if (Math.abs(value) > 1e12) {
        warnings.push(`Extreme value at index ${i}: ${value}`);
        isValid = false;
        break;
      }
      
      min = Math.min(min, value);
      max = Math.max(max, value);
    }
    
    if (isValid) {
      validData.push(item);
    }
  }
  
  return {
    data: validData,
    priceRange: Number.isFinite(min) && Number.isFinite(max) 
      ? { min, max } 
      : null,
    warnings,
  };
}

// Использование
const rawData: MyData[] = fetchData();

const { data, priceRange, warnings } = validateCustomSeriesData(
  rawData,
  (item) => item.value !== undefined ? [item.value] : []
);

if (warnings.length > 0) {
  console.warn('Data validation warnings:', warnings);
}

customSeries.setData(data);
```

**Плюсы:**
- Предотвращает проблемы с некорректными данными
- Логирование проблем
- Переиспользуемый код

**Минусы:**
- Не решает проблему в самом custom series
- Дополнительный overhead

---

### Решение 5: Wrapper для безопасного custom series

**Оценка:** ⭐⭐⭐⭐⭐ (5/5) - Комплексное решение

```typescript
import { 
  ICustomSeriesPaneView, 
  CustomData, 
  Time,
  AutoscaleInfo,
  Logical,
  ISeriesPrimitivePaneRenderer
} from 'lightweight-charts';

/**
 * Базовый класс для безопасных custom series
 */
abstract class SafeCustomSeries<TData extends CustomData<Time>> 
  implements ICustomSeriesPaneView<Time, TData> {
  
  protected _data: TData[] = [];
  
  // Абстрактные методы для реализации в подклассах
  abstract extractPriceValues(item: TData): number[];
  abstract createRenderer(): ISeriesPrimitivePaneRenderer;
  abstract isDataWhitespace(item: TData): boolean;
  
  // Реализация ICustomSeriesPaneView
  priceValueBuilder(plotRow: TData): number[] {
    try {
      const values = this.extractPriceValues(plotRow);
      
      // Фильтруем невалидные значения
      return values.filter(v => 
        v !== undefined && 
        v !== null && 
        Number.isFinite(v) &&
        Math.abs(v) < 1e12
      );
    } catch (error) {
      console.warn('Error in priceValueBuilder:', error);
      return [];
    }
  }
  
  isWhitespace(data: TData): boolean {
    try {
      return this.isDataWhitespace(data);
    } catch {
      return true; // В случае ошибки считаем whitespace
    }
  }
  
  renderer(): ISeriesPrimitivePaneRenderer {
    return this.createRenderer();
  }
  
  // Автоматический расчёт autoscale
  autoscaleInfo(startTimePoint: Logical, endTimePoint: Logical): AutoscaleInfo | null {
    try {
      const visibleData = this._data.filter((_, idx) => 
        idx >= startTimePoint && idx <= endTimePoint
      );
      
      if (visibleData.length === 0) return null;
      
      let min = Infinity;
      let max = -Infinity;
      
      for (const item of visibleData) {
        if (this.isDataWhitespace(item)) continue;
        
        const values = this.extractPriceValues(item);
        for (const v of values) {
          if (Number.isFinite(v)) {
            min = Math.min(min, v);
            max = Math.max(max, v);
          }
        }
      }
      
      if (!Number.isFinite(min) || !Number.isFinite(max)) {
        return null;
      }
      
      const range = max - min || 1;
      const margin = range * 0.05;
      
      return {
        priceRange: {
          minValue: min - margin,
          maxValue: max + margin,
        },
      };
    } catch (error) {
      console.warn('Error in autoscaleInfo:', error);
      return null;
    }
  }
  
  // Метод для обновления данных
  updateData(data: TData[]): void {
    this._data = data;
  }
}

// ==================== ПРИМЕР ИСПОЛЬЗОВАНИЯ ====================

interface BollingerData extends CustomData<Time> {
  time: Time;
  middle: number;
  upper: number;
  lower: number;
}

class BollingerBandsSeries extends SafeCustomSeries<BollingerData> {
  
  extractPriceValues(item: BollingerData): number[] {
    return [item.middle, item.upper, item.lower];
  }
  
  isDataWhitespace(item: BollingerData): boolean {
    return item.middle === undefined;
  }
  
  createRenderer(): ISeriesPrimitivePaneRenderer {
    return new BollingerRenderer(this._data);
  }
}

// Использование
const bollingerSeries = new BollingerBandsSeries();
const series = chart.addCustomSeries(bollingerSeries);

const data: BollingerData[] = [
  { time: '2024-01-01', middle: 100, upper: 110, lower: 90 },
  { time: '2024-01-02', middle: 102, upper: 112, lower: 92 },
];

bollingerSeries.updateData(data);
series.setData(data);
```

## ✅ Рекомендуемое решение

Используйте **решение 5 (SafeCustomSeries wrapper)** как основу для всех custom series:

```typescript
// Минимальный пример правильной реализации
class MyCustomSeries implements ICustomSeriesPaneView<Time, MyData> {
  private _data: MyData[] = [];
  
  // 1. Корректный priceValueBuilder
  priceValueBuilder(plotRow: MyData): number[] {
    if (plotRow.value === undefined || !Number.isFinite(plotRow.value)) {
      return [];
    }
    return [plotRow.value];
  }
  
  // 2. Правильная проверка whitespace
  isWhitespace(data: MyData): boolean {
    return data.value === undefined;
  }
  
  // 3. Реализация autoscaleInfo для надёжности
  autoscaleInfo(start: Logical, end: Logical): AutoscaleInfo | null {
    const visible = this._data.slice(start, end + 1);
    const values = visible.flatMap(d => 
      d.value !== undefined ? [d.value] : []
    );
    
    if (values.length === 0) return null;
    
    return {
      priceRange: {
        minValue: Math.min(...values) * 0.95,
        maxValue: Math.max(...values) * 1.05,
      },
    };
  }
  
  renderer(): ISeriesPrimitivePaneRenderer {
    return new MyRenderer(this._data);
  }
  
  update(data: MyData[]): void {
    this._data = data;
  }
}
```

## 📊 Сравнительная таблица решений

| Решение | Надёжность | Сложность | Производительность | Рекомендация |
|---------|------------|-----------|-------------------|--------------|
| **#1 Правильный priceValueBuilder** | ⭐⭐⭐⭐⭐ | Низкая | Высокая | ✅ Обязательно |
| **#2 autoscaleInfo()** | ⭐⭐⭐⭐⭐ | Средняя | Высокая | ✅ Рекомендуется |
| **#3 Принудительный range** | ⭐⭐⭐ | Средняя | Средняя | Workaround |
| **#4 Валидация данных** | ⭐⭐⭐⭐ | Низкая | Высокая | Дополнительно |
| **#5 SafeCustomSeries** | ⭐⭐⭐⭐⭐ | Средняя | Высокая | ✅ Лучший выбор |

## 🔧 Чеклист для custom series

1. ✅ `priceValueBuilder()` возвращает **все числовые значения** для price scale
2. ✅ `priceValueBuilder()` возвращает **пустой массив** `[]` для whitespace
3. ✅ `isWhitespace()` корректно определяет пустые данные
4. ✅ Реализован `autoscaleInfo()` для точного контроля масштаба
5. ✅ Данные валидируются перед передачей в серию
6. ✅ Нет `undefined`, `null`, `NaN`, `Infinity` в числовых значениях

## 🔗 Источники

- [GitHub Issue #1498](https://github.com/tradingview/lightweight-charts/issues/1498) - Price scale issue on adding custom series
- [Custom Series Tutorial](https://tradingview.github.io/lightweight-charts/tutorials/custom_series/overview) - Официальная документация
- [Plugin Examples](https://tradingview.github.io/lightweight-charts/plugin-examples/) - Примеры плагинов
- [CodePen репродукция](https://codepen.io/danishahmedkhan/pen/BabRvQx) - Демонстрация бага

---

**Документ создан:** 2026-01-18  
**Версия lightweight-charts:** v5.1.0 и ранее
