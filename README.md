# hadoop-cluster-ansible

Ansible playbook для автоматического развертывания кластера Hadoop в облачной среде Yandex Cloud Compute VM.

## Описание

Этот Ansible playbook автоматизирует развертывание кластера Hadoop с одним master узлом и несколькими worker узлами. Поддерживает HDFS и YARN компоненты.

### Архитектура
- **1 Master Node**: NameNode, ResourceManager, HistoryServer
- **2 Worker Nodes**: DataNode, NodeManager

### Особенности
- ✅ Использование внутренних IP адресов для коммуникации кластера
- ✅ Автоматическая настройка SSH для межузловой коммуникации
- ✅ Настройка `/etc/hosts` для резолвинга имен
- ✅ Hadoop 3.4.1 с Java 11
- ✅ Полная конфигурация HDFS и YARN

## Требования

- **Платформа**: x86-64 (arm64 не поддерживается!)
- **Ansible**: 2.19.2 или выше
- **Целевые машины**: Ubuntu/Debian
- **SSH доступ**: ко всем узлам
- **Python**: 3.x на целевых машинах

## Быстрый старт

### 1. Настройка inventory

Отредактируйте [`inventory.yml`](inventory.yml:1) и укажите:
- Публичные IP адреса для Ansible подключения (`ansible_host`)
- Внутренние IP адреса для Hadoop коммуникации (`internal_ip`)
- Ваш SSH ключ и пользователя

```yaml
hadoop_master:
  hosts:
    master:
      ansible_host: 158.160.47.11  # Публичный IP
      internal_ip: 10.128.0.10     # Внутренний IP
```

### 2. Настройка SSH ключей

Сгенерируйте новый SSH ключ для Hadoop кластера:
```bash
ssh-keygen -t rsa -b 4096 -f roles/common/files/ssh/hadoop-key -N ""
```

### 3. Проверка подключения

```bash
ansible all -i inventory.yml -m ping -v
```

### 4. Развертывание кластера

```bash
# Проверка синтаксиса
ansible-playbook -i inventory.yml site.yml --syntax-check

# Развертывание
ansible-playbook -i inventory.yml site.yml -v
```

## Структура проекта

```
.
├── inventory.yml              # Инвентарь узлов кластера
├── site.yml                   # Главный playbook
├── README.md                  # Этот файл
├── DEPLOYMENT.md              # Подробная документация по развертыванию
├── roles/
│   ├── common/                # Общая роль для всех узлов
│   │   ├── tasks/main.yml
│   │   └── files/ssh/         # SSH ключи для кластера
│   ├── hadoop-master/         # Роль для master узла
│   │   ├── tasks/main.yml
│   │   ├── templates/         # Конфигурационные файлы Hadoop
│   │   └── handlers/main.yml
│   └── hadoop-worker/         # Роль для worker узлов
│       ├── tasks/main.yml
│       ├── templates/         # Конфигурационные файлы Hadoop
│       └── handlers/main.yml
```

## Конфигурационные файлы

Все узлы используют одинаковые конфигурационные файлы:
- [`core-site.xml`](hadoop-master/templates/core-site.xml.j2:1) - основная конфигурация Hadoop
- [`hdfs-site.xml`](hadoop-master/templates/hdfs-site.xml.j2:1) - конфигурация HDFS
- [`yarn-site.xml`](hadoop-master/templates/yarn-site.xml.j2:1) - конфигурация YARN
- [`mapred-site.xml`](hadoop-master/templates/mapred-site.xml.j2:1) - конфигурация MapReduce

## Проверка установки

### Проверка Java процессов

На master:
```bash
ssh alice3e@158.160.47.11 "jps"
# Ожидается: NameNode, ResourceManager
```

На workers:
```bash
ssh alice3e@158.160.122.31 "jps"
# Ожидается: DataNode, NodeManager
```

### Проверка статуса кластера

```bash
# На master node
hdfs dfsadmin -report
yarn node -list
```

### Веб-интерфейсы

- **HDFS NameNode**: http://158.160.47.11:9870
- **YARN ResourceManager**: http://158.160.47.11:8088

## Документация

Подробная документация по развертыванию и troubleshooting доступна в [`DEPLOYMENT.md`](DEPLOYMENT.md:1)

## Важные замечания

1. **Внутренние IP адреса**: Кластер использует внутренние IP для коммуникации между узлами
2. **SSH ключи**: Master использует отдельный SSH ключ (`hadoop-key`) для подключения к workers
3. **Файл workers**: Автоматически генерируется с внутренними IP адресами workers
4. **Конфигурация одинакова**: Все узлы имеют идентичные конфигурационные файлы Hadoop
