# Лабораторная работа №3

## Тема
Использование принципов проектирования на уровне методов и классов

## Цель работы
Получить опыт проектирования и реализации модулей с использованием принципов KISS, YAGNI, DRY, SOLID и других принципов разработки программного обеспечения.

## Диаграмма контейнеров

![Containers](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork2/Lab%20Work%20%E2%84%962/docs/containers.jpg)

### Описание элементов и выбор архитектурного стиля
Система реализована в виде **двух независимых микросервисов** (архитектурный стиль — микросервисная архитектура с асинхронным обменом через брокер сообщений):

| Контейнер                          | Технология       | Описание |
|------------------------------------|------------------|---------|
| Proxy Service                      | .NET 6 Worker Service  | Приём сообщений из Kafka (входной топик), валидация метаданных, отправка HTTP-запроса в Verification, получение результата, публикация результата в Kafka (топик result) |
| Verification Service           | .NET 6 Web API | Основная бизнес-логика: скачивание архива из S3, распаковка, валидация XML по XSD, запуск cryptcp для каждой подписи, формирование протокола, сохранение в БД, перемещение архива при успехе |
| Apache Kafka                       | Kafka  | Брокер сообщений |
| Object Storage (S3-совместимое)    | MinIO  | Хранилище исходных и проверенных архивов |
| PostgreSQL                         | PostgreSQL 15+   | Хранение протоколов проверок и метаданных |

## Диаграмма компонентов — Verification Service

![Components — Verification](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork2/Lab%20Work%20%E2%84%962/docs/components-verification.jpg)

## Диаграмма компонентов — Proxy Service

![Components — Proxy](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork2/Lab%20Work%20%E2%84%962/docs/components-proxy.jpg)

## Диаграмма последовательностей

![Components — Proxy](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork2/Lab%20Work%20%E2%84%962/docs/components-proxy.jpg)

Диаграмма последовательностей отражает сценарий проверки электронной подписи документа.
Пользователь загружает документы через приложение, после чего запрос передается на сервер.

## Модель базы данных

Модель данных представлена в виде UML-диаграммы классов. Для хранения логов результата проверки подписей используется одна сущность.

