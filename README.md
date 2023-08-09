ELK (Elasticsearch + Logstash + Kibana)
 

ELK — это аббревиатура из названий трех продуктов: Elasticsearch, Logstash и Kibana.

Elasticsearch (ES) - масштабируемая утилита полнотекстового поиска и аналитики, которая позволяет быстро в режиме реального времени хранить, искать и анализировать большие объемы данных. Он используется в качестве NoSQL-базы данных для приложений со сложными функциями поиска. 

Logstash -  это инструмент получения, преобразования и сохранения данных в общем хранилище. Его первой задачей является прием данных в каком-либо виде: из файла, базы данных, логов и т.д. Далее полученная информация может модифицироваться с помощью фильтров.

Kibana - визуальный (UI) инструмент для Elasticsearch, чтобы взаимодействовать с данными, которые хранятся в индексах ES. Веб-интерфейс Kibana позволяет быстро создавать и обмениваться динамическими панелями мониторинга, включая таблицы, графики и диаграммы и т.д.
 
Как это все вместе работает ?
![image](https://github.com/Nekipel/ELK/assets/88710417/a6e3e55b-e450-4570-98ea-9b40f4f30184)
Logstash представляет собой конвейер обработки данных на стороне сервера, который одновременно получает данные из нескольких источников. Здесь выполняется первичное преобразование, фильтрация, агрегация или парсинг логов, а затем обработанные данные отправляется в Elasticsearch.

Elasticsearch играет роль ядра всей системы, сочетая функции базы данных, поискового и аналитического движков. Наличие REST API позволяет добавлять, просматривать, модифицировать и удалять данные.

Kibana позволяет визуализировать данные Elasticsearch, а также администрировать базу данных.

-"Слева в меню (самая последняя) — Management — там выбираем Index Patterns — и там Create index pattern." - нет там такой ссылки. Грустно из-за такого неуважения к времени обучающихся. Материал только выложили, а он уже "протухший".

-ELK из России не скачаешь. Но можно через VPN. Например, через UrbanVPN из "Турции".

-в image docker-compose.yml сократить до elasticsearch:7.10.1(доступная версия), аналогично и для logstash и kibana, иначе 403 от амазона



Цель: логирование с помощью ELK (Elasticsearch + Logstash + Kibana)

Задача: установи по очереди оба инструмента. Нужна только установка без каких-либо настроек, указанных в некоторых из этих ссылок.

elasticsearch

https://www.elastic.co/downloads/elasticsearch

logstash

https://www.elastic.co/downloads/logstash

kibana

https://www.elastic.co/downloads/kibana

В /etc/logstash/ создай файл logstash.conf со следующим содержимым.


input {
  file {
    type => "java"
    # Logstash insists on absolute paths...
    path => "/home/myName/logs/application.log"
    codec => multiline {
      pattern => "^%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME}.*"
      negate => "true"
      what => "previous"
    }
  }
}
filter {
  #If log line contains tab character followed by 'at' then we will tag that entry as stacktrace
  if [message] =~ "\tat" {
    grok {
      match => ["message", "^(\tat)"]
      add_tag => ["stacktrace"]
    }
  }
  #Grokking Spring Boot's default log format
  grok {
    match => [ "message",
               "(?<timestamp>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME})  %{LOGLEVEL:level} %{NUMBER:pid} --- \[(?<thread>[A-Za-z0-9-]+)\] [A-Za-z0-9.]*\.(?<class>[A-Za-z0-9#_]+)\s*:\s+(?<logmessage>.*)",
               "message",
               "(?<timestamp>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME})  %{LOGLEVEL:level} %{NUMBER:pid} --- .+? :\s+(?<logmessage>.*)"
             ]
  }
  #Parsing out timestamps which are in timestamp field thanks to previous grok section
  date {
    match => [ "timestamp" , "yyyy-MM-dd HH:mm:ss.SSS" ]
  }
}
output {
    stdout {
        codec => rubydebug
    }
    elasticsearch{
        hosts=>["localhost:9200"]
        index=>"todo-logstash-%{+YYYY.MM.dd}"
    }
}

Здесь /home/myName — это путь на ПК в корневую директорию. Напомню, что на всех примерах я использую macOS, в Linux всё аналогично.

Индекс называется todo-logstash. Аналогичный файл с таким же названием создай в корне проекта в IDEA. Только укажи свой путь к сохраняемому файлу с логами.

У меня это path => "/home/myName/logs/application.log"

В коде в IDEA мы должны указать логгер и залогировать то, что нам нужно.

Например, в контроллере мы залогируем стартовую страницу.

private static final Logger LOG = Logger.getLogger(MyTestController.class.getName());
@RequestMapping("/")
public String home() {
   String home = "Client-Service running at port: " + env.getProperty("local.server.port");
   LOG.log(Level.INFO, home);

   return home;
}
Далее запускаем:

elasticsearch

В консоли из-под рута

sudo service elasticsearch start

2. logstash

В консоли переходим в папку

cd /usr/share/logstash

и там в консоли пишем

bin/logstash --verbose -f /etc/logstash/logstash.conf

то есть запускаем logstash с нашими новыми настройками, которые мы создали (logstash.conf)

3. kibana

В новой консоли

service kibana start

Идем в браузере на

http://localhost:9200/

и видим, что elasticsearch работает.

Здесь

http://localhost:9200/_cat/indices

видим все наши индексы.

Запускаем приложение в IDEA, здесь мы увидим новый индекс todo-logstash

http://localhost:9200/_cat/indices

Также можем протестировать:

в консоли пишем

curl -XGET http://127.0.0.1:9200

если получим ответ наподобие


{
 
   "name" : "ks-pc",
 
   "cluster_name" : "elasticsearch",
 
   "cluster_uuid" : "dZ2ldEdWSRGHwdMHydMslQ",
 
   "version" : {
 
   "number" : "7.3.1",
 
   	"build_flavor" : "default",
 
   	"build_type" : "deb",
 
   	"build_hash" : "4749ba6",
 
   	"build_date" : "2019-08-19T20:19:25.651794Z",
 
   	"build_snapshot" : false,
 
   	"lucene_version" : "8.1.0",
 
   	"minimum_wire_compatibility_version" : "6.8.0",
 
   	"minimum_index_compatibility_version" : "6.0.0-beta1"
 
},
 
   "tagline" : "You Know, for Search"
 
}

значит, все хорошо. Идем дальше.

Теперь идем в kibana

http://localhost:5601

Слева в меню (последний пункт — Management, в нем выбираем Index Patterns,  а затем — Create index pattern.

Пишем имя нашего индекса

todo-logstash

. Жмем Next step — выбираем в списке timestamp — Ok.

Отлично, мы создали индекс.

Идем слева в меню в Discover (верхний пункт в меню слева), в этом пункте слева в окошке выбираем созданный индекс.
Отправляем запрос любым удобным способом (браузер, Postman, консоль) и нажимаем на кнопку Refresh.

Мы увидим все наши запросы. Они будут попадать сюда посредством логера, который мы указали в классах в коде.

Ничего сложного!



https://www.youtube.com/watch?v=5s9pR9UUtAU


user avatar
Данил Белоусов
09.05.2023 02:17
Упростим код, написав все эти файлы и конфиги прямо в spring boot приложении.
Добавить зависимости в файл pom.xml проекта:
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-to-slf4j</artifactId>
    <version>2.15.0</version>
    <scope>runtime</scope>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.6</version>
</dependency>

Настроить Logstash для получения логов из Spring Boot приложения. Например, можно создать конфигурационный файл logstash.conf следующего содержания:
javascriptCopy codeinput {
  tcp {
    port => 5000
    codec => json_lines
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "spring-boot-logs"
  }
}

Этот же файл с таким же названием ( logstash.conf) кладем в разархивированную папку Logstash.
Данный файл настраивает Logstash на прослушивание TCP порта 5000 и ожидание JSON-логов. Полученные логи будут отправлены в Elasticsearch в индекс с именем spring-boot-logs.
В Spring Boot приложении настроить логирование. Например, можно добавить в файл application.properties следующие настройки:
cCopy codelogging.level.root=INFO
logging.level.com.example=DEBUG
logging.file.name=myapp.log
Эти настройки задают уровень логирования для корневого пакета приложения (INFO) и для пакета com.example (DEBUG), а также задают имя файла для логов myapp.log.
Настроить Logback, чтобы он использовал Logstash для отправки логов. Для этого нужно создать файл logback-spring.xml со следующим содержанием:
phpCopy code<configuration>
  <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>localhost:5000</destination>
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <pattern>
          <pattern>
            {
              "timestamp": "%d{ISO8601}",
              "level": "%level",
              "logger": "%logger",
              "message": "%message"
            }
          </pattern>
        </pattern>
      </providers>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="LOGSTASH" />
  </root>
</configuration>

Этот файл настраивает Logback на отправку логов в Logstash по TCP на localhost:5000. Формат логов задается JSON-шаблоном.
Можно добавить обычный лог прямо в главный класс logger.info("Hello, world!");
Запустить Logstash командой bin\logstash -f ВАШ_ПУТЬ\logstash.conf внеся абсолютный путь до файла, который внутри папки logstash

Запустить Elasticsearch.
Запустить Spring Boot приложение.
Открыть Kibana и создать индекс spring-boot-logs.
Просмотреть логи в Kibana.
ВАЖНО. я использовал всю эту троицу под версией 7.13.4 т.к в новых версиях по умолчанию включена аутентификация и SSL соединение, которое долго настраивать или отключать
