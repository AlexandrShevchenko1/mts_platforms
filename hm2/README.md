**Развертывание YARN и публикация веб-интерфейсов**

## **1. Настройка Nginx для проксирования веб-интерфейсов Hadoop**

На **Jump Node** необходимо внести изменения в конфигурационный файл Nginx `/etc/nginx/sites-available/default`:

- Изменить строку `listen 80 default_server;` на `listen 9870 default_server;`
- Закомментировать `listen [::]:80 default_server;`
- В секции `server/location` закомментировать `try_files $uri $uri/ =404;`
- Добавить в `server/location`:
    
    ```nginx
    proxy_pass http://team-5-nn:9870;
    ```
    

После этих изменений подключаемся к **Jump Node** с созданием SSH-туннеля, который перенаправит локальный порт `9870` на удалённый порт `9870` на сервере **NameNode**:

```sh
ssh -L 9870:team-5-nn:9870 team@176.109.91.7
```

---

## **2. Подключение к NameNode и запуск HDFS**

После успешного подключения к **Jump Node**, выполняем вход на **NameNode**:

```sh
ssh hadoop@team-5-nn
```

Форматируем файловую систему:

```sh
hdfs namenode -format
```

Запускаем HDFS:

```sh
start-dfs.sh
```

Теперь веб-интерфейс **NameNode** доступен по адресу `http://localhost:9870/`.

---

## **3. Конфигурация YARN**

### **3.1. Настройка `yarn-site.xml`**

В файл `$HADOOP_HOME/etc/hadoop/yarn-site.xml` добавляем следующее:

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>team-5-nn</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>team-5-nn:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>team-5-nn:8031</value>
    </property>
</configuration>
```

### **3.2. Настройка `mapred-site.xml`**

В файле `$HADOOP_HOME/etc/hadoop/mapred-site.xml` указываем:

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

### **3.3. Распространение конфигурации на все узлы**

Копируем обновлённые файлы конфигурации с **Jump Node** на остальные серверы:

```sh
pushd $HADOOP_HOME/etc/hadoop
scp yarn-site.xml team-5-nn:$HADOOP_HOME/etc/hadoop
scp yarn-site.xml team-5-dn-00:$HADOOP_HOME/etc/hadoop
scp yarn-site.xml team-5-dn-01:$HADOOP_HOME/etc/hadoop
scp mapred-site.xml team-5-nn:$HADOOP_HOME/etc/hadoop
scp mapred-site.xml team-5-dn-00:$HADOOP_HOME/etc/hadoop
scp mapred-site.xml team-5-dn-01:$HADOOP_HOME/etc/hadoop
popd
```

---

## **4. Запуск YARN и проверка работы**

Подключаемся к **NameNode**:

```sh
ssh hadoop@team-5-nn
```

Запускаем YARN:

```sh
start-yarn.sh
```

Проверяем работающие сервисы:

```sh
jps
```

Ожидаемый результат:

```
102802 ResourceManager
101928 SecondaryNameNode
103291 Jps
102956 NodeManager
101471 NameNode
```

Запускаем `historyserver`, который позволит просматривать выполнение задач:

```sh
mapred --daemon start historyserver
```

---

## **5. Настройка Nginx для веб-интерфейсов YARN**

Создаём отдельный конфигурационный файл для **YARN UI**:

```sh
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/ya
```

Редактируем `/etc/nginx/sites-available/ya`:

- Изменяем `listen 9870 default_server;` на `listen 8088;`
- Заменяем `proxy_pass http://team-5-nn:9870;` на `proxy_pass http://team-5-nn:8088;`

Настраиваем отдельный конфиг для **History Server**:

```sh
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/dh
```

Редактируем `/etc/nginx/sites-available/dh`:

- Меняем `listen 9870 default_server;` на `listen 19888;`
- Изменяем `proxy_pass http://team-5-nn:9870;` на `proxy_pass http://team-5-nn:19888;`

Создаём симлинки в `/etc/nginx/sites-enabled`:

```sh
sudo ln -s /etc/nginx/sites-available/ya /etc/nginx/sites-enabled/ya
sudo ln -s /etc/nginx/sites-available/dh /etc/nginx/sites-enabled/dh
```

Перезапускаем **Nginx**:

```sh
sudo systemctl reload nginx.service
```

---

## **6. Проброс портов и тестирование веб-интерфейсов**

Переподключаемся к **Jump Node**, добавив проброску необходимых портов:

```sh
ssh -L 9870:team-5-nn:9870 -L 8088:team-5-nn:8088 -L 19888:team-5-nn:19888 team@176.109.91.7
```

Теперь доступны следующие веб-интерфейсы:

- **YARN ResourceManager UI**: `http://localhost:8088/`
- **Просмотр запущенных задач в YARN**: `http://localhost:19888/`