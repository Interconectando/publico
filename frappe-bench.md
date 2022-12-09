```sh
; Notes:
; priority=1 --> Lower priorities indicate programs that start first and shut down last
; killasgroup=true --> send kill signal to child processes too

[program:avc-server-frappe-web]
command=/opt/bench/avc-server/env/bin/gunicorn -b 127.0.0.1:8000 -w 9 --max-requests 5000 --max-requests-jitter 500 -t 120 frappe.app:application --preload
priority=4
autostart=true
autorestart=true
stdout_logfile=/opt/bench/avc-server/logs/web.log
stderr_logfile=/opt/bench/avc-server/logs/web.error.log
user=avc-server
directory=/opt/bench/avc-server/sites


[program:avc-server-frappe-schedule]
command=/home/avc-server/.local/bin/bench schedule
priority=3
autostart=true
autorestart=true
stdout_logfile=/opt/bench/avc-server/logs/schedule.log
stderr_logfile=/opt/bench/avc-server/logs/schedule.error.log
user=avc-server
directory=/opt/bench/avc-server

[program:avc-server-frappe-default-worker]
command=/home/avc-server/.local/bin/bench worker --queue default
priority=4
autostart=true
autorestart=true
stdout_logfile=/opt/bench/avc-server/logs/worker.log
stderr_logfile=/opt/bench/avc-server/logs/worker.error.log
user=avc-server
stopwaitsecs=1560
directory=/opt/bench/avc-server
killasgroup=true
numprocs=1
process_name=%(program_name)s-%(process_num)d

[program:avc-server-frappe-short-worker]
command=/home/avc-server/.local/bin/bench worker --queue short
priority=4
autostart=true
autorestart=true
stdout_logfile=/opt/bench/avc-server/logs/worker.log
stderr_logfile=/opt/bench/avc-server/logs/worker.error.log
user=avc-server
stopwaitsecs=360
directory=/opt/bench/avc-server
killasgroup=true
numprocs=1
process_name=%(program_name)s-%(process_num)d

[program:avc-server-frappe-long-worker]
command=/home/avc-server/.local/bin/bench worker --queue long
priority=4
autostart=true
autorestart=true
stdout_logfile=/opt/bench/avc-server/logs/worker.log
stderr_logfile=/opt/bench/avc-server/logs/worker.error.log
user=avc-server
stopwaitsecs=1560
directory=/opt/bench/avc-server
killasgroup=true
numprocs=1
process_name=%(program_name)s-%(process_num)d
```
