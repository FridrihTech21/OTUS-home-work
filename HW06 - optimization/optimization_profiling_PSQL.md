### 1. Разверните инстанс PostgreSQL на виртуальной машине в Яндекс.Облаке или любом другом месте.

Виртуальная машина создана:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/1.jpg)

PostgreSQL установлен:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/2.jpg)

Не забываем выполнить:
```
1. Правим строку в файле /etc/postgresql/18/main/postgresql.conf:  ---> listen_addresses = '*'		# what IP address(es) to listen on;
2. Добавляем строку в /etc/postgresql/16/main/pg_hba.conf дабы разрешить аутентификацию из других сетей   --->		host    all             all             0.0.0.0/0	        scram-sha-256
3. Перезагружаем экземпляр  --->	sudo pg_ctlcluster 16 main restart
```

### 2. Протестируйте производительность с помощью pgbench.

Данный раздел будет поделена на подразделы:
```
2.1 Создание БД benchmark для тестов
2.2 Запуск базовог теста pgbench
2.3 Тестирование пула соединений
```

Фиксируем исходные(дефолтные) параметры БД PostgreSQL:
<details>
<summary>postgresql.conf</summary>
  
```
 allow_alter_system                          on                                      
 allow_in_place_tablespaces                  off                                     
 allow_system_table_mods                     off                                     
 application_name                            psql                                    
 archive_cleanup_command                                                             
 archive_command                             (disabled)                              
 archive_library                                                                     
 archive_mode                                off                                     
 archive_timeout                             0                                       
 array_nulls                                 on                                      
 authentication_timeout                      1min                                    
 autovacuum                                  on                                      
 autovacuum_analyze_scale_factor             0.1                                     
 autovacuum_analyze_threshold                50                                      
 autovacuum_freeze_max_age                   200000000                               
 autovacuum_max_workers                      3                                       
 autovacuum_multixact_freeze_max_age         400000000                               
 autovacuum_naptime                          1min                                    
 autovacuum_vacuum_cost_delay                2ms                                     
 autovacuum_vacuum_cost_limit                -1                                      
 autovacuum_vacuum_insert_scale_factor       0.2                                     
 autovacuum_vacuum_insert_threshold          1000                                    
 autovacuum_vacuum_max_threshold             100000000                               
 autovacuum_vacuum_scale_factor              0.2                                     
 autovacuum_vacuum_threshold                 50                                      
 autovacuum_work_mem                         -1                                      
 autovacuum_worker_slots                     16                                      
 backend_flush_after                         0                                       
 backslash_quote                             safe_encoding                           
 backtrace_functions                                                                 
 bgwriter_delay                              200ms                                   
 bgwriter_flush_after                        512kB                                   
 bgwriter_lru_maxpages                       100                                     
 bgwriter_lru_multiplier                     2                                       
 block_size                                  8192                                    
 bonjour                                     off                                     
 bonjour_name                                                                        
 bytea_output                                hex                                     
 check_function_bodies                       on                                      
 checkpoint_completion_target                0.9                                     
 checkpoint_flush_after                      256kB                                   
 checkpoint_timeout                          5min                                    
 checkpoint_warning                          30s                                     
 client_connection_check_interval            0                                       
 client_encoding                             UTF8                                    
 client_min_messages                         notice                                  
 cluster_name                                18/main                                 
 commit_delay                                0                                       
 commit_siblings                             5                                       
 commit_timestamp_buffers                    256kB                                   
 compute_query_id                            auto                                    
 config_file                                 /etc/postgresql/18/main/postgresql.conf 
 constraint_exclusion                        partition                               
 cpu_index_tuple_cost                        0.005                                   
 cpu_operator_cost                           0.0025                                  
 cpu_tuple_cost                              0.01                                    
 createrole_self_grant                                                               
 cursor_tuple_fraction                       0.1                                     
 data_checksums                              on                                      
 data_directory                              /var/lib/postgresql/18/main             
 data_directory_mode                         0700                                    
 data_sync_retry                             off                                     
 DateStyle                                   ISO, MDY                                
 deadlock_timeout                            1s                                      
 debug_assertions                            off                                     
 debug_discard_caches                        0                                       
 debug_io_direct                                                                     
 debug_logical_replication_streaming         buffered                                
 debug_parallel_query                        off                                     
 debug_pretty_print                          on                                      
 debug_print_parse                           off                                     
 debug_print_plan                            off                                     
 debug_print_rewritten                       off                                     
 default_statistics_target                   100                                     
 default_table_access_method                 heap                                    
 default_tablespace                                                                  
 default_text_search_config                  pg_catalog.english                      
 default_toast_compression                   pglz                                    
 default_transaction_deferrable              off                                     
 default_transaction_isolation               read committed                          
 default_transaction_read_only               off                                     
 dynamic_library_path                        $libdir                                 
 dynamic_shared_memory_type                  posix                                   
 effective_cache_size                        4GB                                     
 effective_io_concurrency                    16                                      
 enable_async_append                         on                                      
 enable_bitmapscan                           on                                      
 enable_distinct_reordering                  on                                      
 enable_gathermerge                          on                                      
 enable_group_by_reordering                  on                                      
 enable_hashagg                              on                                      
 enable_hashjoin                             on                                      
 enable_incremental_sort                     on                                      
 enable_indexonlyscan                        on                                      
 enable_indexscan                            on                                      
 enable_material                             on                                      
 enable_memoize                              on                                      
 enable_mergejoin                            on                                      
 enable_nestloop                             on                                      
 enable_parallel_append                      on                                      
 enable_parallel_hash                        on                                      
 enable_partition_pruning                    on                                      
 enable_partitionwise_aggregate              off                                     
 enable_partitionwise_join                   off                                     
 enable_presorted_aggregate                  on                                      
 enable_self_join_elimination                on                                      
 enable_seqscan                              on                                      
 enable_sort                                 on                                      
 enable_tidscan                              on                                      
 escape_string_warning                       on                                      
 event_source                                PostgreSQL                              
 event_triggers                              on                                      
 exit_on_error                               off                                     
 extension_control_path                      $system                                 
 external_pid_file                           /var/run/postgresql/18-main.pid         
 extra_float_digits                          1                                       
 file_copy_method                            copy                                    
 file_extend_method                          posix_fallocate                         
 from_collapse_limit                         8                                       
 fsync                                       on                                      
 full_page_writes                            on                                      
 geqo                                        on                                      
 geqo_effort                                 5                                       
 geqo_generations                            0                                       
 geqo_pool_size                              0                                       
 geqo_seed                                   0                                       
 geqo_selection_bias                         2                                       
 geqo_threshold                              12                                      
 gin_fuzzy_search_limit                      0                                       
 gin_pending_list_limit                      4MB                                     
 gss_accept_delegation                       off                                     
 hash_mem_multiplier                         2                                       
 hba_file                                    /etc/postgresql/18/main/pg_hba.conf     
 hot_standby                                 on                                      
 hot_standby_feedback                        off                                     
 huge_page_size                              0                                       
 huge_pages                                  try                                     
 huge_pages_status                           off                                     
 icu_validation_level                        warning                                 
 ident_file                                  /etc/postgresql/18/main/pg_ident.conf   
 idle_in_transaction_session_timeout         0                                       
 idle_replication_slot_timeout               0                                       
 idle_session_timeout                        0                                       
 ignore_checksum_failure                     off                                     
 ignore_invalid_pages                        off                                     
 ignore_system_indexes                       off                                     
 in_hot_standby                              off                                     
 integer_datetimes                           on                                      
 IntervalStyle                               postgres                                
 io_combine_limit                            128kB                                   
 io_max_combine_limit                        128kB                                   
 io_max_concurrency                          64                                      
 io_method                                   worker                                  
 io_workers                                  3                                       
 jit                                         on                                      
 jit_above_cost                              100000                                  
 jit_debugging_support                       off                                     
 jit_dump_bitcode                            off                                     
 jit_expressions                             on                                      
 jit_inline_above_cost                       500000                                  
 jit_optimize_above_cost                     500000                                  
 jit_profiling_support                       off                                     
 jit_provider                                llvmjit                                 
 jit_tuple_deforming                         on                                      
 join_collapse_limit                         8                                       
 krb_caseins_users                           off                                     
 krb_server_keyfile                          FILE:/etc/postgresql-common/krb5.keytab 
 lc_messages                                 C.UTF-8                                 
 lc_monetary                                 C.UTF-8                                 
 lc_numeric                                  C.UTF-8                                 
 lc_time                                     C.UTF-8                                 
 listen_addresses                            localhost                               
 lo_compat_privileges                        off                                     
 local_preload_libraries                                                             
 lock_timeout                                0                                       
 log_autovacuum_min_duration                 10min                                   
 log_checkpoints                             on                                      
 log_connections                                                                     
 log_destination                             stderr                                  
 log_directory                               log                                     
 log_disconnections                          off                                     
 log_duration                                off                                     
 log_error_verbosity                         default                                 
 log_executor_stats                          off                                     
 log_file_mode                               0600                                    
 log_filename                                postgresql-%Y-%m-%d_%H%M%S.log          
 log_hostname                                off                                     
 log_line_prefix                             %m [%p] %q%u@%d                         
 log_lock_failures                           off                                     
 log_lock_waits                              off                                     
 log_min_duration_sample                     -1                                      
 log_min_duration_statement                  -1                                      
 log_min_error_statement                     error                                   
 log_min_messages                            warning                                 
 log_parameter_max_length                    -1                                      
 log_parameter_max_length_on_error           0                                      
 log_parser_stats                            off                                     
 log_planner_stats                           off                                     
 log_recovery_conflict_waits                 off                                     
 log_replication_commands                    off                                     
 log_rotation_age                            1d                                      
 log_rotation_size                           10MB                                    
 log_startup_progress_interval               10s                                     
 log_statement                               none                                    
 log_statement_sample_rate                   1                                       
 log_statement_stats                         off                                     
 log_temp_files                              -1                                      
 log_timezone                                Etc/UTC                                 
 log_transaction_sample_rate                 0                                       
 log_truncate_on_rotation                    off                                     
 logging_collector                           off                                     
 logical_decoding_work_mem                   64MB                                    
 maintenance_io_concurrency                  16                                      
 maintenance_work_mem                        64MB                                    
 max_active_replication_origins              10                                      
 max_connections                             100                                     
 max_files_per_process                       1000                                    
 max_function_args                           100                                     
 max_identifier_length                       63                                      
 max_index_keys                              32                                      
 max_locks_per_transaction                   64                                      
 max_logical_replication_workers             4                                       
 max_notify_queue_pages                      1048576                                 
 max_parallel_apply_workers_per_subscription 2                                       
 max_parallel_maintenance_workers            2                                       
 max_parallel_workers                        8                                       
 max_parallel_workers_per_gather             2                                       
 max_pred_locks_per_page                     2                                       
 max_pred_locks_per_relation                 -2                                      
 max_pred_locks_per_transaction              64                                      
 max_prepared_transactions                   0                                       
 max_replication_slots                       10                                      
 max_slot_wal_keep_size                      -1                                      
 max_stack_depth                             2MB                                     
 max_standby_archive_delay                   30s                                     
 max_standby_streaming_delay                 30s                                   
 max_sync_workers_per_subscription           2                                       
 max_wal_senders                             10                                      
 max_wal_size                                1GB                                     
 max_worker_processes                        8                                       
 md5_password_warnings                       on                                      
 min_dynamic_shared_memory                   0                                       
 min_parallel_index_scan_size                512kB                                   
 min_parallel_table_scan_size                8MB                                     
 min_wal_size                                80MB                                    
 multixact_member_buffers                    256kB                                   
 multixact_offset_buffers                    128kB                                   
 notify_buffers                              128kB                                   
 num_os_semaphores                           174                                     
 oauth_validator_libraries                                                           
 parallel_leader_participation               on                                      
 parallel_setup_cost                         1000                                    
 parallel_tuple_cost                         0.1                                     
 password_encryption                         scram-sha-256                           
 plan_cache_mode                             auto                                    
 port                                        5432                                    
 post_auth_delay                             0                                       
 pre_auth_delay                              0                                       
 primary_conninfo                                                                    
 primary_slot_name                                                                   
 quote_all_identifiers                       off                                     
 random_page_cost                            4                                       
 recovery_end_command                                                                
 recovery_init_sync_method                   fsync                                   
 recovery_min_apply_delay                    0                                       
 recovery_prefetch                           try                                     
 recovery_target                                                                     
 recovery_target_action                      pause                                   
 recovery_target_inclusive                   on                                      
 recovery_target_lsn                                                                 
 recovery_target_name                                                                
 recovery_target_time                                                                
 recovery_target_timeline                    latest                                  
 recovery_target_xid                                                                 
 recursive_worktable_factor                  10                                      
 remove_temp_files_after_crash               on                                      
 reserved_connections                        0                                       
 restart_after_crash                         on                                      
 restore_command                                                                     
 restrict_nonsystem_relation_kind                                                    
 row_security                                on                                      
 scram_iterations                            4096                                    
 search_path                                 "$user", public                         
 segment_size                                1GB                                     
 send_abort_for_crash                        off                                     
 send_abort_for_kill                         off                                     
 seq_page_cost                               1                                       
 serializable_buffers                        256kB                                   
 server_encoding                             UTF8                                    
 server_version                              18.2 (Ubuntu 18.2-1.pgdg24.04+1)        
 server_version_num                          180002                                  
 session_preload_libraries                                                           
 session_replication_role                    origin                                  
 shared_buffers                              128MB                                   
 shared_memory_size                          150MB                                   
 shared_memory_size_in_huge_pages            75                                      
 shared_memory_type                          mmap                                    
 shared_preload_libraries                                                            
 ssl                                         on                                      
 ssl_ca_file                                                                         
 ssl_cert_file                               /etc/ssl/certs/ssl-cert-snakeoil.pem    
 ssl_ciphers                                 HIGH:MEDIUM:+3DES:!aNULL                
 ssl_crl_dir                                                                         
 ssl_crl_file                                                                        
 ssl_dh_params_file                                                                  
 ssl_groups                                  X25519:prime256v1                       
 ssl_key_file                                /etc/ssl/private/ssl-cert-snakeoil.key  
 ssl_library                                 OpenSSL                                 
 ssl_max_protocol_version                                                            
 ssl_min_protocol_version                    TLSv1.2                                 
 ssl_passphrase_command                                                              
 ssl_passphrase_command_supports_reload      off                                     
 ssl_prefer_server_ciphers                   on                                      
 ssl_tls13_ciphers                                                                   
 standard_conforming_strings                 on                                      
 statement_timeout                           0                                       
 stats_fetch_consistency                     cache                                   
 subtransaction_buffers                      256kB                                   
 summarize_wal                               off                                     
 superuser_reserved_connections              3                                       
 sync_replication_slots                      off                                     
 synchronize_seqscans                        on                                      
 synchronized_standby_slots                                                          
 synchronous_commit                          on                                      
 synchronous_standby_names                                                           
 syslog_facility                             local0                                  
 syslog_ident                                postgres                                
 syslog_sequence_numbers                     on                                      
 syslog_split_messages                       on                                      
 tcp_keepalives_count                        0                                       
 tcp_keepalives_idle                         0                                       
 tcp_keepalives_interval                     0                                       
 tcp_user_timeout                            0                                       
 temp_buffers                                8MB                                     
 temp_file_limit                             -1                                      
 temp_tablespaces                                                                    
 TimeZone                                    Etc/UTC                                 
 timezone_abbreviations                      Default                                 
 trace_connection_negotiation                off                                     
 trace_notify                                off                                     
 trace_sort                                  off                                     
 track_activities                            on                                      
 track_activity_query_size                   1kB                                     
 track_commit_timestamp                      off                                     
 track_cost_delay_timing                     off                                     
 track_counts                                on                                      
 track_functions                             none                                    
 track_io_timing                             off                                     
 track_wal_io_timing                         off                                     
 transaction_buffers                         256kB                                   
 transaction_deferrable                      off                                   
 transaction_isolation                       read committed                          
 transaction_read_only                       off                                     
 transaction_timeout                         0                                       
 transform_null_equals                       off                                     
 unix_socket_directories                     /var/run/postgresql                     
 unix_socket_group                                                                   
 unix_socket_permissions                     0777                                    
 update_process_title                        on                                      
 vacuum_buffer_usage_limit                   2MB                                     
 vacuum_cost_delay                           0                                       
 vacuum_cost_limit                           200                                     
 vacuum_cost_page_dirty                      20                                      
 vacuum_cost_page_hit                        1                                       
 vacuum_cost_page_miss                       2                                       
 vacuum_failsafe_age                         1600000000                              
 vacuum_freeze_min_age                       50000000                                
 vacuum_freeze_table_age                     150000000                               
 vacuum_max_eager_freeze_failure_rate        0.03                                    
 vacuum_multixact_failsafe_age               1600000000                              
 vacuum_multixact_freeze_min_age             5000000                                 
 vacuum_multixact_freeze_table_age           150000000                               
 vacuum_truncate                             on                                      
 wal_block_size                              8192                                    
 wal_buffers                                 4MB                                     
 wal_compression                             off                                     
 wal_consistency_checking                                                            
 wal_decode_buffer_size                      512kB                                   
 wal_init_zero                               on                                      
 wal_keep_size                               0                                       
 wal_level                                   replica                                 
 wal_log_hints                               off                                     
 wal_receiver_create_temp_slot               off
 wal_receiver_status_interval                10s                                     
 wal_receiver_timeout                        1min                                    
 wal_recycle                                 on                                      
 wal_retrieve_retry_interval                 5s                                      
 wal_segment_size                            16MB                                    
 wal_sender_timeout                          1min                                    
 wal_skip_threshold                          2MB                                     
 wal_summary_keep_time                       10d                                     
 wal_sync_method                             fdatasync                               
 wal_writer_delay                            200ms                                   
 wal_writer_flush_after                      1MB                                     
 work_mem                                    4MB                                     
 xmlbinary                                   base64                                  
 xmloption                                   content                                  
 zero_damaged_pages                          off                                     
(398 rows)
```
</details>

