# Атомарные операции, предоставляемые в стандартном пакете `sync/atomic`

В статье о [техниках синхронизации из пакета `sync`](sync-package-ru.md) представлены высокоуровневые техники синхронизации конкурентности. Пакет `sync/atomic` предоставляет низкоуровневые атомарные операции памяти, которые могут использоваться для синхронизации конкурентности. Эти операции обычно быстрее, чем операции с мьютексами, поскольку они не требуют блокировок.

Для некоторых простых случаев использования атомарные операции более эффективны и просты, чем использование каналов и техник синхронизации из пакета `sync`. Однако атомарные операции должны использоваться с осторожностью, поскольку они не гарантируют порядок выполнения других операций. Кроме того, атомарные операции имеют ограниченный диапазон применения по сравнению с другими техниками синхронизации.

## Обзор атомарных операций, предоставляемых до Go 1.19

Пакет `sync/atomic` предоставляет следующие функции для атомарных операций над целочисленными типами:

* Для типа `int32`: `AddInt32`, `LoadInt32`, `StoreInt32`, `SwapInt32`, `CompareAndSwapInt32`
* Для типа `int64`: `AddInt64`, `LoadInt64`, `StoreInt64`, `SwapInt64`, `CompareAndSwapInt64`
* Для типа `uint32`: `AddUint32`, `LoadUint32`, `StoreUint32`, `SwapUint32`, `CompareAndSwapUint32`
* Для типа `uint64`: `AddUint64`, `LoadUint64`, `StoreUint64`, `SwapUint64`, `CompareAndSwapUint64`
* Для типа `uintptr`: `AddUintptr`, `LoadUintptr`, `StoreUintptr`, `SwapUintptr`, `CompareAndSwapUintptr`

Каждая функция из приведенных выше списков выполняет соответствующую атомарную операцию:
* `Add`: атомарно добавляет значение к переменной
* `Load`: атомарно загружает значение из переменной
* `Store`: атомарно сохраняет значение в переменную
* `Swap`: атомарно обменивает новое значение со старым и возвращает старое значение
* `CompareAndSwap`: атомарно сравнивает значение с ожидаемым и, если они равны, заменяет его новым значением, возвращая `true`, в противном случае возвращает `false`

Для указателей пакет `sync/atomic` предоставляет следующие функции, использующие тип `unsafe.Pointer`:
* `LoadPointer`, `StorePointer`, `SwapPointer`, `CompareAndSwapPointer`

Кроме того, пакет `sync/atomic` предоставляет тип `Value`, который позволяет выполнять атомарные операции с любыми типами данных.

## Примеры использования атомарных операций

### Атомарное увеличение счетчика

В следующем примере используется `AddInt32` для атомарного увеличения значения переменной `n` в нескольких горутинах:

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
			defer wg.Done()
			atomic.AddInt32(&n, 1)
		}()
	}
	
	wg.Wait()
	fmt.Println(n) // Гарантированно выведет 1000
}
```

В этом примере создается 1000 горутин, каждая из которых увеличивает значение `n` на 1. Использование `atomic.AddInt32` гарантирует отсутствие гонок данных, и в итоге программа выводит `1000`.

### Атомарное чтение и запись

В следующем примере демонстрируется использование `LoadInt32` и `StoreInt32` для атомарного чтения и записи:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var value int32
	var wg sync.WaitGroup
	
	// Писатель
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			atomic.StoreInt32(&value, int32(i))
			fmt.Println("Записано:", i)
			time.Sleep(time.Millisecond)
		}
	}()
	
	// Читатель
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			v := atomic.LoadInt32(&value)
			fmt.Println("Прочитано:", v)
			time.Sleep(time.Millisecond)
		}
	}()
	
	wg.Wait()
}
```

### Атомарный обмен значений

Функция `SwapInt32` позволяет атомарно обменять значение переменной:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var value int32 = 10
	
	old := atomic.SwapInt32(&value, 20)
	fmt.Println("Старое значение:", old) // 10
	fmt.Println("Новое значение:", value) // 20
}
```

### Compare-and-Swap операция

Функция `CompareAndSwapInt32` выполняет атомарное сравнение и замену значения:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var value int32 = 10
	
	swapped := atomic.CompareAndSwapInt32(&value, 10, 20)
	fmt.Println("Замена выполнена:", swapped) // true
	fmt.Println("Значение:", value) // 20
	
	swapped = atomic.CompareAndSwapInt32(&value, 10, 30)
	fmt.Println("Замена выполнена:", swapped) // false (текущее значение 20, а не 10)
	fmt.Println("Значение:", value) // 20
}
```

