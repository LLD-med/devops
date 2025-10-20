# Настройка систем мониторинга

>Настройка ВМ остается той же (проект №1).

Необходимые сервисы на ВМ1:
1) Prometheus
2) Node_exporter
3) Alertmanager
4) Blackbox_exporter

Необходимые сервисы на ВМ2:
1) Grafana (через Docker)
2) Node_exporter

## ВМ1

### Настройки Nginx:
Для получения статистики nginx необходимо добавить в файл `/etc/nginx/conf.d/status.conf`:
![](./img/nginx_status.png)

Перезапускаем Nginx.
Получаем метрики по `curl http://127.0.0.1/nginx_status`, которые можно распарсить:
![](./img/nginx_statistic.png)

### Установка Prometheus:
После загрузки и установки prometheus (создаем пользователя prometheus и настраиваем права, см в настройке alertmanager) создаем systemd unit файл:
![](./img/s1_systemd_prometheus.png)

`--storage.tsdb.retention.time=15d` - циклическая запись 15 дней
`--storage.tsdb.retention.size=10GB` - ограничение по размеру в 10 гигов

Включаем и запускаем:
![](./img/s1_prometheus.png)

### Настройка сбора метрик
Создаем скрипты (+ сделать исполняемыми) для:
- проверки доступности nginx
![](./img/nginx_status_bash.png)

- парсинга полученной статистики с `curl http://127.0.0.1/nginx_status`
![](./img/nginx_metrics_bash.png)

- получения размера папки Prometheus
![](./img/prometheus_size_script.png)

Добавляем задания в cron для автоматического обновления: `sudo crontab -e`
![](./img/cron.png)

Cron будет каждую минуту выполнять скрипты, node_exporter по пути, указанному во флаге `--collector.textfile.directory`, будет читать текстовые файлы с метриками, а prometheus отображать в графиках.


### Настройки Alertmanager:
После загрузки и установки Alertmanager создаем пользователя alertmanager с целью изоляции от root пользователя:
![](./img/s1_alert_own.png)
Изменяем права на изменение/запись/чтение данных, относящихся к Alertmanager, только для пользователя alertmanager.

Создаем конфигурационный файл `/etc/alertmanager/alertmanager.yml`:
![](./img/alert.png)

`group_interval` - отправка отправлений каждые 5 минут

`receivers` - получатели уведомлений. В нашем случаем это telegram, в настройке необходимо указать токен бота (скриншот отредактирован, токен выдаст BotFather) и id для отправки уведомлений.

Для определения id:
`curl https://api.telegram.org/botBOT_TOKEN/getUpdates`
вместо BOT_TOKEN указать токен своего бота, нужно поле `id`  внутри `chat`.

`send_resolved` - отправлять уведомления о восстановлении сервиса

Создаем systemd unit файл для старта после загрузки системы:
![](./img/s1_systemd_alert.png)

Включаем и перезапускаем Alertmanager
![](./img/s1_alert.png)

Нужно настроить правила оповещений в prometheus.
Создаем файл правил:
![](./img/nginx_rules.png)
`expr: nginx_up == 0` - PromQL выражение для срабатывание алерта, будет следить за значением переменной nginx_up

`for: 1m` - ждем одну минуту до срабатывания

В `severity` можно указать уровни warning и info

В конфигурацию прометеуса добавить путь до файла правил в секцию `rule_files` и указать куда отправлять алерт (9093 - стандартный порт alertmanager)
![](./img/prom_alert.png)

Перезапустить прометеус.

### Настройки Node_exporter:
Загружаем и устанавливаем Node_exporter, создаем пользователя node_exporter, настраиваем права.

systemd unit файл:
![](./img/s1_systemd_node.png)

Включаем и запускаем:
![](./img/s1_node.png)

В конфигурации прометеуса добавляем отслеживание node_exporter:
![](./img/prom_node.png)

Перезапускаем прометеус

### Настройки Blackbox_exporter:
Загружаем и устанавливаем blackbox_exporter, создаем пользователя blackbox_exporter, настраиваем права.

systemd unit файл:
![](./img/s1_systemd_black.png)

Включаем и запускаем:
![](./img/s1_black.png)

В конфигурацию прометеус нужно добавить мониторинг внешнего ресурса (секция blackbox):
![](./img/prometheus_black.png)

Проверяем только успешные ответы (код 2xx).
В качестве целей для мониторинга были выбраны сайты google и github.

Перезапускаем прометеус.


## ВМ2

На ВМ2 поднимаем графана через Docker:
![](./img/s2_docker.png)
 
Аналогично ВМ1 устанавливаем node_exporter.

Для просмотра метрик, получаемых с ВМ2, нужно добавить в конфигурацию прометеуса с указанием ip ВМ2 и перезапустить его:
```yaml
  - job_name: 'node_vm2'
    static_configs:
    - targets: ['20.211.55.26:9100']
```

![](./img/prometheus_job_vm2.png)

### Просмотр результатов:
На хосте подключаемся с использованием SSH туннелей.

Подключение к Prometheus: `ssh -L 9090:localhost:9090 pc1@10.211.55.23`

Подключение к Grafana: `ssh -L 3000:192.168.100.2:3000 pc1@10.211.55.23`


### Результаты:

![](./img/grafana_1.png)
![](./img/grafana_blackbox.png)
![](./img/grafana_dir_size.png)
![](./img/grafana_probe.png)


Алертинг
![](./img/prometheus_alert_up.png)
![](./img/prometheus_alert_pending.png)
![](./img/prometheus_alert_down.png)

Выводы в телеграм:
![](./img/telegram.png)