## 2.1 Создание БД benchmark для тестов

Для прохождения теста понадобятся данные:
- Владелец БД: `test_bench`;
- Пароль владельца: `qwerty@123`;
- Адрес тетируемого экземпляра PostgreSQL: `10.92.36.102`;
- Порт: `5432`;
- База данных: `benchdb`.
Данные параметры понадобятся для работы с pgbench и клиентом psql.

# 2.1.1 Создаем БД для тестов:

Создадим пользователя `test_bench`, БД `benchdb`:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/3.jpg)

# 2.1.2 Заполнение фиктивными данными
   
Для заполнения БД тестовыми данными, воспользуемся:
`pgbench -h [ip_PostgreSQL] -p [port] -U [user_db] -i -s [num_connections_clients] [name_db]`

где: 
- `h: конечная точка кластера PostgreSQL(адрес тетируемого экземпляра PostgreSQL)`;
- `p: порт кластера PostgreSQL`;
- `U: имя пользователя базы данных`;
- `i: инициализирует базу данных с помощью таблиц и их фиктивных данных`;
- `s: устанавливает масштабный коэффициент, который умножит размеры таблицы на указанную единицу`.

Используемая команда:
`pgbench -h 10.92.36.102 -p 5432 -U test_bench -i -s 100 benchdb`:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/4.jpg)

