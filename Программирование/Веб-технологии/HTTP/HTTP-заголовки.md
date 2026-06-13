---
tags:
  - Программирование
  - HTTP
---
# HTTP-заголовки

HTTP-заголовки являются важной частью протокола HTTP и используются для передачи дополнительной информации между клиентом и сервером. Они состоят из пары "имя-значение" и разделяются двоеточием.

## Основные категории заголовков

### 1. Заголовки содержимого

#### Content-Type
**Назначение**: Определяет тип передаваемых данных  
**Форматы**:
- `text/html` - HTML документы
- `application/json` - JSON данные
- `application/xml` - XML данные
- `multipart/form-data` - Загрузка файлов
- `application/x-www-form-urlencoded` - Формы

**Примеры**:
```
Content-Type: application/json; charset=utf-8
Content-Type: multipart/form-data; boundary=something
```

#### Content-Length
**Назначение**: Указывает размер тела запроса/ответа в байтах  
**Пример**:
```
Content-Length: 348
```

#### Content-Encoding
**Назначение**: Указывает метод сжатия данных  
**Значения**:
- `gzip`
- `deflate`
- `br` (Brotli)

### 2. Заголовки аутентификации

#### Authorization
**Назначение**: Содержит учетные данные для аутентификации  
**Схемы**:
- `Basic` - Base64-кодированные логин:пароль
- `Bearer` - Токены доступа (JWT, OAuth)
- `Digest` - Хеш-аутентификация

**Примеры**:
```
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### WWW-Authenticate
**Назначение**: Указывает схему аутентификации для доступа к ресурсу  
**Пример**:
```
WWW-Authenticate: Basic realm="Access to staging site"
```

### 3. Заголовки кэширования

#### Cache-Control
**Назначение**: Управляет кэшированием на всех уровнях  
**Директивы**:
- `no-cache` - Проверять актуальность
- `no-store` - Не кэшировать
- `max-age=3600` - Время жизни кэша
- `public`/`private` - Где можно кэшировать

**Пример**:
```
Cache-Control: no-cache, no-store, must-revalidate
```

#### ETag
**Назначение**: Контрольная сумма ресурса для проверки изменений  
**Пример**:
```
ETag: "737060cd8c284d8af7ad3082f209582d"
```

### 4. Заголовки клиента

#### User-Agent
**Назначение**: Идентифицирует клиентское приложение  
**Пример**:
```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
```

#### Accept
**Назначение**: Указывает предпочитаемые форматы ответа  
**Пример**:
```
Accept: text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8
```

#### Accept-Encoding
**Назначение**: Указывает поддерживаемые методы сжатия  
**Пример**:
```
Accept-Encoding: gzip, deflate, br
```

### 5. CORS-заголовки

#### Access-Control-Allow-Origin
**Назначение**: Указывает разрешенные домены для кросс-доменных запросов  
**Пример**:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Origin: https://example.com
```

#### Access-Control-Allow-Methods
**Назначение**: Указывает разрешенные HTTP-методы  
**Пример**:
```
Access-Control-Allow-Methods: GET, POST, OPTIONS
```

## Важные особенности заголовков

1. **Регистр имен**: Имена заголовков нечувствительны к регистру, но принято использовать Upper-Camel-Case (Content-Type)

2. **Множественные значения**: Некоторые заголовки могут содержать несколько значений через запятую

3. **Собственные заголовки**: Можно создавать собственные заголовки с префиксом X-
   ```
   X-Custom-Header: value
   ```

4. **Длина заголовков**: Общая длина всех заголовков обычно ограничена 8-16KB

## Пример полного HTTP-запроса с заголовками

```
GET /api/data HTTP/1.1
Host: example.com
User-Agent: MyApp/1.0
Accept: application/json
Accept-Encoding: gzip, deflate
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Cache-Control: no-cache
```

## Пример полного HTTP-ответа с заголовками

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 127
Cache-Control: max-age=3600
ETag: "abc123"
Access-Control-Allow-Origin: *
Server: nginx/1.18.0
Date: Wed, 21 Oct 2023 07:28:00 GMT

{"status":"success","data":{}}
```

