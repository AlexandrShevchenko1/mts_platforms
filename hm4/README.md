Инструкция настройки и запуска Apache Spark в кластере Yarn, а также пример чтения, обработки и записи данных с последующей проверкой всего в Hive.

---

## **1. Установка и подготовка Spark**

1. **Скачать дистрибутив Spark**:
    
    ```bash
    wget https://dlcdn.apache.org/spark/spark-3.3.2/spark-3.3.2-bin-hadoop3.tgz
    ```
    
2. **Распаковать архив**:
    
    ```bash
    tar -xzvf spark-3.3.2-bin-hadoop3.tgz
    ```
    
3. **Установить переменные окружения** в `~/.profile`:
    
    ```bash
    export SPARK_HOME=~/spark-3.3.2-bin-hadoop3
    export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
    ```
    
4. **Активировать изменения**:
    
    ```bash
    source ~/.profile
    ```

---

## **2. Настройка Spark для работы с YARN**

1. Открыть/создать файл `spark-defaults.conf` в директории `$SPARK_HOME/conf`.
2. Добавить строки для интеграции с YARN:
    
    ```properties
    spark.master                     yarn
    spark.submit.deployMode         client
    spark.yarn.queue                default
    spark.driver.memory             2g
    spark.executor.memory           2g
    spark.executor.cores            2
    ```
    
3. Сохранить файл `spark-defaults.conf`.

---

## **3. Запуск Spark-сессии в YARN-кластере**

1. Проверить работу YARN:
    
    ```bash
    jps  # Должны увидеть ResourceManager, NodeManager
    ```
    
2. Запустить Spark Shell:
    
    ```bash
    spark-shell --master yarn --deploy-mode client
    ```
    
3. Убедиться, что shell запустился и работает под управлением YARN.

---

## **4. Подключение к HDFS и чтение данных**

`hdfs://tmpl-nn:9000/user/hadoop/input/data.csv` будет путём  для загруженного CSV-файл в HDFS

Внутри `spark-shell`:

```scala
val df = spark.read
  .option("header", "true")
  .csv("hdfs://tmpl-nn:9000/user/hadoop/input/data.csv")
```

- `df` теперь содержит DataFrame с данными, загруженными с HDFS.
- Можно проверить схему:
    
    ```scala
    df.printSchema()
    ```
    

---

## **5. Применение трансформаций**

Исполним несколько операций над DataFrame. Агрегатная функция и смена типа столбца:

```scala
import org.apache.spark.sql.functions._

val transformedDf = df
  .withColumn("amount", col("amount").cast("double"))  // Преобразуем тип
  .groupBy("category")
  .agg(sum("amount").alias("total_amount"))
```

- Здесь привели столбец `amount` к типу `double`.
- Затем, используя `groupBy`, получили сумму значений `amount` по полю `category`.

---

## **6. Применение партиционирования при сохранении**

Сохраним результат в партиционированном виде по `category`. В формате Parquet:

```scala
transformedDf.write
  .partitionBy("category")
  .mode("overwrite")
  .parquet("hdfs://tmpl-nn:9000/user/hadoop/output/partitioned_results")
```

- Задаём `partitionBy("category")`.
- Режим перезаписи (`overwrite`) можно менять при необходимости.

---

## **7. Создание таблицы и проверка в Hive**

1. Используем Spark для сохранения данных как Hive-совместимой таблицы:
    
    ```scala
    spark.sql("CREATE DATABASE IF NOT EXISTS mydb")
    spark.sql("USE mydb")
    
    transformedDf.write
      .mode("overwrite")
      .saveAsTable("my_partitioned_table")
    ```
    
2. Таблица в Spark:
    
    ```scala
    val hiveDf = spark.sql("SELECT * FROM mydb.my_partitioned_table")
    hiveDf.show()
    ```
    
3. Таблица в Hive:
    - Открыть Beeline или Hive CLI:
        
        ```bash
        beeline -u "jdbc:hive2://tmpl-nn:10000"
        ```
        
    - Проверка:
        
        ```sql
        USE mydb;
        SHOW TABLES;
        SELECT * FROM my_partitioned_table;
        ```
        