Масштабный коэффициент 100 создаст таблицу pgbench_accounts в 10 000 000 строк:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/5.jpg)

## 2.2 Запуск базового теста pgbench

Для демонстрации теста можно выбрать симуляцию с пулом соединений и без него к БД. По мере подключений к БД расходуется ресурсы машины, соответсвенно чем больше подключений, тем больше нагрузка на БД(машину, где находится БД, а также на саму БД). Также, в PostgreSQL имеется парметр для ограничения коннектов к БД: max_connections. Данный параметр резервирует 3 коннекта для подключения системного администртора в случае аварийных ситуаций, то есть: 
```
кол-во_коннектов - 3_резерв_коннекта = настоящее_кол-во_коннектов к БД
```
max_connections обычно сбрасывает или отклоняет коннекты, если уже квота на количество разрешенных превышена. Чтобы купировать и улучшить ситуацию с количеством обрботанных коннектов к БД можно использовать такой инструмент как пул соединений. Данный иструмент сохраняет открытым фиксированное кол-во коннектов к БД. То есть пул может ставить запросы на коннект в очередь, таким образом коннект не отклоняется как в случае с max_connections, а ждет своей очереди для подключения. 

Прежде чем начнем имитацию с пулом соединений, требуется показать ситуацию до его включения, что и будет рассмотрено в данном разделе.

