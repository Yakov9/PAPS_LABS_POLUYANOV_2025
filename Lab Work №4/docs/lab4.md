# Лабораторная работа №4  
Тема: Проектирование REST API  
Цель работы: Получить опыт проектирования программного интерфейса.

## Принятые проектные решения (8 решений)

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

### 1. POST /api/v1/signVerification
**Описание**: Проверка откреплённой подписи + оригинального документа + сертификата.
**Метод**: POST
**Content-Type**: application/json
**Тело запроса**:
```json
{
  "signatureBase64": "string (обязательно)" - открепленная подпись в base64,
  "documentBase64": "string (обязательно)" - оригинал документа в base64,
  "certificateBase64": "string (обязательно) - сертификат, испольхованный для подписания в base64"
}
```
Коды ответа:

200 OK
```json
{
  "isValid": true | false - валидна или нет,
  "message": "string" - сообщение ошибки
}
```
400 Bad Request — не base64 или не указаны параметры

### 2. POST /api/v1/documentVerififcation/detached/twoFiles
**Описание**: Проверка подписи и ориг. документа, загруженных напрямую в теле запроса.
**Метод**: POST
**Content-Type**: multipart/form-data
**Параметры**:

signature - подписанный файл
data - файл оригинального документа
Коды ответа:
200 OK
```json
{
  "status": "PASS" | "FAIL" - валиден или нет,
}
```
400 Bad Request — файл не передан или неверный формат

### 3. POST /api/v1/documentVerififcation/detached
**Описание**: Запускает проверку архива, расположенного в S3.  
**Метод**: POST  
**Входные данные** (application/json):  
```json
{
  "messageId": "string (обязательно)" - идентификатор сообщения запроса на проверку,
  "s3BucketFilePath": "string (обязательно, пример: s01_my-bucket_path_archive.zip) - путь до файла в хранилище s3"
}
```
200 OK — проверка завершена
```json
{
  "messageId": "string" - идентификатор сообщения запроса на проверку,
  "fileId": "string | null - (новый путь в S3 при успехе)",
  "errorMessage": "string | null" - ошибка при валидации (если все упешно, то null)
}
```
400 Bad Request — некорректные данные
500 Internal Server Error — внутренняя ошибка

### 4. GET /api/v1/verificationLogs/byMessageId/{messageId}
**Описание**: Получение всех логов проверок по messageId (идентификатор запроса на проверку документов).
**Метод**: GET
**Параметры пути** / **Входные**: messageId (string)

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

### 5. GET /api/v1/verificationLogs/byId/{Id}
**Описание**: Получение логов по Id (идентификатор записей логов в БД).
**Метод**: GET
**Параметры запроса (query)** / **Входные**::

Id — Guid
Коды ответа:
200 OK — массив объектов логов (в виде списка в JSON)

### 6. DELETE /api/v1/verificationLogs/clean
**Описание**: Удаление логов по фильтрам (дата и/или тип операции).
**Метод**: DELETE
**Параметры запроса (query)** / **Входные**::

olderThan — DateTime (опционально) — удалить все логи до этой даты
operationType — "DOC" | "CER" | "SIGN" (опционально) - удалить записи логов с проверками этого типа
Коды ответа:
200 - с текстом Удалено n записей
400 Bad Request — некорректные фильтры

### 7. POST /api/v1/certificateVerification
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
  "isValid": true | false - валиден или нет
}
```
400 - некорректный формат входных файлов или null

### 8. POST /api/v1/certificateVerification/file
**Описание**: Проверка сертификата, загруженного файлом.
**Метод**: POST
**Content-Type**: multipart/form-data
**Параметры**:

certFile — IFormFile (.cer файл) - файл сертификата на проверку
Коды ответа:
200 OK
```json
{
  "isValid": true | false
}
```
400 Bad Request — файл не передан или неверного формата

### 9. PUT /api/v1/verification/logs
**Описание**: Обновление/замена записи лога проверки по Id. Остальные поля указывать на новые значения, если их надо поменять
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

## Тестирование API
Для каждого эндпоинта проведено минимум 2 теста в Postman (позитивный сценарий + негативный).

Результаты написанных автотестов:
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/tests.png)
Далее будут расписаны конкретные написанные запросы (к которым написаны тесты)

### 1. POST /api/v1/verification/signature

Позитивный: валидные подпись с оригинальным документом -> 200 true
тело:
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/1signVerify_pos.jpg)
хедеры:
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/1signVerify_pos_headers.jpg)

Негативный 1: невалидная base64-строка в подписи -> 400 Bad Request
тело:
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/1signVerify400.jpg)
хедеры:
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/1signVerify400_headers.jpg)

Негативный 2: невалидная подпись -> 200 false
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/1signVerify_neg.jpg)

Далее не буду предоставлять скрины хедеров (они почти везде одни и те же)

### 2. POST /api/v1/documentVerififcation/detached/twoFiles

Позитивный: файл валиден -> 200 PASS
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/8two_pos.jpg)
Негативный: файл (подпись) не валиден -> 200 FAIL
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/8two_neg.jpg)

### 3. POST /api/v1/documentVerification/detached

Позитивный: путь s3 на валидный архив -> 200 OK, fileId заполнен
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/9s3_pos.jpg)
Негативный: путь s3 на невалидный архив -> 200 OK, fileId не заполнен, указан errorMessage
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/9s3_neg.jpg)

### 4. GET  /api/v1/verificationLogs/byMessageId/{messageId}

Позитивный: существующий messageId -> 200 + записи по этому messageId
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/3byMessageId_pos.jpg)
Негативный: не существующий messageId -> 200 с пустым телом
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/3byMessageId_none.jpg)

### 5. GET /api/v1/verificationLogs/byId/{Id}

Позитивный: верный формат Id (существующий) - 200 + запись по этому Id
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/2byId_pos.jpg)
Негативный: неверный формат Guid -> 400
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/2byId_400.jpg)
Негативный: верный формат Id (но не существующий) - 200 + пустое тело
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/2byIdNone.jpg)

### 6. DELETE /api/v1/verificationLogs/clean

Позитивный: olderThan -> 200 и информация о кол-ве удаленных записей
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/4del_older.jpg)
Позитивный: operationType -> 200 и информация о кол-ве удаленных записей
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/4del_type.jpg)
Позитивный: olderThan + operationType -> 200 и информация о кол-ве удаленных записей
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/4del_comb.jpg)
Негатиыный: olderThan -> 200 и ни одной записи не удалилось
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/4del_none.jpg)

### 7. POST /api/v1/certificateVerification

Позитивный: валидный сертификат в виде строки base64 -> 200 true
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/5cert_pos.jpg)
Негативный: некорректный base64 -> 400 Bad Request
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/5cert_400.jpg)
Негативный: невалидный сертификат в виде строки base64 -> 200 false
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/5cert_neg.jpg)

### 8. POST /api/v1/certificateVerification/file

Позитивный: валидный сертификат в виде файла -> 200 true
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/6certfile_neg.jpg)
Негативный: невалидный сертификат в виде файла -> 200 false
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/6certfile_pos.jpg)

### 9. PUT /api/v1/verification/logs/{logId}

Позитивный: указание существующего Id и верный формат параметров лога для изменения -> 200 Данные обновлены
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/7put_pos.jpg)
Негативный: указание несуществующего Id -> 200 запись не найдены
![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork4/Lab%20Work%20%E2%84%964/docs/7put_NF.jpg)
