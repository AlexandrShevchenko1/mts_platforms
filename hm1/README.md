
В нашем распоряжении 4 виртуальные машины:  
**dn-0** (DataNode) **dn-1** (DataNode) **jn** (JumpNode) **nn** (NameNode, SecondaryNameNode, DataNode)


## Настройка виртуальных машин  

0. Переименовать каждую машину, заменив `team-25` на `tmpl`, отредактировав `/etc/hostname`, затем выполнить перезагрузку.  

1. Сгенерировать SSH-ключи на JumpNode и раздать их остальным узлам:  

```bash
ssh-keygen  
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys  
scp ~/.ssh/id_ed25519.pub 192.168.1.103:.ssh/authorized_keys  
scp ~/.ssh/id_ed25519.pub 192.168.1.104:.ssh/authorized_keys  
scp ~/.ssh/id_ed25519.pub 192.168.1.105:.ssh/authorized_keys  
```

2. Добавить привязку IP-адресов к именам узлов во всех `/etc/hosts`:  

```plaintext
127.0.0.1 localhost  
127.0.0.1 tmpl-jn  

192.168.1.102 tmpl-jn  
192.168.1.103 tmpl-nn  
192.168.1.104 tmpl-dn-00  
192.168.1.105 tmpl-dn-01  
```

На каждой машине должен быть свой локальный идентификатор:

```plaintext
127.0.0.1 tmpl-dn-00  
127.0.0.1 tmpl-dn-01  
```

3. Создать пользователя `hadoop` на всех узлах:  

```bash
sudo adduser hadoop  
```

4. Сгенерировать SSH-ключи для пользователя `hadoop` и скопировать их на узлы:  

```bash
ssh-keygen  
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys  
scp -r ~/.ssh/ tmpl-nn:/home/hadoop  
scp -r ~/.ssh/ tmpl-dn-00:/home/hadoop  
scp -r ~/.ssh/ tmpl-dn-01:/home/hadoop  
```

5. Установить Hadoop на всех узлах:  

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz  
scp hadoop-3.4.0.tar.gz tmpl-jn:/home/hadoop  
scp hadoop-3.4.0.tar.gz tmpl-nn:/home/hadoop  
scp hadoop-3.4.0.tar.gz tmpl-dn-00:/home/hadoop  
scp hadoop-3.4.0.tar.gz tmpl-dn-01:/home/hadoop  
```

6. Разархивировать архив Hadoop на всех машинах:  

```bash
tar -xzvf hadoop-3.4.0.tar.gz  
```

## Конфигурация Hadoop  

1. Добавить в `.profile` на всех узлах:  

```bash
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0  
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64  
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin  
```

2. Применить изменения и распространить `.profile`:  

```bash
source ~/.profile  
hadoop version  
scp ~/.profile tmpl-nn:/home/hadoop  
scp ~/.profile tmpl-dn-00:/home/hadoop  
scp ~/.profile tmpl-dn-01:/home/hadoop  
```

3. Указать путь к Java в `hadoop-env.sh`:  

```bash
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64  
```

4. Добавить настройки в `core-site.xml`:  

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://tmpl-nn:9000</value>
  </property>
</configuration>
```

5. Настроить `hdfs-site.xml`:  

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
</configuration>
```

6. В файле `workers` указать узлы рабочих машин:  

```plaintext
tmpl-nn  
tmpl-dn-00  
tmpl-dn-01  
```

7. Скопировать конфигурационные файлы Hadoop на все узлы:  

```bash
scp hadoop-env.sh tmpl-nn:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp hadoop-env.sh tmpl-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp hadoop-env.sh tmpl-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop  

scp core-site.xml tmpl-nn:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp core-site.xml tmpl-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp core-site.xml tmpl-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop  

scp hdfs-site.xml tmpl-nn:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp hdfs-site.xml tmpl-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp hdfs-site.xml tmpl-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop  

scp workers tmpl-nn:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp workers tmpl-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop  
scp workers tmpl-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop  
```

## Запуск кластера Hadoop  

1. Подключиться к **NameNode** и выполнить:  

```bash
hadoop-3.4.0/bin/hdfs namenode -format  
hadoop-3.4.0/sbin/start-dfs.sh  
```
