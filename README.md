## In the Psql console copy and paste the following to create the tables
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
```


## In the Cloud Shell editor copy and paste (use Contorl+V key to paste in the editor) the following to create the http_request load generator
```
-- loop continuously writing records every 1/4 second
DO $$
BEGIN LOOP
    INSERT INTO http_request (
    site_id, ingest_time, url, request_country,
    ip_address, status_code, response_time_msec
    ) VALUES (
    trunc(random()*32), clock_timestamp(),
    concat('http://example.com/', md5(random()::text)),
    ('{China,India,USA,Indonesia}'::text[])[ceil(random()*4)],
    concat(
        trunc(random()*250 + 2), '.',
        trunc(random()*250 + 2), '.',
        trunc(random()*250 + 2), '.',
        trunc(random()*250 + 2)
    )::inet,
    ('{200,404}'::int[])[ceil(random()*2)],
    5+trunc(random()*150)
    );
    COMMIT;
    PERFORM pg_sleep(random() * 0.25);
END LOOP;
END $$;
```

## In the Psql console copy and paste the following to see average response time for sites

SELECT site_id, ingest_time as minute, request_count,
    success_count, error_count, sum_response_time_msec/request_count as average_response_time_msec
FROM http_request_1min
WHERE ingest_time > date_trunc('minute', now()) - '5 minutes'::interval
LIMIT 15;
