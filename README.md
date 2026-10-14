# Vulnerability Tracker API

Репозиторий на GitHub: https://github.com/fdsclex/SharapovF.A.Test_Task_C-.git

Автор: Шарапов Ф.А.

REST API на C# (.NET 8, ASP.NET Core) для учёта уязвимостей и запуска асинхронных задач сканирования репозиториев.

## Требования
- .NET SDK 8.0+

## Запуск
```bash
dotnet run --project src/VulnerabilityTracker.Api
```
Сервис стартует на `http://localhost:5080`, Swagger UI доступен по адресу `http://localhost:5080/swagger`.

## Тесты
```bash
dotnet test
```

## Эндпоинты
| Метод | URL | Описание |
|---|---|---|
| POST | `/api/vulnerabilities` | Создать уязвимость. Тело: `{"name":"XSS","severity":"High"}` (`Low`/`Medium`/`High`/`Critical`). Ответ 201 с `id` и `createdAt`. |
| GET | `/api/vulnerabilities/{id}` | Полная информация об уязвимости. |
| POST | `/api/scans` | Запустить сканирование. Тело: `{"repositoryUrl":"https://gitlab.com/group/project"}`. Ответ 202 с `id` задачи. |
| GET | `/api/scans/{id}` | Статус (`Queued`/`Running`/`Completed`/`Failed`/`Cancelled`), `updatedAt`, `result` (если завершена), `error` (если ошибка). |
| POST | `/api/scans/{id}/cancel` | Отмена (в очереди или выполняющейся). Ответ: `id`, `time`, `success`, `status`. `success=false`, если задача уже завершена. |

Ошибки возвращаются в формате `ProblemDetails`: 400 (валидация), 404 (не найдено), 500 (непредвиденная ошибка).

## Пример
```bash
curl -X POST http://localhost:5080/api/vulnerabilities -H "Content-Type: application/json" \
     -d '{"name":"SQL injection","severity":"Critical"}'
curl -X POST http://localhost:5080/api/scans -H "Content-Type: application/json" \
     -d '{"repositoryUrl":"https://gitlab.com/group/project"}'
curl http://localhost:5080/api/scans/<id>
curl -X POST http://localhost:5080/api/scans/<id>/cancel
```

## Конфигурация (`src/VulnerabilityTracker.Api/appsettings.json`)
```json
"Scan": { "MaxConcurrentScans": 2, "SimulatedDurationSeconds": 10 }
```
- `MaxConcurrentScans` — лимит параллельных сканирований; остальные получают статус `Queued` и ждут свободного слота.
- `SimulatedDurationSeconds` — длительность имитации сканирования.

Значения можно переопределить переменными окружения, например `Scan__MaxConcurrentScans=4`.

## Где хранятся данные и логи
- Постоянного хранилища нет: уязвимости и задачи лежат в памяти процесса (`ConcurrentDictionary`) и пропадают при перезапуске. Файлы сервис не создаёт.
- Логи (каждый запрос и ответ с кодом и временем, старт/завершение/отмена сканирований, ошибки) пишутся в консоль через `ILogger`.

## Архитектура
- `Controllers` — HTTP-слой.
- `Services/VulnerabilityService` — потокобезопасное хранилище уязвимостей.
- `Services/ScanService` — управление задачами: фоновый запуск (`Task.Run`), очередь через `SemaphoreSlim`, отмена через `CancellationToken`, защита от гонок (статус меняется только пока задача не в конечном состоянии).
- `Services/IScanner` — абстракция сканера. `SimulatedScanner` имитирует работу (задержка + детерминированные находки); реальный сканер подключается заменой реализации в `Program.cs`.
- `Middleware` — логирование запросов и централизованная обработка ошибок.
- `tests` — unit-тесты сервисов и интеграционные тесты API (xUnit).
