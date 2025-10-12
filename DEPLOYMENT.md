# Hadoop Cluster Deployment Guide

## Архитектура кластера

### Узлы кластера:
- **Master Node** (10.128.0.10 / 158.160.47.11)
  - NameNode (HDFS)
  - ResourceManager (YARN)
  - HistoryServer (MapReduce)
  
- **Worker01** (10.128.0.11 / 158.160.122.31)
  - DataNode (HDFS)
  - NodeManager (YARN)
  
- **Worker02** (10.128.0.4 / 158.160.100.168)
  - DataNode (HDFS)
  - NodeManager (YARN)

## Ключевые изменения

### 1. Использование внутренних IP адресов
Все узлы кластера общаются между собой через **внутренние IP адреса** (10.128.0.x), а не публичные. Это обеспечивает:
- Более быструю коммуникацию внутри облака
- Безопасность (трафик не выходит в интернет)
- Отсутствие дополнительных расходов на внешний трафик

### 2. SSH конфигурация
- Ansible подключается к узлам через **публичные IP** используя ключ `homelab-only-key`
- Master node подключается к worker nodes через **внутренние IP** используя ключ `hadoop-key`
- Файл `/etc/hosts` на всех узлах содержит маппинг имен на внутренние IP

### 3. Общая роль (common)
Роль `common` выполняется на всех узлах и настраивает:
- Установку Java и необходимых пакетов
- SSH ключи для межузловой коммуникации
- Файл `/etc/hosts` с внутренними IP адресами

### 4. Автоматический запуск сервисов
- Master: автоматически запускаются NameNode, ResourceManager, HistoryServer
- Workers: автоматически запускаются DataNode и NodeManager после того, как NameNode станет доступен

## Порядок развертывания

### Шаг 0: Подготовка SSH ключей

Если ключ `hadoop-key` еще не создан:
```bash
ssh-keygen -t rsa -b 4096 -f roles/common/files/ssh/hadoop-key -N ""
```

### Шаг 1: Проверка подключения
```bash
ansible all -i inventory.yml -m ping -v
```

Ожидаемый результат: все узлы отвечают `pong`

### Шаг 2: Проверка синтаксиса
```bash
ansible-playbook -i inventory.yml site.yml --syntax-check
```

### Шаг 3: Развертывание кластера

**ВАЖНО:** Сначала нужно остановить все запущенные Hadoop процессы на всех узлах:

```bash
# На master node
ssh alice3e@158.160.47.11 "pkill -f hadoop; pkill -f yarn"

# На worker nodes
ssh alice3e@158.160.122.31 "pkill -f hadoop; pkill -f yarn"
ssh alice3e@158.160.100.168 "pkill -f hadoop; pkill -f yarn"
```

Затем запустите развертывание:

```bash
# Полное развертывание всех узлов
ansible-playbook -i inventory.yml site.yml -v
```

Playbook выполнит следующие действия:
1. Настроит общую конфигурацию на всех узлах (роль common)
2. Установит и настроит Hadoop на master
3. Отформатирует NameNode (если еще не отформатирован)
4. Запустит сервисы на master (NameNode, ResourceManager, HistoryServer)
5. Установит и настроит Hadoop на workers
6. Дождется доступности NameNode
7. Запустит сервисы на workers (DataNode, NodeManager)

## Проверка установки

### 1. Проверка Java процессов

На master node:
```bash
ssh alice3e@158.160.47.11 "jps"
```
Должны быть запущены:
- NameNode
- ResourceManager
- JobHistoryServer
- Jps

На worker nodes:
```bash
ssh alice3e@158.160.122.31 "jps"
ssh alice3e@158.160.100.168 "jps"
```
Должны быть запущены:
- DataNode
- NodeManager
- Jps

### 2. Проверка портов

На master node:
```bash
ssh alice3e@158.160.47.11 "ss -tulpn | grep java"
```

