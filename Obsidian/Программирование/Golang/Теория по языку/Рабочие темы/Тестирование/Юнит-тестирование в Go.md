---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Рабочие_темы_Golang
  - Тестирование
Связанные темы:
---

# `Юнит-тесты` в Golang


### **Юнит-тестирование в Go**

---

#### **1. Основы юнит-тестирования**
**Что такое юнит-тест?**  
Тестирование отдельных компонентов (функций, методов) в изоляции от зависимостей.

**Принципы:**
- Быстрые (<100мс на тест)
- Детерминированные (одинаковый результат при одинаковых условиях)
- Изолированные (не зависят от внешних сервисов)

**Структура теста:**
```go
func TestFunctionName(t *testing.T) {
    // Подготовка данных (Arrange)
    input := 5
    expected := 10

    // Вызов тестируемого кода (Act)
    result := Double(input)

    // Проверка результата (Assert)
    if result != expected {
        t.Errorf("Double(%d) = %d; want %d", input, result, expected)
    }
}
```

---

#### **2. Стандартный пакет testing**
**Основные методы:**
```go
t.Error()    // Некритичная ошибка (тест продолжается)
t.Fatal()    // Критичная ошибка (тест останавливается)
t.Log()      // Логирование (видно при -v)
t.Helper()   // Помечает функцию как вспомогательную
```

**Пример с субтестами:**
```go
func TestParseInt(t *testing.T) {
    tests := []struct{
        name  string
        input string
        want  int
        err   bool
    }{
        {"valid", "42", 42, false},
        {"invalid", "abc", 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseInt(tt.input)
            if (err != nil) != tt.err {
                t.Fatalf("unexpected error: %v", err)
            }
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

---

#### **3. Популярные библиотеки**
**1. Testify (assert/mock)**
```go
import "github.com/stretchr/testify/assert"

func TestAdd(t *testing.T) {
    assert.Equal(t, 4, Add(2, 2), "should add numbers")
    assert.NotNil(t, InitializeService())
}
```

**2. GoMock (генерация моков)**
```go
// Генерация интерфейса
//go:generate mockgen -source=db.go -destination=db_mock.go -package=main

func TestUserService(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockDB := NewMockDatabase(ctrl)
    mockDB.EXPECT().GetUser(1).Return(&User{Name: "Alice"}, nil)

    service := NewUserService(mockDB)
    user, err := service.GetUser(1)
    // Проверки...
}
```

**3. Ginkgo/Gomega (BDD-стиль)**
```go
Describe("Calculator", func() {
    Context("when adding numbers", func() {
        It("should return correct sum", func() {
            Expect(Add(2, 3)).To(Equal(5))
        })
    })
})
```

---

#### **4. Тестирование разных компонентов**
**1. Функции:**
```go
func TestSortSlice(t *testing.T) {
    input := []int{3, 1, 2}
    expected := []int{1, 2, 3}
    SortSlice(input)
    if !reflect.DeepEqual(input, expected) {
        t.Error("slice not sorted")
    }
}
```

**2. Методы структур:**
```go
func TestUser_Validate(t *testing.T) {
    u := User{Email: "test@example.com"}
    if err := u.Validate(); err != nil {
        t.Errorf("validation failed: %v", err)
    }
}
```

**3. Приватные функции:**
```go
// В файле module_test.go
func Test_privateHelper(t *testing.T) {
    result := privateHelper("input")
    // Проверки...
}
```

---

#### **5. Полезные практики**
**1. Тестовые данные:**
```go
func createTestUser() *User {
    return &User{
        ID:    1,
        Name: "Test User",
        Email: "test@example.com",
    }
}
```

**2. Табличные тесты:**
```go
func TestCalculate(t *testing.T) {
    tests := []struct{
        a, b int
        op   string
        want int
        err  bool
    }{
        {1, 2, "+", 3, false},
        {5, 0, "/", 0, true},
    }

    for _, tt := range tests {
        got, err := Calculate(tt.a, tt.b, tt.op)
        // Проверки...
    }
}
```

**3. Cleanup:**
```go
func TestWithCleanup(t *testing.T) {
    tempDir, err := os.MkdirTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { os.RemoveAll(tempDir) })

    // Используем tempDir в тестах...
}
```

---

#### **6. Покрытие кода**
**Запуск с анализом покрытия:**
```bash
go test -cover -coverprofile=coverage.out
go tool cover -html=coverage.out
```

**Целевые показатели:**
- 70-80% для бизнес-логики
- 50% для утилитарных пакетов
- 100% для критических компонентов

---

**Дополнительные материалы:**  
- `t.Parallel()` для параллельных тестов  
- Тестирование с таймаутами (`-timeout`)  
- Генерация тестовых данных (go-fakeit, go-randomdata)