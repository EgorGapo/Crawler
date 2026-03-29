---
# Многопоточный Map-Reduce Crawler файлов (Go)

Проект реализует **многопоточный Map-Reduce краулер** для обработки файлов.
Краулер ищет файлы, обрабатывает их содержимое и агрегирует результаты с помощью пула воркеров.

Примеры использования можно посмотреть в:

* `cmd/app/app.go`
* `tests/`
* `internal/workerpool/pool_test.go`
* `internal/filecrawler/crawler_test.go`

---

## Структура проекта

Система состоит из двух основных компонентов:

### 1. Пул воркеров

Пул воркеров предоставляет три базовые операции:

* **Transform** – преобразование элементов
* **Accumulate** – агрегирование значений
* **List** – рекурсивный обход иерархий

### 2. File Crawler

Краулер координирует:

1. Поиск файлов
2. Чтение и обработку файлов
3. Агрегацию результатов

Используется **Map-Reduce модель** для эффективной обработки большого числа файлов.

---

## Интерфейсы
### Accumulator

```go
type Accumulator[T, R any] func(current T, accum R) R
```

* Агрегирует значения типа `T` в результат типа `R`.
* Должен быть **потокобезопасным**.
* Используется несколькими воркерами одновременно.

---

### Transformer

```go
type Transformer[T, R any] func(current T) R
```

* Преобразует элемент типа `T` в элемент типа `R`.
* Должен быть **потокобезопасным**.
* Вызывается несколькими воркерами параллельно.

---

### Searcher

```go
type Searcher[T any] func(parent T) []T
```

* Используется для обхода иерархий.
* Возвращает дочерние элементы для заданного родителя.
* Должен быть **потокобезопасным**.

---

## Интерфейс Worker Pool

```go
type Pool[T, R any] interface {
    Transform(ctx context.Context, workers int, input <-chan T, transformer Transformer[T, R]) <-chan R
    Accumulate(ctx context.Context, workers int, input <-chan T, accumulator Accumulator[T, R]) <-chan R
    List(ctx context.Context, workers int, start T, searcher Searcher[T])
}
```

* **Transform** – применяет функцию к каждому элементу и возвращает результат.
* **Accumulate** – агрегирует элементы и возвращает промежуточные результаты.
* **List** – выполняет иерархический обход, начиная с корневого элемента.

---

## File Crawler

```go
type Crawler[T, R any] interface {
    Collect(
        ctx context.Context,
        fileSystem fs.FileSystem,
        root string,
        conf Configuration,
        accumulator workerpool.Accumulator[T, R],
        combiner Combiner[R],
    ) (R, error)
}
```

* Координирует поиск и обработку файлов.
* Результат типа `R` формируется через **аккумулятор и комбайнер**.
* Должен корректно обрабатывать отмену контекста.

---

## Конфигурация

```go
type Configuration struct {
    SearchWorkers      int
    FileWorkers        int
    AccumulatorWorkers int
}
```

* `SearchWorkers` – количество воркеров для поиска файлов
* `FileWorkers` – количество воркеров для обработки файлов
* `AccumulatorWorkers` – количество воркеров для агрегации результатов

Все значения обязательны для корректной работы.

---

## Combiner

```go
type Combiner[R any] func(current R, accum R) R
```

* Комбинирует два значения в одно.
* Не требуется быть потокобезопасным.
* Тип `R` должен обладать нейтральным элементом (свойства моноида).

Комбайнер может:

* Модифицировать один из аргументов и вернуть его
* Или создать новый результат на основе обоих аргументов

---

## Map-Reduce Pipeline

Краулер выполняет следующие шаги:

1. **Search phase** – обход директорий и поиск файлов
2. **Map phase** – чтение файлов и десериализация в тип `T`
3. **Reduce phase** – агрегация значений с помощью `Accumulator`
4. **Combine phase** – объединение промежуточных результатов через `Combiner`

---

## Требования к реализации

### Worker Pool

* Ошибки можно не обрабатывать
* Должна поддерживаться отмена контекста
* Использовать только **небуферизованные каналы**

### Crawler

* `FileSystem` и `Accumulator` должны быть **потокобезопасными**
* `Combiner` потокобезопасность не требуется
* Ошибки при десериализации JSON обрабатываются внутри воркеров

---

## Файловая система

Абстракция файловой системы предоставлена для тестирования:

* `internal/fs/filesystem.go`
* Существуют готовые реализации

---

## Тесты

Тесты находятся в:

* `internal/workerpool/pool_test.go`
* `internal/filecrawler/crawler_test.go`
* `tests/`

Они демонстрируют ожидаемое поведение и проверяют граничные случаи.

---

## Ограничения

* Нельзя использовать буферизованные каналы
* Запрещено менять ветку `main` без Pull Request
* Реализация должна находиться только в требуемых файлах

---

## Файлы для реализации

* `internal/workerpool/pool.go`
* `internal/filecrawler/crawler.go`

---

## Makefile

Запуск полного цикла (lint + тесты):

```bash
make all
```

Только тесты:

```bash
make test
```

Только линтер:

```bash
make lint
```

Обновление тестов:

```bash
make update
```