Должны быть открыты порты:
- 9000 (NameNode RPC)
- 9870 (NameNode Web UI)
- 8088 (ResourceManager Web UI)
- 8030-8033 (ResourceManager RPC)
- 10020 (HistoryServer RPC)
- 19888 (HistoryServer Web UI)

### 3. Проверка HDFS

Подключитесь к master node:
```bash
ssh alice3e@158.160.47.11
```

Проверьте статус HDFS:
```bash
hdfs dfsadmin -report
```

Должны отображаться:
- 2 живых DataNode (Live datanodes: 2)
- Доступное и используемое пространство
- Количество блоков

### 4. Проверка YARN

На master node:
```bash
yarn node -list
```

Должны отображаться 2 активных NodeManager:
```
Total Nodes:2
         Node-Id             Node-State Node-Http-Address       Number-of-Running-Containers
10.128.0.11:xxxxx                RUNNING 10.128.0.11:8042                                   0
10.128.0.4:xxxxx                 RUNNING 10.128.0.4:8042                                    0
```

### 5. Проверка веб-интерфейсов

**HDFS NameNode UI:**
```
http://158.160.47.11:9870
```
Проверьте:
- Overview → Live Nodes: должно быть 2
- Datanodes → должны быть видны worker01 и worker02

**YARN ResourceManager UI:**
```
http://158.160.47.11:8088
```
Проверьте:
- Cluster → Nodes → Active Nodes: должно быть 2

**MapReduce HistoryServer UI:**
```
http://158.160.47.11:19888
```

## Управление кластером

### Остановка сервисов

На master node:
```bash
# Остановить все сервисы master
/usr/local/hadoop/bin/mapred --daemon stop historyserver
/usr/local/hadoop/bin/yarn --daemon stop resourcemanager
/usr/local/hadoop/bin/hdfs --daemon stop namenode
```

На worker nodes:
```bash
# На каждом worker
/usr/local/hadoop/bin/yarn --daemon stop nodemanager
/usr/local/hadoop/bin/hdfs --daemon stop datanode
```

### Запуск сервисов

На master node:
```bash
# Запустить все сервисы master
/usr/local/hadoop/bin/hdfs --daemon start namenode
/usr/local/hadoop/bin/yarn --daemon start resourcemanager
/usr/local/hadoop/bin/mapred --daemon start historyserver
```

На worker nodes (после запуска NameNode):
```bash
# На каждом worker
/usr/local/hadoop/bin/hdfs --daemon start datanode
/usr/local/hadoop/bin/yarn --daemon start nodemanager
```

### Перезапуск сервисов

Используйте Ansible handlers:
```bash
# Перезапустить только master
ansible-playbook -i inventory.yml site.yml --limit hadoop_master --tags restart -v

# Перезапустить только workers
ansible-playbook -i inventory.yml site.yml --limit hadoop_worker --tags restart -v
```

## Тестирование кластера

### Тест 1: Создание директории в HDFS
```bash
ssh alice3e@158.160.47.11
hdfs dfs -mkdir -p /user/alice3e/test
hdfs dfs -ls /user/alice3e
```

### Тест 2: Загрузка файла
```bash
echo "Hello Hadoop Cluster" > test.txt
hdfs dfs -put test.txt /user/alice3e/test/
hdfs dfs -cat /user/alice3e/test/test.txt
hdfs dfs -ls /user/alice3e/test/
```

### Тест 3: Проверка репликации
```bash
# Создать файл и проверить его репликацию
echo "Test replication" > replication-test.txt
hdfs dfs -put replication-test.txt /user/alice3e/test/
hdfs fsck /user/alice3e/test/replication-test.txt -files -blocks -locations
```

Вы должны увидеть, что файл реплицирован на 2 DataNode.

### Тест 4: Запуск MapReduce job
```bash
# Создать тестовый файл с текстом
cat > input.txt << EOF
Hello Hadoop
Hello World
Hadoop is great
World of Big Data
EOF

# Загрузить в HDFS
hdfs dfs -mkdir -p /user/alice3e/wordcount/input
hdfs dfs -put input.txt /user/alice3e/wordcount/input/

# Запустить WordCount
hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar wordcount /user/alice3e/wordcount/input /user/alice3e/wordcount/output

# Проверить результат
hdfs dfs -cat /user/alice3e/wordcount/output/part-r-00000
```