Проверим количество разрешенных подключений к БД:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/6.jpg)

Из скриншота видно, что разрешенных соединение равно 100: `(100 - 3 = 97)`. Не забываем, что в PostgreSQL имеется резерв в количестве 3-х едениц подключения для администртора, в итоге общее количество клиентских подключений = `97`. 

Запустим тест в 45 подключений для того, чтобы симулировать среднюю нагрузку подключений к разрешенным в max_connections. 
Для запуска используем данную команду:
```
pgbench -h 10.92.36.102 -p 5432 -U test_bench -c 50 -j 2 -P 60 -T 600 benchdb
```

где:
- `с: количество имитрованных соединений к БД`;
- `j: количество потоков для используемых pgbench`ем для проведения теста`;
- `P: отображение метрик в секндах`;
- `Т: время, на протяжении которого проводится тест, также в секундах`.

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/7.jpg)

На кртинке выше видно, что симуляция подключений прошла успешно. За время выполнения симуляции`(-T = 600 секунд => 10 минут)` удалось произвести `192381` транзакций с пропускной способностью `320(tps)` транзакций в секунду. 

Далее увеличим количество одновременных подключений более максимума: `-с = 120`. Данный тест покажет как поведет себя БД при массовом притоке подключений:
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/8.jpg)
На картинке видно, что `pgbench` не может создать 50 коннектов к тестовой БД. Судя по всему можно решить данную проблему подключением `pgBouncer`, либо его аналогов. Также в теории будет прирост TPS, что проверим а практике.

## 2.3 Подключение pgBouncer

Установим `pgBouncer`:
```
sudo apt-get -y install pgbouncer
```

Далее отредактируем файл инициализации `pgBouncer` `/etc/pgbouncer/pgbouncer.ini`:
<details>
<summary>pgbouncer.ini</summary>
  
```
* = host=localhost port=5432 ; Добавим запись в секцию [databases] хоста где лежит PostgreSQL, а также его порт.
listen_addr = * ; Адрес, на котором будет крутится pgBouncer
max_client_conn = 500 ; Изменим количество максимально открытых сессий для pgBouncer
listen_port = 6432 ; Изменим номер порта для pgBouncer
```
</details>

После чего, нужно создать список пользователей в `/etc/pgbouncer/userlist.txt`, которые будут иметь доступ для открытия коннекта к БД. Файл в формате:
```
"DB_USER" "PASSWORD"
```

Настройка пуллера окончена. Запускаем его через: `systemctl enable --now pgbouncer`
![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/9.jpg)

## 2.4 Тестирование пула соединений

<details>
<summary>97 коннектов без пуллера</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 5432 -U test_bench -c 97 -j 2 -P 10 -T 60 benchdb
Password:
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
progress: 10.0 s, 329.7 tps, lat 260.038 ms stddev 105.418, 0 failed
progress: 20.0 s, 369.6 tps, lat 263.312 ms stddev 98.985, 0 failed
progress: 30.0 s, 320.6 tps, lat 299.074 ms stddev 128.362, 0 failed
progress: 40.0 s, 273.4 tps, lat 358.147 ms stddev 179.145, 0 failed
progress: 50.0 s, 258.8 tps, lat 368.680 ms stddev 177.490, 0 failed
progress: 60.0 s, 288.1 tps, lat 341.715 ms stddev 179.898, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 97
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 18499
number of failed transactions: 0 (0.000%)
latency average = 310.155 ms
latency stddev = 151.459 ms
initial connection time = 1006.665 ms
tps = 312.176658 (without initial connection time)
```
</details>

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password:
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 432.7 tps, lat 114.207 ms stddev 85.885, 0 failed
progress: 20.0 s, 433.4 tps, lat 115.241 ms stddev 44.894, 0 failed
progress: 30.0 s, 338.0 tps, lat 148.215 ms stddev 64.865, 0 failed
progress: 40.0 s, 270.7 tps, lat 184.369 ms stddev 74.058, 0 failed
progress: 50.0 s, 349.2 tps, lat 143.831 ms stddev 63.508, 0 failed
progress: 60.0 s, 372.2 tps, lat 134.213 ms stddev 63.232, 0 failed
NOTICE:  No server connection available in postgres backend, client being queued
NOTICE:  No server connection available in postgres backend, client being queued
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 22082
number of failed transactions: 0 (0.000%)
latency average = 326.867 ms
latency stddev = 3377.046 ms
initial connection time = 40.584 ms
tps = 366.613519 (without initial connection time)
```
</details>


Запуск теста без пуллера: tps = 312.176658 (without initial connection time)
Запуск теста с пуллером: tps = 366.613519 (without initial connection time)
Как выидно, при массовом притоке коннектов к БД с пуллером, обработанных транзакций больше. Плюсом данного решения является расширение количества обрбатываемых запросов за счет пуллера, если БД столкнется с количеством коннектов, привышающим `max_connections`, то в таком случае запросы(не влезающие в количество разрешенных) на коннет будут отклонены. 

Для большей производительности PostgreSQL можно оптимизировать его настройки, что будет описано в следующем разделе. 

### 3. Оптимизируйте настройки PostgreSQL для максимальной производительности.

Данная статья будет описывать что нужно сделать, перед тем как оптимизировать настройки PostgreSQL. И состоит из нескольких подпунктов:

- 3.1 Проверка памяти
- 3.2 Тюнинг настроек PostgreSQL

Перед дальнейшим тестированием зафиксируем резльтаты от pgbench:

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password: 
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 527.9 tps, lat 93.660 ms stddev 74.516, 0 failed
progress: 20.0 s, 430.8 tps, lat 116.107 ms stddev 54.415, 0 failed
progress: 30.0 s, 340.0 tps, lat 147.349 ms stddev 66.303, 0 failed
progress: 40.0 s, 256.6 tps, lat 192.896 ms stddev 84.797, 0 failed
progress: 50.0 s, 325.1 tps, lat 154.401 ms stddev 72.313, 0 failed
progress: 60.0 s, 354.7 tps, lat 141.092 ms stddev 63.909, 0 failed
NOTICE:  No server connection available in postgres backend, client being queued
NOTICE:  No server connection available in postgres backend, client being queued
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 22471
number of failed transactions: 0 (0.000%)
latency average = 321.066 ms
latency stddev = 3346.807 ms
initial connection time = 34.159 ms
tps = 373.173324 (without initial connection time)
```
</details>

