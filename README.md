# Задача
Есть большая таблица(12kk) представимая в kv и маленькая(300к), представимая из полуструктурированных данных json в кафке. 

Сама задача сджоинить их в спарке и слить в целевое хранилище. 

Тянуть сразу все данные из большой таблицы - не хотим, т.к. супер неэффективно в экономическом плане, будем считать, 
что данных в большой таблицы на столько много, что при попытке загрузить их все в спарк - мы получим OOM.

Так же кидать данные из малой таблицы в kv для джоина тоже не ок - двойная запись.(Ну и редис не умеет даже чуть-чуть в скл, получается не совсем коррректное сравнение, но все же, кидать данные туда сюда - не ок).

# Join с Ignite

Ignite имеет некоторую поддержку для spark 2.3.1 от русских разработчиков в виде библиотеки ignite-spark, которая расширяет функционал взаимодействия с ignite через igniteContext(оберткой с sparkContext).
Чтобы не тянуть все данные из игнайт мы можем воспользоваться кешем : igniteContext.fromCache .

При запуске мы заметим, что данные не пытаются залиться разом, а загружаются постепенно батчами, и пытаются сджоиниться, таким образом мы не получаем OOM, но к сожалению тянем все данные.

#Join с Redis 

Redis легок в инсталяции и весит всего 500кб, и супер прост в использовании и своей логике. Имеется подобная библиотека spark-redis с RedisContext.
Методы выгрузки из KV имеет следующий вид:
```
  def fromRedisKV[T](keysOrKeyPattern: T,
                     partitionNum: Int = 3)
                    (implicit redisConfig: RedisConfig = new RedisConfig(new RedisEndpoint(sc.getConf))):
  RDD[(String, String)] = {
    keysOrKeyPattern match {
      case keyPattern: String => fromRedisKeyPattern(keyPattern, partitionNum)(redisConfig).getKV
      case keys: Array[String] => fromRedisKeys(keys, partitionNum)(redisConfig).getKV
      case _ => throw new scala.Exception("KeysOrKeyPattern should be String or Array[String]")
    }
  }
```
То есть мы можем выбрать значения которые нам нужны подобным образом:
```redisContext
       .fromRedisKV(df.select("key").rdd.map(r=>r.getString(0)).collect())
       .toDF("key", "value")
```

А затем быстро сджоинить 1к1 + Narrow Dependency.

Попробуем провернуть тоже самое в Ignite:
