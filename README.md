# Введение в ETL 

<details>
<summary>Требования</summary>
  
- Базовый уровень Python
- Здравый смысл
- Понимание проектирования DWH - Data Warehouse, инструментов для реализации ETL (чтобы правильно забирать данные или складывать в хранилище)
</details>

<details>
<summary>1. Что такое ETL</summary>

- Это перенос данных из одного или нескольких источников в большое хранилище данных, он нужен не всегда (когда небольшой проект, 1-2 БД с репликами).
- Когда необходимо внедрить ETL - если бизнес состоит из многих частей (АБС, СРМ, ПРМ, Терминалы, ИБ, ПРО, МОБИ,...) и есть связи между ними, несколько БД в разных местах
- Когда не обязательно внедрять ETL - если бизнес состоит из 1-3 небольших частей, записей мало
  
- **E**xtract - извлечение (из CSV, DB Table, API…)
- **T**ransform - преобразование (с помощью Python удаление дубликатов, изменение форматов…)
- **L**oad - загрузка (insert в DWH)

- Порядок действий соответсвует порядку букв в аббревиатуре: 1 - E, 2 - T, 3 - L. Минус такого порядка в том, что при неправильном преобразовании сырых данных приходится заново извлекать эти данные.
- Поэтому в некоторых случаях порядок меняют на ELT - сначала извлекают сырые данные, потом загружают их в хранилище и в конце преобразовывают в нужный формат. При таком подходе, если будут ошибки в преобразовании, то сырые данные не надо заново извлекать, достаточно обращаться в хранилище, что экономит время и ресурсы.
- Когда горят "ETL", то обычно имеют ввиду общий процесс, то есть это может быть либо ETL, либо ELT, но когда говорят "ELT", то точно имеют ввиду ELT
</details>

<details>
<summary>2. Как правильно готовить ETL</summary>
  
    1. Принципы построения ETL
        1. Простой и чистый код
        2. Единообразие пайплайна (пайплайн - этапы работы с данными)
        3. Время выполнения пайплайна (если долго, то что-то не так)
        4. Меньше сетевого трафика (экономия ресурсов)
        5. Работа с репликой (чтобы не ломать мастер БД)
        6. Оптимизация забора (запроса) данных
        7. Партицирование
        8. Инкрементальный пересчет витрин (снепшоты, не обязательно каждый раз пересчитывать данные с самого начала)
        9. Загрузка всего без ограничений (сырые данные из источников)
        10. Избавляться от неактуального (аудит пайплайнов - оставлять только нужные)
        11. Идемпотентность (лучше использовать merge, чем insert)
        12. Аудиторский след (сырые данные хранить в DWH, чтобы в случае ошибки заново на месте пересчитать (типа ELT))
    2. Будьте готовы
        1. Отсутствие целостности (данные в источниках не всегда идеальны, мелкие несоответствия будут)
        2. Сетевые проблемы (идемпотентность должно решать эту проблему)
        3. Незапланированные изменения (в БД или АПИ, когда разработчики проектов не сообщают дата-инженеру об изменениях) 
        4. Пайплайны будут задерживаться (акции продукта, заполнение памяти, ...), необходимо контролировать важные пайплайны
        5. Данные из разных системах противоречивы (для одной записи одна система хранит - дни, другая - сумму, другая - сумму фрода)
</details>

<details>
<summary>3. Обзор планировщиков (scheduler - запускает в нужный момент задачу)</summary>
  
    1. CRON
        1. «+» Максимально простой, «-» максимально простой
    2. Jenkins/gitlab CI
        1.  Предназначено больше для. CI/CD
    3. Написать свой (google, yandex,...)
    4. Платные - дорогие, нет доступа к коду, есть поддержка, визуальный редактор
    5. Опен сорс - бесплатно, можно посмотреть код, можно контрибютить, риск ошибок в коде (Apache Oozie, NiFi, Luigi, Airflow (Python); Talend (Java)) 
</details>

<details>
<summary>4. Почему Airflow</summary>
  
    1. Open source
    2. Отличная документация
    3. Простий код на Python 
    4. Удобный UI
    5. Алертинг и мониторинг
    6. Интеграция с основными источниками
    7. Кастомизация
    8. Масштабирование (Докер, кластеры)
    9. Большое комьюнити
</details>

# Знакомство с Airflow

<details>
<summary>1. История Airflow</summary>

    1. Октябрь 2014 - создание Airflow в Airbnb (Open source)
    2. Март 2016 - передали в Apache Incubator 
    3. Январь 2019 - top-level проект 
    4. Конец 2020 - Airflow 2.0