Зафиксировали: `tps = 373.173324` и `latency average = 321.066 ms`

## 3.1 Проверка памяти. 

Если БД работает с большими данными, то естьт смысл включить huge pages. В Linux работа с памятью основывается на обращении к страницам, размер которых равен 4 кВ. А значит, когда объем памяти становиться большим управление нею становится сложнее. Поэтому разумно использовать большие страницы, размер которых начинается с 2 МВ. За счет использования huge pages можно получить прирост к быстродействию системы. 
И так, проверяем ядро на поддержку huge pages:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/10.jpg)

Текущие значения больших страниц:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/11.jpg)


Получаем PID PostgreSQL:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/12.jpg)

Пиковая виртуальная память для PostgreSQL:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/13.jpg)

Расчитываем прибилизительное количество huge pages и устанавливаем его:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/14.jpg)

Влючаем huge pages:

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/15.jpg)

# Отключение THP:

THP — это механизм ядра Linux, который автоматически пытается использовать огромные страницы памяти вместо стандартных маленьких страниц для любого процесса в системе или по запросу.
Отключение THP часто рекомендуют для систем, где работает PostgreSQL чтобы избежать потенциальных проблем с производительностью, связанных с автоматическим управлением огромными страницами ядром.

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/16.jpg)

# swappiness:

