# Домашнее задание: Тестирование сервиса обработки данных

### Цель задания

Написать на Java интеграционный тест, который проверяет полный цикл работы системы: 
от отправки данных в Kafka до проверки сохранения этих данных в базе PostgreSQL и получения ответного сообщения.

### Используемый стек

* Docker / Docker Compose
* Java
* Gradle
* JUnit 5
* Kafka
* PostgreSQL
* JDBI
* Awaitility

-----

### Пошаговая инструкция

#### **Этап 1: Развертывание окружения с помощью Docker**

1.  **Изучите `docker-compose.yml`**: Ознакомьтесь с файлом, который описывает все сервисы системы:
    * `zookeeper` и `kafka`: брокер сообщений.
    * `postgres`: база данных.
    * `app`: приложение, которое связывает всё вместе.
2.  **Запустите окружение**: В терминале, в папке с файлом `docker-compose.yml`, выполните команду:
    ```bash
    docker-compose up -d
    ```
3.  **Проверьте запуск**: Убедитесь, что все контейнеры успешно запустились командой `docker ps`.

#### **Этап 2: Настройка тестового проекта**

1.  **Создайте проект**: Создайте новый проект на Java с использованием системы сборки Gradle.
2.  **Добавьте зависимости**: В файл `build.gradle` добавьте все необходимые зависимости для тестирования:
    * JUnit 5 (`org.junit.jupiter:junit-jupiter`)
    * Клиент Kafka (`org.apache.kafka:kafka-clients`)
    * JDBI для работы с БД (`org.jdbi:jdbi3-core`, `org.jdbi:jdbi3-postgres`)
    * Драйвер PostgreSQL (`org.postgresql:postgresql`)
    * Awaitility для работы с ожиданиями (`org.awaitility:awaitility`)

#### **Этап 3: Написание интеграционного теста**

Создайте тестовый класс который будет выполнять следующие действия:

1.  **Подготовка перед тестом (`@BeforeEach`)**:

    * **Очистка таблицы БД**: Используя JDBI, выполните SQL-команду `TRUNCATE TABLE market_data`, чтобы очистить таблицу от старых данных.

2.  **Реализация тестового метода**:

    * ***Arrange (Подготовка)***: Сформируйте JSON-строку с тестовыми данными.
    * ***Act (Действие)***: С помощью `KafkaProducer` отправьте эту JSON-строку в топик `markets`. В качестве ключа используйте `id` из JSON.
    * ***Assert (Проверка)***:
        1.  **Проверка Базы Данных**:
            * Используя `Awaitility`, подождите несколько секунд.
            * С помощью JDBI подключитесь к PostgreSQL и выполните `SELECT` запрос к таблице `market_data`.
            * Убедитесь, что данные из JSON (в частности, из вложенного списка `selections`) были корректно сохранены в таблицу: проверьте `event_id`, `price`, `probability`, `status` и другие поля.
        2.  **Проверка Выходного Топика**:
            * С помощью `KafkaConsumer` подпишитесь на топик `processed_markets`.
            * Дождитесь и прочтите из него сообщение.
            * Убедитесь, что ключ и значение сообщения соответствуют ожидаемому результату (например, содержат `id` исходного события).
-----

### Критерии выполнения

* Тест успешно выполняется и проходит.
* В коде теста понятно разделены этапы подготовки, действия и проверки.
* Дополнительно можно реализовать ещё несколько тестов с проверкой логики сервиса.
