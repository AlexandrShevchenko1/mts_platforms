Развертывание Apache Hive с поддержкой нескольких клиентов

---
## **1. Загрузка и установка Hive**

1. **Скачать дистрибутив**:
    
    ```bash
    wget https://downloads.apache.org/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
    ```
    
2. **Распаковать** архив:
    
    ```bash
    tar -xzvf apache-hive-4.0.0-alpha-2-bin.tar.gz
    ```
    
3. **Создать переменную окружения** в `~/.profile` или аналогичном файле:
    
    ```bash
    export HIVE_HOME=~/apache-hive-4.0.0-alpha-2-bin
    export PATH=$PATH:$HIVE_HOME/bin
    ```
    
4. **Активировать изменения**:
    
    ```bash
    source ~/.profile
    ```
    

---

## **2. Настройка Metastore (Server Mode)**

1. **Установить и настроить СУБД** (например, MySQL или PostgreSQL). Ниже пример для MySQL:
    
    ```bash
    sudo apt-get install mysql-server
    ```
    
2. **Создать базу** и пользователя для Hive:
    
    ```sql
    CREATE DATABASE hive_metastore;
    CREATE USER 'hiveuser'@'%' IDENTIFIED BY 'hivepassword';
    GRANT ALL PRIVILEGES ON hive_metastore.* TO 'hiveuser'@'%';
    FLUSH PRIVILEGES;
    ```
    
3. **Скачать JDBC-драйвер** (MySQL Connector) и поместить в `HIVE_HOME/lib`:
    
    ```bash
    wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.0.32.tar.gz
    tar -xzvf mysql-connector-j-8.0.32.tar.gz
    cp mysql-connector-j-8.0.32/mysql-connector-j-8.0.32.jar $HIVE_HOME/lib
    ```
    
4. **Настроить `hive-site.xml`** (в каталоге `$HIVE_HOME/conf` или `$HIVE_HOME/apache-hive-4.0.0-alpha-2-bin/conf`), добавив параметры подключения к базе:
    
    ```xml
    <configuration>
      <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://<DB_HOST>:3306/hive_metastore</value>
      </property>
      <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.cj.jdbc.Driver</value>
      </property>
      <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hiveuser</value>
      </property>
      <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hivepassword</value>
      </property>
      <property>
        <name>hive.metastore.uris</name>
        <value>thrift://0.0.0.0:9083</value>
      </property>
    </configuration>
    ```
    
5. **Инициализировать Metastore**:
    
    ```bash
    schematool -dbType mysql -initSchema
    ```
    

---

## **3. Запуск Hive Metastore и HiveServer2**

1. **Запустить Metastore**:
    
    ```bash
    hive --service metastore &
    ```
    
2. **Запустить HiveServer2** (для нескольких клиентов):
    
    ```bash
    hive --service hiveserver2 &
    ```
    
3. **Проверить**, что процессы работают:
    
    ```bash
    jps  # Ожидаем увидеть RunJar (metastore) и RunJar (HiveServer2)
    ```
    

---

## **4. Подключение к Hive**

1. Использовать **Beeline** для подключения к HiveServer2:
    
    ```bash
    beeline -u "jdbc:hive2://localhost:10000"
    ```
    
2. Создать базу:
    
    ```sql
    CREATE DATABASE IF NOT EXISTS mydb;
    USE mydb;
    ```
    

---

## **5. Создание партиционированной таблицы**

1. **Создать таблицу** с партиционированием:
    
    ```sql
    CREATE TABLE sales_partitioned (
      product_id    INT,
      product_name  STRING,
      sale_amount   DECIMAL(10,2)
    )
    PARTITIONED BY (sale_date STRING)
    STORED AS ORC;
    ```
    
2. **Добавить разделы**:
    
    ```sql
    ALTER TABLE sales_partitioned ADD PARTITION (sale_date='2025-01-01');
    ALTER TABLE sales_partitioned ADD PARTITION (sale_date='2025-01-02');
    ```
    

---

## **6. Загрузка данных в партиционированную таблицу**

1. **Подготовить исходный файл**:
    
    ```plaintext
    101,Monitor,150.00
    102,Keyboard,20.00
    ```
    
2. **Загрузить файл** в нужный раздел:
    
    ```sql
    LOAD DATA LOCAL INPATH '/path/to/sales_2025_01_01.csv'
    INTO TABLE sales_partitioned
    PARTITION (sale_date='2025-01-01');
    ```
    
3. **Проверить**, что данные загружены:
    
    ```sql
    SELECT * FROM sales_partitioned WHERE sale_date='2025-01-01';
    ```
    
