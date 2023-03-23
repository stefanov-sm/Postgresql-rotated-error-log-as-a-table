# Postgresql rotated errorlog as a view
Access Postgresql rotated error log files as a single view  

### Setup steps:
- **Enable error log rotation on a day-of week basis. Edit `postgresql.conf` file with the values below and restart the sevice.**
```
#------------------------------------------------------------------------------
# postgresql.conf extract, section REPORTING AND LOGGING
#------------------------------------------------------------------------------

log_destination = 'csvlog'
logging_collector = on
log_directory = '/path/to/'
log_filename = 'postgreslog.%a.log'  # One file for each DOW
log_file_mode = 0640
log_truncate_on_rotation = on
log_rotation_age = 1440
log_rotation_size = 0
log_line_prefix = '%t '
log_timezone = 'Europe/Sofia'  # Use your timezone here
```
- **Attach error log files as foreign tables**
```sql
-- Setup file_fdw if not already there
create extension file_fdw;
create server file_server foreign data wrapper file_fdw;

-- Create 7 foreign tables for MON - SUN log files
do language plpgsql
$body$
declare
  DYNDDL_TEMPLATE constant text default
  $dynddl_template$
  create foreign table errlog___DOW__
  (
   log_time timestamp(3) with time zone,
   user_name text,
   database_name text,
   process_id integer,
   connection_from text,
   session_id text,
   session_line_num bigint,
   command_tag text,
   session_start_time timestamp with time zone,
   virtual_transaction_id text,
   transaction_id bigint,
   error_severity text,
   sql_state_code text,
   message text,
   detail text,
   hint text,
   internal_query text,
   internal_query_pos integer,
   context text,
   query text,
   query_pos integer,
   location text,
   application_name text,
   ignored_a text, ignored_b text, ignored_c text
  )
  server file_server
  options
  (
    filename '/path/to/postgreslog.__DOW__.csv',
    format 'csv'
  )
  $dynddl_template$;
  dow text;
begin
  foreach dow in array '{Mon,Tue,Wed,Thu,Fri,Sat,Sun}'::text[] loop
    execute replace(DYNDDL_TEMPLATE, '__DOW__', dow);
  end loop;
end;
$body$;
```
- **Create the `error_log` view**
```sql

CREATE VIEW error_log AS
WITH t AS
(
 SELECT * FROM errlog_mon UNION ALL
 SELECT * FROM errlog_tue UNION ALL
 SELECT * FROM errlog_wed UNION ALL
 SELECT * FROM errlog_thu UNION ALL
 SELECT * FROM errlog_fri UNION ALL
 SELECT * FROM errlog_sat UNION ALL
 SELECT * FROM errlog_sun
)
SELECT 
  error_severity AS "Severity",
  log_time::timestamp AS "Log time",
  split_part(connection_from, ':', 1) AS "Client",
  application_name AS "Application",
  database_name AS "Database",
  message AS "Log message",
  sql_state_code AS "SQL state",
  query AS "Query",
  query_pos AS "Char position",
  detail AS "Message details",
  hint
FROM t
ORDER BY "Log time" DESC;
```
### Important notes
- Replace `/path/to/` with the actual log files' location in `postgresql.conf` and the dynamic DDL template;
- Fields `ignored_a text`, `ignored_b text` and `ignored_c text` in the DDL template are PG 14 version specific. PG 9.5 has none. Newer versions may have more.
