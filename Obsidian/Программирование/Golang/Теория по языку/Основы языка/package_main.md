---
tags:
  - Программирование
  - "#Основы_языка_Golang"
  - "#Теория_по_языку_Golang"
---

# package main — точка входа в программу

## Теория
- В Go программа начинается с объявления пакета. Файл с `package main` — это главный модуль, который собирается в исполняемый файл.

## Примеры
### Базовый пример
```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello world")
}
```


Для сборки нужно написать команду:
```bash
go build path/to/file/main.go
```
Для компиляции без сборки:
```bash
go run path/to/file/main.go
```
