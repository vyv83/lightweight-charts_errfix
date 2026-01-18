# БАГ #32: subscribeCrosshairMove возвращает некорректные координаты

> **Критичность:** 🟡 СРЕДНЯЯ  
> **Issue:** [#1697](https://github.com/tradingview/lightweight-charts/issues/1697)  
> **Версии:** Все версии  
> **Статус:** 🔴 Open (Needs Investigation)  
> **Дата документирования:** 18 января 2026

---

## 📋 Описание проблемы

### Суть проблемы
При использовании `subscribeCrosshairMove` с включённым режимом `magnet` возникает несоответствие между:

- **Координатами в `param.point`** — это позиция мыши (mouse position)
- **Позицией кроссхэра** — это позиция на ближайшей свече (snapped position)

Когда режим `magnet` активен, кроссхэр "притягивается" к ближайшей свече, но `param.point` всё ещё возвращает координаты курсора мыши. Это приводит к несоответствию при попытке нарисовать элементы на позиции кроссхэра.

### Пример проблемы

```javascript
const chart = createChart(container, {
  crosshair: {
    mode: CrosshairMode.Magnet  // Кроссхэр притягивается к свечам
  }
});

const series = chart.addSeries(CandlestickSeries);
series.setData(candleData);

chart.subscribeCrosshairMove((param) => {
  if (!param.point) return;
  
  // ❌ ПРОБЛЕМА: param.point содержит координаты МЫШИ, а не кроссхэра!
  const x = param.point.x;  // Позиция мыши по X
  const y = param.point.y;  // Позиция мыши по Y
  
  // При попытке нарисовать линию на этих координатах
  // она будет на позиции мыши, а не на позиции кроссхэра
  drawLineAt(x, y);
});
```

### Визуальное проявление
```
Мышь:     ─────────●──────────────  (курсор здесь)
                   │
Кроссхэр: ─────────────┼───────────  (кроссхэр здесь, на свече)
                       │
Линия:    ─────────────────────────  (рисуется на позиции мыши, не кроссхэра)
```

### Ожидаемое поведение
- В режиме `magnet` координаты должны соответствовать позиции кроссхэра (на свече)
- ИЛИ должны предоставляться отдельные координаты для мыши и кроссхэра

---

## 🔍 Найденные решения

### Решение 1: Использование seriesPrices для получения цены на кроссхэре
**Оценка: 8/10** ⭐ РЕКОМЕНДУЕМОЕ

Для получения ТОЧНОЙ цены на позиции кроссхэра используйте `param.seriesPrices`.

```javascript
chart.subscribeCrosshairMove((param) => {
  if (!param.point || !param.time) return;
  
  // ✅ Получаем цену из seriesPrices - это цена НА кроссхэре
  const seriesPrice = param.seriesPrices.get(series);
  
  if (seriesPrice) {
    // Для OHLC данных seriesPrice - это объект
    if (typeof seriesPrice === 'object') {
      const ohlc = seriesPrice as CandlestickData;
      console.log('OHLC на кроссхэре:', ohlc);
      
      // Получаем координаты КРОССХЭРА (не мыши!)
      const crosshairY = series.priceToCoordinate(ohlc.close);
      const crosshairX = chart.timeScale().timeToCoordinate(param.time);
      
      if (crosshairX !== null && crosshairY !== null) {
        drawMarkerAt(crosshairX, crosshairY);
      }
    } else {
      // Для line/area series - это число
      const price = seriesPrice as number;
      const crosshairY = series.priceToCoordinate(price);
      const crosshairX = chart.timeScale().timeToCoordinate(param.time);
      
      if (crosshairX !== null && crosshairY !== null) {
        drawMarkerAt(crosshairX, crosshairY);
      }
    }
  }
});
```

**Плюсы:**
- Получает точную цену данных на кроссхэре
- Работает корректно с режимом `magnet`
- Официальный API

**Минусы:**
- Требует дополнительного преобразования priceToCoordinate
- Сложнее для OHLC данных (нужно выбрать open/high/low/close)

---

### Решение 2: Комбинация time + priceToCoordinate
**Оценка: 7/10**

Использование `param.time` для X-координаты и `priceToCoordinate` для Y.

```javascript
chart.subscribeCrosshairMove((param) => {
  if (!param.time) return;
  
  // X-координата - из временной метки кроссхэра
  const crosshairX = chart.timeScale().timeToCoordinate(param.time);
  if (crosshairX === null) return;
  
  // Y-координата - из данных серии
  const seriesData = param.seriesPrices.get(series);
  if (!seriesData) return;
  
  let price: number;
  if (typeof seriesData === 'object' && 'close' in seriesData) {
    price = seriesData.close;  // Используем close для свечей
  } else if (typeof seriesData === 'number') {
    price = seriesData;
  } else {
    return;
  }
  
  const crosshairY = series.priceToCoordinate(price);
  if (crosshairY === null) return;
  
  // Теперь crosshairX и crosshairY - это координаты КРОССХЭРА
  positionTooltip(crosshairX, crosshairY);
});
```

**Плюсы:**
- Точные координаты кроссхэра
- Универсальное решение

**Минусы:**
- Больше кода
- Необходимость обрабатывать разные типы данных

---

### Решение 3: Отключение режима Magnet
**Оценка: 5/10**

Если точные координаты мыши — это то, что вам нужно.

```javascript
const chart = createChart(container, {
  crosshair: {
    mode: CrosshairMode.Normal  // Кроссхэр следует за мышью
  }
});

chart.subscribeCrosshairMove((param) => {
  if (!param.point) return;
  
  // ✅ В режиме Normal param.point = позиции кроссхэра = позиции мыши
  const x = param.point.x;
  const y = param.point.y;
  
  // Можно также получить цену на этой координате
  const price = series.coordinateToPrice(y);
  const time = chart.timeScale().coordinateToTime(x);
  
  console.log(`Курсор: price=${price}, time=${time}`);
});
```

**Плюсы:**
- Простое решение
- Нет несоответствия координат

**Минусы:**
- Теряется функционал "притягивания" к свечам
- Может не подходить для большинства use-cases

---

### Решение 4: Создание вспомогательной функции для получения координат кроссхэра
**Оценка: 9/10** ⭐ ЛУЧШЕЕ РЕШЕНИЕ

Универсальная функция для получения координат кроссхэра.

```typescript
import {
  IChartApi,
  ISeriesApi,
  MouseEventParams,
  Time,
  CandlestickData,
  BarData,
  LineData,
  AreaData
} from 'lightweight-charts';

interface CrosshairCoordinates {
  x: number;
  y: number;
  price: number;
  time: Time;
}

/**
 * Получает координаты кроссхэра (а не мыши) из события crosshairMove
 */
function getCrosshairCoordinates<T extends Time>(
  chart: IChartApi,
  series: ISeriesApi<'Candlestick' | 'Bar' | 'Line' | 'Area'>,
  param: MouseEventParams<T>,
  priceField: 'open' | 'high' | 'low' | 'close' = 'close'
): CrosshairCoordinates | null {
  
  // Проверяем наличие времени (кроссхэр на данных)
  if (!param.time) {
    return null;
  }
  
  // Получаем данные серии в точке кроссхэра
  const seriesData = param.seriesPrices.get(series);
  if (seriesData === undefined) {
    return null;
  }
  
  // Определяем цену в зависимости от типа данных
  let price: number;
  
  if (typeof seriesData === 'number') {
    // Line или Area series
    price = seriesData;
  } else if (typeof seriesData === 'object') {
    // Candlestick или Bar series
    const ohlcData = seriesData as CandlestickData | BarData;
    price = ohlcData[priceField];
  } else {
    return null;
  }
  
  // Конвертируем time и price в экранные координаты
  const x = chart.timeScale().timeToCoordinate(param.time);
  const y = series.priceToCoordinate(price);
  
  if (x === null || y === null) {
    return null;
  }
  
  return {
    x,
    y,
    price,
    time: param.time
  };
}

// === Использование ===

const chart = createChart(container, {
  crosshair: { mode: CrosshairMode.Magnet }
});

const series = chart.addSeries(CandlestickSeries);
series.setData(candleData);

chart.subscribeCrosshairMove((param) => {
  // ✅ Получаем координаты КРОССХЭРА
  const coords = getCrosshairCoordinates(chart, series, param, 'close');
  
  if (coords) {
    console.log(`Кроссхэр: x=${coords.x}, y=${coords.y}`);
    console.log(`Данные: time=${coords.time}, price=${coords.price}`);
    
    // Позиционируем tooltip на координатах КРОССХЭРА
    tooltip.style.left = `${coords.x + 10}px`;
    tooltip.style.top = `${coords.y}px`;
    tooltip.textContent = `Price: ${coords.price.toFixed(2)}`;
  }
});
```

**Плюсы:**
- Универсальное решение
- Работает со всеми типами серий
- Возвращает и координаты, и данные
- Легко переиспользовать

**Минусы:**
- Дополнительный код
- Требуется передача параметров

---

### Решение 5: Использование coordinateToPrice для мыши + seriesPrices для данных
**Оценка: 6/10**

Получение обеих позиций: мыши и данных.

```javascript
chart.subscribeCrosshairMove((param) => {
  if (!param.point) return;
  
  // Координаты МЫШИ
  const mouseX = param.point.x;
  const mouseY = param.point.y;
  const mousePrice = series.coordinateToPrice(mouseY);
  const mouseTime = chart.timeScale().coordinateToTime(mouseX);
  
  console.log('Мышь:', { x: mouseX, y: mouseY, price: mousePrice, time: mouseTime });
  
  // Координаты КРОССХЭРА (данные на свече)
  if (param.time) {
    const crosshairX = chart.timeScale().timeToCoordinate(param.time);
    const seriesData = param.seriesPrices.get(series);
    
    if (seriesData && crosshairX !== null) {
      const crosshairPrice = typeof seriesData === 'number' 
        ? seriesData 
        : (seriesData as CandlestickData).close;
      const crosshairY = series.priceToCoordinate(crosshairPrice);
      
      console.log('Кроссхэр:', { 
        x: crosshairX, 
        y: crosshairY, 
        price: crosshairPrice, 
        time: param.time 
      });
    }
  }
});
```

**Плюсы:**
- Полная информация о обеих позициях
- Гибкость использования

**Минусы:**
- Много кода
- Избыточность для большинства случаев

---

## ✅ Рекомендуемое решение

### Решение 4: Вспомогательная функция для координат кроссхэра

Создайте переиспользуемый модуль для работы с координатами:

```typescript
// crosshair-utils.ts
import {
  IChartApi,
  ISeriesApi,
  MouseEventParams,
  Time,
  CandlestickData,
  BarData,
  SeriesType
} from 'lightweight-charts';

export interface CrosshairPosition {
  // Экранные координаты кроссхэра
  screenX: number;
  screenY: number;
  // Данные в точке кроссхэра
  price: number;
  time: Time;
  // OHLC для свечных серий
  ohlc?: {
    open: number;
    high: number;
    low: number;
    close: number;
  };
}

export interface MousePosition {
  screenX: number;
  screenY: number;
  price: number | null;
  time: Time | null;
}

export interface CrosshairEventData {
  crosshair: CrosshairPosition | null;
  mouse: MousePosition | null;
}

/**
 * Извлекает полные данные о позиции кроссхэра и мыши
 */
export function getCrosshairEventData<T extends Time>(
  chart: IChartApi,
  series: ISeriesApi<SeriesType>,
  param: MouseEventParams<T>
): CrosshairEventData {
  const result: CrosshairEventData = {
    crosshair: null,
    mouse: null
  };

  // Данные о мыши
  if (param.point) {
    result.mouse = {
      screenX: param.point.x,
      screenY: param.point.y,
      price: series.coordinateToPrice(param.point.y),
      time: chart.timeScale().coordinateToTime(param.point.x)
    };
  }

  // Данные о кроссхэре (на свече)
  if (param.time) {
    const seriesData = param.seriesPrices.get(series);
    if (seriesData !== undefined) {
      const screenX = chart.timeScale().timeToCoordinate(param.time);
      
      if (screenX !== null) {
        let price: number;
        let ohlc: CrosshairPosition['ohlc'];

        if (typeof seriesData === 'number') {
          price = seriesData;
        } else {
          const data = seriesData as CandlestickData | BarData;
          price = data.close;
          ohlc = {
            open: data.open,
            high: data.high,
            low: data.low,
            close: data.close
          };
        }

        const screenY = series.priceToCoordinate(price);
        
        if (screenY !== null) {
          result.crosshair = {
            screenX,
            screenY,
            price,
            time: param.time,
            ohlc
          };
        }
      }
    }
  }

  return result;
}

// === Практический пример: Tooltip ===

export class CrosshairTooltip {
  private _element: HTMLDivElement;
  private _chart: IChartApi;
  private _series: ISeriesApi<SeriesType>;
  private _unsubscribe: (() => void) | null = null;

  constructor(
    container: HTMLElement,
    chart: IChartApi,
    series: ISeriesApi<SeriesType>
  ) {
    this._chart = chart;
    this._series = series;

    // Создаём tooltip element
    this._element = document.createElement('div');
    this._element.className = 'crosshair-tooltip';
    this._element.style.cssText = `
      position: absolute;
      display: none;
      background: rgba(0, 0, 0, 0.85);
      color: #fff;
      padding: 8px 12px;
      border-radius: 4px;
      font-family: -apple-system, BlinkMacSystemFont, sans-serif;
      font-size: 12px;
      pointer-events: none;
      z-index: 1000;
      white-space: nowrap;
      box-shadow: 0 2px 8px rgba(0,0,0,0.3);
    `;
    container.appendChild(this._element);

    // Подписываемся на события
    this._subscribe();
  }

  private _subscribe() {
    const handler = (param: MouseEventParams<Time>) => {
      const data = getCrosshairEventData(this._chart, this._series, param);

      if (data.crosshair) {
        this._element.style.display = 'block';
        // Позиционируем на координатах КРОССХЭРА (не мыши!)
        this._element.style.left = `${data.crosshair.screenX + 15}px`;
        this._element.style.top = `${data.crosshair.screenY - 30}px`;

        // Формируем контент
        if (data.crosshair.ohlc) {
          this._element.innerHTML = `
            <div><strong>Time:</strong> ${data.crosshair.time}</div>
            <div><strong>O:</strong> ${data.crosshair.ohlc.open.toFixed(2)}</div>
            <div><strong>H:</strong> ${data.crosshair.ohlc.high.toFixed(2)}</div>
            <div><strong>L:</strong> ${data.crosshair.ohlc.low.toFixed(2)}</div>
            <div><strong>C:</strong> ${data.crosshair.ohlc.close.toFixed(2)}</div>
          `;
        } else {
          this._element.innerHTML = `
            <div><strong>Time:</strong> ${data.crosshair.time}</div>
            <div><strong>Price:</strong> ${data.crosshair.price.toFixed(2)}</div>
          `;
        }
      } else {
        this._element.style.display = 'none';
      }
    };

    this._chart.subscribeCrosshairMove(handler);
    this._unsubscribe = () => this._chart.unsubscribeCrosshairMove(handler);
  }

  destroy() {
    this._unsubscribe?.();
    this._element.remove();
  }
}

// === Использование ===

const chart = createChart(document.getElementById('chart')!, {
  width: 800,
  height: 600,
  crosshair: {
    mode: CrosshairMode.Magnet  // Magnet режим
  }
});

const series = chart.addSeries(CandlestickSeries);
series.setData([
  { time: '2025-01-01', open: 100, high: 105, low: 98, close: 103 },
  { time: '2025-01-02', open: 103, high: 110, low: 102, close: 108 },
  { time: '2025-01-03', open: 108, high: 112, low: 105, close: 107 }
]);

// ✅ Создаём tooltip с правильным позиционированием
const tooltip = new CrosshairTooltip(
  chart.chartElement().parentElement!,
  chart,
  series
);

// При необходимости можно уничтожить
// tooltip.destroy();
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Точность | Универсальность |
|---------|--------|-----------|----------|-----------------|
| 1. seriesPrices + priceToCoordinate | 8/10 | Средняя | ✅ Высокая | ✅ Да |
| 2. time + priceToCoordinate | 7/10 | Средняя | ✅ Высокая | ✅ Да |
| 3. CrosshairMode.Normal | 5/10 | Низкая | ⚠️ Разная | ❌ Нет magnet |
| 4. **Вспомогательная функция** | **9/10** | Высокая | ✅ **Высокая** | ✅ **Да** |
| 5. Обе позиции (мышь + кроссхэр) | 6/10 | Высокая | ✅ Высокая | ✅ Да |

### Рекомендации:

- **Для tooltips** → Решение 4 (вспомогательная функция)
- **Для простых случаев** → Решение 1 (seriesPrices)
- **Если magnet не нужен** → Решение 3 (Normal mode)

---

## 🔗 Источники

1. [GitHub Issue #1697](https://github.com/tradingview/lightweight-charts/issues/1697) - Оригинальный баг-репорт
2. [Tooltips Tutorial](https://tradingview.github.io/lightweight-charts/tutorials/how_to/tooltips) - Официальная документация
3. [CrosshairMode Documentation](https://tradingview.github.io/lightweight-charts/docs/api/enums/CrosshairMode) - Режимы кроссхэра
4. [ISeriesApi.coordinateToPrice](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#coordinatetoprice) - API Reference
5. [ISeriesApi.priceToCoordinate](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/ISeriesApi#pricetocoordinate) - API Reference
6. [subscribeCrosshairMove](https://tradingview.github.io/lightweight-charts/docs/api/interfaces/IChartApi#subscribecrosshairmove) - Подписка на события

---

## 📝 Примечания

### Ключевые понятия

| Свойство | Описание |
|----------|----------|
| `param.point` | Координаты МЫШИ (всегда позиция курсора) |
| `param.time` | Время на КРОССХЭРЕ (на ближайшей свече в режиме magnet) |
| `param.seriesPrices` | Данные серий на позиции КРОССХЭРА |

### Когда что использовать

| Задача | Используйте |
|--------|-------------|
| Tooltip на свече | `param.time` + `param.seriesPrices` + `priceToCoordinate` |
| Рисование по позиции мыши | `param.point` |
| Рисование по позиции кроссхэра | `timeToCoordinate(param.time)` + `priceToCoordinate(price)` |

### Статус
Issue #1697 помечена как "needs investigation". Официального фикса нет, используйте workarounds из этого документа.
