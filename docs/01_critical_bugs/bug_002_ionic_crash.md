# БАГ #2: Полный crash в Ionic + Chrome Mobile Simulator

> **Критичность:** 🔴 КРИТИЧЕСКАЯ (для Ionic developers)  
> **GitHub Issue:** [#2046](https://github.com/tradingview/lightweight-charts/issues/2046)  
> **Версии:** v5.1.0  
> **Статус:** 🔴 Open (15 января 2026)

---

## 📋 Описание проблемы

### Симптомы
- Полное завершение приложения при попытке отобразить график
- Происходит при инициализации графика
- Приложение крашится без понятных error messages

### Причина
Библиотека обращается к `navigator.userAgentData.brands`, который в Chrome Mobile Simulator **не содержит ожидаемый массив `brands`**. Это приводит к crash при попытке итерации по undefined.

### Сценарии воспроизведения
1. Ionic framework + Chrome Mobile Simulator extension
2. Тестирование мобильной версии через DevTools

### Частота и платформы
- **Частота:** 100% в данной комбинации
- **Платформы:** Chrome Mobile Simulator + Ionic
- **На реальных устройствах:** Работает нормально

---

## 🔍 Найденные решения

### Решение 1: Полифилл navigator.userAgentData.brands
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Добавить пустой массив `brands` перед инициализацией графика.

**Плюсы:**
- Простое однострочное решение
- Рекомендовано разработчиком библиотеки (SlicedSilver)
- Не влияет на production

**Минусы:**
- Workaround, а не исправление в библиотеке
- Нужно добавлять в каждый проект

```javascript
// Добавить в начало приложения (перед импортом lightweight-charts)
if (navigator.userAgentData && !navigator.userAgentData.brands) {
  navigator.userAgentData.brands = [];
}
```

**Для Ionic/Angular:**

```typescript
// src/main.ts (перед bootstrapApplication)
if (typeof navigator !== 'undefined' && 
    navigator.userAgentData && 
    !navigator.userAgentData.brands) {
  (navigator.userAgentData as any).brands = [];
}

import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
// ...
```

**Для Ionic/React:**

```typescript
// src/index.tsx (в самом начале)
if (navigator.userAgentData && !navigator.userAgentData.brands) {
  (navigator.userAgentData as any).brands = [];
}

import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
// ...
```

---

### Решение 2: Environment-specific polyfill
**Оценка: 8/10**

**Суть:** Создать отдельный polyfill файл для development.

**Плюсы:**
- Чистое разделение dev/prod кода
- Легко удалить после исправления в библиотеке

**Минусы:**
- Дополнительный файл
- Нужна настройка build system

```typescript
// src/polyfills/chrome-simulator-fix.ts
export function applyChromeMobileSimulatorFix(): void {
  if (typeof navigator === 'undefined') return;
  
  // Проверяем, что мы в Chrome Mobile Simulator
  const isSimulator = 
    navigator.userAgent.includes('Chrome') &&
    navigator.userAgentData && 
    !navigator.userAgentData.brands;
  
  if (isSimulator) {
    console.warn('[DevFix] Applied Chrome Mobile Simulator workaround');
    (navigator.userAgentData as any).brands = [];
  }
}
```

```typescript
// src/main.ts
import { applyChromeMobileSimulatorFix } from './polyfills/chrome-simulator-fix';

// Применяем только в development
if (process.env.NODE_ENV === 'development') {
  applyChromeMobileSimulatorFix();
}

// Остальной код инициализации...
```

---

### Решение 3: Тестирование на реальном устройстве
**Оценка: 7/10**

**Суть:** Избегать Chrome Mobile Simulator для тестирования графиков.

**Плюсы:**
- Не требует изменения кода
- Более точное тестирование на реальных устройствах

**Минусы:**
- Неудобно для быстрого development
- Требует физические устройства или эмуляторы

```bash
# Android
npx cap run android

# iOS  
npx cap run ios

# Или подключить реальное устройство через USB debugging
```

---

### Решение 4: Обновление WebView версии
**Оценка: 6/10**

**Суть:** Убедиться, что используется актуальная версия Chrome WebView.

**Плюсы:**
- Может решить множество других проблем

**Минусы:**
- Не всегда возможно контролировать версию WebView
- Может не помочь с данной конкретной проблемой

```javascript
// Проверка версии WebView в консоли
console.log('WebView version:', window.navigator.userAgent);
```

---

## ✅ Рекомендуемое решение

Используйте **Решение #1** (полифилл) как наиболее простое и эффективное:

```typescript
// === ПОЛНЫЙ ПРИМЕР ДЛЯ IONIC/ANGULAR ===

// src/main.ts
// Chrome Mobile Simulator fix (GitHub issue #2046)
if (typeof navigator !== 'undefined' && 
    navigator.userAgentData && 
    !navigator.userAgentData.brands) {
  (navigator.userAgentData as any).brands = [];
}

import { enableProdMode } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { environment } from './environments/environment';

if (environment.production) {
  enableProdMode();
}

bootstrapApplication(AppComponent);
```

```typescript
// === ПОЛНЫЙ ПРИМЕР ДЛЯ IONIC/REACT ===

// src/index.tsx
// Chrome Mobile Simulator fix (GitHub issue #2046)
if (navigator.userAgentData && !navigator.userAgentData.brands) {
  (navigator.userAgentData as any).brands = [];
}

import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
const root = createRoot(container!);
root.render(<App />);
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Надёжность | Рекомендация |
|---------|--------|-----------|------------|--------------|
| #1 Полифилл brands | 9/10 | Очень низкая | Высокая | ⭐ Рекомендуется |
| #2 Environment-specific | 8/10 | Низкая | Высокая | Альтернатива |
| #3 Реальные устройства | 7/10 | Нет кода | - | Для финального QA |
| #4 Обновление WebView | 6/10 | Низкая | Неизвестно | Как дополнение |

---

## 🔗 Источники

1. [GitHub Issue #2046 - Ionic project Chrome Simulator crashes](https://github.com/tradingview/lightweight-charts/issues/2046)
2. [SlicedSilver's workaround suggestion](https://github.com/tradingview/lightweight-charts/issues/2046#issuecomment-xxx)
3. [Ionic Capacitor WebView debugging](https://ionicframework.com/docs/troubleshooting/debugging)

---

**Документ создан:** 18 января 2026  
**Последнее обновление:** 18 января 2026