![Components — Proxy](https://github.com/Yakov9/PAPS_LABS_POLUYANOV_2025/blob/LabWork2/Lab%20Work%20%E2%84%962/docs/components-proxy.jpg)

## Применение основных принципов разработки

В рамках разработки сервиса проверки электронных подписей были применены
базовые принципы проектирования программного обеспечения:
KISS, YAGNI, DRY и SOLID.
Ниже приведены фрагменты кода (псевдокода), демонстрирующие применение данных принципов.

### KISS (Keep It Simple, Stupid)

Принцип KISS реализован за счёт простых и легко читаемых методов,
каждый из которых выполняет одну конкретную задачу без избыточной логики.

public VerificationResult VerifySignature(byte[] document, byte[] signature)
{
    if (!_cryptoService.Verify(document, signature))
    {
        return VerificationResult.Invalid("Подпись недействительна");
    }

    return VerificationResult.Valid();
}

Метод содержит минимальное количество условий и возвращает результат
проверки без дополнительной обработки, что упрощает понимание и сопровождение кода.

### YAGNI (You Aren't Gonna Need It)

Принцип YAGNI соблюдён за счёт реализации только той функциональности,
которая необходима для текущего варианта использования.

public class SignatureVerificationService
{
    public VerificationResult Verify(byte[] document, byte[] signature)
    {
        return _cryptoService.Verify(document, signature)
            ? VerificationResult.Valid()
            : VerificationResult.Invalid("Ошибка проверки подписи");
    }
}

В системе отсутствует преждевременная реализация дополнительных функций,
таких как поддержка нескольких алгоритмов подписи, расширенные отчёты
или сложная система ролей пользователей.

### DRY (Don't Repeat Yourself)

Принцип DRY реализован путём вынесения повторяющейся логики
криптографической проверки в отдельный сервис.

public interface ICryptoService
{
    bool Verify(byte[] data, byte[] signature);
}

public class CryptoService : ICryptoService
{
    public bool Verify(byte[] data, byte[] signature)
    {
        // Единая реализация криптографической проверки
        return true;
    }
}

Данный сервис используется всеми компонентами системы,
что исключает дублирование логики и упрощает поддержку кода.

### SOLID

При проектировании системы были применены все основные принципы SOLID.

#### Single Responsibility Principle (SRP)

Каждый класс системы отвечает только за одну зону ответственности.

public class SignatureVerificationService
{
    public VerificationResult VerifySignature(byte[] document, byte[] signature)
    {
        // Проверка подписи
    }
}

public class CertificateRepository
{
    public Certificate GetById(Guid id)
    {
        // Работа с хранилищем сертификатов
    }
}

#### Open/Closed Principle (OCP)

Система открыта для расширения и закрыта для модификации
за счёт использования абстракций.

public interface ISignatureAlgorithm
{
    bool Verify(byte[] data, byte[] signature);
}

public class RsaSignatureAlgorithm : ISignatureAlgorithm
{
    public bool Verify(byte[] data, byte[] signature)
    {
        return true;
    }
}

#### Liskov Substitution Principle (LSP)

Любая реализация интерфейса ISignatureAlgorithm
может использоваться без изменения логики системы.

public class VerificationService
{
    private readonly ISignatureAlgorithm _algorithm;

    public VerificationService(ISignatureAlgorithm algorithm)
    {
        _algorithm = algorithm;
    }
}

#### Interface Segregation Principle (ISP)

Интерфейсы разделены по назначению,
что предотвращает реализацию лишних методов.

public interface ICertificateReader
{
    Certificate GetById(Guid id);
}

public interface ICertificateValidator
{
    bool IsValid(Certificate certificate);
}

#### Dependency Inversion Principle (DIP)

Высокоуровневые модули не зависят от конкретных реализаций,
а работают с абстракциями.

public class VerificationService
{
    private readonly ICryptoService _cryptoService;

    public VerificationService(ICryptoService cryptoService)
    {
        _cryptoService = cryptoService;
    }
}

## Дополнительные принципы разработки

В рамках лабораторной работы были рассмотрены дополнительные
принципы разработки программного обеспечения.

### BDUF (Big Design Up Front)

От применения принципа BDUF было решено отказаться.
Проектирование выполнялось итеративно,
с фокусом на текущий вариант использования — проверку электронной подписи.

Это позволило избежать избыточной архитектурной сложности
и снизить затраты на разработку.

### SoC (Separation of Concerns)

Принцип разделения ответственности был применён на уровне архитектуры системы.
- Клиентское приложение отвечает за пользовательский интерфейс (реализована в сторонних сервисах).
- Серверная часть реализует бизнес-логику проверки подписи.
- Компонент доступа к данным отвечает за работу с хранилищем.

Такое разделение упрощает сопровождение и расширение системы.

### MVP (Minimum Viable Product)

На ранних этапах разработки система могла рассматриваться как минимально жизнеспособный продукт (MVP),
однако на текущий момент проект вышел за рамки MVP.

Архитектура и функциональность сервиса были расширены с целью
демонстрации принципов проектирования, модульности и масштабируемости.
При этом основной вариант использования — проверка электронной подписи документа —
по-прежнему остаётся центральной функцией системы.

### PoC (Proof of Concept)

Лабораторная работа также выполняет роль Proof of Concept,
подтверждая возможность построения сервиса проверки электронных подписей
с использованием принципов проектирования и модульной архитектуры.

### PoC (Proof of Concept)

В рамках лабораторной работы система рассматривается как Proof of Concept,
демонстрирующий практическую реализуемость сервиса проверки электронных подписей.

Результаты работы подтверждают возможность построения
модульной архитектуры с чётким разделением ответственности компонентов,
а также применимость принципов проектирования
(KISS, DRY, SOLID и др.) для решения поставленной задачи.

Таким образом, лабораторная работа подтверждает корректность выбранных
архитектурных решений и подходов к проектированию системы.
