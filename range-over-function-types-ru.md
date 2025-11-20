# Range over function types — перевод статьи

Range over function types — новое языковое улучшение в Go 1.23. В заметке разбираем, почему появилась эта возможность, что именно добавили и как ей пользоваться.

## Зачем вообще что-то менять

С Go 1.18 мы умеем писать обобщённые контейнеры. В качестве примера возьмём упрощённый `Set`, построенный поверх `map`.

```go
// Set хранит набор элементов.
type Set[E comparable] struct {
	m map[E]struct{}
}

// New создаёт новый Set.
func New[E comparable]() *Set[E] {
	return &Set[E]{m: make(map[E]struct{})}
}
```

Тип набора очевидно требует метод добавления и проверку наличия элемента.

```go
// Add добавляет элемент.
func (s *Set[E]) Add(v E) {
	s.m[v] = struct{}{}
}

// Contains проверяет наличие элемента.
func (s *Set[E]) Contains(v E) bool {
	_, ok := s.m[v]
	return ok
}
```

Хочется уметь строить объединение двух множеств:

```go
// Union возвращает объединение двух Set.
func Union[E comparable](s1, s2 *Set[E]) *Set[E] {
	r := New[E]()
	for v := range s1.m {
		r.Add(v)
	}
	for v := range s2.m {
		r.Add(v)
	}
	return r
}
```

Чтобы реализовать `Union`, приходится итерироваться по неэкспортируемому полю `m`. Это возможно только внутри пакета `set`. Снаружи же единственный способ получить элементы — через предоставленное API.

## Первый подход: push

Можно добавить метод `Push`, который принимает функцию и вызывает её для каждого значения. Если функция вернёт `false`, перебор прекращаем.

```go
func (s *Set[E]) Push(f func(E) bool) {
	for v := range s.m {
		if !f(v) {
			return
		}
	}
}
```

Такой паттерн уже встречается в стандартной библиотеке: `sync.Map.Range`, `flag.Visit`, `filepath.Walk`. Чтобы вывести все элементы, передаём функцию, которая печатает значение и продолжает обход:

```go
func PrintAllElementsPush[E comparable](s *Set[E]) {
	s.Push(func(v E) bool {
		fmt.Println(v)
		return true
	})
}
```

## Второй подход: pull

Альтернатива — вернуть функцию, которая при каждом вызове отдаёт следующий элемент плюс флаг валидности. Здесь также нужен `stop`, чтобы корректно завершить горутину, которая поставляет значения.

```go
func (s *Set[E]) Pull() (func() (E, bool), func()) {
	ch := make(chan E)
	stopCh := make(chan bool)

	go func() {
		defer close(ch)
		for v := range s.m {
			select {
			case ch <- v:
			case <-stopCh:
				return
			}
		}
	}()

	next := func() (E, bool) {
		v, ok := <-ch
		return v, ok
	}

	stop := func() {
		close(stopCh)
	}

	return next, stop
}
```

Использование:

```go
func PrintAllElementsPull[E comparable](s *Set[E]) {
	next, stop := s.Pull()
	defer stop()
	for v, ok := next(); ok; v, ok = next() {
		fmt.Println(v)
	}
}
```

## Нужна стандартизация

Отсутствие единообразия усложняет использование контейнеров и мешает писать функции, работающие с разными структурами данных. Цель Go 1.23 — дать стандартную форму обхода.

## Что добавили в Go 1.23

### Range по функциям

Теперь `for range` поддерживает некоторые типы функций. Требования:

- функция принимает ровно один аргумент;
- этот аргумент — функция-yield с 0–2 параметрами и возвращаемым `bool`.

Варианты:

```go
func(yield func() bool)
func(yield func(V) bool)
func(yield func(K, V) bool)
```

Такие функции называем итераторами (push-итераторами). Они «проталкивают» значения, вызывая `yield`.

### Пакет `iter`

В стандартной библиотеке появился `iter`, где определены типы:

```go
package iter

type Seq[V any] func(yield func(V) bool)

type Seq2[K, V any] func(yield func(K, V) bool)
```

`Seq` — последовательность значений, `Seq2` — последовательность пар (например, ключ/значение словаря).

### Пример: `Set.All`

Метод возвращает итератор, который вызывает `yield` для каждого элемента множества и останавливается, если тот вернёт `false`.

```go
func (s *Set[E]) All() iter.Seq[E] {
	return func(yield func(E) bool) {
		for v := range s.m {
			if !yield(v) {
				return
			}
		}
	}
}
```

Использование выглядит максимально естественно:

```go
func PrintAllElements[E comparable](s *Set[E]) {
	for v := range s.All() {
		fmt.Println(v)
	}
}
```

Компилятор сам создаёт нужный `yield`, корректно обрабатывает `break`, `panic` и т.д.

