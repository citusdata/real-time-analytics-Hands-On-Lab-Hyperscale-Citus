```
-- this is run on the coordinator
CREATE TABLE http_request (
site_id INT,
ingest_time TIMESTAMPTZ DEFAULT now(),

url TEXT,
request_country TEXT,
ip_address TEXT,

status_code INT,
response_time_msec INT
) PARTITION BY RANGE (ingest_time);

-- Configure pgpartman to create daily partitions
SELECT partman.create_parent('public.http_request', 'ingest_time', 'native', 'daily');
UPDATE partman.part_config SET infinite_time_partitions = true;

-- Automatically create daily partitions
SELECT partman.run_maintenance(p_analyze := false);

-- Schedule automatic creation of partions on a daily basis
SELECT cron.schedule('@daily', $$SELECT partman.run_maintenance(p_analyze := false)$$);


CREATE TABLE http_request_1min (
site_id INT,
ingest_time TIMESTAMPTZ, -- which minute this row represents
request_country TEXT,

error_count INT,
success_count INT,
request_count INT,
sum_response_time_msec INT,
CHECK (request_count = error_count + success_count),
CHECK (ingest_time = date_trunc('minute', ingest_time)),
PRIMARY KEY (site_id, ingest_time,request_country)
);

CREATE TABLE latest_rollup (
minute timestamptz PRIMARY KEY,

CHECK (minute = date_trunc('minute', minute))
);


SELECT create_distributed_table('http_request',      'site_id');
SELECT create_distributed_table('http_request_1min', 'site_id');
```
