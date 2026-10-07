# Возможности и ограничения WAL-G для PostgreSQL

Исследование по тикету [#3](https://github.com/dimda7/backup/issues/3). Дата: 2026-10-07.

## К чему относятся факты

- **WAL-G v3.0.9** — последний релиз на дату исследования (опубликован 2026-08-20, тег `v3.0.9`, коммит `3e493188`). Все ссылки на документацию и исходный код WAL-G ведут на этот тег.
- **PostgreSQL** — официальная документация, страницы `/docs/current/` (на дату исследования это версия 18). Построчно со страницами для версии 13 не сверялось; см. раздел «Что не удалось подтвердить».
- Ничего не запускалось: выводы сделаны по документации и чтению исходного кода, без стенда.

Пометки в тексте:

- **[Д]** — документированный факт (документация WAL-G или PostgreSQL).
- **[К]** — факт из исходного кода WAL-G v3.0.9, в документации не описан.
- **[В]** — вывод автора из фактов, а не утверждение первоисточника.

## Краткий ответ

WAL-G закрывает всю «механику» бэкапа: полный и дельта-бэкап, архивация и получение WAL, удаление по правилам хранения, проверка непрерывности архива WAL, скачивание бэкапа в каталог. Состояние отдаётся в JSON тремя командами (`backup-list --detail --json`, `wal-show --detailed-json`, `wal-verify --json`).

WAL-G **не** делает: расписание, PITR как отдельную команду (точку восстановления задаёт сам PostgreSQL через `recovery_target_*`), подготовку каталога и запуск PostgreSQL при восстановлении, проверку бэкапа тестовым восстановлением, блокировку от параллельных запусков, отчёт о ходе выполняющегося бэкапа. Всё это должен взять на себя сервер управления.

## 1. Команды

### 1.1. Бэкап: `backup-push`

- **[Д]** `wal-g backup-push $PGDATA` делает бэкап и выгружает его в хранилище; флаг `--permanent` защищает бэкап от `delete` ([PostgreSQL.md, backup-push](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#backup-push)).
- **[Д]** Дельта-бэкап включается настройкой `WALG_DELTA_MAX_STEPS` — «сколько дельта-бэкапов может быть между полными», по умолчанию 0 (дельты выключены). База для дельты — `WALG_DELTA_ORIGIN`: `LATEST` (цепочка) или `LATEST_FULL` (каждая дельта от последнего полного). Дельта считается по времени изменения файлов и LSN страниц ([PostgreSQL.md, Configuration](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#configuration)).
- **[Д]** Базу для дельты можно указать явно: `--delta-from-name`, `--delta-from-user-data` ([PostgreSQL.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#create-delta-backup-from-specific-backup)).
- **[К]** Есть флаг `--full` / `-f` («Make full backup-push») — принудительно полный бэкап; в документации не описан ([cmd/pg/backup_push.go](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/pg/backup_push.go#L158)). Там же флаг `--add-user-data` — записать произвольный JSON в метаданные бэкапа.
- **[Д]** Два режима чтения данных ([PostgreSQL.md, Remote backup](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#remote-backup)):
  1. локально, от пользователя `postgres`, прямо с файловой системы — быстрее, доступны дельта-бэкапы и частичное восстановление;
  2. удалённо через протокол `BASE_BACKUP` (нужна роль с правом репликации) — однопоточно, **дельта-бэкап не реализован**.
- **[Д]** Проверка контрольных сумм страниц при бэкапе: `--verify` / `WALG_VERIFY_PAGE_CHECKSUMS`; номера повреждённых блоков (не более 10 на файл) пишутся в JSON-сентинел бэкапа ([PostgreSQL.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#page-checksums-verification)). **[К]** Флаг `--store-all-corrupt-blocks` снимает ограничение в 10.
- **[Д]** `WALG_SENTINEL_USER_DATA` — произвольный JSON в сентинеле, «для средств автоматизации»; по нему потом можно выбирать бэкап в `backup-fetch --target-user-data` и `delete target --target-user-data`.
- **[Д]** `--without-files-metadata` экономит память на базах с >100 тыс. файлов, но **несовместим с дельта-бэкапами**.
- **[Д]** `WALG_ALIVE_CHECK_INTERVAL` (по умолчанию 1 мин): если PostgreSQL перестал отвечать, `backup-push` завершается. `WALG_STOP_BACKUP_TIMEOUT` — таймаут `pg_stop_backup()`, по умолчанию без таймаута.
- **[Д]** Бэкап со standby возможен; при смене таймлайна во время бэкапа данные выгружаются, но бэкап не финализируется, WAL-G выходит с ошибкой.

### 1.2. Архивация и получение WAL

- **[Д]** `wal-g wal-push <путь>` — для `archive_command`; `wal-g wal-fetch <имя> <куда>` — для `restore_command` ([PostgreSQL.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#wal-push)).
- **[Д]** `wal-fetch` возвращает код **74**, если сегмента нет в хранилище; все остальные ошибки — код 1. Документация WAL-G прямо говорит, что для остановки PostgreSQL при «настоящей» ошибке нужен код 126–255, и это «достигается простым скриптом-обёрткой» ([PostgreSQL.md, wal-fetch](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#wal-fetch), [PR #1195](https://github.com/wal-g/wal-g/pull/1195)).
- **[Д]** `wal-fetch` скачивает сегменты с упреждением (prefetch) в `./.wal-g/prefetch` внутри `pg_wal`; каталог меняется через `WALG_PREFETCH_DIR`.
- **[Д]** За один вызов `wal-push` может выгрузить в фоне до 32 готовых сегментов (`TOTAL_BG_UPLOADED_LIMIT`), параллельность — `WALG_UPLOAD_CONCURRENCY` (по умолчанию 16).
- **[Д]** `WALG_PREVENT_WAL_OVERWRITE`: перед выгрузкой проверять, нет ли в хранилище другого файла с тем же именем; если есть — ненулевой код. **По умолчанию выключено.**
- **[Д]** `wal-receive` — альтернатива `archive_command`: приём WAL по потоковой репликации через слот (`WALG_SLOTNAME`, по умолчанию `walg`).
- **[Д]** `wal-g daemon <сокет>` + лёгкий клиент `walg-daemon-client` — чтобы не платить за запуск полного бинарника на каждый сегмент; для PostgreSQL 15+ есть расширение `walg_archive` (через `archive_library`). Клиент собирается отдельно (`make build_client`) ([PostgreSQL.md, daemon](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#daemon)).

### 1.3. Удаление по правилам хранения: `delete`

Источник: [README.md, delete](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/README.md#delete), [PostgreSQL.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#delete-retention-ordering-and-use-sentinel-time).

- **[Д]** `delete` удаляет бэкапы **и WAL до них**. По умолчанию — «сухой» прогон; реальное удаление только с `--confirm`. Постоянные (permanent) бэкапы не удаляются.
- **[Д]** Режимы:
  - `retain [FULL|FIND_FULL] N [--after имя|время]` — оставить N последних; `FULL` — считать только полные и хранить все их дельты; `--after` — дополнительно сохранить всё, что новее метки;
  - `before [FIND_FULL] имя` — удалить всё до указанного бэкапа;
  - `target [FIND_FULL] имя | --target-user-data` — удалить конкретный бэкап и зависящие от него дельты; **WAL не удаляет**;
  - `everything [FORCE]`;
  - `garbage [ARCHIVES|BACKUPS]` — подчистить устаревшие WAL и остатки неудачных или частично удалённых бэкапов.
- **[Д]** `retain 5` **завершится ошибкой**, если пятый бэкап — дельта; нужно `retain FULL 5` или `retain FIND_FULL 5`.
- **[Д]** Порядок бэкапов по умолчанию берётся из таймлайна и номера сегмента в имени, а не из времени. После `pg_upgrade` таймлайн начинается заново, и `retain` может удалить **новые** бэкапы вместо старых ([issue #636](https://github.com/wal-g/wal-g/issues/636)). Защита: флаг `--use-sentinel-time` и отдельный префикс на каждую мажорную версию PostgreSQL.
- **[Д]** `backup-mark <имя>` делает бэкап и всю его цепочку постоянными, `-i` — снимает отметку.

### 1.4. Проверка целостности

- **[Д]** `wal-verify integrity` проверяет, что в хранилище есть все сегменты от начала самого старого бэкапа до текущего сегмента кластера. Статусы сегментов: `FOUND`, `MISSING_DELAYED`, `MISSING_UPLOADING`, `MISSING_LOST`; итог — `OK` / `WARNING` / `FAILURE` ([PostgreSQL.md, wal-verify](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#wal-verify)).
- **[Д]** `wal-verify timeline` сравнивает текущий таймлайн кластера с максимальным в хранилище (обнаружение split-brain). Корректно работает, только если хранилище создано заново или очищено после восстановления из бэкапа или `pg_upgrade`.
- **[К]** `wal-verify` подключается к работающему PostgreSQL, чтобы узнать текущий сегмент ([wal_verify_handler.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/databases/postgres/wal_verify_handler.go#L91)); без флагов `--timeline` и `--lsn` на остановленном хосте БД не выполнится.
- **[Д]** `wal-show` показывает таймлайны в хранилище, бэкапы на них и пропущенные сегменты; итог `OK` или `LOST_SEGMENTS`. **[К]** Работает только с хранилищем, подключение к PostgreSQL не нужно ([cmd/pg/wal_show.go](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/pg/wal_show.go)).
- **[К]** `wal-g st check read` / `st check write` — проверка доступа к хранилищу ([cmd/common/st/check.go](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/common/st/check.go)).
- **Команды, которая проверяла бы сам бэкап (скачала, сверила контрольные суммы, попробовала поднять), в WAL-G для PostgreSQL нет** — в `cmd/pg` v3.0.9 такой команды нет, в документации она не описана. `--verify` проверяет страницы на исходном хосте БД в момент бэкапа, а не то, что лежит в хранилище.

### 1.5. Восстановление и PITR

- **[Д]** `wal-g backup-fetch <каталог> <имя | LATEST>` скачивает бэкап; для дельты сам подтягивает полный бэкап и все промежуточные дельты. Выбор по метаданным — `--target-user-data` ([PostgreSQL.md, backup-fetch](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#backup-fetch)).
- **[К]** Каталог назначения должен быть **пустым**, иначе ошибка `Directory ... for delta base must be empty` ([backup_fetch_handler_new.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/databases/postgres/backup_fetch_handler_new.go#L30-L37)).
- **[К]** `backup-fetch` для PostgreSQL **не создаёт `recovery.signal` и не прописывает `restore_command`** — в коде PostgreSQL-части таких действий нет (они есть только в коде Greenplum). В бэкап не попадают `pg_wal`, `postmaster.pid`, `recovery.conf`, `standby.signal`, `pg_replslot`, каталоги логов ([bundle.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/databases/postgres/bundle.go#L46-L52)).
- **[К]** Флаг `--restore-spec <файл>` — спецификация размещения табличных пространств при восстановлении; в документации не описан.
- **[Д]** Экспериментальное частичное восстановление: `--restore-only=база,база/таблица`. Бета: `--reverse-unpack`, `--skip-redundant-tars`.
- **У WAL-G нет команды PITR.** Восстановление на момент времени делает PostgreSQL:
  - **[Д, PostgreSQL]** после раскладки файлов нужно задать параметры восстановления в `postgresql.conf` и создать `recovery.signal`; `restore_command` обязателен ([Continuous Archiving, 25.3.5](https://www.postgresql.org/docs/current/continuous-archiving.html#BACKUP-PITR-RECOVERY));
  - **[Д, PostgreSQL]** цель задаётся одним из `recovery_target`, `recovery_target_time`, `recovery_target_lsn`, `recovery_target_name`, `recovery_target_xid`; `recovery_target_timeline` по умолчанию `latest`; `recovery_target_action` по умолчанию `pause` ([Recovery Target](https://www.postgresql.org/docs/current/runtime-config-wal.html#RUNTIME-CONFIG-WAL-RECOVERY-TARGET));
  - **[Д, PostgreSQL]** точка остановки должна быть позже окончания бэкапа: «нельзя использовать базовый бэкап для восстановления на момент, когда этот бэкап ещё выполнялся».
- **[Д]** `wal-g copy --from=... --to=... [--backup-name] [--with-history]` копирует бэкапы и нужные им WAL между хранилищами без перешифровки и пережатия; повторный запуск докачивает новое ([PostgreSQL.md, copy](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#copy)).

## 2. Параметры конфигурации

### 2.1. Способ задания

- **[Д]** Переменные окружения либо файл конфигурации (`--config <путь>` или `WALG_CONFIG_PATH`); форматы — JSON, YAML, envfile и прочие, что умеет viper. Любой параметр можно задать любым из двух способов ([README.md, Configuration](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/README.md#configuration)).
- **[Д]** Подключение к PostgreSQL — стандартные переменные libpq (`PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`/`PGPASSFILE`); для локального подключения рекомендуется сокет, `PGHOST=/var/run/postgresql`.

### 2.2. Хранилище (S3-совместимое)

Источник: [STORAGES.md, S3](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/STORAGES.md#s3).

- **[Д]** Обязателен `WALG_S3_PREFIX` (`s3://bucket/path/to/folder`) — это и есть префикс хоста БД.
- **[Д]** Учётные данные: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (или `~/.aws/credentials`).
- **[Д]** Для не-AWS: `AWS_ENDPOINT`, `AWS_S3_FORCE_PATH_STYLE` (по умолчанию `false`), `AWS_REGION` (иначе WAL-G вызывает `GetBucketLocation`), `WALG_S3_USE_LIST_OBJECTS_V1` (если хранилище не умеет ListObjectsV2), `S3_SKIP_VALIDATION`, `S3_ENABLE_VERSIONING`.
- **[Д]** Надёжность и нагрузка: `WALG_S3_MAX_RETRIES` (15), `WALG_S3_RANGE_BATCH_ENABLED` (докачка по HTTP Range), `S3_DELETE_BATCH_SIZE` (1000), `WALG_S3_STORAGE_CLASS`.
- **[Д]** Защита от удаления на стороне хранилища: `S3_RETENTION_PERIOD`, `S3_RETENTION_MODE` (`GOVERNANCE` / `COMPLIANCE`) — Object Lock.
- **[К]** Раскладка в префиксе: `basebackups_005/` и `wal_005/`; завершённый бэкап отмечен файлом `<имя>_backup_stop_sentinel.json` ([utility.go](https://github.com/wal-g/wal-g/blob/v3.0.9/utility/utility.go#L61-L69)).
- **[Д]** Несколько хранилищ: `WALG_FAILOVER_STORAGES` — **экспериментально**, только через файл конфигурации (вложенная структура), поддерживается в `wal-push`, `wal-fetch`, `wal-prefetch`, `backup-push`, `backup-fetch`, `backup-list`, `delete`. Основное хранилище используется всегда, пока живо; иначе — первое живое запасное. Явный выбор — `--target-storage` ([FailoverStorages.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/FailoverStorages.md)).

### 2.3. Сжатие

- **[Д]** `WALG_COMPRESSION_METHOD`: `lz4` (по умолчанию), `lzma`, `zstd`, `brotli`, `none`. `WALG_ZSTD_LEVEL`: `fastest`, `default`, `better`, `best` ([README.md, Compression](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/README.md#compression)).
- **[Д]** Brotli, libsodium и LZO — опции сборки (`USE_BROTLI`, `USE_LIBSODIUM`, `USE_LZO`). **[К]** Релизные бинарники собраны со всеми тремя ([release.yml](https://github.com/wal-g/wal-g/blob/v3.0.9/.github/workflows/release.yml#L9-L11)).

### 2.4. Шифрование

Источник: [README.md, Encryption](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/README.md#encryption), [STORAGES.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/STORAGES.md#s3).

- **[Д]** По умолчанию шифрования нет.
- **[Д]** На стороне клиента:
  - libsodium — `WALG_LIBSODIUM_KEY` или `WALG_LIBSODIUM_KEY_PATH`, ключ 32 байта, `WALG_LIBSODIUM_KEY_TRANSFORM` = `base64` / `hex` / `none`;
  - OpenPGP — `WALG_PGP_KEY` или `WALG_PGP_KEY_PATH`, `WALG_PGP_KEY_PASSPHRASE`; для `*-push` достаточно открытого ключа, для `*-fetch` нужен закрытый;
  - `WALG_GPG_KEY_ID` — **устарел**;
  - AWS KMS (`WALG_CSE_KMS_ID`) и Yandex Cloud KMS (`YC_CSE_KMS_KEY_ID`, конвертные PGP-ключи) — привязаны к облачным KMS.
- **[Д]** На стороне хранилища: `WALG_S3_SSE` (`AES256`, `aws:kms`), `WALG_S3_SSE_KMS_ID`.

### 2.5. Ограничение нагрузки

- **[Д]** `WALG_NETWORK_RATE_LIMIT` (байт/с, для `backup-push` и `backup-fetch`), `WALG_DISK_RATE_LIMIT` (чтение диска при `backup-push`), `WALG_UPLOAD_CONCURRENCY` (16), `WALG_UPLOAD_DISK_CONCURRENCY` (1), `WALG_DOWNLOAD_CONCURRENCY` (до 10), `WALG_TAR_SIZE_THRESHOLD` (1 ГБ).

## 3. Что отдаётся в машиночитаемом виде

| Источник | Что даёт | Требует PostgreSQL |
|---|---|---|
| `wal-g backup-list --detail --json` | список завершённых бэкапов | нет |
| `wal-g wal-show --detailed-json` | таймлайны, диапазоны сегментов, пропуски | нет |
| `wal-g wal-verify integrity timeline --json` | статус непрерывности архива и таймлайна | да |
| сентинел бэкапа в хранилище | метаданные, `user_data`, повреждённые блоки | нет |
| statsd (`WALG_STATSD_ADDRESS`) | метрики операций по UDP | — |
| Prometheus-экспортёр `cmd/pg/exporter` | метрики поверх трёх команд выше | да |
| `pg_stat_archiver` (PostgreSQL) | последний архивированный сегмент, сбои | да |

Подробности:

- **[К]** Поля `backup-list --detail --json`: `backup_name`, `time`, `wal_file_name`, `storage_name`, `start_time`, `finish_time`, `date_fmt`, `hostname`, `data_dir`, `pg_version`, `start_lsn`, `finish_lsn`, `is_permanent`, `system_identifier`, `uncompressed_size`, `compressed_size`, `user_data` (если задан) ([backup_time.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/backup_time.go#L11-L14), [backup_sentinel_dto.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/databases/postgres/backup_sentinel_dto.go#L71-L87), пример — [testdata/backup-list.json](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/pg/exporter/testdata/backup-list.json)).
- **[Д]** Тип бэкапа отдельным полем не отдаётся: дельта определяется по `_D_` в имени, после `_D_` стоит имя базового бэкапа ([exporter/README.md](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/pg/exporter/README.md#backup-type-detection)).
- **[К]** Поля `wal-show --detailed-json` по каждому таймлайну: `id`, `parent_id`, `switch_point_lsn`, `start_segment`, `end_segment`, `segments_count`, `missing_segments`, `backups`, `segment_range_size`, `status` ([wal_show_handler.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/databases/postgres/wal_show_handler.go#L21-L30)).
- **[Д]** `wal-verify --json`: по `integrity` — `status` и список диапазонов (`timeline_id`, `start_segment`, `end_segment`, `segments_count`, `status`); по `timeline` — `status`, `current_timeline_id`, `highest_storage_timeline_id` ([пример в PostgreSQL.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md#wal-verify)).
- **[Д]** Метрики экспортёра: `walg_backup_info`, `walg_backup_start_timestamp`, `walg_backup_finish_timestamp`, размеры, `walg_wal_verify_status`, `walg_wal_integrity_status`, `walg_storage_up`, `walg_storage_latency_seconds`, `walg_pitr_window_seconds`, `walg_errors_total` ([exporter/README.md](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/pg/exporter/README.md)). **[К]** Экспортёр — отдельная программа, которая просто вызывает `backup-list --detail --json`, `wal-verify integrity timeline --json` и `st check read`; в релиз v3.0.9 готовым бинарником **не входит**, собирается из исходников.
- **[Д, PostgreSQL]** `pg_stat_archiver`: `archived_count`, `last_archived_wal`, `last_archived_time`, `failed_count`, `last_failed_wal`, `last_failed_time`, `stats_reset` ([monitoring-stats](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ARCHIVER-VIEW)).
- **[К]** При пустом хранилище `backup-list` не считает это ошибкой ([backup_util.go](https://github.com/wal-g/wal-g/blob/v3.0.9/internal/backup_util.go#L38-L47)).

Чего в машиночитаемом виде **нет**:

- **Отставания архивации WAL во времени.** WAL-G сообщает только о пропущенных сегментах. **[В]** Отставание нужно считать самому: `pg_stat_archiver.last_archived_time` и `last_failed_time`, плюс число файлов `.ready` в `pg_wal/archive_status`.
- **Неудачных и выполняющихся бэкапов.** **[К]** Список строится по сентинелам, сентинел пишется в конце, поэтому незавершённый бэкап в `backup-list` не появляется. **[В]** Статус «идёт / упал» сервер управления узнаёт только из кода возврата и вывода процесса, который сам запустил.
- **Хода выполнения** `backup-push` и `backup-fetch` (проценты, байты) — в документации не описано.

## 4. Ограничения и подводные камни для архитектуры

1. **WAL-G должен работать на самом хосте БД, от `postgres`.** **[Д]** Удалённый режим не умеет дельта-бэкапы и однопоточен. **[В]** Значит, сервер управления не может «бэкапить по сети»: ему нужен способ исполнять команды на хосте БД (агент либо SSH) и класть туда файл конфигурации.

2. **Расписания в WAL-G нет.** **[В]** В документации и списке команд планировщика нет; `daemon` обслуживает только WAL. Расписание политики бэкапа — это либо systemd-таймеры на хосте БД, либо запуск с сервера управления. От выбора зависит, продолжатся ли бэкапы при недоступности сервера управления.

3. **Полный или дельта — выбирает WAL-G, а не вызывающий.** **[Д]** Тип определяется `WALG_DELTA_MAX_STEPS` и наличием базы. **[В]** Запуск «дельты» при исчерпанном числе шагов или без подходящей базы даст полный бэкап — с другим временем и объёмом. Для полного бэкапа по расписанию надёжнее недокументированный `--full` **[К]**; фактический тип проверять по имени бэкапа после завершения.

4. **Блокировки от параллельного запуска не найдено.** **[К]** Поиск по коду PostgreSQL-части не нашёл файловой блокировки или проверки «бэкап уже идёт» — это отсутствие находки, а не доказательство. **[В]** Сервер управления должен сам исключать наложение `backup-push`, `delete` и восстановления на одном префиксе.

5. **`delete` опасен при автоматизации.**
   - **[Д]** `retain N` падает, если N-й бэкап — дельта: использовать `FULL` или `FIND_FULL`.
   - **[Д]** После `pg_upgrade` без `--use-sentinel-time` могут быть удалены новые бэкапы.
   - **[Д]** Правила хранения — «по количеству» и «новее метки»; правила «N дневных / недельных / месячных» нет. **[В]** Такие схемы придётся собирать из `backup-mark` (постоянные бэкапы) и `delete target`.
   - **[Д]** `delete target` не удаляет WAL — после него нужен `delete garbage`.
   - **[Д]** Удаление бэкапа удаляет и WAL до него, то есть окно PITR определяется самым старым сохранённым бэкапом.

6. **Восстановление — это сценарий из многих шагов, WAL-G делает один.** **[В]** Серверу управления нужно: остановить PostgreSQL, очистить каталог данных (он обязан быть пустым **[К]**), выполнить `backup-fetch`, прописать `restore_command` и `recovery_target_*`, создать `recovery.signal`, запустить PostgreSQL, дождаться окончания восстановления, обработать `recovery_target_action` (по умолчанию `pause` **[Д, PostgreSQL]** — без явного `promote` база останется на паузе).

7. **`restore_command` нужна обёртка.** **[Д]** `wal-fetch` возвращает 74 при отсутствии сегмента и 1 при любой другой ошибке. **[Д, PostgreSQL]** Ненулевой код PostgreSQL понимает как «файла нет» и может закончить восстановление раньше времени; остановить его может только завершение команды по сигналу или ошибка оболочки. **[В]** Без обёртки, превращающей код 1 в 126+, сетевой сбой хранилища при PITR выглядит как «WAL закончился» — база поднимется в более раннем состоянии, чем просили.

8. **Восстановление на другой хост БД.** **[В]** Технически нужны только конфигурация хранилища, ключ шифрования и пустой каталог. Но:
   - **[Д, PostgreSQL]** восстановленный кластер по умолчанию идёт по `recovery_target_timeline = latest` и после повышения начинает новый таймлайн;
   - **[В]** если на восстановленном хосте остался тот же префикс и включена архивация, он начнёт писать WAL в архив исходного хоста БД. `WALG_PREVENT_WAL_OVERWRITE` по умолчанию выключен **[Д]**, а документация PostgreSQL прямо называет «два сервера в один каталог архива» типовой ошибкой администратора. Для тестового восстановления архивацию нужно отключать или давать другой префикс, и включать `WALG_PREVENT_WAL_OVERWRITE` везде;
   - **[Д, PostgreSQL]** `CREATE TABLESPACE` записан в WAL с абсолютным путём и воспроизведётся с тем же путём на другой машине.

9. **Проверку бэкапа тестовым восстановлением нужно строить целиком.** **[В]** WAL-G даёт только кирпичи: `backup-fetch`, `wal-fetch`, `--verify` при бэкапе, `wal-verify`. Поднять экземпляр, довести до согласованного состояния, проверить данные, убрать за собой — задача сервера управления. Нужна площадка с местом под полный размер базы (`uncompressed_size` есть в `backup-list`).

10. **Включение архивации требует перезапуска PostgreSQL.** **[Д, PostgreSQL]** `archive_mode` задаётся только при старте сервера и не включается при `wal_level = minimal`; `archive_command` меняется без перезапуска. **[В]** Сервер управления не может включить архивацию незаметно — нужен согласованный перезапуск. Практичный приём: один раз включить `archive_mode = on`, дальше менять только `archive_command`.

11. **Сбой архивации заполняет диск.** **[Д, PostgreSQL]** Пока архивация не работает, `pg_wal/` растёт; при заполнении файловой системы PostgreSQL аварийно останавливается. **[В]** Отставание архивации — главный сигнал мониторинга, важнее статуса бэкапов.

12. **Малоактивные базы.** **[Д, PostgreSQL]** Архивируются только завершённые сегменты; без `archive_timeout` последние транзакции могут долго не попадать в хранилище. **[В]** `archive_timeout` должен входить в настройку архивации, иначе «отставание» на тихой базе неотличимо от сбоя.

13. **Секреты лежат на хосте БД.** **[В]** `archive_command` выполняется процессом PostgreSQL, поэтому ключи хранилища и шифрования должны быть доступны пользователю `postgres` — на практике в файле конфигурации. Потеря ключа шифрования делает бэкапы невосстановимыми; сервер управления должен хранить ключи у себя, иначе не сможет восстановить на другой хост. С OpenPGP можно оставить на хосте БД только открытый ключ **[Д]**, но тогда этот хост не сможет сам восстановиться.

14. **Чтение состояния возможно прямо с сервера управления.** **[В]** `backup-list` и `wal-show` работают только с хранилищем **[К]**, значит их можно запускать с сервера управления, имея конфигурацию хранилища, — и видеть бэкапы даже недоступного хоста БД. `wal-verify` и `pg_stat_archiver` требуют работающего PostgreSQL.

15. **Несколько хранилищ.** **[Д]** Один файл конфигурации — одно основное хранилище; запасные экспериментальны. **[В]** «Несколько хранилищ» в нашем смысле (разные хосты БД в разных хранилищах) — это разные файлы конфигурации, штатно. А вот запись одного хоста БД в два хранилища сразу опирается на экспериментальную функцию; более надёжный путь — `wal-g copy` по расписанию.

16. **Готовые бинарники только под Ubuntu.** **[К]** Релиз v3.0.9 содержит `wal-g-pg-{20.04,22.04,24.04}-{amd64,aarch64}` с файлами `.sha256`; `walg-daemon-client` и экспортёра в релизе нет. **[В]** Для других дистрибутивов возможна сборка из исходников (Go, `GOEXPERIMENT=jsonv2`, libsodium, brotli, lzo) — установка WAL-G сервером управления должна это учитывать.

17. **Зрелость функций.** **[Д]** Отмечены как экспериментальные или бета: запасные хранилища, частичное восстановление, `--reverse-unpack`, `--skip-redundant-tars`, `WALG_DIRECT_IO`, составители `rating` / `copy` / `database`, поддержка pgBackRest. **[В]** В первой версии сервера управления на них лучше не опираться.

## 5. Что не удалось подтвердить

- **Список поддерживаемых версий PostgreSQL.** В документации WAL-G v3.0.9 его нет. В коде есть ветки для версий до 15 и для 15+ (`pg_backup_start`), то есть 13 и 14 идут по старой ветке, 15+ — по новой; явного подтверждения работы с PostgreSQL 17 и 18 в документации не найдено.
- **Отсутствие блокировки параллельных запусков** — вывод из поиска по коду, не из документации и не из опыта запуска.
- **Формат вывода `backup-list --json` при пустом хранилище** (пустой массив или пустой вывод) — не запускалось.
- **Точный смысл поля `time`** в `backup-list` (предположительно время изменения сентинела) — не проверялось.
- **Формат и структура логов WAL-G**, а также наличие какого-либо отчёта о ходе выполнения — в документации не описаны, код на этот счёт не изучался.
- **Совместимость релизных бинарников с дистрибутивами, отличными от Ubuntu** — не проверялась.
- **Поведение с конкретными S3-совместимыми хранилищами** (MinIO, Ceph и др.) — документация перечисляет только параметры совместимости, не список проверенных хранилищ.
- **Требование включённых контрольных сумм данных для `--verify`** — в документации WAL-G не оговорено.
- **Цитаты из документации PostgreSQL** взяты со страниц `/docs/current/` (версия 18) и получены через автоматическое извлечение текста. Описанные механизмы (`recovery.signal`, `restore_command`, `recovery_target_*`, `pg_stat_archiver`) существуют с версии 12 и ранее, а `archive_library` — с версии 15, но построчной сверки со страницами версии 13 не было.

## Источники

WAL-G, тег `v3.0.9`:

- [docs/PostgreSQL.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/PostgreSQL.md)
- [docs/README.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/README.md)
- [docs/STORAGES.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/STORAGES.md)
- [docs/FailoverStorages.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/FailoverStorages.md)
- [docs/StorageTools.md](https://github.com/wal-g/wal-g/blob/v3.0.9/docs/StorageTools.md)
- [cmd/pg/exporter/README.md](https://github.com/wal-g/wal-g/blob/v3.0.9/cmd/pg/exporter/README.md)
- [исходный код `cmd/pg`](https://github.com/wal-g/wal-g/tree/v3.0.9/cmd/pg) и [`internal/databases/postgres`](https://github.com/wal-g/wal-g/tree/v3.0.9/internal/databases/postgres)
- [примечания к релизу v3.0.9](https://github.com/wal-g/wal-g/releases/tag/v3.0.9)

PostgreSQL:

- [Continuous Archiving and Point-in-Time Recovery](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [Write Ahead Log: параметры архивации и восстановления](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- [The Cumulative Statistics System: `pg_stat_archiver`](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ARCHIVER-VIEW)
