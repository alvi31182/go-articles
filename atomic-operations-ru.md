# Атомарные операции, предоставляемые в стандартном пакете `sync/atomic`

Атомарные операции более примитивны, чем другие техники синхронизации. Они не требуют блокировок и обычно реализованы непосредственно на аппаратном уровне. Фактически, они часто используются при реализации других техник синхронизации.

Обратите внимание, многие примеры ниже не являются конкурентными программами. Они предназначены только для демонстрации и объяснения, чтобы показать, как использовать атомарные функции, предоставляемые в стандартном пакете `sync/atomic`.

## Обзор атомарных операций, предоставляемых до Go 1.19

Стандартный пакет `sync/atomic` предоставляет следующие пять атомарных функций для целочисленного типа `T`, где `T` должен быть любым из `int32`, `int64`, `uint32`, `uint64` и `uintptr`.

```go
func AddT(addr *T, delta T)(new T)
func LoadT(addr *T) (val T)
func StoreT(addr *T, val T)
func SwapT(addr *T, new T) (old T)
func CompareAndSwapT(addr *T, old, new T) (swapped bool)
```

Например, для типа `int32` предоставляются следующие пять функций:

```go
func AddInt32(addr *int32, delta int32)(new int32)
func LoadInt32(addr *int32) (val int32)
func StoreInt32(addr *int32, val int32)
func SwapInt32(addr *int32, new int32) (old int32)
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
```

Для (безопасных) типов указателей предоставляются следующие четыре атомарные функции. Когда эти функции были введены в стандартную библиотеку, Go еще не поддерживал пользовательские дженерики, поэтому эти функции реализованы через тип небезопасного указателя `unsafe.Pointer` (Go-аналог `void*` в C).

```go
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
```

Для указателей нет функции `AddPointer`, поскольку указатели Go (безопасные) не поддерживают арифметические операции.

Стандартный пакет `sync/atomic` также предоставляет тип `Value`, соответствующий указательный тип `*Value` которого имеет четыре метода (перечислены ниже, последние два были введены в Go 1.17). Мы можем использовать эти методы для выполнения атомарных операций со значениями любого типа.

```go
func (*Value) Load() (x interface{})
func (*Value) Store(x interface{})
func (*Value) Swap(new interface{}) (old interface{})
func (*Value) CompareAndSwap(old, new interface{}) (swapped bool)
```

## Обзор новых атомарных операций, предоставляемых с Go 1.19

Go 1.19 ввел несколько типов, каждый из которых имеет набор методов атомарных операций, чтобы достичь тех же эффектов, что и функции уровня пакета, перечисленные в предыдущем разделе.

Среди этих типов `Int32`, `Int64`, `Uint32`, `Uint64` и `Uintptr` предназначены для целочисленных атомарных операций. Методы типа `atomic.Int32` перечислены ниже. Методы остальных четырех типов представлены аналогичным образом.

```go
func (*Int32) Add(delta int32) (new int32)
func (*Int32) Load() int32
func (*Int32) Store(val int32)
func (*Int32) Swap(new int32) (old int32)
func (*Int32) CompareAndSwap(old, new int32) (swapped bool)
```

Начиная с Go 1.18, Go уже поддерживал пользовательские дженерики. И некоторые стандартные пакеты начали использовать пользовательские дженерики с Go 1.19. Пакет `sync/atomic` — один из таких пакетов. Тип `Pointer[T any]`, введенный в этот пакет Go 1.19, является дженерик-типом. Его методы перечислены ниже.

```go
(*Pointer[T]) Load() *T
(*Pointer[T]) Store(val *T)
(*Pointer[T]) Swap(new *T) (old *T)
(*Pointer[T]) CompareAndSwap(old, new *T) (swapped bool)
```

Go 1.19 также ввел тип `Bool` для выполнения булевых атомарных операций.

## Атомарные операции для целых чисел

Оставшаяся часть этой статьи показывает некоторые примеры того, как использовать атомарные операции, предоставляемые в Go.

В следующем примере показано, как выполнить атомарную операцию `Add` для значения `int32`, используя функцию `AddInt32`. В этом примере основная горутина создает 1000 новых конкурентных горутин. Каждая из новых горутин увеличивает целое число `n` на единицу. Атомарные операции гарантируют отсутствие гонок данных среди этих горутин. В конце гарантированно будет выведено `1000`.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var n int32
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			atomic.AddInt32(&n, 1)
			wg.Done()
		}()
	}
	wg.Wait()

	fmt.Println(atomic.LoadInt32(&n)) // 1000
}
```

Если оператор `atomic.AddInt32(&n, 1)` заменить на `n++`, то вывод может быть не `1000`.

Следующий код перереализует вышеуказанную программу, используя тип `atomic.Int32` и его методы (с Go 1.19). Этот код выглядит немного аккуратнее.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var n atomic.Int32
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			n.Add(1)
			wg.Done()
		}()
	}
	wg.Wait()

	fmt.Println(n.Load()) // 1000
}
```