## Использование типа `atomic.Value`

Тип `atomic.Value` позволяет выполнять атомарные операции с значениями любого типа. Он предоставляет методы `Load()`, `Store()`, `Swap()` и `CompareAndSwap()`.

Важно отметить, что тип `Value` не должен быть скопирован после первого использования. Значение типа `Value` должно быть создано через объявление или конструктор, а не через копирование.

Пример использования `atomic.Value`:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var config atomic.Value
	
	// Инициализация
	config.Store(map[string]int{
		"timeout": 30,
		"retries": 3,
	})
	
	var wg sync.WaitGroup
	
	// Несколько читателей
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 0; j < 3; j++ {
				cfg := config.Load().(map[string]int)
				fmt.Printf("Горутина %d: timeout=%d, retries=%d\n", 
					id, cfg["timeout"], cfg["retries"])
				time.Sleep(time.Millisecond)
			}
		}(i)
	}
	
	// Писатель
	wg.Add(1)
	go func() {
		defer wg.Done()
		time.Sleep(time.Millisecond * 50)
		config.Store(map[string]int{
			"timeout": 60,
			"retries": 5,
		})
		fmt.Println("Конфигурация обновлена")
	}()
	
	wg.Wait()
}
```

В этом примере несколько горутин-читателей атомарно читают конфигурацию, а одна горутина-писатель обновляет её. Все операции с `atomic.Value` являются потокобезопасными.

## Новые типы атомарных операций в Go 1.19

Начиная с Go 1.19, пакет `sync/atomic` предоставляет новые типы, которые делают работу с атомарными операциями более удобной:

* `atomic.Int32` — для атомарных операций с `int32`
* `atomic.Int64` — для атомарных операций с `int64`
* `atomic.Uint32` — для атомарных операций с `uint32`
* `atomic.Uint64` — для атомарных операций с `uint64`
* `atomic.Uintptr` — для атомарных операций с `uintptr`
* `atomic.Bool` — для атомарных операций с булевыми значениями
* `atomic.Pointer[T]` — обобщенный тип для атомарных операций с указателями

Эти типы предоставляют методы `Add`, `Load`, `Store`, `Swap` и `CompareAndSwap`.

### Пример использования новых типов

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var counter atomic.Int32
	var wg sync.WaitGroup
	
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Add(1)
		}()
	}
	
	wg.Wait()
	fmt.Println("Счетчик:", counter.Load()) // Гарантированно выведет 100
}
```

Использование новых типов более удобно, так как не требует передачи указателя:

```go
// Старый способ (до Go 1.19)
var n int32
atomic.AddInt32(&n, 1)
v := atomic.LoadInt32(&n)

// Новый способ (Go 1.19+)
var n atomic.Int32
n.Add(1)
v := n.Load()
```

### Использование `atomic.Bool`

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var flag atomic.Bool
	var wg sync.WaitGroup
	
	// Установка флага
	wg.Add(1)
	go func() {
		defer wg.Done()
		time.Sleep(time.Second)
		flag.Store(true)
		fmt.Println("Флаг установлен")
	}()
	
	// Проверка флага
	wg.Add(1)
	go func() {
		defer wg.Done()
		for !flag.Load() {
			time.Sleep(time.Millisecond * 100)
		}
		fmt.Println("Флаг обнаружен")
	}()
	
	wg.Wait()
}
```

### Использование `atomic.Pointer`

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

type Config struct {
	Timeout int
	Retries int
}

func main() {
	var config atomic.Pointer[Config]
	var wg sync.WaitGroup
	
	// Инициализация
	config.Store(&Config{Timeout: 30, Retries: 3})
	
	// Читатели
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			cfg := config.Load()
			fmt.Printf("Горутина %d: timeout=%d, retries=%d\n", 
				id, cfg.Timeout, cfg.Retries)
		}(i)
	}
	
	// Писатель
	wg.Add(1)
	go func() {
		defer wg.Done()
		config.Store(&Config{Timeout: 60, Retries: 5})
		fmt.Println("Конфигурация обновлена")
	}()
	
	wg.Wait()
}
```