</details>
<details>
<summary>2. Основные понятия</summary>
  
    1. DAG (Directed Acyclic Graph) - однонаправленный ацикличный (без циклов) граф, то есть всегда будет один конечный результат
        1. Каждая вершина - одна задача (Task)
        2. Рёбра - зависимости между Task-ами
        3. Task
            Виды:
            1. Сущность Operator - выполняет конкретную задачу
            2. Сущность Sensor (специальный тип Operator-а) - дожидается выполнения события
            3. Сначала запускается Task, не имеющий предшественников, после его отработки выполняются те Task-и, которые зависят от предыдущего, до тех пор пока не доходят до последнего
            4. Task-и объединяются в DAG по смыслу (Task1 - ждём появление записи, Task2 - забираем к себе, Task3 - преобразовываем, Task4 - отправляем уведомление о выполнении DAG-a)
            5. DAG-ов может быть очень много
            6. Task-и время от времени  падают (по какой-то причине), после падения Task переходит в состояние «RETRY», перезапускается (по умолчанию 3 раза). После 3-ей безуспешной попытки переходит в состояние «FAILED», а последующие за ним Task-и в состояние «UPSTREAM-FAILED», потом сам DAG переходит в состояние «FAILED», об этом получаем уведомление или видим в UI
        4. После объявления DAG-а можем поставить его на расписание (под капотом Airflow работает CRON), можем использовать alias-ы для указывание времени типа @none, @once, @daily
</details>
<details>
<summary>3. Компоненты Airflow</summary>
  
    1. Webserver (Страница Airflow)
        1. Показывает внешний вид DAG-ов (берёт данные из DAG Directory)
        2. Показывает статусы выполнения DAG-ов (берёт данные из Metadata)
        3. Есть кнопки перезапуска, отладки
        
    2. Scheduler (Планировщик)
        1. По умолчанию 1 раз в минуту анализирует DAG-и (DAG Directory)
        2. Создаёт DAG Run (экземпляр DAG-а) в момент когда должен запуститься DAG (DAG Run имеет параметром «execution_date» - начала предыдущего периода (если запуск 15 сентября, то значение будет 14-ое))
        3. Создаёт Task Instance - каждый Task генерируется в отдельный Task Instance и этот instance привязывается к DAG-у, для них тоже прокидывается «execution_date»
        4. Ставит Task-и в очередь
        5. Для выполнения активных Task-ов планировщик (scheduler) использует указанный у нас в настройках «executor»
    3. Executor (Исполнитель Task Instance-а)
        1. Механизм с помощью которого запускаются Task Instance-ы
        2. Работает в одной связке с планировщиком, то есть когда запускаете процесс планировщика, executor запускается в том же самом процессе
        3. Категории 
            1. Локальные (исполняются на той же машине, на котором есть Scheduler)
                1. SequentialExecutor - последовательно запускает задачи и на время их выполнения приостанавливает планировщик, другие задачи не ставятся в очередь, что неудобно (по умолчанию Airflow подсказывает заменить его на хотя бы LocalExecutor)
                2. LocalExecutor - на каждую задачу запускает отдельный процесс, позволяет параллельно запускать столько задач, сколько позволяет генерировать машина. Тоже не рекомендуется на проде, так как низкоустойчив - если машина остановится, то и планировщик остановится и в конце Airflow остановится 
                3. DebugExecutor - нужен только для того, чтобы запускать DAG-и из среды разработки
            2. Нелокальные (могут запускать таски удаленно, Scheduler на другой машине)
                1. CeletyExecutor
                    1. Может иметь несколько Worker-ов на разных машинах, требует дополнительные настройки брокер-сообщений (Redis, RabbitMQ)
                    2. Позволяет масштабировать Airfow подключением нового Worker-а
                    3. При подключении нового Worker-а часть задач переходят к нему, если с одним Worker-ом что-то пошло не так, то эта задача переадресует на другие работающие Worker-ы
                2. DaskExecutor (делает тоже самое что и CeletyExecutor только билиотекой Dask)
                3. KubernetesExecutor - на каждый Task Instance запускает новый Worker на отдельном pod-e в k8. «+» Появляется динамическое распределение ресурсов, «-» - необходимо уметь поднять и настроить k8
                4. CeleryKubernetesExecutor - одноввременно держит 2 executor-a и в зависимости от Task-а (а именно, параметра queue в Task-e) выполняется либо 1-ым, либо 2-ым executor-ом
                5. Custom
    4. Worker (Обработчик задач)
            1. Процесс, в котором исполняются задачи
            2. В зависимости от executor-а может быть запущен локально на той же машине что и scheduler или на другой машине
    5. METADATA DATABASE (Информация о состоянии всех пайплайнов)
            1. DAG (Инфо об абстрактном DAG-е)
            2. DAG Run (Инфо о конкретных запусках DAG-a - DAG Run-ов)
            3. Task Instance (Инфо когда запустился, как завершился, сколько попыток,…)
            4. Variable (Глобальные переменные)
            5. Connection (Связи с БД, API, ...)
            6. XCom
            7. ….
</details>