Начиная с Go 1.25, можно использовать метод `WaitGroup.Go`, который упрощает использование `WaitGroup` и делает код более читаемым. Вместо явного вызова `wg.Add(1)` и `defer wg.Done()` можно использовать метод `Go`, который автоматически запускает функцию в новой горутине и управляет счетчиком `WaitGroup`:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var n int32
	var wg sync.WaitGroup
	for range 1000 {
		wg.Go(func() {
			atomic.AddInt32(&n, 1)
		})
	}
	wg.Wait()

	fmt.Println(atomic.LoadInt32(&n)) // 1000
}
```

Атомарные функции/методы `StoreT` и `LoadT` часто используются для реализации методов setter и getter (соответствующего указательного типа) типа, если значения типа должны использоваться конкурентно. Например, версия функции:

```go
type Page struct {
	views uint32
}

func (page *Page) SetViews(n uint32) {
	atomic.StoreUint32(&page.views, n)
}

func (page *Page) Views() uint32 {
	return atomic.LoadUint32(&page.views)
}
```

И версия с типом+методами (с Go 1.19):

```go
type Page struct {
	views atomic.Uint32
}

func (page *Page) SetViews(n uint32) {
	page.views.Store(n)
}

func (page *Page) Views() uint32 {
	return page.views.Load()
}
```

Для знакового целочисленного типа `T` (`int32` или `int64`) второй аргумент вызова функции `AddT` может быть отрицательным значением для выполнения атомарной операции уменьшения. Но как выполнить атомарные операции уменьшения для значений беззнакового типа `T`, такого как `uint32`, `uint64` и `uintptr`? Есть два обстоятельства для второго беззнакового аргумента.

1. Для беззнаковой переменной `v` типа `T`, `-v` является допустимым в Go. Поэтому мы можем просто передать `-v` в качестве второго аргумента вызова `AddT`.
2. Для положительной целочисленной константы `c`, `-c` недопустимо использовать в качестве второго аргумента вызова `AddT` (где `T` обозначает беззнаковый целочисленный тип). Мы можем использовать `^T(c-1)` в качестве второго аргумента вместо этого.

Этот трюк `^T(v-1)` также работает для беззнаковой переменной `v`, но `^T(v-1)` менее эффективен, чем `T(-v)`.

В трюке `^T(c-1)`, если `c` — типизированное значение и его тип точно `T`, то форма может быть сокращена как `^(c-1)`.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var (
		n uint64 = 97
		m uint64 = 1
		k int    = 2
	)
	const (
		a        = 3
		b uint64 = 4
		c uint32 = 5
		d int    = 6
	)

	// Эти строки компилируются.
	atomic.AddUint64(&n, -m)           // -m вычитается из n
	fmt.Println(n)                      // 96
	atomic.AddUint64(&n, ^uint64(0))   // -1 вычитается из n
	fmt.Println(n)                      // 95
	atomic.AddUint64(&n, ^uint64(a-1)) // -3 вычитается из n
	fmt.Println(n)                      // 92
	atomic.AddUint64(&n, ^(b-1))        // -4 вычитается из n
	fmt.Println(n)                      // 88
	atomic.AddUint32(&n, ^uint32(c-1))  // -5 вычитается из n
	fmt.Println(n)                      // 83

	// Эти строки не компилируются.
	// atomic.AddUint64(&n, -k)
	// atomic.AddUint64(&n, a)
	// atomic.AddUint64(&n, -a)
	// atomic.AddUint32(&n, -c)
	// atomic.AddUint64(&n, d)
	// atomic.AddUint64(&n, -d)
}
```

## Атомарные операции для указателей

Следующий пример показывает, как использовать функции атомарных указателей для выполнения атомарных операций над указателями.

