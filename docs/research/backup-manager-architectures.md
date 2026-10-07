# Как существующие решения с центральным сервером общаются с хостами БД

Исследование по тикету [#4](https://github.com/dimda7/backup/issues/4). Дата: 2026-10-07.

**Вопрос.** Как существующие решения с центральным сервером (веб-интерфейсы и оркестраторы над WAL-G, pgBackRest, Barman и подобные) общаются с хостами БД: свой агент, SSH, Ansible или иное? Какие плюсы и минусы каждого подхода видны на практике: безопасность, установка, выполнение долгих операций, сбор состояния?

**Короткий ответ.** Все рассмотренные продукты с веб-интерфейсом, которые запускают физические бэкапы на хосте БД (PPEM, Percona PMM, Платформа Tantor, temBoard как ближайший аналог без бэкапов), используют свой агент на хосте БД. SSH используют сами инструменты бэкапа для связи «хост с копиями — хост БД» (Barman, pgBackRest, pg_probackup) и ClusterControl; Ansible — Pigsty, причём только для раскладки конфигурации и разовых операций, а расписание и сбор состояния там живут на самом хосте (cron и exporter). Готового веб-интерфейса именно над WAL-G в первичных источниках не найдено.

Как читать документ:

- **Факт** — утверждение, подтверждённое первичным источником (официальная документация или исходный код продукта); ссылка стоит рядом.
- **Вывод** — моё умозаключение из фактов; помечено словом «Вывод».
- **Не подтверждено** — то, что не удалось проверить; собрано в последнем разделе.

Термины «Сервер управления», «Хост БД», «Хранилище», «Политика бэкапа» — из `CONTEXT.md`. Для чужих продуктов сохранены их собственные названия компонентов (repository host, manager, PMM Server и т. п.).

---

## 1. Продукты

### 1.1. Barman (EnterpriseDB) — SSH и протокол репликации, без агента и без веб-интерфейса

Факты:

- Центральный сервер Barman ходит на хост PostgreSQL двумя способами: по протоколу репликации PostgreSQL (`pg_basebackup`, `pg_receivewal`) и по SSH (rsync). «If you plan to use Rsync backups or WAL archiving via `archive_command`, then SSH connections are required» [B1].
- SSH нужен в обе стороны и без пароля: ключ пользователя `barman` кладётся в `authorized_keys` пользователя `postgres` на хосте PostgreSQL («The SSH connection between from barmanhost to pghost is used for the traditional backup using Rsync»), а ключ `postgres` — в `authorized_keys` пользователя `barman` («The SSH connection from pghost to barmanhost is needed to correctly archive WAL files using the archive_command»). Ключи создаются без парольной фразы [B1].
- Вариант через потоковую репликацию проще в настройке: «It is much simpler to configure, as no manual configuration is required on the database server» [B2].
- Для S3-совместимых хранилищ есть отдельный режим без сервера Barman: пакет `barman-cloud` ставится на тот же хост, где PostgreSQL, и сам читает PGDATA и пишет в объектное хранилище («Using the utility provided by the barman cloud client package in the Postgres host, without a Barman server. Both the base backup and the WALs are read from the local host») [B3]. Скрипт `barman-cloud-backup` «requires read access to PGDATA and tablespaces, typically run as the postgres user» [B3].

Вывод: у Barman нет своего демона на хосте PostgreSQL; при работе с S3 напрямую центральный сервер вообще исчезает, и вопрос «кто запускает и кто собирает состояние» остаётся на стороне пользователя. Веб-интерфейса в составе Barman нет (в документации описаны только CLI-команды).

### 1.2. pgBackRest — SSH по умолчанию, собственный TLS-сервер как альтернатива

Факты:

- При выделенном repository host команды `backup` и `expire` запускаются на нём, а не на хосте БД: «some commands such as backup and expire are run from the repository host instead of the database host» [P1]. Восстановление запускается на хосте БД: «run pgBackRest with the restore command on the database host» [P1].
- Транспорт между хостами — SSH (по умолчанию) или собственный TLS-сервер: параметры `repo-host-type` и `pg-host-type` принимают `ssh` («Secure Shell») и `tls` («pgBackRest TLS server»), `default: ssh` [P2].
- SSH настраивается без пароля в обе стороны, причём ключи ограничиваются запуском только pgBackRest: в `authorized_keys` добавляется `no-agent-forwarding,no-X11-forwarding,no-port-forwarding,command="/usr/bin/pgbackrest ${SSH_ORIGINAL_COMMAND#* }"`. Документация поясняет: «ssh has been configured to only allow pgBackRest to be run via passwordless ssh. This enhances security in the event that one of the service accounts is hijacked» [P1].
- В TLS-режиме на каждом хосте работает постоянный демон: «The pgBackRest TLS server must be configured and started on each host»; запускается как systemd-сервис `ExecStart=/usr/bin/pgbackrest server` от пользователя `postgres` на хосте БД [P3]. Порт по умолчанию 8432 [P2].
- Авторизация в TLS-режиме — по клиентскому сертификату: «Clients are authorized on the server by verifying their certificate and checking their certificate CN (Common Name) against a list on the server configured with the tls-server-auth option», с привязкой CN к конкретным stanza [P2].
- Состояние отдаётся командой `info`: «For machine-readable output use `--output=json`. The JSON output contains far more information than the text output and is kept stable unless a bug is found» [P1].

Вывод: pgBackRest показывает оба транспорта в одном продукте. SSH-режим не требует демона на хосте БД, но требует обмена ключами в обе стороны; TLS-режим по сути и есть узкоспециализированный агент (постоянный сервис на хосте БД, mTLS, список разрешённых клиентов). Официального веб-интерфейса у pgBackRest нет; поиск по GitHub популярных веб-надстроек не дал (см. раздел 4).

### 1.3. pg_probackup (Postgres Professional) — только SSH, временные «remote agents»

Факты:

- «pg_probackup supports the remote mode that allows you to perform backup, restore and WAL archiving operations remotely… Currently the only supported remote protocol is SSH» [PB1].
- Бинарник нужен на обеих машинах, и версии должны совпадать: «Operating in remote mode requires pg_probackup binary to be installed on both local and remote systems. The versions of local and remote binary must be the same» [PB1].
- Агент здесь не демон, а процесс на время операции: «the main pg_probackup process on the local system connects to the remote system via SSH and launches one or more agent processes on the remote system, which are called remote agents… Once data transfer is complete, remote agents are terminated and SSH connections are closed» [PB1].
- Практическое ограничение SSH при параллелизме: при числе потоков больше 10 число SSH-соединений может превысить лимит сервера, и операция падает с «kex_exchange_identification: Connection closed by remote host»; лечится уменьшением потоков или правкой `MaxStartups` в sshd [PB1].
- Документация сама советует ужесточать SSH: «You can impose additional restrictions on SSH settings to protect the system in the event of account compromise» [PB1].

### 1.4. Postgres Pro Enterprise Manager (PPEM) — свой агент, HTTPS в обе стороны

Ближайший аналог нашей задачи: веб-интерфейс, который настраивает и запускает бэкапы (pg_probackup) на многих хостах, в том числе в S3.

Факты:

- Состав: «PPEM is a DBMS infrastructure management service comprising web application, manager, repository, and agents» [E1]. «The agent interacts with the manager: sends environment data (OS and DBMS instance information) and receives control instructions» [E2].
- Соединения открывают обе стороны: «Networking between the manager and agent can be initiated by either side: The manager can send control instructions to agents, and agents return the execution results in response. Agents can send to the manager requests to register themselves and DBMS instances as well as to update the state of instances» [E3].
- Протокол и порты: «The manager and agent use the HTTPS protocol for communication. By default, the manager uses the tcp/8080 (no mTLS) or tcp/9443 (mTLS) port, and the agent uses the tcp/8081 port»; дополнительно можно включить mutual TLS [E3].
- Аутентификация — общий API-ключ на агента: manager получает токены у агента через `POST /v1/sessions` с параметрами `name` и `api_key`; агент так же аутентифицируется на manager ключом из `ppem-agent.yml` (`agent.manager.api_key`) [E4].
- Права на хосте: «To implement most features, the agent only requires the access level of a non-privileged operating system user. There is a small number of features that require privileged access»; «the optimal operational approach is to run the agent under the postgres system user» [E5]. В «защищённом» варианте установки агент работает не от суперпользователя и ему через sudo выдаются отдельные команды (`chmod`, `chown`, `initdb`, `pg_ctl`); часть функций при этом недоступна [E6].
- Бэкап: сам pg_probackup ставится на хосты вручную, агент его обнаруживает: «After pg_probackup is installed, the agent automatically detects it and notifies the manager. The server is now ready for backup operations. You must manually configure the installed pg_probackup» [E7]. S3-совместимое хранилище — штатное место для копий [E1].
- Долгие операции оформлены как задачи: «A user or the manager can initiate the execution of tasks on agents and instances, as well as create schedules for such tasks» [E8]; в разделе Backup есть панель с задачами бэкапа, журнал бэкапа и расписания бэкапов [E9].

### 1.5. Percona Monitoring and Management (PMM) — свой агент, исходящий gRPC-поток от хоста к серверу

Факты:

- Управление бэкапами в PMM есть только для MongoDB (GA) и MySQL (Technical Preview): «MongoDB (Generally Available), MySQL (in Technical Preview)»; PostgreSQL в списке нет [M1].
- Бэкап выполняет агент на хосте БД, вызывая локально установленный инструмент. Требования для MySQL: «Install and run the PMM Client on the node», «Grant pmm-agent read/write permissions to /var/lib/mysql», установить `xtrabackup`, `xbcloud`, `xbstream` и добавить их в `$PATH` [M2]. В исходниках агента задача MySQL-бэкапа запускает `xtrabackup` и передаёт поток в `xbcloud` через `exec.CommandContext` [M3]. Для MongoDB дополнительно на всех узлах должен работать отдельный агент самого инструмента бэкапа — `pbm-agent` [M4].
- Связь инициирует агент, и она постоянная: «pmm-agent keeps a two-way gRPC stream open to pmm-managed. pmm-managed sends configuration and on-demand actions, and pmm-agent sends status and QAN data» [M5]. «By default, pmm-agent and vmagent connect to PMM Server over HTTPS, on port 443 or 8443» [M5].
- Долгие операции — «jobs» с таймаутом и обратным каналом: интерфейс `Job` в агенте имеет методы `Timeout()` и `Run(ctx, send)`, где `send` — «function that used by jobs to send messages back to pmm-server»; типы задач: `mysql_backup`, `mongodb_backup`, `mongodb_restore`, `mysql_restore` [M6].
- Жёсткие предположения об окружении хоста: один экземпляр MySQL на узле, сервис systemd с именем `mysql`/`mysqld`, данные в `/var/lib/mysql` [M2]; для MongoDB — `mongod` под `systemctl` и «PMM only works with a single mongod installed on a node» [M4].

Вывод: PMM не решает нашу задачу напрямую (нет PostgreSQL), но это хорошо документированный образец модели «агент сам держит соединение с сервером, сервер шлёт по нему задачи, агент локально запускает CLI инструмента бэкапа и шлёт обратно ход и результат».

### 1.6. Платформа Tantor — свой агент с исходящим соединением через NATS; бэкапы отданы RuBackup

Факты:

- Агент — «легковесное приложение на Golang — собирает информацию с базы данных и операционной системы и отправляет данные с изменениями на серверную часть Платформы… Агент обеспечивает управление и конфигурацию СУБД, а также выполнение заранее запрограммированных задач и задач по расписанию» [T1].
- Соединение исходящее, от агента к платформе: «Внешний агент (Platform Agent) подключается к nginx через TCP:4222 и HTTPS:443»; по этим каналам идут «управляющие команды, телеметрия, пакеты установки новых версий агента». TCP:4222 — это публикация брокера сообщений NATS; «Все входящие внешние подключения (от агента, браузера) проходят через nginx (Reverse Proxy)» [T1].
- Установка агента — вручную на хосте БД по командам, которые показывает веб-интерфейс («Зайдите на сервер базы данных, на котором будет установлен Агент, как пользователь из группы sudo»), затем регистрация экземпляра командой `pmaagent instances add` [T2]. Есть отдельный способ «Установка Агента с помощью Ansible» и офлайн-установка из локального архива [T2].
- Бэкапы платформа сама не выполняет: «Для создания резервных копий баз данных и их восстановления в Платформе реализована интеграция с Rubackup… Настроить резервное копирование и жизненный цикл резервных копий можно в системе Rubackup» [T3]. Подключение к RuBackup задаётся адресом, портом, логином и паролем сервера системы резервного копирования [T4].

Вывод: Tantor — пример «агент для управления и мониторинга, а бэкап — внешняя система со своим собственным клиентом на хосте». Ansible у них — способ доставить агента, а не канал управления.

### 1.7. temBoard (Dalibo) — свой агент с HTTPS API, сервер ходит к агенту; бэкапами не управляет

Факты:

- «temBoard is composed of 2 services: A lightweight agent to install on every PostgreSQL server to monitor and manage. A central server controlling the agents, collecting metrics and presenting it on a web UI» [TB1]. В списке возможностей управления бэкапами нет [TB1].
- Агент обязан стоять на том же хосте: «The temBoard agent must run on the same host as the PostgreSQL instance. Running an agent on a remote host is not yet supported» [TB2].
- Агент слушает порт: `port` — «port number that the agent will listen on to serve its HTTP API. Default: 2345», со своим TLS-сертификатом (`ssl_cert_file`, `ssl_key_file`) [TB3].
- Доверие построено на асимметричной подписи запросов, а не на общем секрете: «An agent accepts requests from a single temBoard UI. temBoard UI signs each requests using an asymetric key. Use `temboard-agent fetch-key` to download the signing key from the configured UI. Or push the signing using a configuration management service» [TB2]. В коде: UI добавляет заголовок `X-TemBoard-Signature: v1:…` [TB4], агент отвергает запросы без подписи или с неверной подписью (HTTP 400 / 403) [TB5].
- Агент подключается к PostgreSQL локально через unix-сокет суперпользователем: «temboard-agent requires superuser access using local UNIX socket only» [TB3]. Запускается как systemd-сервис `temboard-agent@<версия>-<кластер>` [TB2].
- Установка: пакет из репозитория Dalibo (есть инструкция для офлайн-установки), скрипт `auto_configure.sh` от root, затем `fetch-key` и `register` [TB2].

### 1.8. Pigsty — Ansible с admin-узла; расписание и состояние живут на хосте

Факты:

- Вся система описывается инвентарём и выполняется плейбуками: «Pigsty uses config inventory and config parameters to describe the entire deployment environment, implemented via Ansible playbooks» [G1].
- Требование к доступу: «Pigsty requires an OS admin user with passwordless SSH and Sudo privileges on all managed nodes»; «Ansible relies on sudo to execute commands with root privileges on managed nodes» [G2]. Документация признаёт, что это конфликтует с некоторыми политиками безопасности: «Some enterprise security policies may prohibit passwordless ssh or sudo» [G2].
- Бэкап — pgBackRest; Ansible используется для настройки и для восстановления (плейбук `pgsql-pitr.yml`), а регулярные бэкапы запускает cron на самом узле: `pg_crontab: [ '00 01 * * * /pg/bin/pg-backup full' ]` [G3].
- Состояние собирается не через Ansible, а экспортёром метрик на узле: «The `pgbackrest_exporter` service on port 9854 continuously exports metrics such as the latest backup time, type, size, and error status» [G4].
- Плейбук не является надёжным источником правды о результате: начальный бэкап «the playbook ignores a failed attempt and leaves no marker… always verify actual repository state with `pig pb info` or `pgbackrest info`» [G4].

Вывод: даже в продукте, целиком построенном на Ansible, Ansible отвечает только за «привести хост в нужное состояние» и за редкие разовые операции. Долгие регулярные операции и наблюдение вынесены в локальные механизмы хоста (cron, exporter).

### 1.9. ClusterControl (Severalnines) — SSH без агента для управления, агенты (exporters) для мониторинга

Факты (цитаты получены через автоматический пересказ страницы, а не прямым чтением HTML — страница отрисовывается JavaScript; см. раздел 4):

- «ClusterControl executes jobs by sending remote commands via SSH to the designated target node»; «This agentless management relies on passwordless SSH (key-based authentication), utilizing the `ssh_identity` key and `ssh_user` defined in the CMON configuration»; «This mechanism requires no agent which simplifies the configuration of the whole infrastructure» [C1].
- Для мониторинга продукт от SSH отказался: «ClusterControl used to support both agentless (SSH sampling) and agent-based monitoring. However, since v1.9.8, ClusterControl defaults to agent-based monitoring using Prometheus. Agentless monitoring is no longer recommended» [C1].

Вывод: это единственный найденный коммерческий продукт с веб-интерфейсом, который управляет хостами по SSH, и он же показывает практический предел подхода — регулярный сбор состояния по SSH они сами перестали рекомендовать.

### 1.10. PG Back Web — без доступа к хосту вообще, только логические дампы

Факты: продукт поставляется как один Docker-образ, работает через `pg_dump` («backed by the robust pg_dump tool») и складывает дампы локально или в S3 («Local & S3 storage») [W1]. Никакого компонента для установки на хост БД README не описывает.

Вывод: раз используется `pg_dump` и на хост ничего не ставится, сервис подключается к базам по обычному SQL-соединению. Это другой класс задач (логический бэкап по SQL-соединению). К WAL-G, который должен читать PGDATA на хосте БД, такой подход неприменим; упомянут только чтобы закрыть вопрос «а нельзя ли вообще без доступа к хосту».

### 1.11. WAL-G — как он сам соотносится с этими моделями

Факты:

- WAL-G — CLI, запускаемый на хосте БД; собственного режима «центральный сервер — хост» с SSH или агентом, как у pgBackRest/pg_probackup, в документации нет. Есть «remote backup» через протокол `BASE_BACKUP` PostgreSQL, но с ограничениями: «the BASE_BACKUP protocol does not allow for multithreaded streaming, and … Delta backup currently is not implemented» [WG1].
- Машиночитаемое состояние отдаётся командами: `wal-g backup-list --pretty --detail --json`, `wal-show --detailed-json`, `wal-verify --json` [WG1].
- `wal-g daemon` — это локальный процесс для ускорения архивации WAL через unix-сокет («archives and fetches WAL segments in response to commands sent over a UNIX socket»), а не сетевой агент управления [WG1].

Вывод: при нашей политике с дельта-бэкапами WAL-G обязан запускаться локально на хосте БД, значит, Серверу управления в любом случае нужен способ выполнить команду на хосте БД; режим «только по сети через протокол репликации» отпадает. При этом список бэкапов Сервер управления теоретически может получать и без хоста БД — читая Хранилище напрямую тем же `wal-g backup-list --json` (это следствие из того, что WAL-G хранит всё в S3; отдельно не проверялось).

---

## 2. Сводная таблица

| Продукт | Что управляет | Канал до хоста БД | Что работает на хосте БД | Кто открывает соединение | Бэкапы PostgreSQL |
|---|---|---|---|---|---|
| Barman | CLI на сервере Barman | SSH (rsync) и/или протокол репликации | ничего постоянного; `barman-cli`/`barman-cloud` по вызову | обе стороны (SSH в обе стороны) | да |
| pgBackRest | CLI на repository host | SSH (по умолчанию) или свой TLS-сервер | в TLS-режиме — демон `pgbackrest server` | обе стороны | да |
| pg_probackup | CLI на хосте с каталогом копий | только SSH | временные remote agents на время операции | обе стороны | да |
| PPEM | веб + manager | HTTPS, опционально mTLS | постоянный агент (`ppem-agent`) + pg_probackup | обе стороны | да (pg_probackup) |
| Percona PMM | веб + PMM Server | gRPC-поток поверх HTTPS | постоянный `pmm-agent` + CLI инструмента | хост БД → сервер | нет (MySQL, MongoDB) |
| Платформа Tantor | веб + backend | NATS (TCP:4222) и HTTPS:443 | постоянный агент (`pmaagent`) | хост БД → платформа | через RuBackup |
| temBoard | веб + сервер | HTTPS API агента, подписанные запросы | постоянный агент | сервер → хост БД | нет |
| Pigsty | Ansible с admin-узла | SSH + sudo (Ansible) | cron, `pgbackrest_exporter` | admin-узел → хост БД | да (pgBackRest) |
| ClusterControl | веб + контроллер | SSH без пароля; для метрик — exporters | exporters Prometheus | контроллер → хост БД | да |
| PG Back Web | веб | SQL-соединение (`pg_dump`) | ничего | сервер → БД | только логические |

---

## 3. Сравнение подходов

Ниже — выводы; в скобках указано, на какие факты они опираются.

### 3.1. Свой агент на хосте БД

Кто так делает: PPEM, PMM, Tantor, temBoard; по сути — и TLS-режим pgBackRest.

Плюсы:

- **Безопасность.** На хосте нет интерактивного входа для Сервера управления: агент выполняет только заранее запрограммированные действия (Tantor: «выполнение заранее запрограммированных задач» [T1]). Агент может работать от `postgres` без root (PPEM [E5]). Если соединение открывает агент (PMM [M5], Tantor [T1]), на хосте БД не нужно открывать входящий порт. Доверие можно сделать точечным: подпись запросов одним сервером (temBoard [TB2]), mTLS с проверкой CN (pgBackRest [P2], PPEM [E3]).
- **Долгие операции.** Операция живёт в процессе агента на хосте и не зависит от того, держится ли соединение с Сервером управления; ход и результат передаются как задача/job (PMM [M6], PPEM [E8]). Вывод: это главный практический довод — бэкап на часы не должен умирать от разрыва сети или перезапуска Сервера управления.
- **Сбор состояния.** Агент сам регулярно шлёт состояние (PPEM [E3], PMM [M5], Tantor [T1]); свежесть данных не зависит от опроса.

Минусы:

- **Установка и сопровождение.** На каждый хост надо доставить пакет, конфиг, сервис, секрет/ключ и потом обновлять (temBoard: пакет + `auto_configure.sh` + `fetch-key` + `register` [TB2]; Tantor: ручные команды или Ansible, отдельная страница про обновление агента [T2]). Вывод: у продуктов с агентом всё равно появляется второй механизм — для доставки самого агента (Tantor прямо предлагает Ansible [T2]; temBoard — «push the signing using a configuration management service» [TB2]).
- **Свой протокол и свои баги.** Нужно проектировать аутентификацию, версии, совместимость агента и сервера. Вывод: это собственный код с привилегиями на хосте БД, то есть новая поверхность атаки, которую надо защищать самим (в отличие от sshd).
- **Агент не снимает требований к хосту.** Инструмент бэкапа всё равно ставится отдельно (PPEM: pg_probackup ставится и настраивается на серверах вручную [E7]; PMM: `xtrabackup` в `$PATH` [M2]), и продукты накладывают жёсткие предположения об окружении (PMM [M2], [M4]).

Вариант «кто открывает соединение»:

- сервер → агент (temBoard, PPEM в одном из направлений): проще модель «запрос-ответ», но на каждом хосте БД открыт порт;
- агент → сервер (PMM, Tantor): на хосте БД нет входящих портов, один открытый адрес на Сервере управления, но нужен постоянный поток или брокер (gRPC-поток у PMM, NATS у Tantor).

### 3.2. SSH без агента

Кто так делает: Barman, pgBackRest (по умолчанию), pg_probackup — между хостом с копиями и хостом БД; ClusterControl — для управления.

Плюсы:

- **Установка.** На хосте БД не нужен новый сервис: sshd уже есть, достаточно ключа (ClusterControl: «requires no agent which simplifies the configuration» [C1]).
- **Безопасность транспорта.** Проверенный протокол, не надо писать свой. Ключ можно сузить до одной программы через `command=` в `authorized_keys` (pgBackRest [P1]).

Минусы:

- **Безопасность по умолчанию.** Беспарольный ключ без ограничений — это полноценная оболочка пользователя `postgres` (а у ClusterControl/Pigsty — пользователя с sudo) на всех хостах БД с одной машины. И pgBackRest [P1], и pg_probackup [PB1] отдельно пишут про ограничение SSH на случай компрометации учётной записи; Pigsty признаёт, что беспарольные ssh/sudo запрещены частью корпоративных политик [G2]. Вывод: компрометация Сервера управления при неограниченных ключах означает компрометацию всех хостов БД.
- **Долгие операции.** Вывод: процесс, запущенный через SSH-сеанс, привязан к этому сеансу; чтобы бэкап пережил разрыв, его всё равно надо отвязывать на стороне хоста (например, запускать как transient-юнит systemd и потом опрашивать) — то есть на хосте появляется «полуагент» из юнитов и скриптов. У самих инструментов бэкапа эта проблема решена тем, что данные идут по тем же SSH-соединениям и обрыв просто прерывает операцию с ошибкой (pg_probackup: «all agents are terminated and error details are reported» [PB1]).
- **Сбор состояния.** Только опросом. ClusterControl от регулярного сбора по SSH отказался в пользу агентов-экспортёров: «Agentless monitoring is no longer recommended» [C1].
- **Мелкие практические ограничения.** Лимиты sshd на число соединений (`MaxStartups`) при параллельной работе [PB1]; необходимость одинаковых версий инструмента на обеих сторонах [PB1].

### 3.3. Ansible

Кто так делает: Pigsty (всё управление); Tantor — только для установки агента.

Плюсы:

- **Установка и настройка.** Декларативно и идемпотентно раскладывает пакеты, конфиги, юниты, cron; хорошо подходит для первичной настройки WAL-G и применения Политики бэкапа как файлов на хосте (Pigsty [G1], [G3]).
- **Не нужен свой код на хосте.**

Минусы:

- **Безопасность.** Те же SSH-ключи плюс sudo: «Ansible relies on sudo to execute commands with root privileges on managed nodes» [G2]. Права шире, чем у агента под `postgres`.
- **Долгие операции и состояние.** Pigsty сам не использует Ansible ни для регулярных бэкапов (cron на узле [G3]), ни для сбора состояния (exporter [G4]), и предупреждает, что результат плейбука не равен реальному состоянию [G4]. Вывод: Ansible — инструмент доставки конфигурации, а не канал оперативного управления и наблюдения.
- **Вывод:** для веб-сервиса Ansible означает запуск внешнего процесса с разбором его вывода; отзывчивого статуса «идёт бэкап, 40%» из него не получить.

### 3.4. Гибриды, которые реально встречаются

- **«Настроил один раз — дальше хост сам»** (Pigsty): управляющая машина только раскладывает конфигурацию и расписание (cron/systemd timer), хост сам запускает бэкапы, состояние читается отдельным лёгким каналом (exporter) [G3], [G4]. Вывод: это снимает проблему долгих операций без своего агента, но запуск по требованию и восстановление всё равно требуют канала команд.
- **«Агент управляет, но доставляется Ansible/пакетами»** (Tantor, temBoard) [T2], [TB2].
- **«Состояние — из хранилища, а не с хоста»**: pgBackRest и WAL-G отдают список бэкапов в JSON [P1], [WG1]. Вывод: для WAL-G с S3 Сервер управления может сверять фактическое наличие бэкапов в Хранилище независимо от канала до хоста БД — полезная независимая проверка при любом выбранном подходе (см. оговорку в 1.11).

### 3.5. Что из этого следует для выбора (вывод, не факт)

1. Отраслевой образец для задачи «веб-интерфейс запускает и контролирует физические бэкапы на десятках хостов» — постоянный агент на хосте БД (PPEM, PMM, Tantor). Чисто SSH-управление из веб-продукта нашлось одно (ClusterControl), и для сбора состояния он от SSH ушёл.
2. Реальный выбор для нас — не «агент против SSH против Ansible», а разделение трёх задач, которые продукты решают разными средствами: (а) доставка и настройка на хосте — пакеты/Ansible/SSH; (б) запуск долгих операций — процесс на хосте (агент или systemd/cron), не привязанный к соединению; (в) сбор состояния — пуш с хоста либо чтение Хранилища.
3. Если идти через SSH, минимум, который показывают первичные источники: отдельный ключ с `command=` и без форвардинга (pgBackRest [P1]), запуск операций так, чтобы они переживали разрыв сеанса, и отказ от частого опроса по SSH для состояния.
4. Если идти через агента, проверенные решения: агент под `postgres` без root (PPEM [E5]), исходящее соединение от хоста (PMM [M5], Tantor [T1]) или подпись запросов единственным сервером (temBoard [TB2]), задачи с таймаутом и обратным каналом (PMM [M6]).

---

## 4. Что не удалось подтвердить

- **Веб-интерфейсы над WAL-G.** Поиск по GitHub (`gh search repos "wal-g"`) и веб-поиск не дали ни одного проекта-надстройки с веб-интерфейсом и центральным сервером над WAL-G. Это отсутствие находки, а не доказательство отсутствия: закрытые внутренние системы (например, у облачных провайдеров, использующих WAL-G) публично не описаны в первичных источниках, которые я смотрел.
- **Веб-интерфейсы над pgBackRest и Barman.** Официальных нет; поиск по GitHub сторонних надстроек над pgBackRest вернул пустой результат. Целенаправленно сторонние проекты я дальше не искал.
- **ClusterControl.** Страница документации отрисовывается JavaScript; цитаты в разделе 1.9 получены через автоматический пересказ страницы инструментом WebFetch, дословность не сверена с исходным HTML. Как именно ClusterControl запускает и отслеживает долгий бэкап PostgreSQL по SSH, я не проверял.
- **PPEM.** Не нашёл в документации, как именно агент отслеживает ход долгого бэкапа и что происходит с задачей при перезапуске агента или manager. Способ установки агента (пакеты) подробно не изучал. Смотрел документацию версии 2.
- **Платформа Tantor.** Аутентификация агента в NATS/HTTPS и права, с которыми работает агент на хосте, в прочитанных страницах не описаны. Сайт документации отвечал нестабильно; страницу «Мониторинг бэкапов» и страницу установки через Ansible прочитать не удалось. Как устроен клиент RuBackup на хосте БД, не изучал (это отдельный продукт).
- **temBoard.** Направление передачи метрик мониторинга (агент шлёт на сервер или сервер забирает у агента) по прочитанным страницам однозначно не установил; подтверждено только, что агент слушает порт и сервер шлёт ему подписанные запросы.
- **PMM.** Поведение job при обрыве gRPC-потока (продолжается ли бэкап, как доставляется результат) не проверял — в исходниках смотрел только интерфейс `Job` и задачу MySQL-бэкапа.
- **Pigsty.** Не проверял, есть ли у Pigsty веб-интерфейс для запуска бэкапов; по прочитанным страницам управление идёт из командной строки (плейбуки и утилита `pig`).
- **WAL-G: чтение состояния из Хранилища с другой машины** — логическое следствие из устройства WAL-G, на практике не проверялось.
- Не рассматривались: EDB Postgres Enterprise Manager, pgmoneta, Databasus/Postgresus, Bacula/Bareos, решения для Kubernetes (операторы). Операторы Kubernetes вне области (у нас одиночные серверы на Linux с systemd).

---

## Источники

Barman 3.14 (EnterpriseDB):

- [B1] Pre-requisites — https://docs.pgbarman.org/release/3.14.0/user_guide/pre_requisites.html
- [B2] Concepts — https://docs.pgbarman.org/release/3.14.0/user_guide/concepts.html
- [B3] Barman for the cloud — https://docs.pgbarman.org/release/3.14.0/user_guide/barman_cloud.html

pgBackRest:

- [P1] User Guide (Debian/Ubuntu), разделы «Dedicated Repository Host», «Monitoring», «Information» — https://pgbackrest.org/user-guide.html
- [P2] Configuration Reference, опции `repo-host-type`, `pg-host-type`, `tls-server-auth`, `repo-host-port` — https://pgbackrest.org/configuration.html
- [P3] User Guide (RHEL), разделы «Setup TLS», «Setup TLS Server» — https://pgbackrest.org/user-guide-rhel.html

pg_probackup (Postgres Professional):

- [PB1] pg_probackup, разделы «Configuring the Remote Mode», «Using pg_probackup in the Remote Mode» — https://postgrespro.com/docs/postgrespro/17/app-pgprobackup

Postgres Pro Enterprise Manager 2:

- [E1] High-Level Architecture — https://postgrespro.com/docs/ppem/2/ppem-high-level-architecture.html
- [E2] Manager and Agent Architecture — https://postgrespro.com/docs/ppem/2/ppem-manager-and-agent-architecture.html
- [E3] Security: Networking — https://postgrespro.com/docs/ppem/2/ppem-networking.html
- [E4] Authentication and Authorization Operation Scheme — https://postgrespro.com/docs/ppem/2/ppem-authentication-and-authorization-operation-scheme.html
- [E5] Security: Manager and Agent — https://postgrespro.com/docs/ppem/2/ppem-manager-and-agent.html
- [E6] Installation in Secure Environments — https://postgrespro.com/docs/ppem/2/ppem-installation-in-secure-environments.html
- [E7] Installation and Configuration of Backup and Restore Tools — https://postgrespro.com/docs/ppem/2/ppem-installation-and-configuration-of-backup-and-restore-tools.html
- [E8] Task Console — https://postgrespro.com/docs/ppem/2/ppem-task-console.html
- [E9] Backup — https://postgrespro.com/docs/ppem/2/ppem-backup.html

Percona Monitoring and Management 3:

- [M1] Backup and restore — https://docs.percona.com/percona-monitoring-and-management/3/backup/index.html
- [M2] MySQL backup prerequisites — https://github.com/percona/pmm/blob/main/documentation/docs/backup/mysql-backup/mysql_prerequisites.md
- [M3] Исходный код задачи MySQL-бэкапа в агенте — https://github.com/percona/pmm/blob/main/agent/runner/jobs/mysql_backup_job.go
- [M4] MongoDB backup prerequisites — https://github.com/percona/pmm/blob/main/documentation/docs/backup/mongodb-backup/mongo_prerequisites.md
- [M5] Reference: PMM architecture, «How PMM Client and PMM Server interact», «Connection security» — https://docs.percona.com/percona-monitoring-and-management/3/reference/index.html
- [M6] Интерфейс `Job` в агенте — https://github.com/percona/pmm/blob/main/agent/runner/jobs/job.go

Платформа Tantor 7.0:

- [T1] Архитектура Платформы — https://docs.tantorlabs.ru/tp/7.0/platform_architecture.html
- [T2] Добавление экземпляра и установка Агента — https://docs.tantorlabs.ru/tp/7.0/instances/instance_adding.html (оглавление с пунктом «Установка Агента с помощью Ansible»: https://docs.tantorlabs.ru/tp/7.0/index.html)
- [T3] Резервное копирование (обзор) — https://docs.tantorlabs.ru/tp/7.0/admin/backup_info.html
- [T4] Резервное копирование (настройка подключения) — https://docs.tantorlabs.ru/tp/7.0/admin/backup.html

temBoard (Dalibo):

- [TB1] Главная страница документации — https://github.com/dalibo/temboard/blob/master/docs/index.md
- [TB2] Agent installation — https://github.com/dalibo/temboard/blob/master/docs/agent_install.md
- [TB3] Configure the agent — https://github.com/dalibo/temboard/blob/master/docs/agent_configure.md
- [TB4] Клиент UI → агент — https://github.com/dalibo/temboard/blob/master/ui/temboardui/agentclient.py
- [TB5] Проверка подписи в агенте — https://github.com/dalibo/temboard/blob/master/agent/temboardagent/web/app.py

Pigsty:

- [G1] Architecture — https://doc.pigsty.io/docs/concept/arch/
- [G2] Admin user, sudo, SSH — https://doc.pigsty.io/docs/deploy/admin/
- [G3] Backup & Restore — https://doc.pigsty.io/docs/pgsql/backup/
- [G4] Backup: Admin Commands — https://doc.pigsty.io/docs/pgsql/backup/admin/

ClusterControl (Severalnines):

- [C1] How it works — https://docs.severalnines.com/clustercontrol/latest/getting-started/overview/how-it-works

PG Back Web:

- [W1] README — https://github.com/eduardolat/pgbackweb/blob/main/README.md

WAL-G:

- [WG1] WAL-G for PostgreSQL — https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md
