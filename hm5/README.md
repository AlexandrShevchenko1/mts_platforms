Пошаговый подход к созданию пайплайна обработки данных в Airflow, который запускает Spark на YARN, читает данные из HDFS, выполняет трансформации и сохраняет результат.

---

## **1. Предварительные условия**

1. HDFS и YARN развернуты и настроены
2. Apache Spark установлен и готов работать в Yarn-кластере
3. Airflow установлен и инициализирован
    
    ```bash
    pip install apache-airflow
    airflow db init
    airflow users create \
      --username admin \
      --password admin \
      --firstname First \
      --lastname Last \
      --role Admin \
      --email admin@example.com
    ```
    
4. В файле `airflow.cfg` или через Web UI Airflow можно настроить Spark Connection и указать путь к Spark при помощи `SPARK_HOME` или вручную.

---

## **2. DAG для Spark на YARN**

### **2.1. Структура файлов**

1. Создать папку для DAG, например `~/airflow/dags/`.
2. Внутри создать файл `spark_processing_dag.py`.

### **2.2. DAG с SparkSubmitOperator**

```python
from datetime import datetime
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 1, 1),
    'retries': 1
}

with DAG(
    dag_id='spark_processing_dag',
    default_args=default_args,
    schedule_interval='@once',
    catchup=False
) as dag:
    
    spark_job = SparkSubmitOperator(
        application='/home/airflow/dags/spark_app.py', 
        task_id='spark_job',
        conn_id='spark_default',           
        conf={'spark.master': 'yarn'},     
        executor_cores=2,
        executor_memory='2g',
        name='spark_processing_task'
    )

    spark_job
```

---

## **3. Реализация логики в Spark**

В файле `/home/airflow/dags/spark_app.py` (или другой в `application`):

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum

def main():
    spark = SparkSession.builder \
        .appName("AirflowSparkProcessing") \
        .getOrCreate()

    # 1. чтение данных из HDFS
    df = spark.read \
        .option("header", "true") \
        .csv("hdfs://tmpl-nn:9000/user/hadoop/input/data.csv")

    # 2. трансформации
    transformed_df = df \
        .withColumn("amount", col("amount").cast("double")) \
        .groupBy("category") \
        .agg(sum("amount").alias("total_amount"))

    # 3. сохранение
    transformed_df.write \
        .mode("overwrite") \
        .parquet("hdfs://tmpl-nn:9000/user/hadoop/output/transformed_parquet")

    spark.stop()

if __name__ == "__main__":
    main()
```

---

## **4. Запуск пайплайна**

1. Airflow Web Server и Scheduler должны быть запущены:
    
    ```bash
    airflow webserver -p 8080
    airflow scheduler
    ```
    
2. Зайти в Airflow UI по адресу `http://<Host_IP>:8080` и включите DAG `spark_processing_dag`.
3. Запустить DAG вручную или дождаться планового запуска.

---

## **5. Проверка результата**

1. В Airflow UI все задачи `spark_processing_dag` должны быть выполнены без ошибок
2. Надо подключиться к HDFS или Hive и проверить, что данные успешно записаны:
    
    ```bash
    hdfs dfs -ls /user/hadoop/output/transformed_parquet
    ```
    
3. Если таблица в Hive, то запустить Beeline:
    
    ```bash
    beeline -u "jdbc:hive2://tmpl-nn:10000"
    USE mydb;
    SELECT * FROM my_table;
    ```