swappiness–определяет частоту сброса данных из RAM в SWAP(значение от 0 до 200). Для PostgreSQLрекомендуется от 1 до 5.

![КАРТИНКА](https://github.com/FridrihTech21/OTUS-home-work/blob/main/project/lab_6/17.jpg)



## 3.2 Тюнинг настроек PostgreSQL

Изменим параметры в `postgresql.auto.conf` для большей производительности нашей БД с учетом аппаратной части сервера.

<details>
<summary>postgresql.auto.conf</summary>
  
```
max_connections = 100
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
```
</details>


### 4. Проверьте, насколько выросла производительность.

## 4.1.1 Поcле применения page huges показатели улучшились:

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password: 
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 513.6 tps, lat 96.405 ms stddev 71.452, 0 failed
progress: 20.0 s, 461.7 tps, lat 108.269 ms stddev 48.497, 0 failed
progress: 30.0 s, 386.5 tps, lat 129.372 ms stddev 60.398, 0 failed
progress: 40.0 s, 332.1 tps, lat 149.880 ms stddev 70.113, 0 failed
progress: 50.0 s, 314.7 tps, lat 159.068 ms stddev 75.112, 0 failed
progress: 60.0 s, 412.2 tps, lat 121.752 ms stddev 57.637, 0 failed
NOTICE:  No server connection available in postgres backend, client being queued
NOTICE:  No server connection available in postgres backend, client being queued
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 24328
number of failed transactions: 0 (0.000%)
latency average = 296.209 ms
latency stddev = 3213.131 ms
initial connection time = 33.417 ms
tps = 404.099289 (without initial connection time)
```
</details>

----------------------------------------------------
Было:

- Среднее время выполнения: `latency average = 321.066 ms`

- `TPS`: `tps = 373.173324`  

----------------------------------------------------

Стало: 

- Среднее время выполнения сократилось: `latency average = 296.209 ms`

- `TPS` поднялся: `tps = 404.099289 (without initial connection time)`

----------------------------------------------------

## 4.1.2 Поcле отключения THP показатели улучшились:

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password: 
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 623.5 tps, lat 79.563 ms stddev 66.946, 0 failed
progress: 20.0 s, 470.8 tps, lat 105.476 ms stddev 54.797, 0 failed
progress: 30.0 s, 416.3 tps, lat 120.709 ms stddev 57.628, 0 failed
progress: 40.0 s, 341.0 tps, lat 145.947 ms stddev 72.501, 0 failed
progress: 50.0 s, 284.8 tps, lat 176.133 ms stddev 74.739, 0 failed
progress: 60.0 s, 342.9 tps, lat 145.107 ms stddev 72.674, 0 failed
NOTICE:  No server connection available in postgres backend, client being queued
NOTICE:  No server connection available in postgres backend, client being queued
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 24913
number of failed transactions: 0 (0.000%)
latency average = 289.546 ms
latency stddev = 3180.559 ms
initial connection time = 33.278 ms
tps = 413.593608 (without initial connection time)
```
</details>

----------------------------------------------------
Было:

- Среднее время выполнения: `latency average = 296.209 ms`

- `TPS`: `tps = 404.099289 (without initial connection time)`

----------------------------------------------------

Стало: 

- Среднее время выполнения сократилось: `latency average = 289.546 ms`

- `TPS` поднялся: `tps = 413.593608 (without initial connection time)`

---------------------------------------------------- 

## 4.1.3 Поcле отключения THP показатели улучшились:

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password: 
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 626.1 tps, lat 79.147 ms stddev 64.812, 0 failed
progress: 20.0 s, 483.8 tps, lat 102.670 ms stddev 53.037, 0 failed
progress: 30.0 s, 414.7 tps, lat 121.466 ms stddev 57.377, 0 failed
progress: 40.0 s, 316.7 tps, lat 156.739 ms stddev 80.123, 0 failed
progress: 50.0 s, 294.0 tps, lat 170.026 ms stddev 79.282, 0 failed
progress: 60.0 s, 408.8 tps, lat 122.328 ms stddev 59.381, 0 failed
NOTICE:  No server connection available in postgres backend, client being queued
NOTICE:  No server connection available in postgres backend, client being queued
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25561
number of failed transactions: 0 (0.000%)
latency average = 282.110 ms
latency stddev = 3137.499 ms
initial connection time = 31.711 ms
tps = 424.777859 (without initial connection time)
```
</details>

----------------------------------------------------
Было:

- Среднее время выполнения: `latency average = 289.546 ms`

- `TPS`: `tps = 413.593608 (without initial connection time)`

----------------------------------------------------

Стало: 

- Среднее время выполнения сократилось: `latency average = 282.110 ms`

- `TPS` поднялся: `tps = 424.777859 (without initial connection time)`

---------------------------------------------------- 

## 4.1.3 Поcле отключения THP показатели улучшились:

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password: 
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 426.1 tps, lat 116.318 ms stddev 86.287, 0 failed
progress: 20.0 s, 440.0 tps, lat 113.537 ms stddev 38.240, 0 failed
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29813
number of failed transactions: 0 (0.000%)
latency average = 241.704 ms
latency stddev = 2903.890 ms
initial connection time = 33.072 ms
tps = 495.816196 (without initial connection time)
```
</details>

----------------------------------------------------
Было:

- Среднее время выполнения: `latency average = 282.110 ms`

- `TPS`: `tps = 424.777859 (without initial connection time)`

----------------------------------------------------

Стало: 

- Среднее время выполнения сократилось: `latency average = 241.704 ms`

- `TPS` поднялся: `tps = 495.816196 (without initial connection time)`

Как видим производительность БД поднялась еще выше. Пока это максимальная точка, которй удалось добится без особых рисков.

### 5. Настройте кластер на оптимальную производительность, не обращая внимания на стабильность БД.

Исходя из задания `Настройте кластер на оптимальную производительность, не обращая внимания на стабильность БД`, мы будем максимально увеличивать параметры, влияющие на скорость обработки данных, жертвуя надежностью, устойчивостью к сбоям, безопасностью и возможностью восстановления в случае проблем.
Данную конфигурацию применять на реальных системах НЕЛЬЗЯ.

<details>
<summary>120 коннектов с пуллером</summary>
  
```
huge_pages = 'on'

max_connections = 100
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB



fsync = off                    # Отключает синхронизацию WAL на диск. Критический риск потери данных.
synchronous_commit = off       # Отключает ожидание подтверждения записи WAL. Повышает скорость, но теряются транзакции при сбое.
full_page_writes = off

maintenance_work_mem = 512MB   # Увеличиваем для быстрых операций обслуживания
work_mem = 20MB                # Увеличиваем для более быстрых сортировок/соединений в памяти.

wal_buffers = 64MB             # Увеличиваем буфер WAL для уменьшения частоты синхронизации. Риск потери данных из буфера при сбое.
min_wal_size = 4GB             # Увеличиваем минимальный размер WAL.
max_wal_size = 16GB            # Увеличиваем максимальный размер WAL. Уменьшает частоту контрольных точек, но увеличивает время восстановления.
checkpoint_completion_target = 0.95 # Увеличиваем для более плавного выполнения контрольной точки.

logging_collector = off        # Отключаем сбор логов в файлы. Полезно для отладки, но замедляет работу.
log_checkpoints = off          # Отключаем логирование контрольных точек.
log_connections = off          # Отключаем логирование подключений.
log_disconnections = off       # Отключаем логирование отключений.
log_lock_waits = off           # Отключаем логирование ожиданий блокировок.
log_temp_files = -1            # Не логировать создание временных файлов.
log_min_duration_statement = -1 # Не логировать медленные запросы.

bgwriter_delay = 100ms         # Уменьшаем задержку между циклами фоновой записи.
bgwriter_lru_maxpages = 200    # Увеличиваем количество страниц, записываемых за цикл.
```
</details>

Запуск теста с конфигурацией выше:

<details>
<summary>120 коннектов с пуллером</summary>
  
```
postgres@tarasov-postgre-advance-1:~$ pgbench -h 10.92.36.102 -p 6432 -U test_bench -c 120 -j 2 -P 10 -T 60 benchdb
Password: 
pgbench (18.2 (Ubuntu 18.2-1.pgdg24.04+1))
starting vacuum...end.
NOTICE:  No server connection available in postgres backend, client being queued
progress: 10.0 s, 526.1 tps, lat 94.111 ms stddev 80.914, 0 failed
progress: 20.0 s, 536.3 tps, lat 93.197 ms stddev 54.914, 0 failed
progress: 30.0 s, 534.8 tps, lat 93.476 ms stddev 71.732, 0 failed
progress: 40.0 s, 652.1 tps, lat 76.717 ms stddev 58.884, 0 failed
progress: 50.0 s, 717.4 tps, lat 69.765 ms stddev 54.342, 0 failed
progress: 60.0 s, 725.3 tps, lat 68.903 ms stddev 64.342, 0 failed
NOTICE:  No server connection available in postgres backend, client being queued
NOTICE:  No server connection available in postgres backend, client being queued
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 120
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 37040
number of failed transactions: 0 (0.000%)
latency average = 194.652 ms
latency stddev = 2608.032 ms
initial connection time = 30.451 ms
tps = 615.663889 (without initial connection time)
```
</details>

----------------------------------------------------
Было:

- Среднее время выполнения: `latency average = 241.704 ms`

- `TPS`: `tps = 495.816196 (without initial connection time)`

----------------------------------------------------

Стало: 

- Среднее время выполнения сократилось: `latency average = 194.652 ms`

- `TPS` поднялся: `tps = 615.663889 (without initial connection time)`

Исходя из результата последнего тестирования можно сделать вывод о максимальном приросте обрботки операций БД с данными параметрами.
Но данная конфигурация может привести к печальным результатам на реальных системах. 
