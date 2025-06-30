### Описание сервиса для тестировщиков

Этот документ описывает, как сервис получает, обрабатывает, сохраняет и отправляет данные. Эта информация поможет в написании тестов для сервиса.

#### Получение данных

Сервис получает данные из топика Kafka с именем `markets`. Сообщения должны быть в формате JSON и соответствовать одной из двух структур: `MarketEvent` или `MarketReport`.

За прием сообщений отвечает `MarketKafkaConsumerService`. Ключ каждого сообщения должен быть числом в формате `Long`.

**Типы входных сообщений:**

* **MarketEvent**: Представляет событие на рынке. Включает в себя статус и список рынков с исходами, ценами и вероятностями.
* **MarketReport**: Представляет отчет по рынку. Включает в себя список рынков без коэффициентов.

В случае ошибки десериализации входящего сообщения, сервис отправит `ErrorMessage` в выходной топик.

#### Хранение данных

После получения, данные обрабатываются сервисом `MarketProcessingService` и сохраняются в базу данных PostgreSQL.

Данные сохраняются в таблицу `market_data`, которая содержит следующую информацию:

* `id`: Уникальный идентификатор записи.
* `event_id`: Идентификатор события.
* `market_type_id`: Идентификатор типа рынка.
* `selection_type_id`: Идентификатор типа исхода.
* `price`: Цена.
* `probability`: Вероятность.
* `status`: Статус.

Взаимодействие с базой данных осуществляется через `MarketDataRepository`.

#### Отправка данных

После обработки и сохранения данных, сервис отправляет результирующее сообщение в топик Kafka с именем `processed_markets`.

За отправку сообщений отвечает `MarketKafkaProducerService`. Сообщения сериализуются в формат JSON.

**Типы исходящих сообщений:**

* **ProcessedMarkets**: Сообщение с уникальными идентификаторами типов рынков и исходов после обработки `MarketEvent`.
* **ProcessedReportMarkets**: Сообщение со списками обработанных идентификаторов рынков и исходов после обработки `MarketReport`.
* **ErrorMessage**: Сообщение об ошибке, например, при ошибке десериализации.

-----

### Примеры JSON и логика обработки

#### 1\. Обработка `MarketEvent`

**Пример JSON в топике `markets`:**

```json
{
  "id": 12345,
  "message_type": "market_event",
  "status": "active",
  "markets": [
    {
      "market_type_id": 1,
      "specifiers": [],
      "selections": [
        {
          "selection_type_id": 101,
          "status": 0,
          "odds": {
            "price": 1.85,
            "probability": 0.51
          }
        },
        {
          "selection_type_id": 102,
          "status": 4,
          "odds": {
            "price": 2.10,
            "probability": 0.45
          }
        }
      ]
    }
  ]
}
```

**Логика обработки (`processMarketEvent`):**

1.  **Валидация**: Сервис проверяет, что `key` и `event` не равны `null`, и что `key` совпадает с `event.getId()`.
2.  **Итерация и сохранение**:
    * Сервис проходит по каждому рынку (`market`) и каждому исходу (`selection`) в сообщении.
    * Сохраняются уникальные `marketTypeId` и `selectionTypeId`.
    * Статус исхода (`selection.status`) конвертируется из числового формата в строковый (`SelectionsStatus`). Числовое представление равно индексу в массиве `[active, suspended, disabled, win, loss, return, half_win, half_loss, cancelled]`. Например, `0` станет `"active"`, а `4` — `"loss"`.
    * Если статус является финальным `[disabled, win, loss, return, half_win, half_loss, cancelled]`, то `price` и `probability` устанавливаются в `null`.
    * Для каждого исхода создается и сохраняется объект `MarketData` в базу данных.
3.  **Отправка результата**:
    * После обработки всех исходов сервис создает сообщение `ProcessedMarkets`, содержащее `id` события и наборы уникальных `market_type_id` и `selection_type_id`.
    * Это сообщение отправляется в топик `processed_markets`.

**Пример JSON в топике `processed_markets`:**

```json
{
  "id": 12345,
  "is_success": true,
  "unique_markets_ids": [1],
  "unique_selection_ids": [101, 102]
}
```

#### 2\. Обработка `MarketReport`

**Пример JSON в топике `markets`:**

```json
{
  "id": 67890,
  "message_type": "market_report",
  "markets": [
    {
      "market_type_id": 2,
      "selections": [
        {
          "selection_type_id": 201,
          "status": "active"
        },
        {
          "selection_type_id": 202,
          "status": "suspended"
        }
      ]
    }
  ]
}
```

**Логика обработки (`processMarketReport`):**

1.  **Валидация**: Аналогично `MarketEvent`, сервис проверяет `key` и `report`.
2.  **Итерация, вычисление и сохранение**:
    * Сервис проходит по каждому рынку и исходу в сообщении.
    * **Логика вычисления `price` и `probability`**:
        * Если `selectionTypeId` — четное число, `price` вычисляется как `1.5 + selectionTypeId`, а `probability` — как `0.445 + (selectionTypeId / 10.0)`.
        * Если `selectionTypeId` — нечетное число, `price` вычисляется как `2.5 + selectionTypeId`, а `probability` — как `0.555 + (selectionTypeId / 10.0)`.
    * Статус исхода (`selection.status`) конвертируется в `SelectionsStatus`.
    * Для каждого исхода создается и сохраняется объект `MarketData` в базу данных.
3.  **Отправка результата**:
    * Сервис создает сообщение `ProcessedReportMarkets`, содержащее `id` отчета и списки всех обработанных `market_type_id` и `selection_type_id`.
    * Это сообщение отправляется в топик `processed_markets`.

**Пример JSON в топике `processed_markets`:**

```json
{
  "id": 67890,
  "is_success": true,
  "processed_markets_ids": [2],
  "processed_selections_ids": [201, 202]
}
```

#### 3\. Обработка ошибок

В случае, если входящее JSON-сообщение в топике `markets` не может быть десериализовано (например, из-за неверного формата), обработчик ошибок отправит следующее сообщение в `processed_markets`.

**Пример JSON в топике `processed_markets` (сообщение об ошибке):**

```json
{
  "id": -922337203685477580,
  "is_success": false,
  "error_description": "Deserialization error. Received message with wrong format."
}
```
