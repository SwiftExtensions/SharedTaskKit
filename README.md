# SharedTaskKit

Небольшая библиотека для Swift, которая решает две частые задачи:

1. **Не запускать одну и ту же работу дважды.** Если десять мест в коде
   просят одну и ту же картинку — загрузка выполнится один раз,
   а результат получат все десять.
2. **Не запускать слишком много работ сразу.** Можно ограничить число
   одновременных операций, например «скачивать не больше 4 файлов
   одновременно», остальные подождут в очереди.

## Когда это нужно

Пример: на экране список из ста карточек, у многих одинаковые картинки.
Без библиотеки вы скачаете одну картинку много раз и можете устроить
сто параллельных загрузок. С библиотекой каждая уникальная картинка
скачивается один раз, а загрузки идут не больше заданного числа сразу.

## Требования

- iOS 15+ или macOS 12+
- Swift 5.6+

## Установка

Библиотека — это Swift Package. В Xcode:

1. Откройте свой проект.
2. `File` → `Add Package Dependencies…`
3. Вставьте адрес репозитория (или выберите локальную папку с пакетом
   через `Add Local…`).
4. Нажмите `Add Package`.

В коде подключите модуль:

```swift
import SharedTaskKit
```

## Использование

В библиотеке два типа: `TaskQueue` и `SharedTasks`.

### `TaskQueue` — очередь с ограничением

Запускает асинхронные операции так, чтобы одновременно выполнялось
не больше заданного числа. Кто не поместился — ждёт в очереди
в порядке поступления.

```swift
// одновременно не больше 4 операций
let queue = TaskQueue(limit: 4)

let image = try await queue.run {
    try await loadImage(from: url)
}
```

Без ограничений (всё запускается сразу):

```swift
let queue = TaskQueue(limit: .unlimited)
```

### `SharedTasks` — общие задачи по ключам

Делает так, что по одному ключу работа выполняется только один раз.
Все, кто запросил тот же ключ, получают результат одной и той же задачи.

```swift
let images = SharedTasks<URL, UIImage>(
    taskQueue: TaskQueue(limit: 10),
    operation: { url in try await loadImage(from: url) }
)

// запускаем два запроса одной и той же картинки
async let first = images.value(for: url)
async let second = images.value(for: url)

// картинка скачается ОДИН раз, оба получат её
let (image1, image2) = try await (first, second)
```

Ключ — любой тип, соответствующий `Hashable & Sendable`
(`URL`, `String`, `Int` и т.п.).

### Как работает отмена

- Отменили одну задачу, ожидающую `value(for:)` — отписался только
  этот запрос, остальные ждут дальше.
- Отменили последнего ждущего по ключу — общая задача отменяется:
  доделывать никому не нужную работу незачем.
- Отменили задачу, стоящую в очереди `TaskQueue`, — она снимается
  с ожидания и её операция вообще не запускается.

## Пример: загрузчик картинок

Полный пример по шагам: загрузчик, запуск загрузки и её отмена.

### Шаг 1. Загрузчик

```swift
import Foundation
import SharedTaskKit
import UIKit

/**
 Загрузчик картинок.

 Одинаковые адреса скачиваются один раз: запросы одного адреса
 делят одну загрузку. Одновременно идёт не больше четырёх загрузок,
 остальные ждут в очереди.
 */
final class ImageLoader {
    /**
     Пул общих задач: ключ — адрес картинки,
     результат — загруженное изображение.
     */
    private let tasks = SharedTasks<URL, UIImage>(
        taskQueue: TaskQueue(limit: 4)
    ) { url in
        let (data, _) = try await URLSession.shared.data(from: url)
        guard let image = UIImage(data: data) else {
            throw URLError(.cannotDecodeContentData)
        }
        return image
    }

    /**
     Возвращает картинку по адресу.

     - Parameter url: Адрес картинки.
     - Returns: Загруженное изображение.
     - Throws: `CancellationError`, если запрос отменён;
       ошибка загрузки иначе.
     */
    func image(for url: URL) async throws -> UIImage {
        try await self.tasks.value(for: url)
    }


}
```

### Шаг 2. Запуск загрузки

Вызов `image(for:)` — асинхронный, его запускают внутри `Task`.
Сам `Task` — это «ручка» запроса: запомните его, чтобы потом
было что отменять.

```swift
final class AvatarController {
    private let loader = ImageLoader()
    private let imageView: UIImageView

    /**
     Запомненная задача загрузки: через неё запрос отменяется.
     */
    private var loadTask: Task<Void, Never>?

    /**
     Показывает картинку по адресу.
     */
    func showAvatar(from url: URL) {
        self.loadTask = Task {
            do {
                let avatar = try await self.loader.image(for: url)
                self.imageView.image = avatar
            } catch is CancellationError {
                // запрос отменили: картинка больше не нужна
            }
        }
    }
    /**
     Отменяет текущую загрузку, если она ещё идёт.
     */
    func stopLoading() {
        self.loadTask?.cancel()
    }


}
```

### Шаг 3. Отмена

Чтобы отменить запрос, вызовите `cancel()` у запомненного `Task`:

```swift
controller.stopLoading()   // внутри — loadTask.cancel()
```

Что при этом происходит:

- этот запрос получает `CancellationError` и завершается;
- если тот же адрес ждут другие запросы — загрузка продолжится
  для них: отменились только вы, а не общая задача;
- если ваш запрос был последним по этому адресу — общая загрузка
  отменится целиком: доделывать никому не нужную работу незачем;
- если запрос ещё ждал место в очереди (четыре загрузки уже идут) —
  он выйдет из очереди, его загрузка даже не начнётся.

### Пример в SwiftUI

В SwiftUI запоминать `Task` вручную не нужно — модификатор `.task`
сам отменяет задачу, когда экран исчезает:

```swift
struct AvatarView: View {
    let url: URL
    let loader: ImageLoader

    @State private var image: UIImage?

    var body: some View {
        Group {
            if let image {
                Image(uiImage: image).resizable().scaledToFit()
            } else {
                ProgressView()
            }
        }
        .task {
            // пока вью на экране — грузим;
            // вью исчезла — .task сам вызовет cancel() у запроса
            self.image = try? await self.loader.image(for: url)
        }
    }
}
```

Если нужна ручная отмена — например, по кнопке «Стоп» — запустите
загрузку в собственном `Task` и храните его в `@State`, как в шагах
2–3 выше.

> Если нужно кэшировать уже скачанные картинки, добавьте `NSCache`
> внутрь `image(for:)`: проверили кэш, и только при промахе пошли
> в `tasks.value(for:)`.

## Запуск тестов

Из корня пакета:

```sh
swift test
```

## Состав

| Файл | Что внутри |
| ---- | ---------- |
| `SharedTasks.swift` | Актор `SharedTasks` — дедупликация задач по ключу |
| `TaskQueue.swift` | Актор `TaskQueue` — лимит одновременных операций |
| `Subscribers.swift` | Внутренний счётчик подписок на ключи |

История изменений — в [CHANGELOG.md](CHANGELOG.md).