```go
package main

import (
	"fmt"
	"sync/atomic"
	"unsafe"
)

type T struct{ x int }

func main() {
	var pT unsafe.Pointer

	var ta, tb T
	ta.x = 111
	tb.x = 222

	// store
	atomic.StorePointer(&pT, unsafe.Pointer(&ta))
	fmt.Println(pT)                          // 0x...
	fmt.Println((*T)(pT).x)                  // 111
	fmt.Println((*T)(atomic.LoadPointer(&pT)).x) // 111

	// load
	pa1 := (*T)(atomic.LoadPointer(&pT))
	fmt.Println(pa1.x)     // 111
	fmt.Println(pa1 == &ta) // true

	// swap
	pa2 := atomic.SwapPointer(&pT, unsafe.Pointer(&tb))
	fmt.Println((*T)(pa2).x) // 111
	fmt.Println((*T)(atomic.LoadPointer(&pT)).x) // 222
	fmt.Println(pa2 == unsafe.Pointer(&ta)) // true

	// compare and swap
	pb := atomic.LoadPointer(&pT)
	fmt.Println(pb == unsafe.Pointer(&tb)) // true
	b := atomic.CompareAndSwapPointer(&pT, pb, unsafe.Pointer(&ta))
	fmt.Println(b)                          // true
	fmt.Println((*T)(atomic.LoadPointer(&pT)).x) // 111
	b = atomic.CompareAndSwapPointer(&pT, pb, unsafe.Pointer(&tb))
	fmt.Println(b) // false (pT уже не указывает на tb)
}
```

Да, использование функций атомарных указателей довольно многословно. Фактически, не только использование многословно, они также не защищены рекомендациями совместимости Go 1, поскольку эти использования требуют импорта стандартного пакета `unsafe`.

Напротив, код будет намного проще и чище, если мы используем дженерик-тип `Pointer`, введенный в Go 1.19, и его методы для выполнения атомарных операций с указателями, как показывает следующий код.

```go
package main

import (
	"fmt"
	"sync/atomic"
)

type T struct{ x int }

func main() {
	var pT atomic.Pointer[T]
	var ta, tb = T{1}, T{2}

	// store
	pT.Store(&ta)
	fmt.Println(pT.Load()) // &{1}

	// load
	pa1 := pT.Load()
	fmt.Println(pa1 == &ta) // true

	// swap
	pa2 := pT.Swap(&tb)
	fmt.Println(pa2 == &ta) // true
	fmt.Println(pT.Load())  // &{2}

	// compare and swap
	b := pT.CompareAndSwap(&ta, &tb)
	fmt.Println(b) // false
	b = pT.CompareAndSwap(&tb, &ta)
	fmt.Println(b) // true
}
```

Более того, реализация с использованием дженерик-типа `Pointer` защищена рекомендациями совместимости Go 1.

## Атомарные операции для значений произвольных типов

Тип `Value`, предоставляемый в стандартном пакете `sync/atomic`, может использоваться для атомарной загрузки и сохранения значений любого типа.

Тип `*Value` имеет несколько методов: `Load`, `Store`, `Swap` и `CompareAndSwap` (последние два введены в Go 1.17). Типы входных параметров этих методов — все `interface{}`. Поэтому любое значение может быть передано в вызовы этих методов. Но для адресуемого значения `Value` `v`, после того как вызов `v.Store()` (сокращение для `(&v).Store()`) когда-либо был вызван, последующие вызовы методов для значения `v` также должны принимать аргументы-значения с тем же конкретным типом, что и аргумент первого вызова `v.Store()`, иначе произойдут паники. Аргумент `nil` интерфейса также вызовет панику при вызове `v.Store()`.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	type T struct{ a, b, c int }
	var ta = T{1, 2, 3}
	var v atomic.Value
	v.Store(ta)
	var tb = v.Load().(T)
	fmt.Println(tb)       // {1 2 3}
	fmt.Println(ta == tb) // true

	v.Store("hello") // вызовет панику
}
```

Другой пример (для Go 1.17+):

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	type T struct{ a, b, c int }
	var x = T{1, 2, 3}
	var y = T{4, 5, 6}
	var z = T{7, 8, 9}
	var v atomic.Value
	v.Store(x)
	fmt.Println(v) // {{1 2 3}}
	old := v.Swap(y)
	fmt.Println(v)       // {{4 5 6}}
	fmt.Println(old.(T)) // {1 2 3}
	swapped := v.CompareAndSwap(x, z)
	fmt.Println(swapped, v) // false {{4 5 6}}
	swapped = v.CompareAndSwap(y, z)
	fmt.Println(swapped, v) // true {{7 8 9}}
}
```

Фактически, мы также можем использовать функции атомарных указателей, объясненные в предыдущем разделе, для выполнения атомарных операций со значениями любого типа, с одним дополнительным уровнем косвенности. Оба способа имеют свои соответствующие преимущества и недостатки. Какой способ следует использовать, зависит от требований на практике.

## Гарантии порядка памяти, обеспечиваемые атомарными операциями в Go

Пожалуйста, прочитайте [модель памяти Go](https://go.dev/ref/mem) для получения подробной информации.

---

**Источник:** [Atomic Operations Provided in The sync/atomic Standard Package - Go 101](https://go101.org/article/concurrent-atomic-operation.html)
