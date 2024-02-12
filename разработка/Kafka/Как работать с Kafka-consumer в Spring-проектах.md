# Как работать с Kafka-consumer в Spring-проектах

Эта статья сборник небольших рекомендаций как работать с Kafka в Spring основанных на личном опыте.
Для тех, кто только начинаете работать с Kafka, рекомендую предварительно ознакомится с другими обзорами и потом возвращаться сюда - [Getting start](https://docs.spring.io/spring-kafka/reference/quick-tour.html)

## Что предлагает Spring из коробки

В Spring 2 способа по интегрировать kafka-consumer в код:
1. Через аннотацию [@KafkaListener](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html)
2. Реализовав интерфейс [MessageListener](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/message-listeners.html) или один из его наследников - в статье этот способ рассматриваться не будет

Типичный код обработки вымышленного топика выглядит так:

```kotlin
@Service
class UserLikeActionKafkaListener {

    @KafkaListener(
        topics = ["user.action.like"]
    )
    fun onUserLikeAction(action: UserActionDto) {

    }
}
```

Spring-стартер предлагает работать с Kafka по такой схеме:
1. Реализован глобальный конфиг, который распространяется на все консьюмеры/контейнеры
2. Фабрики `KafkaListenerContainerFactory`/`ConsumerFactory` общие для всех обработчиков
3. Для переопределения настроек из пункта 1 в конкретном обработчике используются параметры аннотации `@KafkaListener`

## Что не так с этой схемой

В продакшн-коде я сталкивался со следующими ситуациями:

1. Одно приложение работает с несколькими Kafka-кластерами
2. У топиков разнородный формат сообщений - xml/json/avro или внутри одного формата отличается стиль написания ключей внутри json(CamelCase vs SnakeCase)
3. В топики иногда попадают не валидные сообщения, которые нельзя обработать(ошибки при десериализации)
4. Временное отключение обработки топика
5. Изменение offset для топика в консьюмер-группе
6. Переопределение настроек только для обработчика одного топика
7. Необходим минимальный мониторинг и поиск проблем

На основе этих ситуаций формализую требования для Kafka-консьюмеров:

1. Приложение поддерживает работу с несколькими кластерами
2. Обработчики топиков автономны и изолированы. У каждого топика свои правила десериализации
3. Отключение обработчика топика и изменение offset без полной остановки приложения
4. Минимальный мониторинг обработки сообщений и локализация проблем

## Разбор требований

### Поддержка нескольких кластеров в приложении

Из коробки starter не умеет работать с несколькими кластерами сразу.
Можно попрбовать переиспользовать класс настроек `KafkaProperties` для описания других кластеров, но мне такой вариант не нравится по 2 причинам:
1. В этом классе часть настроек Kafka выделена в отдельные поля. Для чего это было сделано непонятно, потому что потом все собирается в обычную мапу
2. Иногда одно приложение обрабатывает несколько топиков из кластера, а если кластеров будет больше одного, то получится дублирование настроек

Настройки kafka-кластера сводятся к `Map<String, Object>`.
Чтобы добавить поддержку нескольких кластеров в приложение сделаем такой конфиг:

```kotlin
@ConfigurationProperties("kafka")
data class KafkaClusterProperties(
    val clusters: Map<String, KafkaCluster>
)

class KafkaCluster {
    lateinit var bootstrapServers: List<String>
    lateinit var properties: Map<String, String>
}
```

Пример заполнения(настройками кластера будем считать общие настройки, которые походят для consumer/producer одновременно)

```yaml
kafka:
  clusters:
    prod:
      bootstrap-servers: ${embedded.kafka.brokerList}
      properties:
        client.id: my-application
        #security.protocol: SASL_PLAINTEXT
        #sasl.mechanism: PLAIN
        #sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="${embedded.kafka.saslPlaintext.user}" password="${embedded.kafka.saslPlaintext.password}";
        # Все настройки с префиксом - ssl.*
```

### Автономность и изоляция обработчиков топиков

Spring по умолчанию создает общие фабрики для всех контейнеров `KafkaListenerContainerFactory`/`ConsumerFactory`.
Какие это вызывает проблемы:
1. В топиках разнородные форматы сообщений и ключей(начиная с версии 2.8 стал доступен `DelegatingByTopicDeserializer`)
2. Для топиков иногда нужны специфичные интерцепторы, которые зависят от модели сообщения
3. При измение настроек бина специфичных для одного топика можно поломать другие обработчики
4. У топиков разное кол-во партиций. Для параллельной обработки для каждой партиции создается отдельный контейнер.

Один контейнер способен читать из нескольких партиций, но это будет происходить последовательно, так как контейнер по очереди будет переключаться на другие партиции. В Kafka количество партиций в топике - это единица параллельности, поэтому сразу Spring для ускорения обработки на каждую партицию создает отдельный контейнер(кол-во создаваемых контейнеров зависит от параметра concurrency и не может быть больше кол-ва партиций)

Решение этих проблем - это создание отдельных фабрик `KafkaListenerContainerFactory`/`ConsumerFactory` для обработки каждого топика .


Но перед этим ненадолго вернемся к аннотации `@KafkaListener`. Выше я уже упоминал какую схему предлагает starter из коробки(общий конфиг на все консьюмеры с возможностью переопределить их в аннотации).
То есть получается мешанина - часть настроек будет в конфиге, а часть в коде.
Но я обычно вижу следующие варианты:

```kotlin
@KafkaListener(
    topics = ["user.action.like"],
    groupId = "\${ser.action.like.kafka.group-id}"
)
fun onUserLikeAction(action: UserActionDto) = Unit
```
В Spring нет возможности использовать готовые настройки приложения для одного контейнера, но так прибивать их гвоздями в коде не лучшая идея, то они обычно заменяются плейсхолдерами.
Поэтому рекомендую не указывать параметры в аннотации и выносить их в настройки тоже:

```kotlin
class KafkaContainerProperties {
    lateinit var cluster: String
    lateinit var properties: Map<String, String>
    var autoStartup: Boolean = true
	var batchListener: Boolean = false
    var concurrency: Int = 1
}
```

Возвращаемся к настройке фабрик:

```kotlin
const val USER_LIKE_ACTION_KAFKA_LISTENER_CONTAINER_FACTORY = "userLikeActionKafkaListenerContainerFactory"

@EnableKafka
@Configuration(proxyBeanMethods = true)
class KafkaConfiguration(
    private val kafkaClusterProperties: KafkaClusterProperties
) {

    @Bean
    @ConfigurationProperties("user.like.action.kafka.consumer")
    fun userLikeActionKafkaConsumerProperties() = KafkaContainerProperties()

    @Bean
    fun userLikeActionKafkaConsumerFactory(): DefaultKafkaConsumerFactory<String, UserActionDto> {
        return DefaultKafkaConsumerFactory(
            kafkaClusterProperties.buildConsumerProperties(userLikeActionKafkaConsumerProperties()),
            StringDeserializer(),
            buildDefaultJsonDeserializer<UserActionDto>()
        )
    }


    @Bean(USER_LIKE_ACTION_KAFKA_LISTENER_CONTAINER_FACTORY)
    fun userLikeActionKafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, UserActionDto> {
        return with(userLikeActionKafkaConsumerProperties()) {
            ConcurrentKafkaListenerContainerFactory<String, UserActionDto>().also {
                it.consumerFactory = userLikeActionKafkaConsumerFactory()
                it.setConcurrency(concurrency)
                it.setAutoStartup(autoStartup)
                it.setBatchListener(batchListener)
            }
        }
    }
}

val FILE_PROPERTIES_NAMES = hashSetOf(    SslConfigs.SSL_KEYSTORE_LOCATION_CONFIG,
    SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG,
)


fun KafkaClusterProperties.getCluster(name: String): KafkaCluster {
    return this.clusters[name] ?: throw IllegalArgumentException("Not found kafka-cluster")
}

fun KafkaClusterProperties.buildConsumerProperties(
    containerProperties: KafkaContainerProperties
): Map<String, Any> {
    val cluster = getCluster(containerProperties.cluster)

    cluster.properties

    val commonProperties = hashMapOf(
        CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG to cluster.bootstrapServers,
    )

    val result = (commonProperties + cluster.properties + containerProperties.properties).mapValues {
        if (FILE_PROPERTIES_NAMES.contains(it.key)) {
            resourceToPath(it.value as String)
        } else {
            it.value
        }
    }

    return result
}


private fun resourceToPath(path: String): String {
    try {
        return ResourceUtils.getFile(path).absolutePath
    } catch (ex: IOException) {
        throw IllegalStateException("Resource '$path' must be on a file system", ex)
    }
}
```

### Отключение обработки топика/изменение offset топика в группе

С этой проблемой уже не все так плохо. В Spring доступна фича отключать листенеры через параметр `autoStartup` в аннотации `@KafkaListener`.
И это работает как полагается, если добавить это в код заранее. При задании значения `autoStartup=false` Spring не будет создавать контейнеры для обработки топиков(в примерах выше этот параметр вынесен в класс `KafkaContainerProperties`).

Иногда возникает ситуация, когда требуется изменить offset в группе(например, если обработка топика встала, то это решается пропуском сообщения).
Это делается только в случае полной остановки группы-консьюмеров. Для этого как раз пригодится параметр autoStartup.

Но в случае использования стартера это не сработает. В стартере id-группы задается для всех консьюмеров сразу, а не для конретного топика.
И если остановить обработчик одного топика, то это ни к чему не приведет, так как в консьюмер-группе консьюмеры других топиков продолжат работать и Kafka вернет ошибку при изменение offset.

Поэтому реализация в Spring не очень удачная (этот параметр переопределяется в новом классе с настройками или в аннотации `@KafkaListener`).

```yaml 
spring:
  kafka:
    consumer:
      group-id: my-application
```

Для группы консьюмеров сформируем следующие правило - "За одной группой консьюмеров закрепляется только один топик".
Если за одной группой консьюмеров закрепляется больше одного топика, то возникнет еще одна проблема с перфомансом - чем больше топиков в одной коньсюмер-группе, тем дольше выполнятся коммит offset`а

### Мониторинг

Spring поддерживает два типа [метрик](https://docs.spring.io/spring-kafka/reference/kafka/micrometer.html): для контейнера и для консьюмера.
Рассмотрим только метрики для контейнера - `spring.kafka.listener`.
У этой метрики тип таймер, но она так же подходит и для мониторинга кол-ва обработанных сообщений(так как таймер состоит из 2-х частей: кол-во вызовов таймера и сумма времени работы).

Метрика информативная, но есть один ньюанс связанный с тэгом `name`. По документации значение у этого тэга - это имя бина созданного контейнера.
И если специально не настроить контейнер, то у него будет значение `org.springframework.kafka.KafkaListenerEndpointContainer#`.
Но это легко исправляется следующим образом - в аннотацию `@KafkaListener` добавим 2 параметра: `id` и `idIsGroup=false`.
Значение из `id` станет значением тэга `name`, а `idIsGroup` нужен, чтобы Spring не переопределил название консьюмер группы значением параметра `id`.

Итоговый обработчик получится таким:
```kotlin
@KafkaListener(
    id = "onUserLikeAction",
    idIsGroup = false,
    topics = ["user.action.like"],
    containerFactory = USER_LIKE_ACTION_KAFKA_LISTENER_CONTAINER_FACTORY,
)
fun onUserLikeAction(action: UserActionDto) = Unit
```

Параметр `id` указывается так же и для корректного формирования названия поток(в Spring каждый созданный контейнер это отдельный поток).
При снятии thread-dump это позволит получать информацию в полноценном виде.

Помимо метрик так же полезно делать следующие вещи:
1. Указывать ключ сообщения, даже если в этом нет явной необходимости(например, для упорядочивания сообщений).
Ключ сообщения - это мета-информация и нужен, чтобы быстро понимать к какой бизнес-сущности(бизнес-операции) относится сообщение. Это полезно, когда в топике бинарный формат сообщения или было записано не валидное сообщение.
2. Каждое сообщение так или иначе относится к бизнес-операции. Добавление в заголовки сообщения мета-информации упрощает поддержку приложений, так как с ними локализация проблем ускоряется
3. Подключение трейсинга. Трейсинг лучше запускать с этапа десериализации 

### Другие рекомендации

1. Параметр `auto.offset.reset` активен только в 2-х ситуациях: при первой подписки группы на топик и при создании партиции в топике.
Чтобы при создании партиций не было потери сообщений этот параметр должен иметь значение `earliest`(если при первой подписке нужно другое значение, то его потом следует заменить)
2. В Spring несколько стратегий коммита offset. Одна из них это ручной коммит. Если делать коммит после каждой записи, то это будет сильно влиять на перформанс обработки.
3. Kafka не поддерживает семантику `exactly-once` и нужно заранее определить, что нужно при повторной обработке сообщения
4. Рано или поздно возникнут ошибки обработки сообщений, которые можно решать так:
	+ пропускать сообщения и после устранения проблемы сделать переотправку события
	+ полностью останавливать обработку топика/партиции до устранения проблемы
	+ необработанные сообщения переотправлять в [retryable-topic](https://docs.spring.io/spring-kafka/reference/retrytopic.html) и продолжать обработку


Полный [пример кода](https://github.com/bespaltovyj/kafka-in-spring-samples/tree/main)/[текст на github](https://github.com/bespaltovyj/articles/blob/main/%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B0/Kafka/%D0%9A%D0%B0%D0%BA%20%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%B0%D1%82%D1%8C%20%D1%81%20Kafka-consumer%20%D0%B2%20Spring-%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B0%D1%85.md)

![](https://habrastorage.org/webt/8l/f7/oc/8lf7oc4eaugy1lonnrwj37epc7e.jpeg "Вместо вывода")

***

*Kafka, как и другие брокеры, сложный инструмент. Добавление брокера в продукт без обдуманных причин ни к чему хорошему не приведет*
