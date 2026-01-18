# БАГ #33: График не обновляется при detach примитива

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1594](https://github.com/tradingview/lightweight-charts/issues/1594)  
> **Версии:** v4.1.x  
> **Статус:** ✅ Closed (Исправлено в v5.0)  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы
При вызове `series.detachPrimitive(primitive)` график не обновляется, и примитив продолжает отображаться на экране, хотя логически он уже отсоединён.

Проблема заключалась в том, что метод `detachPrimitive` не вызывал `requestUpdate()` перед уничтожением ссылок на функцию обновления.

### Причина бага

В файле `plugin-base.ts`:

```typescript
// ❌ Проблемный код (v4.1.x)
detached(): void {
  // Не вызывается requestUpdate перед очисткой!
  this._requestUpdate = undefined;  // Ссылка уничтожается
  this._series = undefined;
  this._chart = undefined;
}
```

### Пример проблемы

```tsx
import React, { useEffect, useRef, useState } from 'react';
import { createChart, IChartApi, ColorType, LineData } from 'lightweight-charts';
import { BandsIndicator } from './bands-indicator';

const ChartComponent = ({ data, showBands }: { data: LineData[]; showBands: boolean }) => {
  const chartContainerRef = useRef<HTMLDivElement>(null);
  const lineSeries = useRef<ReturnType<IChartApi["addLineSeries"]>>(null);
  const bandIndicator = useRef(new BandsIndicator());

  useEffect(() => {
    const chart = createChart(chartContainerRef.current!, {
      layout: {
        background: { type: ColorType.Solid, color: 'black' },
        textColor: 'white'
      },
      width: chartContainerRef.current!.clientWidth,
      height: 300
    });

    lineSeries.current = chart.addLineSeries();
    lineSeries.current.setData(data);
    chart.timeScale().fitContent();

    return () => chart.remove();
  }, [data]);

  useEffect(() => {
    if (!lineSeries.current) return;

    if (showBands) {
      lineSeries.current.attachPrimitive(bandIndicator.current);
    } else {
      // ❌ В v4.1.x график НЕ обновляется!
      lineSeries.current.detachPrimitive(bandIndicator.current);
      // Примитив визуально остаётся на месте
    }
  }, [showBands]);

  return <div ref={chartContainerRef} />;
};

const ChartWidget = () => {
  const [showBands, setShowBands] = useState(false);
  const sampleData = useMemo(() => generateLineData(), []);

  return (
    <>
      <ChartComponent data={sampleData} showBands={showBands} />
      <button onClick={() => setShowBands(!showBands)}>
        Toggle Bands
      </button>
    </>
  );
};
```

### Ожидаемое поведение
- При вызове `detachPrimitive()` график должен автоматически перерисоваться
- Примитив должен исчезнуть с экрана

### Фактическое поведение (в v4.1.x)
- График НЕ обновляется
- Примитив остаётся видимым до следующего взаимодействия с графиком

---

## 🔍 Найденные решения

### Решение 1: Обновление до v5.0+
**Оценка: 10/10** ⭐ ОФИЦИАЛЬНЫЙ ФИКС

Баг официально исправлен в версии 5.0. Release notes:
> "Fixed primitive detachment update issues (#1594)"

```bash
npm update lightweight-charts
# или
npm install lightweight-charts@^5.0.0
```

**Плюсы:**
- Официальное решение
- Без дополнительного кода
- Гарантированная стабильность

**Минусы:**
- Требует обновления версии
- Возможны breaking changes при миграции

---

### Решение 2: Ручной вызов requestUpdate перед detach
**Оценка: 8/10**

Если примитив реализует паттерн с сохранением `requestUpdate`.

```typescript
class MyPrimitive implements ISeriesPrimitive<Time> {
  private _requestUpdate: (() => void) | null = null;
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._requestUpdate = param.requestUpdate;
  }
  
  // Публичный метод для корректного отсоединения
  safeDetach(): void {
    // ✅ Вызываем requestUpdate ПЕРЕД очисткой
    this._requestUpdate?.();
    this._requestUpdate = null;
  }
  
  detached(): void {
    this._requestUpdate = null;
  }
}

// Использование
const primitive = new MyPrimitive();
series.attachPrimitive(primitive);

// При отсоединении:
primitive.safeDetach();       // Сначала вызываем обновление
series.detachPrimitive(primitive);  // Потом отсоединяем
```

**Плюсы:**
- Работает в v4.x
- Контроль над процессом

**Минусы:**
- Нужно модифицировать примитивы
- Дополнительный код

---

### Решение 3: Форсированное обновление графика после detach
**Оценка: 7/10**

Использование хака с изменением опций для принудительной перерисовки.

```typescript
// После detach
series.detachPrimitive(primitive);

// ✅ Форсируем перерисовку графика
chart.applyOptions({});  // Пустой вызов триггерит обновление

// Альтернатива - изменение timeScale
chart.timeScale().applyOptions({});
```

**Плюсы:**
- Простое решение
- Не требует модификации примитивов

**Минусы:**
- Хак, не официальный подход
- Может вызвать избыточные перерисовки

---

### Решение 4: "Скрытие" примитива вместо detach
**Оценка: 6/10**

Для случаев с проблемами производительности — не отсоединять примитив, а скрывать его.

```typescript
class ToggleablePrimitive implements ISeriesPrimitive<Time> {
  private _visible: boolean = true;
  private _requestUpdate: (() => void) | null = null;
  
  attached(param: SeriesAttachedParameter<Time>): void {
    this._requestUpdate = param.requestUpdate;
  }
  
  // Вместо detach — просто скрываем
  hide(): void {
    this._visible = false;
    this._requestUpdate?.();
  }
  
  show(): void {
    this._visible = true;
    this._requestUpdate?.();
  }
  
  paneViews(): IPrimitivePaneView[] {
    // Если скрыт — возвращаем пустой массив
    if (!this._visible) return [];
    return [this._paneView];
  }
  
  detached(): void {
    this._requestUpdate = null;
  }
}

// Использование
const primitive = new ToggleablePrimitive();
series.attachPrimitive(primitive);

// Вместо detachPrimitive:
primitive.hide();  // ✅ График обновится и примитив исчезнет

// Показать обратно:
primitive.show();
```

**Плюсы:**
- Обходит проблему полностью
- Лучшая производительность при частых toggle
- Не требует detach/attach cycles

**Минусы:**
- Примитив остаётся прикреплённым (memory)
- Нужно изменить архитектуру

---

### Решение 5: Патч plugin-base.ts (для своих примитивов)
**Оценка: 7/10**

Если вы используете примитивы из plugin-examples и не можете обновить библиотеку.

```typescript
// patched-plugin-base.ts
import { 
  ISeriesPrimitive, 
  SeriesAttachedParameter, 
  Time 
} from 'lightweight-charts';

export abstract class PatchedPluginBase<T extends Time> implements ISeriesPrimitive<T> {
  protected _chart: Parameters<ISeriesPrimitive<T>['attached']>[0]['chart'] | undefined;
  protected _series: Parameters<ISeriesPrimitive<T>['attached']>[0]['series'] | undefined;
  protected _requestUpdate: (() => void) | undefined;

  attached(param: SeriesAttachedParameter<T>): void {
    this._chart = param.chart;
    this._series = param.series;
    this._requestUpdate = param.requestUpdate;
  }

  detached(): void {
    // ✅ ФИКС: Вызываем requestUpdate ПЕРЕД очисткой
    if (this._requestUpdate) {
      this._requestUpdate();
    }
    
    this._chart = undefined;
    this._series = undefined;
    this._requestUpdate = undefined;
  }

  protected requestUpdate(): void {
    this._requestUpdate?.();
  }

  abstract updateAllViews(): void;
  abstract paneViews(): readonly any[];
}

// Использование
class MyIndicator extends PatchedPluginBase<Time> {
  updateAllViews(): void {
    // ...
  }
  
  paneViews() {
    // ...
  }
}
```

**Плюсы:**
- Точное исправление проблемы
- Работает для своих примитивов

**Минусы:**
- Нужно унаследовать все примитивы от нового класса
- Дублирование кода из библиотеки

---

## ✅ Рекомендуемое решение

### Для всех: Обновление до v5.0+

```bash
npm install lightweight-charts@latest
```

Это решает проблему официально и без workarounds.

### Для проектов на v4.x (без возможности обновления)

Используйте комбинацию решений 2+3:

```typescript
// Обёртка для безопасного detach
function safeDetachPrimitive<T extends ISeriesPrimitive<Time>>(
  chart: IChartApi,
  series: ISeriesApi<SeriesType>,
  primitive: T
): void {
  // Если примитив имеет safeDetach — используем его
  if ('safeDetach' in primitive && typeof primitive.safeDetach === 'function') {
    (primitive as any).safeDetach();
  }
  
  // Отсоединяем примитив
  series.detachPrimitive(primitive);
  
  // Форсируем обновление графика
  chart.applyOptions({});
}

// Использование
safeDetachPrimitive(chart, series, myPrimitive);
```

### Полный пример для React (v4.x)

```tsx
import React, { useEffect, useRef, useState, useCallback } from 'react';
import { 
  createChart, 
  IChartApi, 
  ISeriesApi, 
  Time 
} from 'lightweight-charts';

// Custom hook для управления примитивами
function usePrimitive<T extends ISeriesPrimitive<Time>>(
  chart: IChartApi | null,
  series: ISeriesApi<SeriesType> | null,
  primitive: T,
  isVisible: boolean
) {
  const isAttached = useRef(false);

  useEffect(() => {
    if (!chart || !series) return;

    if (isVisible && !isAttached.current) {
      series.attachPrimitive(primitive);
      isAttached.current = true;
    } else if (!isVisible && isAttached.current) {
      // ✅ WORKAROUND для v4.x
      series.detachPrimitive(primitive);
      chart.applyOptions({});  // Форсируем обновление
      isAttached.current = false;
    }
  }, [chart, series, primitive, isVisible]);

  // Cleanup при размонтировании
  useEffect(() => {
    return () => {
      if (series && isAttached.current) {
        series.detachPrimitive(primitive);
        isAttached.current = false;
      }
    };
  }, [series, primitive]);
}

// Использование в компоненте
function ChartWithToggleablePrimitive() {
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<'Line'> | null>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  
  const [showIndicator, setShowIndicator] = useState(false);
  const indicator = useRef(new MyIndicator());

  useEffect(() => {
    if (!containerRef.current) return;
    
    const chart = createChart(containerRef.current, { width: 800, height: 400 });
    chartRef.current = chart;
    
    const series = chart.addLineSeries();
    seriesRef.current = series;
    series.setData(generateData());
    
    return () => {
      chart.remove();
    };
  }, []);

  // Управление примитивом с workaround
  usePrimitive(
    chartRef.current,
    seriesRef.current,
    indicator.current,
    showIndicator
  );

  return (
    <div>
      <div ref={containerRef} />
      <button onClick={() => setShowIndicator(!showIndicator)}>
        {showIndicator ? 'Hide' : 'Show'} Indicator
      </button>
    </div>
  );
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Для версии | Надёжность |
|---------|--------|-----------|------------|------------|
| 1. **Обновление до v5.0** | **10/10** | Низкая | v5.0+ | ✅ Максимальная |
| 2. safeDetach внутри примитива | 8/10 | Средняя | v4.x | ✅ Высокая |
| 3. chart.applyOptions({}) | 7/10 | Низкая | v4.x | ⚠️ Хак |
| 4. Скрытие вместо detach | 6/10 | Средняя | Все | ✅ Надёжная |
| 5. Патч plugin-base | 7/10 | Высокая | v4.x | ✅ Высокая |

### Рекомендации:

- **Новые проекты** → v5.0+ (Решение 1)
- **Не можете обновиться** → Решение 2 + 3 (комбинация)
- **Частый toggle** → Решение 4 (скрытие)

---

## 🔗 Источники

1. [GitHub Issue #1594](https://github.com/tradingview/lightweight-charts/issues/1594) - Оригинальный баг-репорт
2. [v5.0 Release Notes](https://tradingview.github.io/lightweight-charts/docs/release-notes) - Описание фикса
3. [Plugin Examples](https://github.com/tradingview/lightweight-charts/tree/master/plugin-examples) - Примеры плагинов
4. [ISeriesPrimitive Interface](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesPrimitive) - API Reference
5. [CodeSandbox Reproduction](https://codesandbox.io/p/sandbox/jolly-christian-lkmqnh) - Пример воспроизведения

---

## 📝 Примечания

### Статус исправления
Баг исправлен в **v5.0**. Фикс заключался в добавлении вызова `requestUpdate()` в метод `detached()` класса `PluginBase`.

```typescript
// ✅ Исправленный код (v5.0+)
detached(): void {
  this._requestUpdate?.();  // ← Добавлена эта строка
  this._requestUpdate = undefined;
  this._series = undefined;
  this._chart = undefined;
}
```

### Миграция на v5.0
При миграции на v5.0 учитывайте:
- ESM-only сборка (CommonJS больше не поддерживается)
- Некоторые изменения в API серий
- Обновлённые типы TypeScript

Проверьте [Migration Guide](https://tradingview.github.io/lightweight-charts/docs/migration) перед обновлением.
