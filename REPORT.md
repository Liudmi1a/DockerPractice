# Отчёт по работе №2: Практика Docker
**Цель:** создать систему мониторинга системы датчиков с использованием Docker, Mosquitto, InfluxDB, Telegraf и Grafana на трёх виртуальных машинах (далее ВМ).

## Подготовка ВМ
1. Взяты три ВМ из работы №1 (предварительно настроены в формате client, server, gateway).
2. Необходимо установить на каждую ВМ Docker и Docker Compose посредством:
```shell
sudo apt update
sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
docker --version
sudo usermod -aG docker $USER
sudo apt install -y docker-compose
docker-compose --version
```

# Шаг 1: Разработка Simulator для генерации данных

В файле `sensor.py` создаем четвертыht типf датчиков.

Далее в файле `main.py` реализовуем клиента, который подключается к mqtt брокеру и публикует сообщения.

В `Dockerfile`, необходимом для того, что создать образ, указываем следуюещие инструкции:

``` Dockerfile
FROM python:alpine3.19
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

Инструкция `FROM` инициализирует новый этап сборки и устанавливает базовый образ для последующих инструкций. `WORKDIR` создает рабочий каталог для последующих инструкций `Dockerfile`. Инструкция `COPY` копирует файл *requirements.txt* из источника в указанное место внутри образа.
Инструкция `RUN`задает команды, которые следует выполнить и поместить в новый образ контейнера. `RUN` описывает команду с аргументами, которую нужно выполнить когда контейнер будет запущен.

Содержимое файла *requirements.txt* приведено ниже:
`paho_mqtt==1.6.1`

*paho_mqtt* предоставляет клиентский класс, который позволяет приложениям подключаться к MQTT-брокеру для публикации сообщений.  

После проделанных выше опреаций можно создать образ командой:

`docker build -t luyda/sensor-sim . `

Таким способом мы создали образ, из которого можно развернуть контейнер.

# Шаг 2: Запуск Mosquitto брокера

Для настройки протокола MQTT необходимо создать конфигурационный файл *mosquitto.conf* со следующим содержимымы:

```c
listener 1883
allow_anonymous true
```
Для более удобного запуска брокера создадим файл *docker-compose.yml*.

Теперь создав контейнер посмотрим его логи:
![example_run_logs](https://github.com/user-attachments/assets/79529d5e-a1a1-4f9a-bb6e-ae171fc12bc2)

Брокер же отображает присоединившегося клиента.
![mqtt-docker_run](https://github.com/user-attachments/assets/ac0feb01-6071-4bf4-8a9c-e969bd2aea44)

Теперь можно запустить несколько датчиков. Но перед этим необходимо прописать *docker-compose.yml*:

С помощью команды `docker compose up` запустим контейнеры.

При этом брокер отображает:
![mqtt-docker-compose](https://github.com/user-attachments/assets/67cae960-53c8-42cc-a962-8eb575bdbbed)

# Шаг 3: Получение данных от симулятора
Первоначально необходимо настроить Telegraf, который  подписывается на MQTT, где датчики публикуют данные. Данные будут сохраняться в InfluxDB. Отображение информации с датчиков будет происходить при помощи Grafana.

## Telegraf
Перед использованием Telegram необходимо настроить его.
Для этого создадим конфигурационный файл *telegraf.conf* 
Тем самым мы настраиваем Telegraf на чтение данных с машины IP адрес которой 192.168.0.101 через порт 1883.

Настройка Telegraf на этом завршена. 

## InfluDB

Для создания базы данных необходимо в конфигурационном файле *influxdb-init.iql* прописать следующее:
```sql
CREATE database sensors
CREATE USER telegraf WITH PASSWORD 'telegraf' WITH ALL PRIVILEGES
```

## Grafana

Данные для отображения датчиков берутся из InfluxDB.
Для настройки в конфигурационном файле необходимо прописать следующее:
```yaml
apiVersion: 1
datasources:
  - name: InfluxDB_v1
    type: influxdb
    access: proxy
    database: sensors
    user: telegraf
    url: http://influxdb:8086
    jsonData:
      httpMode: GET
    secureJsonData:
      password: telegraf
```

Для запуска всех трех контенйеров воспользуемся docker-compose. создадим файл docker-compose.yml

После чего можно выполнить команду `docker compose up`

# Настройка дашборда
После запуска всех необходимых контейнеров, в браузере переходим по `192.168.0.101:3000`

Для проверки соединения перейдем в Menu -> Connections -> Data sources. Находим в истончиках InfluxDB и проверяем подключение:
![grafana-influxdb-test](https://github.com/user-attachments/assets/db9150a7-259a-4186-94d3-167383dd9e25)

После переходим через Меню в раздел Dashboards, где создаем собственный дашбоард.
![grafana-new-view](https://github.com/user-attachments/assets/ee1b4398-b5e2-4910-b340-9d1a33185fb5)

Для отображения информации с датчиков необходимо создать запрос (query).
![query](https://github.com/user-attachments/assets/3dbd7945-7190-470d-bf3c-f6f1b537e77e)

После создания необходимо количества графиков, отображающих инфомрацию с Simluator, экспортируем дашборд как JSON-файл.

Для этого находим функцию `Share` -> ``Export` -> `Save to file`. Сохраненный файл помещаем в папку `vms\server\infra\grafana\provisioning\dashboards\mqtt.json`

# Пример выполненной работы

![dashboard](https://github.com/user-attachments/assets/f94c4469-4ef2-4c4a-b2bb-9b2c7464bd79)


