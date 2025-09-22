---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Продвинутые_темы
  - Пакеты
  - json
Связанные темы:
---
# **Пакет encoding/json в Go**

Пакет `encoding/json` предоставляет инструменты для работы с JSON данными, включая сериализацию (маршалинг) и десериализацию (анмаршалинг) структур данных.

## **1. Маршалинг/анмаршалинг**

### **Базовый маршалинг (Go → JSON)**
```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func main() {
    p := Person{Name: "Alice", Age: 30}
    
    // Преобразование структуры в JSON
    jsonData, err := json.Marshal(p)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println(string(jsonData)) // {"name":"Alice","age":30}
}
```

### **Базовый анмаршалинг (JSON → Go)**
```go
jsonStr := `{"name":"Bob","age":25}`

var p Person
err := json.Unmarshal([]byte(jsonStr), &p)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("%+v\n", p) // {Name:Bob Age:25}
```

### **Обработка ошибок**
```go
err := json.Unmarshal([]byte(`{"age": "thirty"}`), &p)
if err != nil {
    if ute, ok := err.(*json.UnmarshalTypeError); ok {
        fmt.Printf("Неверный тип: %v, ожидалось %v\n", 
            ute.Value, ute.Type)
    }
}
```

## **2. Кастомные сериализаторы**

### **Реализация json.Marshaler**
```go
type CustomDate time.Time

func (cd CustomDate) MarshalJSON() ([]byte, error) {
    return []byte(`"` + time.Time(cd).Format("2006-01-02") + `"`), nil
}

func (cd *CustomDate) UnmarshalJSON(data []byte) error {
    t, err := time.Parse(`"2006-01-02"`, string(data))
    if err != nil {
        return err
    }
    *cd = CustomDate(t)
    return nil
}
```

### **Игнорирование полей**
```go
type Config struct {
    Password string `json:"-"`               // Полностью игнорируется
    Visible  bool   `json:"visible,omitempty"` // Пропускается если false
}
```

### **Обработка динамических данных**
```go
var data map[string]interface{}
json.Unmarshal(rawJSON, &data)

// Проверка типа
if value, ok := data["field"].(float64); ok {
    // Обработка числа
}
```

## **3. Потоковая обработка**

### **Декодирование потоков**
```go
dec := json.NewDecoder(resp.Body)
for {
    var item Item
    if err := dec.Decode(&item); err == io.EOF {
        break
    } else if err != nil {
        log.Fatal(err)
    }
    process(item)
}
```

### **Инкрементальное кодирование**
```go
enc := json.NewEncoder(os.Stdout)
for _, item := range items {
    if err := enc.Encode(item); err != nil {
        log.Fatal(err)
    }
}
```

### **Обработка больших файлов**
```go
file, _ := os.Open("large.json")
dec := json.NewDecoder(file)

// Чтение открывающей скобки массива
if _, err := dec.Token(); err != nil {
    log.Fatal(err)
}

// Чтение элементов по одному
for dec.More() {
    var elem Item
    if err := dec.Decode(&elem); err != nil {
        log.Fatal(err)
    }
    process(elem)
}
```

## **4. Производительность и оптимизация**

### **Использование jsoniter**
```go
import jsoniter "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary
```

### **Пул декодеров**
```go
var decoderPool = sync.Pool{
    New: func() interface{} {
        return json.NewDecoder(nil)
    },
}

func decodeFromReader(r io.Reader, v interface{}) error {
    dec := decoderPool.Get().(*json.Decoder)
    dec.Reset(r)
    defer decoderPool.Put(dec)
    return dec.Decode(v)
}
```

### **Предварительное выделение памяти**
```go
type Message struct {
    Text string
    Tags []string `json:",omitempty"`
}

// Указываем начальную емкость для среза
msg := Message{
    Tags: make([]string, 0, 10),
}
```

## **5. Обработка особых случаев**

### **HTML-экранирование**
```go
json.HTMLEscape(os.Stdout, []byte(`{"x":"<div>"}`))
// Вывод: {"x":"\u003cdiv\u003e"}
```

### **Pretty-print**
```go
data := map[string]interface{}{"a": 1, "b": "text"}
json.NewEncoder(os.Stdout).Encode(data) // Компактный

// С отступами
enc := json.NewEncoder(os.Stdout)
enc.SetIndent("", "  ")
enc.Encode(data)
```

### **Кастомные разделители**
```go
dec := json.NewDecoder(strings.NewReader(data))
dec.UseNumber() // Обрабатывать числа как json.Number
```

Пакет `encoding/json` покрывает большинство потребностей в работе с JSON, но для высоконагруженных систем стоит рассмотреть альтернативы вроде `json-iterator/go` или `ffjson`.