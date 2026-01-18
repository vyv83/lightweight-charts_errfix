# БАГ #29: Whitespace не отображается визуально в line series

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issues:** [#1947](https://github.com/tradingview/lightweight-charts/issues/1947), [#699](https://github.com/tradingview/lightweight-charts/issues/699)  
> **Версии:** Все версии (особенно v4.0+)  
> **Статус:** 🔴 Closed as "Working as Intended"  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы

В Lightweight Charts **whitespace данные** (точки только с `time` без `value`) **не создают визуальные разрывы** в line series. Вместо ожидаемого разрыва линии:

- Линия продолжает соединять точки ДО и ПОСЛЕ whitespace
- `null`, `undefined` или отсутствующие `value` не создают gaps
- Пользователи ожидают поведение как в Chart.js (spanGaps: false)

### Почему это происходит

По дизайну библиотеки, "whitespace" означает:
> "Нет значения для этой точки времени, но время должно присутствовать на шкале"

Это **НЕ** означает:
> "Нарисуй разрыв в линии"

Для **candlestick/bar series** whitespace создаёт видимый пропуск, потому что свечей просто нет. Но для **line series** линия соединяет все имеющиеся значения.

### Визуальное сравнение

```
Ожидаемое поведение (как в Chart.js):
    ───○───○───     ───○───○───
              gap (разрыв)

Фактическое поведение Lightweight Charts:
    ───○───○───○───○───○───○───
              ^ whitespace, но линия непрерывна
```

### Сценарии возникновения

```javascript
// Данные с whitespace (нет value)
const data = [
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 105 },
  { time: '2024-01-03' },            // Whitespace - нет value!
  { time: '2024-01-04' },            // Whitespace
  { time: '2024-01-05', value: 110 },
  { time: '2024-01-06', value: 108 },
];

// Линия будет СОЕДИНЯТЬ 105 и 110 напрямую
// Визуального разрыва НЕ БУДЕТ
```

### Когда это критично

1. **Пропуски в данных**: Сенсоры, которые не отправляют данные
2. **Отсутствие торгов**: Периоды без транзакций
3. **Разделение логических сегментов**: Разные торговые сессии
4. **Визуальная чистота**: Не соединять несвязанные данные

---

## 🔍 Найденные решения

### Решение 1: Множественные серии для каждого сегмента

**Оценка: 9/10** ⭐ РЕКОМЕНДУЕМОЕ

**Описание:** Разделение данных на непрерывные сегменты и создание отдельной серии для каждого.

**Преимущества:**
- Чистые визуальные разрывы
- Полный контроль над каждым сегментом
- Работает во всех версиях

**Недостатки:**
- Больше серий в графике
- Немного больше памяти
- Сложнее управление (обновление данных)

```javascript
/**
 * Разделяет данные на непрерывные сегменты
 * @param {Array} data - Исходные данные
 * @returns {Array[]} Массив сегментов
 */
function splitIntoSegments(data) {
  const segments = [];
  let currentSegment = [];

  for (const point of data) {
    if (point.value !== undefined && point.value !== null) {
      currentSegment.push(point);
    } else {
      // Whitespace - завершаем текущий сегмент
      if (currentSegment.length > 0) {
        segments.push(currentSegment);
        currentSegment = [];
      }
    }
  }

  // Добавляем последний сегмент
  if (currentSegment.length > 0) {
    segments.push(currentSegment);
  }

  return segments;
}

/**
 * Создаёт множественные серии для визуализации разрывов
 */
function createSegmentedLineSeries(chart, data, options = {}) {
  const segments = splitIntoSegments(data);
  const series = [];

  segments.forEach((segment, index) => {
    const lineSeries = chart.addLineSeries({
      color: options.color || '#2962FF',
      lineWidth: options.lineWidth || 2,
      // Показываем last value только для последнего сегмента
      lastValueVisible: index === segments.length - 1,
      priceLineVisible: index === segments.length - 1,
      crosshairMarkerVisible: true,
      ...options,
    });
    
    lineSeries.setData(segment);
    series.push(lineSeries);
  });

  return series;
}

// Использование
const allSeries = createSegmentedLineSeries(chart, data, {
  color: '#2962FF',
  lineWidth: 2,
});
```

---

### Решение 2: Area Series с прозрачной заливкой

**Оценка: 8/10**

**Описание:** Использование Area series, которая лучше обрабатывает gaps.

**Преимущества:**
- Одна серия
- Более простое управление

**Недостатки:**
- Визуально это Area, а не Line
- Может не подходить для некоторых случаев

```javascript
const areaSeries = chart.addAreaSeries({
  topColor: 'rgba(41, 98, 255, 0)',  // Прозрачная заливка
  bottomColor: 'rgba(41, 98, 255, 0)',
  lineColor: '#2962FF',
  lineWidth: 2,
});

// Для area series whitespace работает лучше
areaSeries.setData(data);
```

---

### Решение 3: lineType: 2 (WithGaps) - только для v3.8

**Оценка: 5/10** ⚠️ УСТАРЕВШЕЕ

**Описание:** В версии 3.8 существовала опция `lineType: 2` (LineType.WithGaps).

**Преимущества:**
- Простая конфигурация
- Работало как ожидается

**Недостатки:**
- **НЕ работает в v4.0+!**
- Устаревший API

```javascript
// ТОЛЬКО для v3.8 и ранее!
const lineSeries = chart.addLineSeries({
  lineType: 2,  // LineType.WithGaps
});

// null значения создавали визуальные разрывы
lineSeries.setData([
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: null },  // Gap!
  { time: '2024-01-03', value: 110 },
]);
```

---

### Решение 4: Custom Series с поддержкой gaps

**Оценка: 7/10**

**Описание:** Создание custom series plugin с явной поддержкой разрывов.

**Преимущества:**
- Полный контроль над рендерингом
- Можно реализовать любую логику

**Недостатки:**
- Сложная реализация
- Требует глубокого понимания API

```javascript
// Упрощённый пример custom renderer с gaps
class GappedLineRenderer {
  draw(target, priceConverter) {
    const ctx = target.context;
    let isDrawing = false;
    
    ctx.beginPath();
    ctx.strokeStyle = this.options.color;
    ctx.lineWidth = this.options.lineWidth;
    
    for (const point of this.data) {
      if (point.value === undefined || point.value === null) {
        // Gap - заканчиваем текущий путь
        if (isDrawing) {
          ctx.stroke();
          ctx.beginPath();
          isDrawing = false;
        }
        continue;
      }
      
      const x = target.timeToCoordinate(point.time);
      const y = priceConverter.priceToCoordinate(point.value);
      
      if (!isDrawing) {
        ctx.moveTo(x, y);
        isDrawing = true;
      } else {
        ctx.lineTo(x, y);
      }
    }
    
    if (isDrawing) {
      ctx.stroke();
    }
  }
}
```

---

### Решение 5: Обёртка с автоматическим разделением (Production-ready)

**Оценка: 10/10** ⭐⭐ ЛУЧШЕЕ РЕШЕНИЕ

**Описание:** Универсальная обёртка, которая автоматически обрабатывает gaps в line series.

**Преимущества:**
- Drop-in replacement для стандартного API
- Автоматическая сегментация
- Поддержка обновлений в реальном времени

---

## ✅ Рекомендуемое решение

### Полный код реализации

```javascript
import { createChart } from 'lightweight-charts';

// ============================================
// Класс для управления line series с gaps
// ============================================

/**
 * Обёртка над line series с поддержкой визуальных разрывов
 */
class GappedLineSeries {
  constructor(chart, options = {}) {
    this.chart = chart;
    this.options = {
      color: '#2962FF',
      lineWidth: 2,
      lastValueVisible: true,
      priceLineVisible: true,
      crosshairMarkerVisible: true,
      ...options,
    };
    
    this.segments = [];
    this.rawData = [];
  }

  /**
   * Устанавливает данные с автоматическим созданием сегментов
   */
  setData(data) {
    this.rawData = data;
    this._recreateSegments();
  }

  /**
   * Обновляет последнюю точку данных
   */
  update(bar) {
    if (bar.value === undefined || bar.value === null) {
      // Whitespace - начинаем новый сегмент
      this._startNewSegment();
      return;
    }

    // Обновляем последний сегмент
    const lastSegment = this.segments[this.segments.length - 1];
    if (lastSegment) {
      lastSegment.series.update(bar);
    } else {
      // Нет сегментов - создаём первый
      this._createSegment([bar]);
    }
  }

  /**
   * Удаляет все серии
   */
  remove() {
    for (const segment of this.segments) {
      this.chart.removeSeries(segment.series);
    }
    this.segments = [];
  }

  /**
   * Возвращает все внутренние серии
   */
  getAllSeries() {
    return this.segments.map(s => s.series);
  }

  // ============ Private methods ============

  _recreateSegments() {
    this.remove();  // Удаляем старые серии
    
    const segmentedData = this._splitIntoSegments(this.rawData);
    
    segmentedData.forEach((data, index) => {
      const isLast = index === segmentedData.length - 1;
      this._createSegment(data, isLast);
    });
  }

  _splitIntoSegments(data) {
    const segments = [];
    let currentSegment = [];

    for (const point of data) {
      const hasValue = point.value !== undefined && 
                       point.value !== null && 
                       !Number.isNaN(point.value);
      
      if (hasValue) {
        currentSegment.push(point);
      } else {
        if (currentSegment.length > 0) {
          segments.push(currentSegment);
          currentSegment = [];
        }
      }
    }

    if (currentSegment.length > 0) {
      segments.push(currentSegment);
    }

    return segments;
  }

  _createSegment(data, isLast = true) {
    const series = this.chart.addLineSeries({
      ...this.options,
      lastValueVisible: isLast && this.options.lastValueVisible,
      priceLineVisible: isLast && this.options.priceLineVisible,
    });
    
    series.setData(data);
    this.segments.push({ series, data });
    
    return series;
  }

  _startNewSegment() {
    // Следующий update создаст новый сегмент
    this._createSegment([], true);
  }

  // ============ Proxy methods ============

  applyOptions(options) {
    this.options = { ...this.options, ...options };
    this.segments.forEach((segment, index) => {
      const isLast = index === this.segments.length - 1;
      segment.series.applyOptions({
        ...options,
        lastValueVisible: isLast && this.options.lastValueVisible,
        priceLineVisible: isLast && this.options.priceLineVisible,
      });
    });
  }

  priceToCoordinate(price) {
    return this.segments[0]?.series.priceToCoordinate(price);
  }

  coordinateToPrice(coordinate) {
    return this.segments[0]?.series.coordinateToPrice(coordinate);
  }

  setMarkers(markers) {
    // Распределяем маркеры по соответствующим сегментам
    for (const segment of this.segments) {
      const segmentMarkers = markers.filter(m => 
        segment.data.some(d => d.time === m.time)
      );
      segment.series.setMarkers(segmentMarkers);
    }
  }
}

// ============================================
// Фабрика для создания графика с gapped lines
// ============================================

function createChartWithGappedLines(container, options = {}) {
  const chart = createChart(container, {
    width: options.width || 800,
    height: options.height || 400,
    layout: {
      background: { type: 'solid', color: '#1E222D' },
      textColor: '#DDD',
    },
    grid: {
      vertLines: { color: '#2B2B43' },
      horzLines: { color: '#2B2B43' },
    },
    ...options,
  });

  // Расширяем API
  return {
    chart,
    
    /**
     * Создаёт line series с поддержкой gaps
     */
    addGappedLineSeries(seriesOptions = {}) {
      return new GappedLineSeries(chart, seriesOptions);
    },

    // Proxy методы к оригинальному chart
    addCandlestickSeries: chart.addCandlestickSeries.bind(chart),
    addBarSeries: chart.addBarSeries.bind(chart),
    addHistogramSeries: chart.addHistogramSeries.bind(chart),
    addAreaSeries: chart.addAreaSeries.bind(chart),
    addLineSeries: chart.addLineSeries.bind(chart),  // Оригинальный метод
    timeScale: () => chart.timeScale(),
    priceScale: (id) => chart.priceScale(id),
    remove: () => chart.remove(),
  };
}

// ============================================
// Пример использования
// ============================================

// Создание графика
const chartWrapper = createChartWithGappedLines(document.getElementById('chart'));

// Добавление gapped line series
const gappedLine = chartWrapper.addGappedLineSeries({
  color: '#26a69a',
  lineWidth: 2,
});

// Данные с пропусками
const data = [
  { time: '2024-01-01', value: 100 },
  { time: '2024-01-02', value: 105 },
  { time: '2024-01-03', value: 103 },
  // Gap - нет данных за 4-5 января
  { time: '2024-01-04' },  // Whitespace
  { time: '2024-01-05' },  // Whitespace
  // Новый сегмент
  { time: '2024-01-06', value: 110 },
  { time: '2024-01-07', value: 108 },
  { time: '2024-01-08', value: 112 },
];

// Установка данных - автоматическое создание сегментов
gappedLine.setData(data);

// Подгонка временно́й шкалы
chartWrapper.timeScale().fitContent();
```

### React компонент

```tsx
import React, { useEffect, useRef, useMemo } from 'react';
import { createChart, IChartApi } from 'lightweight-charts';

interface DataPoint {
  time: string | number;
  value?: number;
}

interface GappedLineChartProps {
  data: DataPoint[];
  color?: string;
  lineWidth?: number;
  width?: number;
  height?: number;
}

// Хук для разделения данных на сегменты
function useSegmentedData(data: DataPoint[]) {
  return useMemo(() => {
    const segments: DataPoint[][] = [];
    let currentSegment: DataPoint[] = [];

    for (const point of data) {
      if (point.value !== undefined && point.value !== null) {
        currentSegment.push(point);
      } else {
        if (currentSegment.length > 0) {
          segments.push([...currentSegment]);
          currentSegment = [];
        }
      }
    }

    if (currentSegment.length > 0) {
      segments.push(currentSegment);
    }

    return segments;
  }, [data]);
}

export const GappedLineChart: React.FC<GappedLineChartProps> = ({
  data,
  color = '#2962FF',
  lineWidth = 2,
  width = 600,
  height = 400,
}) => {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<any[]>([]);

  const segments = useSegmentedData(data);

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
    });

    return () => {
      chartRef.current?.remove();
    };
  }, [width, height]);

  // Обновление сегментов
  useEffect(() => {
    if (!chartRef.current) return;

    // Удаляем старые серии
    seriesRef.current.forEach(series => {
      try {
        chartRef.current?.removeSeries(series);
      } catch (e) {
        // Series might already be removed
      }
    });
    seriesRef.current = [];

    // Создаём новые серии для каждого сегмента
    segments.forEach((segment, index) => {
      const isLast = index === segments.length - 1;
      
      const series = chartRef.current!.addLineSeries({
        color,
        lineWidth,
        lastValueVisible: isLast,
        priceLineVisible: isLast,
        crosshairMarkerVisible: true,
      });

      series.setData(segment);
      seriesRef.current.push(series);
    });

    chartRef.current.timeScale().fitContent();
  }, [segments, color, lineWidth]);

  return <div ref={containerRef} />;
};

// Пример использования
const ExampleUsage = () => {
  const data = [
    { time: '2024-01-01', value: 100 },
    { time: '2024-01-02', value: 105 },
    { time: '2024-01-03' },  // Gap
    { time: '2024-01-04' },  // Gap
    { time: '2024-01-05', value: 110 },
    { time: '2024-01-06', value: 108 },
  ];

  return (
    <GappedLineChart 
      data={data}
      color="#26a69a"
      lineWidth={2}
    />
  );
};
```

---

## 📊 Сравнительная таблица решений

| Критерий | Multi-Series | Area Series | lineType:2 | Custom Renderer | Wrapper Class |
|----------|:-----------:|:-----------:|:----------:|:---------------:|:-------------:|
| **Работает в v5.x** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Простота** | ⚠️ Средняя | ✅ Простая | ✅ Простая | 🔴 Сложная | ⚠️ Средняя |
| **Гибкость** | ✅ Высокая | ⚠️ Средняя | ❌ | ✅ Полная | ✅ Высокая |
| **Вид линии** | ✅ Line | ⚠️ Area | ✅ Line | ✅ Любой | ✅ Line |
| **Real-time updates** | ⚠️ Сложно | ✅ | - | ✅ | ✅ |
| **Производительность** | ⚠️ Много серий | ✅ | - | ✅ | ⚠️ |
| **Оценка** | **9/10** | **8/10** | **5/10** | **7/10** | **10/10** |

---

## 🎯 Дополнительные рекомендации

### 1. Определение gaps автоматически

```javascript
/**
 * Автоматически определяет gaps по временны́м интервалам
 */
function detectGaps(data, threshold = 1.5) {
  if (data.length < 2) return data;
  
  // Вычисляем медианный интервал
  const intervals = [];
  for (let i = 1; i < data.length; i++) {
    intervals.push(data[i].time - data[i - 1].time);
  }
  intervals.sort((a, b) => a - b);
  const medianInterval = intervals[Math.floor(intervals.length / 2)];
  
  // Помечаем gaps
  return data.map((point, i) => {
    if (i === 0) return point;
    
    const gap = point.time - data[i - 1].time;
    if (gap > medianInterval * threshold) {
      // Это начало после gap
      return { ...point, _afterGap: true };
    }
    return point;
  });
}
```

### 2. Анимированные gaps

```javascript
// Для плавного появления новых сегментов
function animateSegmentAppearance(series, data, duration = 500) {
  const step = duration / data.length;
  
  data.forEach((point, i) => {
    setTimeout(() => {
      series.update(point);
    }, step * i);
  });
}
```

### 3. Стилизация разных сегментов

```javascript
// Разные цвета для разных сегментов
const colors = ['#26a69a', '#ef5350', '#5C6BC0'];

segments.forEach((segment, index) => {
  chart.addLineSeries({
    color: colors[index % colors.length],
    lineWidth: 2,
  }).setData(segment);
});
```

---

## 🔗 Источники

1. **GitHub Issue #1947** - [Whitespace not showing in line series](https://github.com/tradingview/lightweight-charts/issues/1947)
2. **GitHub Issue #699** - [Line series gap/whitespace handling](https://github.com/tradingview/lightweight-charts/issues/699)
3. **Lightweight Charts Documentation** - [Whitespace Data](https://tradingview.github.io/lightweight-charts/docs/api#whitespace-data)
4. **GitHub Discussion** - [Feature Request: connectGaps option](https://github.com/tradingview/lightweight-charts/discussions)
5. **Stack Overflow** - [Line breaks in lightweight-charts](https://stackoverflow.com/questions/tagged/lightweight-charts+line+gaps)
6. **Chart.js Documentation** - [spanGaps Option](https://www.chartjs.org/docs/latest/charts/line.html#spangaps)

---

## 📝 Примечания

- Поведение **"Working as Intended"** - библиотека не планирует менять это в ближайшее время
- В v3.8 был `lineType: 2` (WithGaps), но он удалён в v4.0+
- Feature Request на добавление `connectGaps` опции открыт, но без конкретных сроков
- Multi-series подход - официально рекомендуемый workaround от разработчиков

---

*Документ создан: 18 января 2026*  
*Последнее обновление: 18 января 2026*
