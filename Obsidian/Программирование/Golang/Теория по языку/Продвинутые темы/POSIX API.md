---
tags:
  - Программирование
  - Теория_по_языку_Golang
  - Продвинутые_темы
  - POSIX
Связанные темы:
---
# **Работа с POSIX API в Go**  
*Низкоуровневые системные вызовы и взаимодействие с ОС*

---

### **1. Управление процессами (fork/exec)**
#### **Полноценная замена fork/exec в Go**
```go
// Запуск процесса с перенаправлением ввода/вывода
cmd := exec.Command("python3", "script.py")
cmd.Stdin = strings.NewReader("input data")
cmd.Stdout = &bytes.Buffer{}
cmd.Stderr = os.Stderr

// Настройка окружения
cmd.Env = append(os.Environ(), "GO_ENV=production")

// Запуск с таймаутом
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
cmd = exec.CommandContext(ctx, "sleep", "10")

if err := cmd.Start(); err != nil {
    log.Fatal(err)
}

// Ожидание завершения с обработкой статуса
if err := cmd.Wait(); err != nil {
    if exiterr, ok := err.(*exec.ExitError); ok {
        status := exiterr.Sys().(syscall.WaitStatus)
        fmt.Printf("Exit Status: %d\n", status.ExitStatus())
    }
}
```

#### **Особенности работы с процессами:**
- **Изоляция процессов**:  
  Использование `SysProcAttr` для:
  ```go
  cmd.SysProcAttr = &syscall.SysProcAttr{
      Cloneflags: syscall.CLONE_NEWPID, // Новое PID пространство
      Unshareflags: syscall.CLONE_NEWNS, // Новое mount пространство
  }
  ```
- **Работа с процесс-группами**:  
  ```go
  cmd.SysProcAttr = &syscall.SysProcAttr{
      Setpgid: true,
      Pgid: 0, // Новая группа
  }
  ```
- **Обработка дочерних процессов**:  
  Использование `cmd.Process.Wait()` для мониторинга

---
## **2. Работа с сигналами**  
### **Перехват сигналов**  
```go
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

go func() {
    sig := <-sigChan
    fmt.Printf("Получен сигнал: %v\n", sig)
    // Graceful shutdown
    os.Exit(0)
}()
```
**Основные сигналы:**  
- `SIGINT` (Ctrl+C)  
- `SIGTERM` (завершение процесса)  
- `SIGKILL` (неперехватываемый)  

### **Отправка сигналов**  
```go
process, _ := os.FindProcess(pid)
process.Signal(syscall.SIGUSR1)  // Отправка кастомного сигнала
```

---


### **3. Сокеты (net, syscall.Socket)**
#### **Низкоуровневая работа с сокетами**
```go
// Создание RAW-сокета для ICMP
fd, err := syscall.Socket(
    syscall.AF_INET,
    syscall.SOCK_RAW,
    syscall.IPPROTO_ICMP,
)
if err != nil {
    log.Fatal("Socket creation error:", err)
}
defer syscall.Close(fd)

// Настройка адреса
addr := syscall.SockaddrInet4{
    Port: 0,
    Addr: [4]byte{8,8,8,8}, // Google DNS
}

// Установка таймаутов
tv := syscall.Timeval{Sec: 5}
syscall.SetsockoptTimeval(fd, syscall.SOL_SOCKET, syscall.SO_RCVTIMEO, &tv)

// Отправка пакета
if err := syscall.Sendto(fd, []byte{0x08, 0x00}, 0, &addr); err != nil {
    log.Fatal("Send error:", err)
}
```

#### **Высокоуровневые абстракции net пакета**
```go
// TCP-сервер с graceful shutdown
listener, err := net.Listen("tcp", ":8080")
if err != nil {
    log.Fatal(err)
}

go func() {
    for {
        conn, err := listener.Accept()
        if errors.Is(err, net.ErrClosed) {
            break
        }
        go handleConnection(conn)
    }
}()

// UDP клиент с таймаутом
conn, err := net.DialTimeout("udp", "1.1.1.1:53", 2*time.Second)
udpConn := conn.(*net.UDPConn)
udpConn.SetReadDeadline(time.Now().Add(3 * time.Second))
```

#### **Ключевые особенности:**
- **Различия ОС**:  
  Поведение `SOCK_RAW` отличается на Linux/Windows/macOS
- **Буферизация**:  
  ```go
  // Отключение буферизации Nagle
  tcpConn.(*net.TCPConn).SetNoDelay(true)
  ```
- **Мультиплексирование**:  
  Использование `netpoll` в runtime Go для эффективного IO
- **Безопасность**:  
  ```go
  // Ограничение доступа к сокету
  syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1)
  ```

---

## **4. Каналы и пайпы (os.Pipe)**  
### **Анонимные пайпы**  
```go
r, w, err := os.Pipe()
if err != nil {
    log.Fatal(err)
}

go func() {
    w.Write([]byte("test"))
    w.Close()
}()

buf := make([]byte, 4)
r.Read(buf)  // Получаем "test"
```
**Использование:**  
- Связь между процессами  
- Перенаправление ввода/вывода  

### **Именованные пайпы (FIFO)**  
```go
syscall.Mkfifo("/tmp/myfifo", 0666)
f, _ := os.OpenFile("/tmp/myfifo", os.O_RDWR, 0)
```

---

## **5. Другие системные вызовы**  
### **Управление файловыми дескрипторами**  
```go
// Дублирование fd
newFd := syscall.Dup(oldFd)

// Установка флагов
syscall.FcntlInt(fd, syscall.F_SETFD, syscall.FD_CLOEXEC)
```

### **Работа с временем**  
```go
var tv syscall.Timeval
syscall.Gettimeofday(&tv)  // Точное время
```

---

## **Ограничения и особенности**  
1. **Портативность**  
   - Поведение может отличаться на Linux/Windows/macOS  
   - Используйте `GOOS` для кросс-платформенного кода  

2. **Безопасность**  
   - `syscall` устарел в пользу `golang.org/x/sys`  
   - Некоторые вызовы требуют прав root  

3. **Производительность**  
   - Прямые syscall быстрее высокоуровневых аналогов  
   - Но сложнее в поддержке  

**Пример измерения времени выполнения:**  
```go
start := time.Now()
syscall.Syscall(syscall.SYS_GETPID, 0, 0, 0)
duration := time.Since(start)
```

---

**Дополнительно:**  
- [Документация syscall](https://pkg.go.dev/syscall)  
- [Продвинутое использование в x/sys](https://pkg.go.dev/golang.org/x/sys)  
- [Unix системные вызовы](https://man7.org/linux/man-pages/man2/syscalls.2.html)  

Для production-кода предпочтительнее использовать высокоуровневые аналоги из пакетов `os`, `net` и `exec`, оставляя `syscall` для специфичных задач.