## Атомарные операции с указателями (до Go 1.19)

Для атомарных операций с указателями до Go 1.19 необходимо использовать `unsafe.Pointer`:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"unsafe"
)

type Config struct {
	Timeout int
	Retries int
}

func main() {
	var config unsafe.Pointer
	var wg sync.WaitGroup
	
	// Инициализация
	initial := &Config{Timeout: 30, Retries: 3}
	atomic.StorePointer(&config, unsafe.Pointer(initial))
	
	// Читатели
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			cfg := (*Config)(atomic.LoadPointer(&config))
			fmt.Printf("Горутина %d: timeout=%d, retries=%d\n", 
				id, cfg.Timeout, cfg.Retries)
		}(i)
	}
	
	// Писатель
	wg.Add(1)
	go func() {
		defer wg.Done()
		newConfig := &Config{Timeout: 60, Retries: 5}
		atomic.StorePointer(&config, unsafe.Pointer(newConfig))
		fmt.Println("Конфигурация обновлена")
	}()
	
	wg.Wait()
}
```

## Сравнение производительности

Атомарные операции обычно быстрее, чем использование мьютексов, поскольку они не требуют блокировок на уровне операционной системы. Однако для сложных критических секций мьютексы могут быть более подходящим выбором, поскольку они обеспечивают более четкие гарантии порядка выполнения операций.

## Когда использовать атомарные операции

Атомарные операции подходят для:
* Простых операций с одним значением (инкремент, декремент, установка флага)
* Реализации спинлоков и других низкоуровневых примитивов синхронизации
* Операций, где производительность критична
* Счетчиков и простых состояний, доступных из нескольких горутин

Атомарные операции не подходят для:
* Сложных критических секций с множественными операциями
* Когда требуется координация между несколькими операциями
* Когда требуется гарантированный порядок выполнения операций

В таких случаях лучше использовать мьютексы, каналы или другие техники синхронизации из пакета `sync`.

## Важные замечания

1. **Порядок выполнения:** Атомарные операции не гарантируют порядок выполнения других операций. Если требуется гарантированный порядок, используйте мьютексы или каналы.

2. **Безопасность памяти:** Атомарные операции обеспечивают атомарность операции, но не гарантируют видимость изменений в других горутинах без дополнительных барьеров памяти. В большинстве случаев это работает корректно, но для сложных сценариев может потребоваться использование явных барьеров памяти.

3. **Копирование значений:** Тип `atomic.Value` не должен копироваться после первого использования. Значение должно быть создано через объявление или конструктор.

4. **Типы данных:** До Go 1.19 атомарные операции поддерживаются только для определенных типов: `int32`, `int64`, `uint32`, `uint64`, `uintptr` и указателей (через `unsafe.Pointer`). Для других типов используйте `atomic.Value`.

5. **Алignment (выравнивание):** Переменные, используемые в атомарных операциях, должны быть правильно выровнены в памяти. Обычно это не является проблемой, так как компилятор Go автоматически выравнивает переменные, но стоит помнить об этом при работе с нестандартными структурами данных.

## Пример: реализация простого счетчика с использованием атомарных операций

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

type Counter struct {
	value atomic.Int64
}

func (c *Counter) Increment() int64 {
	return c.value.Add(1)
}

func (c *Counter) Decrement() int64 {
	return c.value.Add(-1)
}

func (c *Counter) Get() int64 {
	return c.value.Load()
}

func (c *Counter) Reset() {
	c.value.Store(0)
}

func main() {
	var counter Counter
	var wg sync.WaitGroup
	
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Increment()
		}()
	}
	
	wg.Wait()
	fmt.Println("Финальное значение:", counter.Get()) // 100
}
```

Этот пример демонстрирует, как создать потокобезопасный счетчик, используя атомарные операции. Такой подход более эффективен, чем использование мьютекса для простого инкремента.

---

**Источник:** [Atomic Operations Provided in The sync/atomic Standard Package - Go 101](https://go101.org/article/concurrent-atomic-operation.html)

