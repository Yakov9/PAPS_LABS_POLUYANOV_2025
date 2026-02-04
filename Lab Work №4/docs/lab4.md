# Лабораторная работа №4  
Тема: Проектирование REST API  
Цель работы: Получить опыт проектирования программного интерфейса.

## Принятые проектные решения (9 решений)

При проектировании REST API были приняты следующие решения:

1. **Версионирование API через путь**  
   Все эндпоинты начинаются с `/api/v1/` — это позволяет в будущем добавлять `/v2/` без нарушения обратной совместимости.

2. **Использование RESTful принципов**  
   - POST — создание/запуск новой операции проверки  
   - GET — получение данных (логов, статуса)  
   - PUT — обновление существующей записи (лога)  
   - DELETE — удаление записей по фильтрам  

3. **Единый формат ошибок**  
   Все ошибки при валидации возвращаются в виде json с указанием сообщения ошибки и типа.

4. **Валидация входных данных**  
   Обязательная проверка через атрибуты `[Required]`, `[RegularExpression]` и кастомные валидаторы. Невалидные данные = 400 Bad Request.

5. **Поддержка base64 и multipart/form-data**  
   Для передачи бинарных данных (сертификаты, подписи, документы) используются два варианта: base64 в JSON и form-data.

6. **Асинхронность и имитация задержек**  
   Все методы асинхронные (`async Task<IActionResult>`)

7. **Отсутствие авторизации в лабораторной версии**  
   API работает без аутентификации (для упрощения тестирования в Postman). В продакшене будет токены/ключи.

8. **Ограничение размера файлов**  
   Для form-data установлен лимит 10 МБ (RequestSizeLimit), чтобы избежать перегрузки сервера.

## Документация по API

### 1. POST /api/v1/documentVerififcation/detached
**Описание**: Запускает проверку архива, расположенного в S3.  
**Метод**: POST  
**Входные данные** (application/json):  
```json
{
  "messageId": "string (обязательно)",
  "s3BucketFilePath": "string (обязательно, пример: s01_my-bucket_path_archive.zip)"
}
```
200 OK — проверка завершена
```json
{
  "messageId": "string",
  "fileId": "string | null (новый путь в S3 при успехе)",
  "errorMessage": "string | null"
}
```
400 Bad Request — некорректные данные
500 Internal Server Error — внутренняя ошибка

###2. POST /api/v1/documentVerififcation/detached/twoFiles
**Описание**: Проверка подписи и ориг. документа, загруженных напрямую в теле запроса.
**Метод**: POST
**Content-Type**: multipart/form-data
**Параметры**:

signature и data — IFormFile
Коды ответа:
200 OK
```json
{
  "status": "PASS" | "FAIL",
}
```
400 Bad Request — файл не передан или неверный формат

###3. POST /api/v1/signVerification
**Описание**: Проверка откреплённой подписи + оригинального документа + сертификата.
**Метод**: POST
**Content-Type**: application/json
**Тело запроса**:
```json
{
  "signatureBase64": "string (обязательно)",
  "documentBase64": "string (обязательно)",
  "certificateBase64": "string (обязательно)"
}
```
Коды ответа:

200 OK
```json
{
  "isValid": true | false,
  "message": "string"
}
```
400 Bad Request — не base64 или не указаны параметры

###4. GET /api/v1/verificationLogs/byMessageId/{messageId}
**Описание**: Получение всех логов проверок по messageId.
**Метод**: GET
**Параметры пути**: messageId (string)
**Коды ответа**:

200 OK — массив объектов логов
```json
{
  { "id": "guid",
  "messageId": "string",
  "finalStatus": "PASS",
  ...
  }
}
```

###5. GET /api/v1/verificationLogs/byId/{Id}
**Описание**: Получение логов по Id.
**Метод**: GET
**Параметры запроса (query)**:

Id — Guid
Коды ответа:
200 OK — массив объектов логов (в виде списка в JSON)

###6. DELETE /api/v1/verificationLogs/clean
**Описание**: Удаление логов по фильтрам (дата и/или тип операции).
**Метод**: DELETE
**Параметры запроса (query)**:

olderThan — DateTime (опционально) — удалить все логи до этой даты
operationType — "DOC" | "CER" | "SIGN" (опционально)
Коды ответа:
200 - с текстом Удалено n записей
400 Bad Request — некорректные фильтры

###7. POST /api/v1/certificateVerification
**Описание**: Проверка валидности сертификата в формате base64.
**Метод**: POST
**Content-Type**: application/json
Тело запроса:
```json
{
  "certRawData": "(base64 сертификата)"
}
```
200 OK
```json
{
  "isValid": true | false
}
```
400 - некорректный формат входных файлов или null

###8. POST /api/v1/certificateVerification/file
**Описание**: Проверка сертификата, загруженного файлом.
**Метод**: POST
**Content-Type**: multipart/form-data
**Параметры**:

certFile — IFormFile (.cer файл)
Коды ответа:
200 OK
```json
{
  "isValid": true | false
}
```
400 Bad Request — файл не передан или неверного формата

###9. PUT /api/v1/verification/logs/{logId}
**Описание**: Обновление/замена записи лога проверки по Id.
**Метод**: PUT
**Параметры пути**: Id (Guid)
**Content-Type**: application/json
Тело запроса:
```json
{
  "id": "guid (обязательно)",
  "completedAt": "2026-02-04T20:23:05.357Z",
  "messageId": "string",
  "thumbprint": "string",
  "subject": "string",
  "finalStatus": "string",
  "signId": "string",
  "errorMessage": "string",
  "operationType": "string",
  "fileUrl": "string"
}
```
Коды ответа:

200 OK
```json{
  "updated": true,
  "logId": "guid",
  "updatedAt": "datetime"
}
```
404 Not Found — запись не найдена
