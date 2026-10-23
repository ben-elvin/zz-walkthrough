# Deploy notes: CloudByte Ops Reporter

## Which machine

The reporter runs on one EC2 instance on AWS (region eu-central-1). Inside AWS its name is
`ip-172-31-41-213.eu-central-1.compute.internal`. It is a Linux server, and the account to use is
`ec2-user`.

Its public address is `<HOST-IP>`. This repository is public, so the address is not written here;
ask the instructor who set the instance up, or read it in their message.

## How to get onto it

```
ssh -i ~/.ssh/cloudbyte-ops.pem ec2-user@<HOST-IP>
```

- The key file is `~/.ssh/cloudbyte-ops.pem` on the laptop. It must not be readable by anyone else:
  `chmod 600 ~/.ssh/cloudbyte-ops.pem`. The key is never kept in this repository (`.gitignore` skips `*.pem`).
- The first connection asks whether to trust the host's fingerprint. Answer `yes`.
- The server only accepts ssh from the laptop's public address. If ssh times out, check that address
  first.
- You are on when the prompt reads `[ec2-user@ip-172-31-41-213 ~]`. Type `exit` to come back.

## The clock

`date` on the host read `Fri Oct 23 13:26:29 UTC 2026`. On the laptop, at the same moment, it read
`Fri Oct 23 15:26:29 CEST 2026`. The host is on UTC all year and never changes for summer time;
the laptop is two hours ahead of it now (CEST).

What this means for a schedule: an hour set with the scheduler on the host is read on the host's
clock, in UTC. If the report must be ready at 07:00 laptop time (CEST, UTC+2), set the job for
05:00 on the host. When the laptop's clocks go back to CET (UTC+1) the gap shrinks to one hour and
the job's laptop-time hour moves with it, because the host does not change.

## The interpreter

| Machine | `python3 --version` |
|---|---|
| Host (Amazon Linux 2023, `/usr/bin/python3`) | `Python 3.9.25` |
| Laptop | `Python 3.13.5` |

The difference is expected. Nothing needs installing on the host: the reporter imports only the
standard library. The virtual environment stays on the laptop and does not travel.

## What travelled, and where it landed

Three files, all in one folder on the host, `/home/ec2-user/ops` (made with `mkdir -p`):

- `/home/ec2-user/ops/cloudbyte_report.py` the reporter
- `/home/ec2-user/ops/cloudbyte_access.log` the access log it reads
- `/home/ec2-user/ops/users.csv` the staff roster it joins

The reporter finds the log and roster by bare name, so it only works when started from
`/home/ec2-user/ops`. Run from any other folder it stops with `FileNotFoundError`.

Copied from the laptop with `scp -i ~/.ssh/cloudbyte-ops.pem <files> ec2-user@<HOST-IP>:/home/ec2-user/ops/`.
Nothing else travelled: not `.venv`, not `requirements.txt`, not `README.md`.

The host's Python is 3.9, so the reporter must not use syntax newer than 3.9. One line did
(`{first["date"]}` inside a double-quoted f-string needs Python 3.12); it now uses single quotes
inside the f-string.

Check: run on the host from `/home/ec2-user/ops`, the report is identical to the laptop's
(`diff laptop-report.txt host-report.txt` printed nothing).

## What starts it, when, and where the output goes

The scheduler is `cron` (package `cronie`, service `crond`). It was installed with
`sudo dnf install -y cronie && sudo systemctl enable --now crond`, and `systemctl is-active crond`
should print `active`. The job is in `ec2-user`'s own crontab (`crontab -l` as `ec2-user`, never
with `sudo`, which would use root's list):

```
0 5 * * * /usr/bin/python3 /home/ec2-user/ops/cloudbyte_report.py -v --roster /home/ec2-user/ops/users.csv /home/ec2-user/ops/cloudbyte_access.log > /home/ec2-user/report.txt 2> /home/ec2-user/cron-report.err
```

- **When:** 05:00 every day, on the host's clock (UTC). That is 07:00 on the laptop's clock while
  the laptop is on CEST (UTC+2), and 06:00 when it is on CET (UTC+1).
- **Report** (standard output): `/home/ec2-user/report.txt`, rewritten on every firing.
- **Notices** (standard error): `/home/ec2-user/cron-report.err`, rewritten on every firing. With
  `-v` it holds two `INFO:` lines.
- Every path in the line is absolute, because a scheduled run starts in the home directory with no
  terminal and a short `PATH`.

## How to check on the job

Nobody watches a firing, so the host's files are the only evidence. Get onto the host (see "How to
get onto it") and ask four questions, in this order:

1. **Is the entry installed?** `crontab -l` should print the `0 5 * * *` line above. If it prints
   `no crontab for ec2-user`, nothing is scheduled; also run `systemctl is-active crond`, which
   should print `active`.
2. **When did it last fire?** `ls -l ~/report.txt ~/cron-report.err`. Both files are rewritten on
   every firing, so their times are the last firing, in the host's clock (UTC). A time from before
   today at 05:00 means today's firing did not happen.
3. **Is the report right?** `head -1 ~/report.txt` must print `CloudByte Access Report`, and
   `wc -l ~/report.txt` must print `118` for the current log and roster.
4. **What did the run say?** `cat ~/cron-report.err` must hold exactly two lines:
   `INFO: loaded 45 records from /home/ec2-user/ops/cloudbyte_access.log` and
   `INFO: no --user given; skipping the per-user report`.

What a bad answer means (seen when `/home/ec2-user/ops/users.csv` was renamed on 23 Oct):

- `report.txt` is **0 bytes** with a fresh time, and `cron-report.err` ends on a traceback
  (`FileNotFoundError: ... '/home/ec2-user/ops/users.csv'`): the job fired, but a data file is
  missing or moved. Put the file back in `/home/ec2-user/ops`; the next firing repairs the report.
- Files older than expected, or `crontab -l` empty: the job did not fire; check questions 1 and 2.
- `report.txt` has a different length or a first line other than the header: something other than
  the report reached it; compare with the log and roster in `/home/ec2-user/ops`.
