# БАГ #5: График blank в iOS wrapper (lightweight-charts-ios)

> **Критичность:** 🔴 КРИТИЧЕСКАЯ  
> **GitHub Issue:** lightweight-charts-ios repo  
> **Версии:** iOS wrapper  
> **Статус:** 🔴 Open

---

## 📋 Описание проблемы

### Симптомы
- View становится полностью чёрным/пустым
- Невозможность восстановить отображение без перезапуска
- Происходит случайно

### Причина
Проблема связана с неправильной установкой frame/constraints для WKWebView, который используется внутри iOS wrapper, или с race conditions при загрузке JavaScript.

### Сценарии воспроизведения
1. Использование в нативных iOS приложениях
2. Переходы между ViewController'ами
3. Rotation устройства

### Частота и платформы
- **Частота:** Редкая, но критическая при проявлении
- **Платформы:** iOS native apps

---

## 🔍 Найденные решения

### Решение 1: Правильная установка frame и constraints
**Оценка: 9/10** ⭐ РЕКОМЕНДУЕТСЯ

**Суть:** Убедиться, что LightweightCharts view имеет правильные размеры.

```swift
import LightweightChartsIOS

class ChartViewController: UIViewController {
    private var chartView: LightweightCharts!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Создаём chart view
        chartView = LightweightCharts()
        view.addSubview(chartView)
        
        // КРИТИЧЕСКИ ВАЖНО: установка constraints
        chartView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            chartView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            chartView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            chartView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            chartView.bottomAnchor.constraint(equalTo: view.safeAreaLayoutGuide.bottomAnchor)
        ])
    }
    
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        
        // Дополнительная проверка размеров
        if chartView.bounds.size.width > 0 && chartView.bounds.size.height > 0 {
            // Размеры корректны
            print("Chart size: \(chartView.bounds.size)")
        } else {
            print("Warning: Chart has zero size!")
        }
    }
}
```

---

### Решение 2: Отложенная инициализация
**Оценка: 8/10**

**Суть:** Инициализировать график после полной загрузки view.

```swift
class ChartViewController: UIViewController {
    private var chartView: LightweightCharts?
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        
        // Инициализация после появления view
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) { [weak self] in
            self?.setupChart()
        }
    }
    
    private func setupChart() {
        guard chartView == nil else { return }
        
        chartView = LightweightCharts()
        chartView!.frame = view.bounds
        chartView!.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        view.addSubview(chartView!)
        
        // Установка данных
        loadChartData()
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        
        // Очистка при уходе с экрана
        chartView?.removeFromSuperview()
        chartView = nil
    }
}
```

---

### Решение 3: WKWebView configuration
**Оценка: 7/10**

**Суть:** Настройка WKWebView для предотвращения blank screen.

```swift
// Если используете custom WKWebView для графика
import WebKit

extension WKWebViewConfiguration {
    static func chartConfiguration() -> WKWebViewConfiguration {
        let config = WKWebViewConfiguration()
        config.allowsInlineMediaPlayback = true
        config.mediaTypesRequiringUserActionForPlayback = []
        
        // Предотвращение белого экрана
        let prefs = WKPreferences()
        prefs.javaScriptEnabled = true
        config.preferences = prefs
        
        return config
    }
}
```

---

## 📊 Сравнительная таблица решений

| Решение | Оценка | Сложность | Надёжность |
|---------|--------|-----------|------------|
| #1 Proper constraints | 9/10 | Низкая | Высокая |
| #2 Delayed init | 8/10 | Низкая | Высокая |
| #3 WKWebView config | 7/10 | Средняя | Средняя |

---

## 🔗 Источники

1. [lightweight-charts-ios GitHub](https://github.com/nicemarcela/lightweight-charts-ios)
2. [WKWebView blank screen issues](https://stackoverflow.com/questions/tagged/wkwebview+blank)

---

**Документ создан:** 18 января 2026
