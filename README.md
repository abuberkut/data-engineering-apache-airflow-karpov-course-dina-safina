# Введение в ETL 

<details>
<summary>Требования</summary>
  
- Базовый уровень Python
- Здравый смысл
- Понимание проектирования DWH - Data Warehouse, инструментов для реализации ETL (чтобы правильно забирать данные или складывать в хранилище)
</details>

<details>
<summary>1. Что такое ETL</summary>

- Это перенос данных из одного или нескольких источников в большое хранилище данных
- Когда необходимо внедрить ETL? Если бизнес состоит из многих частей (АБС, СРМ, ПРМ, Терминалы, ИБ, ПРО, МОБИ,...) и есть связи между ними и их БД в разных местах
- Когда не обязательно внедрять ETL? Если бизнес состоит из 1-3 небольших частей, записей мало
  
Расшифровка аббревиатуры ETL:
- **E**xtract - извлечение (из CSV, DB Table, API…)
- **T**ransform - преобразование (с помощью Python удаление дубликатов, изменение форматов…)
- **L**oad - загрузка (insert в DWH)

- Порядок действий соответсвует порядку букв в аббревиатуре: 1 - E, 2 - T, 3 - L. Минус такого порядка в том, что при неправильном преобразовании сырых данных приходится заново извлекать эти данные.
- Поэтому в некоторых случаях порядок ETL меняют на ELT - сначала извлекают сырые данные, потом загружают их в хранилище и в конце преобразовывают в нужный формат. При таком подходе, если будут ошибки в преобразовании, то сырые данные не надо заново извлекать, достаточно обращаться в хранилище, что экономит время и ресурсы.
- Когда говорят "ETL", то имеют ввиду либо ETL, либо ELT, когда говорят "ELT", то точно имеют ввиду ELT
</details>

<details>
<summary>2. Как правильно готовить ETL</summary>
  
    1. Принципы построения ETL
        1. Простой и чистый код
        2. Единообразные пайплайны (пайплайн - этапы работы с данными, забор, загрузка, преобразование)
        3. Время выполнения пайплайна (если долго, то что-то не так)
        4. Меньше сетевого трафика (экономия ресурсов)
        5. Работа с репликой (чтобы не нагружать основной БД)
        6. Оптимизация забора (запроса) данных
        7. Партицирование
        8. Инкрементальный пересчет витрин (снепшоты, не обязательно каждый раз пересчитывать данные с самого начала)
        9. Загрузка всего без ограничений (сырые данные из источников)
        10. Избавляться от неактуального (аудит пайплайнов - оставлять только нужные)
        11. Идемпотентность
        12. Аудиторский след (сырые данные хранить в DWH, чтобы в случае ошибки заново на месте пересчитать (ELT))
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
    3. Простой код на Python 
    4. Удобный UI
    5. Алертинг и мониторинг
    6. Интеграция с основными источниками
    7. Кастомизация
    8. Масштабирование (докер, кластеры)
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
            1. Сущность Operator - выполняет конкретную задачу
            2. Сущность Sensor (вид Task-a, специальный тип Operator-а) - дожидается выполнения события
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
                1. CeleryExecutor
                    1. Может иметь несколько Worker-ов на разных машинах, требует дополнительные настройки брокер-сообщений (Redis, RabbitMQ)
                    2. Позволяет масштабировать Airfow подключением нового Worker-а
                    3. При подключении нового Worker-а часть задач переходят к нему, если с одним Worker-ом что-то пошло не так, то эта задача переадресует на другие работающие Worker-ы
                2. DaskExecutor (делает тоже самое что и CeleryExecutor только библиотекой Dask)
                3. KubernetesExecutor - на каждый Task Instance запускает новый Worker на отдельном pod-e в k8. «+» Появляется динамическое распределение ресурсов, «-» - необходимо уметь поднять и настроить k8
                4. CeleryKubernetesExecutor - одновременно держит 2 executor-a и, в зависимости от Task-а (а именно, параметра queue в Task-e), выполняется либо 1-ым, либо 2-ым executor-ом
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

# Создаём простой DAG

<details>
<summary>Конвенция написания кода DAG-а</summary>
  