## Pull-итераторы

Иногда нужен параллельный обход двух последовательностей. Для этого пригодны pull-итераторы: функция, которая при каждом вызове «вытягивает» очередное значение. Стандартная библиотека даёт конвертер `iter.Pull`, превращающий push-итератор (`iter.Seq`) в пару `(next, stop)`:

```go
next, stop := iter.Pull(seq)
defer stop()
for {
	v, ok := next()
	if !ok {
		break
	}
	// ...
}
```

Важно всегда вызывать `stop`, чтобы источник смог освободить ресурсы (закрыть каналы, завершить горутины).

### Пример сравнения двух последовательностей

```go
func EqSeq[E comparable](s1, s2 iter.Seq[E]) bool {
	next1, stop1 := iter.Pull(s1)
	defer stop1()
	next2, stop2 := iter.Pull(s2)
	defer stop2()

	for {
		v1, ok1 := next1()
		v2, ok2 := next2()
		if !ok1 {
			return !ok2
		}
		if ok1 != ok2 || v1 != v2 {
			return false
		}
	}
}
```

## Адаптеры

Стандартная форма итератора позволяет писать функции-адаптеры. Например, фильтр:

```go
func Filter[V any](f func(V) bool, s iter.Seq[V]) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range s {
			if f(v) && !yield(v) {
				return
			}
		}
	}
}
```

## Итератор для бинарного дерева

```go
type Tree[E any] struct {
	val         E
	left, right *Tree[E]
}

func (t *Tree[E]) All() iter.Seq[E] {
	return func(yield func(E) bool) {
		t.push(yield)
	}
}

func (t *Tree[E]) push(yield func(E) bool) bool {
	if t == nil {
		return true
	}
	return t.left.push(yield) &&
		yield(t.val) &&
		t.right.push(yield)
}
```

Рекурсия удобна: дополнительный стек не нужен, всё делает стек вызовов.

## Новые функции в `slices` и `maps`

### `slices`

- `All([]E) iter.Seq2[int, E]`
- `Values([]E) iter.Seq[E]`
- `Collect(iter.Seq[E]) []E`
- `AppendSeq([]E, iter.Seq[E]) []E`
- `Backward([]E) iter.Seq2[int, E]`
- `Sorted`, `SortedFunc`, `SortedStableFunc`
- `Repeat`, `Chunk`

### `maps`

- `All(map[K]V) iter.Seq2[K, V]`
- `Keys(map[K]V) iter.Seq[K]`
- `Values(map[K]V) iter.Seq[V]`
- `Collect(iter.Seq2[K, V]) map[K, V]`
- `Insert(map[K, V], iter.Seq2[K, V])`

### Пример с картой

```go
func LongStrings(m map[int]string, n int) []string {
	isLong := func(s string) bool {
		return len(s) >= n
	}
	return slices.Collect(Filter(isLong, maps.Values(m)))
}
```

`maps.Values` → итератор по значениям карты, `Filter` отбирает длинные строки, `slices.Collect` собирает результат в срез. Универсальность в том, что Filter не зависит от конкретного контейнера.

## Итераторы вне контейнеров

Итераторы применимы не только к структурам данных. Например, можно пройтись по строкам `[]byte`, не создавая промежуточных срезов:

```go
func Lines(data []byte) iter.Seq[[]byte] {
	return func(yield func([]byte) bool) {
		for len(data) > 0 {
			line, rest, _ := bytes.Cut(data, []byte{'\n'})
			if !yield(line) {
				return
			}
			data = rest
		}
	}
}
```

Использование:

```go
for line := range Lines(data) {
	handleLine(line)
}
```

## Можно и без `for range`

Push-итератор — обычная функция. Никто не мешает вызвать его вручную и передать кастомный `yield`:

```go
func PrintAllElements[E comparable](s *Set[E]) {
	s.All()(func(v E) bool {
		fmt.Println(v)
		return true
	})
}
```

Пользы немного, но демонстрирует, что `yield` — это просто функция, без магии.

## Итоги

- Go 1.23 позволяет итерироваться по функциям подходящего сигнатуры.
- Пакет `iter` вводит стандартные определения push-итераторов (`Seq`, `Seq2`) и утилиты вроде `Pull`.
- `slices` и `maps` получили набор помощников, понимающих итераторы.
- Push-итераторы удобны для `for range`, pull — для более сложных сценариев (параллельный обход, досрочное завершение).
- Стандартная форма упрощает написание адаптеров (`Filter`, `Chunk`, и т.д.) и унифицирует API контейнеров.

Пользуйтесь итерируемыми функциями, где это повышает читаемость и переиспользуемость, но не забывайте, что обычный цикл зачастую понятнее. Главное — у нас теперь есть выбор и общая договорённость по интерфейсу.