Ожидаемый результат:
```
Big     1
Data    1
Hadoop  2
Hello   2
World   2
great   1
is      1
of      1
```

## Troubleshooting

### Проблема: DataNode не подключается к NameNode

**Симптомы:**
- `hdfs dfsadmin -report` показывает 0 живых DataNode
- В логах DataNode ошибки подключения к NameNode

**Решение:**
1. Проверьте `/etc/hosts` на всех узлах:
```bash
ansible all -i inventory.yml -m shell -a "cat /etc/hosts | grep -E '10.128.0'"
```

2. Проверьте доступность NameNode с worker:
```bash
ssh alice3e@158.160.122.31 "telnet 10.128.0.10 9000"
```

3. Проверьте конфигурацию в `core-site.xml`:
```bash
ssh alice3e@158.160.122.31 "grep -A1 'fs.defaultFS' /usr/local/hadoop/etc/hadoop/core-site.xml"
```

4. Перезапустите DataNode:
```bash
ssh alice3e@158.160.122.31 "/usr/local/hadoop/bin/hdfs --daemon stop datanode && /usr/local/hadoop/bin/hdfs --daemon start datanode"
```

### Проблема: NodeManager не регистрируется в ResourceManager

**Симптомы:**
- `yarn node -list` показывает 0 узлов
- В логах NodeManager ошибки подключения

**Решение:**
1. Проверьте доступность ResourceManager:
```bash
ssh alice3e@158.160.122.31 "telnet 10.128.0.10 8032"
```

2. Проверьте конфигурацию в `yarn-site.xml`:
```bash
ssh alice3e@158.160.122.31 "grep -A1 'yarn.resourcemanager.address' /usr/local/hadoop/etc/hadoop/yarn-site.xml"
```

3. Перезапустите NodeManager:
```bash
ssh alice3e@158.160.122.31 "/usr/local/hadoop/bin/yarn --daemon stop nodemanager && /usr/local/hadoop/bin/yarn --daemon start nodemanager"
```

### Проблема: Сервисы не запускаются

**Решение:**
1. Проверьте логи:
```bash
ssh alice3e@158.160.47.11 "tail -100 /usr/local/hadoop/logs/hadoop-alice3e-namenode-*.log"
```

2. Проверьте переменные окружения:
```bash
ssh alice3e@158.160.47.11 "source /etc/profile.d/hadoop.sh && env | grep HADOOP"
```

3. Проверьте JAVA_HOME:
```bash
ssh alice3e@158.160.47.11 "grep JAVA_HOME /usr/local/hadoop/etc/hadoop/hadoop-env.sh"
```

### Проблема: "Address already in use"

**Решение:**
Остановите все процессы и запустите заново:
```bash
# На всех узлах
ansible all -i inventory.yml -m shell -a "pkill -f hadoop; pkill -f yarn"

# Подождите 5 секунд
sleep 5

# Перезапустите playbook
ansible-playbook -i inventory.yml site.yml -v
```

### Проблема: HistoryServer не может подключиться к HDFS

**Симптомы:**
- HistoryServer постоянно пытается подключиться к HDFS
- В логах: "Waiting for FileSystem at 10.128.0.10:9000 to be available"

**Решение:**
1. Убедитесь, что NameNode запущен:
```bash
ssh alice3e@158.160.47.11 "jps | grep NameNode"
```

2. Проверьте доступность порта 9000:
```bash
ssh alice3e@158.160.47.11 "ss -tulpn | grep 9000"
```

3. Перезапустите HistoryServer:
```bash
ssh alice3e@158.160.47.11 "/usr/local/hadoop/bin/mapred --daemon stop historyserver && sleep 2 && /usr/local/hadoop/bin/mapred --daemon start historyserver"
```