```
1. Создаём питоновский файл dag_name.py
2. Составление кода DAG-а в dag_name.py:
    1. "Шапка описание" - комментарии про то что делает DAG
    2. Импорт необходимых библиотек
		from airflow import DAG
		from airflow.utils.dates import days_ago
		import logging

		from airflow.operators.dummy_operator import DummyOperator
		from airflow.operators.bash import BashOperator
		from airflow.operators.python_operator import PythonOperator

    3. Тело кода DAG-а  
      
      DEFAULT_ARGS = {
        ’start_date’: days_ago(2), # 2 instance-а    
        ‘owner’: ‘abubakr’,    
        ‘poke_interval’: 600 
      }

      with DAG(    
        ‘dag_name’,    
        schedule_interval=‘@daily’,      
        default_args=DEFAULT_ARGS,    
        max_active_runs=1, # 1 Task Instance может быть в активном (running) состоянии     
        tags=[‘dag_tag1’, ‘dag_tag2’] 
      ) as dag:      
        dummy = DummyOperator(task_id=‘dummy’)      
        echo_ds = BashOperator(        
          task_id=‘echo_ds’,        
          bash_command=‘echo {{ ds }}’        
          dag=dag     
        )      
        
        def hello_world_func():         
          logging.info(‘Hello world’)      
        hello_world = PythonOperator(        
          taks_id=‘hello_world’,        
          python_callable=hello_world_func,        
          dag=dag     
        )      
        dummy >> [echo_ds, hello_world]


Документацию DAG-a можно добавить как: 

dag.doc_md = __doc__
dag_name.doc_md

```
</details>


# Сложные пайплайны

<details>
<summary>1. Создание DAG-a</summary>

    1. Способы создания  DAG-a:
        1. Создание переменной класса DAG (dag_name=DAG(…)). Каждый созданный Task надо привязывать к созданному DAG-у (внутри Task-a в параметр dag присваивать переменную DAG: dag=dag_name)  	
	dag_name = DAG(   
		"owner_name",    
		schedule_interval='@daily',    
  		default_args=DEFAULT_ARGS,    
    		max_active_runs=1,    
      		tags=['tag1', 'tag2'] 
	)
	
	wait_until_6am = TimeFeltaSensor(    
 		task_id='wait_until6am',    
   		delta=timedelta(seconds=6*60*60), # 6 часов    
     		dag=dag_name 
     	) 

      
        2. Создание переменной класса DAG через контекстный менеджер (with DAG(…)). DAG автоматически назначается Task-ам внутри контекста (не надо привязывать каждый Task отдельно как в пункте 1.1.1)  
	with DAG(    
 		dag_id='some_id',    
   		schedule_interval='@daily',    
     		default_args=DEFAULT_ARGS,    
       		max_active_runs=1,    
	 	tags=['tag1', 'tag2']    
   	) as dag_name:         
    		wait_until_6am = TimeFeltaSensor(       
      			task_id='wait_until6am',       
	 		delta=timedelta(seconds=6*60*60), # 6 часов    
    		) 
      
        3. Создание DAG-a с помощью декоратора, набрасываем функцию со списком Task-ов внутри, оборачиваем его в декоратор и таким образом получаем переменную класса DAG,  переменную присваиваем глобальной области видимости  (необходимо знать декораторы в Python)  
	
 	@dag(    
  		start_date=days_ago(2),    
    		dag_id='some_id',    
      		schedule_interval='@daily',   
		default_args=DEFAULT_ARGS,    
  		max_active_runs=1,    
    		tags=['tag1', 'tag2']    
    	)  
     	
      	def generate_dag():     
       		wait_until_6am = TimeFeltaSensor(        
	 		task_id='wait_until6am',        
    			delta=timedelta(seconds=6*60*60), # 6 часов     
       		)  
	 
  	dag = generate_dag()  
   
    2. default_args = {    
    	'owner': 'owner_name',    
     	'queue': 'queue_name', # очередь, в которую становится Task    
      	'pool': 'user_pool',    
       	'email': ['name@example.com'],    
	'email_on_failure': False,    
 	'email_on_retry': False,    
  	'depends_on_past': False, # Task в данной DAG Instance будет запущен только в тот момент, когда этот же Task в предыдущем (за предыдущий период) DAG Instanc-e уже был отработан    
   	'wait_for_downstream': False, # Task ждёт окончание работы всех Task-ов, зависящих от этого     
    	'retries': 3,    
     	'retry_delay': timedelta(minutes=5),    
      	'priority_weight': 10,    
       	'start_date': detetime(2024, 1, 1),    
	'end_date': detetime(2026, 1, 1),    
 	'sla': timedelta(hours=2),    
  	'execution_timeout': timedelta(seconds=300),    
   	'on_failure_callback': some_function,    
    	'on_success_callback': some_other_function,    
     	'on_retry_callback': another_function,    
      	'sla_miss_callback': yet_another_function,    
       	'trigger_rule': 'all_success', 
	}

</details>

<details>
<summary>2. Trigger rule</summary>

    В каком  состоянии должны быть предыдущие Task-и, чтобы Task который от них зависит сработал, по умолчанию all_success
    1. all_success
    2. all_failed
    3. all_done (все предыдущие Task-и должны перейти в одно из этих состояний: SUCCESS, SKIPPED, FAILED, UPSTREAM_FAILED)
    4. one_failed (хотя бы один из предыдущих Task-ов перейдёт в состояние FAILED)
    5. one_success (хотя бы один из предыдущих Task-ов перейдёт в состояние SUCCESS)
    6. none_failed
    7. none_failed_or_skepped
    8. none_skipped
    9. dummy (в любом случае должен сработать)
</details>
