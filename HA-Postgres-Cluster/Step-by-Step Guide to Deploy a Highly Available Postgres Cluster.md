<div align=center>
<img width=90% alt="Highly Available Postgres Cluster - Deployment View" src="https://github.com/libremomo/devops-notes/blob/main/HA-Postgres-Cluster/HA-Postgres-Cluster-Deployment-View.png">
</div>

# راهنمای گام به گام استقرار کلاستر پستگرس با معماری HA
در راهنمای حاضر، شیوهٔ راه‌اندازی یک کلاستر پستگرس با معماری Highly Available را توضیح می‌دهیم. برای این کار لازم است چند نمونهٔ پستگرس مستقر کنیم و مدیریت مکانیزم همتاسازی (replication) پستگرس را به پاترونی (Patroni) بسپریم. برای آنکه پاترونی بتواند وضعیت کلاستر را در هر لحظه ثبت و پایش کند، نیاز داریم یک کلاستر etcd راه‌اندازی کنیم و آن را در اختیار پاترونی قرار دهیم. در نهایت ترافیک ورودی را از طریق HAProxy بین نودهای پستگرس توزیع می‌کنیم. 
 
## ۶- ۱. پیش‌نیازها
برای راه‌اندازی این کلاستر دست‌کم به ۵ نود احتیاج داریم. این ۵ نود می‌توانند ماشین‌های لینوکسی فیزیکی یا مجازی باشند. ما در این راهنما از ماشین‌های مجازی استفاده می‌کنیم. حداقل نیازمندی‌های سخت‌افزاری و نرم‌افزاری این ماشین‌ها در زیر آمده است:

| No. | Hostname  |         Role         | CPU (Cores) | RAM (GB) | Disk (GB) | NIC |       IP        |        OS        |
| :-: | :-------: | :------------------: | :---------: | :------: | :-------: | :-: | :-------------: | :--------------: |
|  1  | psql-db-1 | psql, patroni, etcd  |     16      |    16    |    100    |  1  | 192.168.228.141 | Ubuntu 22.04 LTS |
|  2  | psql-db-2 | psql, patroni, etcd  |     16      |    16    |    100    |  1  | 192.168.228.142 | Ubuntu 22.04 LTS |
|  3  | psql-db-3 | psql, patroni, etcd  |     16      |    16    |    100    |  1  | 192.168.228.143 | Ubuntu 22.04 LTS |
|  4  | psql-lb-1 | HAProxy, Keepalived  |      4      |    8     |    25     |  1  | 192.168.228.144 | Ubuntu 22.04 LTS |
|  5  | psql-lb-2 | HAProxy, Keepalived  |      4      |    8     |    25     |  1  | 192.168.228.145 | Ubuntu 22.04 LTS |

- این ماشین‌ها باید بتوانند از حیث شبکه‌‌ای همدیگر را ببینند. بهترین حالت آن است که همگی در یک سابنت قرار داشته باشند تا میزان تأخیر ارتباطات آنها به حداقل برسد و در عملیات replication اختلالی رخ ندهد.
- لازم است یک آی‌پی دیگر نیز در همین سابنت برای استفاده به عنوان Floating IP کنار بگذاریم. (مثلاً: 192.168.228.140)
- کلاستر پستگرس نیازمند آن است که تاریخ و ساعت در همهٔ نودها به درستی و دقت تنظیم شده باشد. پیش از شروع فرایند نصب، از هماهنگ شدن ساعت هر نود با زمان اینترنتی مطمئن شوید.

## ۶- ۲. نصب پستگرس روی نودها

**تذکر:**
آنچه در این قسمت مطرح می‌کنیم عیناً باید روی هر سه نود پستگرس (psql-db-1~3) اجرا شود.

---
پستگرس نسخهٔ ۱۴ به صورت پیش‌فرض روی مخازن اوبونتو وجود دارد، اما ما از نسخهٔ ۱۷ استفاده می‌کنیم. برای نصب این نسخه بایستی نخست مخزن APT خود پستگرس را به ماشین‌ها اضافه کنیم. [طبق مستندات پستگرس](https://www.postgresql.org/download/linux/ubuntu/) از مسیر زیر اقدام ‌می‌کنیم:

```bash
# Import the repository signing key:
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Create the repository configuration file:
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```
پس از ایجاد `pdgd.list` یک بار کش APT را به‌روزرسانی می‌کنیم:
```bash
sudo apt update
```
در نهایت با فرمان زیر، پستگرس را نصب می‌کنیم:
```
sudo apt -y install postgresql-17 postgresql-server-dev-17
```
برای اطلاعات بیشتر در مورد نحوهٔ نصب پستگرس [اینجا](https://www.postgresql.org/download/linux/ubuntu/) را ببینید. 
پس از اتمام عملیات، باید از صحت نصب مطمئن شویم. نخست محل باینری‌های پستگرس را بررسی می‌کنیم:
```bash
which psql
```
خروجی فرمان بالا باید چنین باشد:
```bash
/usr/sbin/psql
```
سپس، با فرمان زیر وضعیت سرویس پستگرس را بررسی می‌کنیم:
```bash
sudo systemctl status postgresql
```
از آنجا که باید کنترل پستگرس را به پاترونی بسپریم، سرویس فعلی را که به صورت خودکار اجرا شده، متوقف می‌کنیم:
```bash
sudo systemctl stop postgresql
```
پاترونی از برخی از ابزارهایی که همراه با پستگرس نصب می‌شوند استفاده می‌کند، در نتیجه لازم است یک پیوند نمادین (symlink) از باینری‌های آن در مسیر `/usr/sbin/` قرار دهیم تا مطمئن شویم پاترونی به آنها دسترسی خواهد داشت:
```bash
ln -s /usr/lib/postgresql/17/bin/* /usr/sbin/
```

## ۶- ۳. نصب پاترونی روی نودها

**تذکر:**
آنچه در این قسمت مطرح می‌کنیم باید عیناً روی هر سه نود پستگرس (psql-db-1~3) اجرا شود.

---
پاترونی یک بستهٔ متن‌باز پایتونی است که پیکربندی پستگرس را مدیریت می‌کند. پاترونی را می‌توان برای انجام وظایفی مانند همتاسازی، پشتیبان‌گیری، و بازیابی پایگاه‌های داده تنظیم کرد. در این راهنما از پاترونی برای موارد زیر استفاده می‌کنیم:
- پیکربندی نمونهٔ پستگرسی که روی همان سرور اجرا می‌شود
- پیکربندی همتاسازی داده‌ها از روی نود پستگرس اصلی (primary) به نودهای آماده‌به‌کار (standby)
- انتخاب نود اصلی جدید به صورت خودکار (automatic failover) در صورت افتادن نود اصلی فعلی

برای نصب پاترونی از دستور زیر استفاده می‌کنیم:
```bash
sudo apt install patroni
```
پس از آن، پکیج `python3-psycopg2` را هم نصب می‌کنیم.`Psycopg` آداپتور پستگرس برای پایتون است و برنامه‌های پایتونی و پایگاه‌های دادهٔ پستگرس را به هم متصل می‌کند:
```bash
sudo apt install python3-psycopg2
```
پس از اتمام نصب، مسیر باینری‌ها و نسخهٔ پاترونی را با دستورهای زیر را بررسی می‌کنیم:
```bash
which patroni

patroni --version
```
## ۶- ۴. نصب etcd روی نودها
پیش از آنکه نودهای پاترونی را پیکربندی کنیم، لازم است زیرساخت لازم برای ثبت و نگهداری وضعیت کلاستر پستگرس راه‌اندازی کنیم. برای این منظور از etcd که یک پایگاه دادهٔ کلید-مقداری است، به منزلهٔ مخزن توزیع‌شده پیکربندی (DCS) استفاده می‌کنیم. برای اینکه این مخزن در برابر بروز خطا تاب‌آوری داشته باشد و بتواند در صورت از کار افتادن یک نود به کار خود ادامه دهد، لازم است که روی هر سه نود یک نمونه etcd راه‌اندازی کنیم و هر سه را با هم تبدیل به یک کلاستر کنیم. توجه کنید که در کلاستر etcd از الگوریتم Raft برای اجماع و انتخاب رهبر استفاده می‌شود، در نتیجه لازم است حتماً تعداد نودها **فرد** باشد.

برای نصب etcd و پکیج‌های مورد نیاز دیگر از دستور زیر استفاده می‌کنیم:
```bash
sudo apt install etcd-server etcd-client python3-etcd
```
- پکیج `etcd-server` حاوی باینری‌های دیمن etcd است.
- پکیج `etcd-client` حاوی باینری‌های کلاینت etcd است.
- پکیج `python3-etcd` یک کلاینت پایتونی برای تعامل با etcd است که به برنامه های پایتونی اجازه می‌دهد تا با خوشه های etcd ارتباط برقرار کرده و مدیریت کنند.

---
**تذکر:**
عملیات بالا را باید عیناً روی هر سه نود پستگرس (psql-db-1~3) اجرا کرد.

---
## ۶- ۵. پیکربندی نودهای etcd
برای پیکربندی هر نود etcd، لازم است نخست فایل‌هایی که در شاخهٔ `/var/lib/etcd/` قرار دارد را پاک می‌کنیم. وجود فایل‌های پیش‌فرض در این مسیر باعث می‌شود نودهای etcd همگی با یک UUID یکسان ایجاد شوند و به همین خاطر نتوانند یکدیگر را به عنوان اعضای یک کلاستر شناسایی کنند:
```bash
sudo rm -rf /var/lib/etcd/*
```
سپس فایلی را که در مسیر `etc/default/etcd/` قرار دارد در ویرایشگر باز می‌کنیم و خطوط زیر را به آن می‌افزاییم. تنها لازم است مقدار `etcd-node-name` و `etcd-node-ip` را برای هر نود مطابق جدول زیر در آن جایگذاری کنیم.

| etcd-node-name | etcd-node-ip |
| ---------------| -------------|
|etcd-1 | 192.168.228.141 |
|etcd-2 | 192.168.228.142 |
|etcd-3 | 192.168.228.143 |


```
ETCD_NAME="<etcd-node-name>"
ETCD_LISTEN_PEER_URLS="http://<etcd-node-ip>:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://<etcd-node-ip>:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<etcd-node-ip>:2380"
ETCD_INITIAL_CLUSTER="etcd-1=http://192.168.228.141:2380,etcd-2=http://192.168.228.142:2380,etcd-3=http://192.168.228.143:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://<etcd-node-ip>:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
 پس از اتمام ویرایش فایل پیکربندی، سرویس etcd را بازنشانی می‌کنیم:
```bash
sudo systemctl restart etcd
```
پس از بازنشانی، وضعیت سرویس را از طریق فرمان‌های زیر چک می‌کنیم:
```bash
sudo systemctl is-enabled etcd 
sudo systemctl status etcd
```
خروجی فرمان اول باید `enabled` و خروجی فرمان دوم باید `active (running)` باشد. اکنون می‌توانیم با فرمان زیر بررسی کنیم که آ‌یا کلاستر به درستی شکل گرفته است یا نه:
```bash
etcdctl member list
```
خروجی این فرمان مشابه این خواهد بود:
```bash
38c2f5960a646674: name=etcd-3 peerURLs=http://192.168.228.143:2380 clientURLs=http://192.168.228.143:2379 isLeader=true                           

659f84d35a13cf2f: name=etcd-1 peerURLs=http://192.168.228.141:2380 clientURLs=http://192.168.228.141:2379 isLeader=false                          

731fb9c8c2f57b00: name=etcd-2 peerURLs=http://192.168.228.142:2380 clientURLs=http://192.168.228.142:2379 isLeader=false
```
**نکته:** اگر پکیج `etcd-client` را نصب نکرده باشید نیز می‌توانید از فرمان زیر استفاده کنید:
```bash
curl http://<etcdnode_ip>:2380/members
```
همان‌طور مشخص است کلاستر etcd به درستی شکل گرفته است و نودها همدیگر را شناخته‌اند و یک نود را به عنوان رهبر انتخاب کرده‌اند. اکنون می‌توانیم پیکربندی پاترونی را آغاز کنیم.

## ۶- ۶. پیکربندی نودهای پاترونی
پیکربندی پاترونی مهم‌ترین بخش راه‌اندازی یک کلاستر Highly Available پستگرس است. بنابر این، بخش‌های مختلف آن را گام به گام اضافه می‌کنیم و پارامتر هر بلاک را جداگانه توضیح می‌دهیم.

---
**تذکر:**
- پیکربندی پاترونی باید روی هر نود پستگرس به صورت جداگانه صورت پذیرد. 
- نمونهٔ کاملی از فایل پیکربندی پاترونی را می‌توان در مخزن [گیتهاب پاترونی](https://github.com/patroni/patroni/blob/master/postgres0.yml) پیدا کرد.

---
نخست، فایل `etc/patroni/config.yml/` را در ویرایشگر دلخواهمان باز می‌کنیم و پیکربندی زیر را به آن می‌افزاییم. در قسمت `name` باید `hostname` هر نود پستگرس را بنویسیم. مثلاً برای نود ۱، `dc1-psql-db-1` را قرار می‌دهیم و برای ۲ نود دیگر نیز به همین ترتیب عمل می‌کنیم:
```yml
scope: postgres
namespace: /db/
name: <hostname>
```
 آدرس API پاترونی را تکمیل می‌کنیم. به جای `<node-ip>` آی‌‌پی هر نود را وارد می‌کنیم:
```yml
restapi:
  listen: <node-ip>:8008
  connect_address: <node-ip>:8008
```
آدرس آی‌پی و پورت نودهای کلاستر etcd را به پاترونی می‌دهیم:
```yml
etcd:
  hosts: 192.168.228.141:2379,192.168.228.142:2379,192.168.228.143:2379
```
سپس پیکربندی‌های زیر را به بخش `bootstrap` می‌افزاییم. در این بخش تعریف می‌کنیم که نود پستگرس باید در هنگام راه‌اندازی اولیه از چه مقادیر و تنظیمات پیش‌فرضی استفاده کند. همچنین نحوهٔ تعامل پاترونی با مخزن توزیع‌شده پیکربندی (DCS) یا همان etcd را در اینجا تنظیم می‌کنیم:
```yml
bootstrap:
  dcs:
    # Time-to-live for the leader lock, in seconds.
    ttl: 30
    # Number of seconds to wait between checks on the cluster state.
    loop_wait: 10
    # Timeout for DCS and PostgreSQL operations, in seconds.
    retry_timeout: 10
    # Maximum allowed lag in bytes for a follower to be considered as a candidate for promotion.
    maximum_lag_on_failover: 1048576
    postgresql:
      # Enables the use of pg_rewind for faster failover recovery.
      use_pg_rewind: true
      # Uses replication slots, which ensures that the primary does not remove WAL segments needed by replicas.
      use_slots: true
      parameters:
        # Enables hot standby mode and ensures that enough info is written to the WAL to support a robust HA setup.
        wal_level: hot_standby
        # Allows read-only queries on standby servers.
        hot_standby: "on"
        # Maximum number of concurrent connections.
        max_connections: 100
        # Maximum number of background worker processes.
        max_worker_processes: 8           
        # Minimum number of past log file segments to keep for standby servers.
        wal_keep_segments: 8
        # Maximum number of concurrent connections for streaming replication.
        max_wal_senders: 10
        # Maximum number of replication slots.
        max_replication_slots: 10
        # Maximum number of prepared transactions (0 disables the feature).
        max_prepared_transactions: 0
        # Maximum number of locks per transaction.
        max_locks_per_transaction: 64
        # Logs additional information in the WAL to support pg_rewind.
        wal_log_hints: "on"
        # Disables tracking of commit timestamps.
        track_commit_timestamp: "off"
        # Enables WAL archiving.
        archive_mode: "on"
        # Forces a switch to a new WAL file if a new file hasn't been started within 1800 seconds.
        archive_timeout: 1800s
        # Command to archive WAL files. This command creates a directory named 'wal_archive', checks if the file doesn't already exist, and then copies it
        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
      recovery_conf:
        # Command used to retrieve archived WAL files during recovery. It copies files from the 'wal_archive' directory.
        restore_command: cp ../wal_archive/%f %p
  initdb:
  # Sets the default authentication method
  - auth: scram-sha-256
  # Sets the default character encoding for the database to UTF-8, which supports a wide range of characters.
  - encoding: UTF8
  # Enables data checksums on this cluster, which helps detect corruption caused by hardware or I/O subsystem faults.
  - data-checksums
```

در ادامه نحوهٔ احراز هویت کلاینت‌های کلاستر را تعیین می‌کنیم (`pg_hba.conf`) و به کاربر ‍`replicator` که برای همتاسازی بین نودهای کلاستر از آن استفاده می‌شود، اجازه می‌دهیم که با مکانیزم `scram-sha-256` که امن‌ترین شیوه احراز هویت از طریق رمز عبور است، به نودهای کلاستر پستگرس دسترسی پیدا کند. رمز عبور کاربر `admin` را نیز در ادامهٔ همین قسمت تعیین کرده و آن را جایگزین `<some-secure-password>` می‌کنیم:
```yml
  # This section configures the pg_hba.conf file, which controls client authentication for PostgreSQL.
  pg_hba:
  - host replication replicator 127.0.0.1/32 scram-sha-256
  - host replication replicator 192.168.228.141/0 scram-sha-256
  - host replication replicator 192.168.228.142/0 scram-sha-256
  - host replication replicator 192.168.228.143/0 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256

  # This section defines database users and their properties.
  users:
    admin:
      password: <some-super-secure-password>
      options:
        - createrole
        - createdb
```

سپس، باقی‌ماندهٔ تنظیمات پستگرس و پاترونی، مانند آدرس پستگرس در هر نود، رمز عبور کاربر `replicator` و `postgres`، و ... را وارد کرده و تغییرات را ذخیره می‌کنیم.
```yml
postgresql:
  # Specifies the IP address and port on which PostgreSQL should listen for connections.	
  listen: <node-ip>:5432
  # The address and port other nodes should use to connect to this instance.
  connect_address: <node-ip>:5432
  # The directory where PostgreSQL will store its data files.
  data_dir: /var/lib/patroni/
  # The location of the .pgpass file, which stores passwords for non-interactive connections.
  pgpass: /tmp/pgpass

  # This defines authentication details for different types of connections.
  authentication:
    replication:
      username: replicator
      password: <some-secure-pass-for-replicator-user>
    superuser:
      username: postgres
      password: <some-secure-pass-for-postgres-user>

  parameters:
    # Sets the current directory as the location for Unix-domain sockets.
    unix_socket_directories: '.'
    password_encryption: 'scram-sha-256'

# These are Patroni-specific tags that control various behaviors
tags:
    # Allows this node to be considered for failover.
    nofailover: false
    # Allows this node to be considered for load balancing.
    noloadbalance: false
    # This node will not be used as a preferred source when adding new nodes.
    clonefrom: false
    # Allows this node to be considered for synchronous replication.
    nosync: false
```

پس از اتمام ویرایش فایل `config.yml` لازم است مالکیت شاخه‌ای را که برای ذخیره‌سازی داده‌های پاترونی در نظر گرفتیم (`/var/lib/patroni/`) به کاربر`postgres` انتقال بدهیم و دسترسی خواندن و نوشتن را فقط به مالک شاخه محدود کنیم:
```bash
mkdir -p /var/lib/patroni

chown -R postgres:postgres /var/lib/patroni

chmod 700 /var/lib/patroni
```

اکنون می‌توانیم سرویس پاترونی را روی هر سه نود راه‌اندازی کنیم. با این کار، نود اول به عنوان رهبر انتخاب شده و دو نود دیگر از طریق API پاترونی به آن می‌پیوندند.
```bash
systemctl start patroni 
systemctl status patroni
```
---
**تذکر:**
چنانچه پس از راه‌اندازی اولیه لازم شد در پارامترهای قسمت `bootstrap.dcs` تغییری ایجاد کنیم، باید از دستور ‍`patronictl edit-config` استفاده کنیم.

---

برای بررسی نهایی نودهای پستگرسی که پاترونی مدیریت می‌کند، فرمان زیر را روی یکی از نودها اجرا کنید:
```bash
patronictl -c /etc/patroni/config.yml list
```

خروجی آن باید چنین چیزی باشد:
```
+ Cluster: postgres (7422247765142374405) --+-----------+----+-----------+
| Member        | Host            | Role    | State     | TL | Lag in MB |
+---------------+-----------------+---------+-----------+----+-----------+
| dc1-psql-db-1 | 192.168.228.141 | Leader  | running   |  1 |           |
| dc1-psql-db-2 | 192.168.228.142 | Replica | streaming |  1 |         0 |
| dc1-psql-db-3 | 192.168.228.143 | Replica | streaming |  1 |         0 |
+---------------+-----------------+---------+-----------+----+-----------+
```

در نهایت اجرای اتوماتیک سرویس پستگرس را پس از بوت شدن مجدد سیستم غیرفعال می‌کنیم تا کنترل کلاستر پستگرس در دست پاترونی باقی بماند. اگر اجرای این فرمان موفقیت‌آمیز باشد، خروجی خاصی نخواهد داشت:
```bash
sudo systemctl disable --now postgresql
```
## ۶- ۷. نصب و پیکربندی HAProxy
پس از استقرار موفقیت‌آمیز کلاستر پستگرس باید ترافیک کلاینت‌ها را بین نودها توزیع کنیم. برای این منظور لازم است از لودبالانسرهای لایهٔ ۴ مانند HAProxy بالای سر کلاستر استفاده کنیم. در محیط‌های تست و توسعه می‌شود HAProxy را مستقیماً بر روی نودهای پستگرس نصب کرد. اما در محیط پروداکشن به دلایل زیر لازم است نودهای مجزایی برای آن در نظر بگیریم:
- **عملکرد**: پستگرس، به‌ویژه زمانی که تحت فشار زیادی باشد، منابع زیادی مصرف می‌کند. از این‌رو، اختصاص دادن ماشین‌های مجزا به لودبالانسر به ما اطمینان می‌دهد که عملکرد دیتابیس بابت عملیات توزیع بار تحت تأثیر قرار نمی‌گیرد.
- **قابلیت نگهداری**: در نظر گرفتن ماشین‌های مجزا برای لودبالانسر به ما امکان می‌دهد که تعمیر، به‌روزرسانی و عیب‌یابی در لایهٔ توزیع بار را بدون درگیر کردن نودهای پستگرس انجام بدهیم.
- **مقیاس‌پذیری**: به اقتضای مقیاس‌پذیری ممکن است لازم باشد لایهٔ توزیع بار را به نحو متفاوتی نسبت به لایهٔ دیتابیس گسترش بدهیم. داشتن ماشین‌های مجزا این انعطاف‌پذیری را به ما می‌دهد.
- **امنیت**: با استفاده از نودهای مجزا برای توزیع بار می‌توانیم در صورت لزوم معماری شبکهٔ ایمن‌تری را اجرا کنیم و از یک سو، نودهای دیتابیس را در یک شبکهٔ خصوصی بگذاریم و از سوی دیگر، نودهای HAProxy را در دسترسی لایه‌ٔ اپلیکیشن قرار بدهیم.
---
**تذکر:**
نصب و پیکربندی HAProxy باید عیناً روی هر کدام از نودهای توزیع بار صورت پذیرد. 

---
برای نصب HAProxy از فرمان زیر استفاده می‌کنیم:
```bash
sudo apt update
sudo apt install haproxy
```
پس از اتمام نصب، در ابتدا از فایل پیکربندی پیش فرض HAProxy یک پشتیبان درست می‌کنیم:
```bash
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
```
سپس فایل جدیدی به نام `etc/haproxy/haproxy.cfg/` ایجاد می‌کنیم و خطوط زیر را به آن اضافه می‌کنیم:
```
# Global configuration settings
global
    # Maximum connections globally
    maxconn 100
    # Logging settings
    log 127.0.0.1 local2

# Default settings
defaults
    # Global log configuration
    log global
    # Set mode to TCP
    mode tcp
    # Number of retries
    retries 2
    # Client timeout
    timeout client 30m
    # Connect timeout
    timeout connect 4s
    # Server timeout
    timeout server 30m
    # Check timeout
    timeout check 5s

# Stats configuration
listen stats
    # Set mode to HTTP
    mode http
    # Bind to port 8080
    bind *:8080
    # Enable stats
    stats enable
    # Stats URI
    stats uri /

# PostgreSQL configuration
listen production
    # Bind to port 5000
    bind *:5000
    # Enable HTTP check
    option httpchk OPTIONS/master
    # Expect status 200
    http-check expect status 200
    # Server settings
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    # Define PostgreSQL servers
    server dc1-psql-db-1 192.168.228.141:5432 maxconn 100 check port 8008
    server dc1-psql-db-2 192.168.228.142:5432 maxconn 100 check port 8008
    server dc1-psql-db-3 192.168.228.143:5432 maxconn 100 check port 8008
listen standby
    bind *:5001
    option httpchk OPTIONS/replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server dc1-psql-db-1 192.168.228.141:5432 maxconn 100 check port 8008
    server dc1-psql-db-2 192.168.228.142:5432 maxconn 100 check port 8008
    server dc1-psql-db-3 192.168.228.143:5432 maxconn 100 check port 8008
```

بر اساس پیکربندی فوق هر نود HAProxy روی پورت `5000` منتظر دریافت کانکشن‌ها به نود Primary و روی پورت `5001` منتظر دریافت کانکشن‌ها به نودهای replica است. بنابر این اکنون می‌توان درخواست‌های write را به نود اصلی فرستاد اما درخواست‌های read را به صورت roundrobin میان سه سروری که در انتهای پیکربندی به عنوان بک‌اند تعریف شده‌اند، توزیع کرد. 
علاوه بر این، طبق تنظیمات بلاک `listen stats`، داشبورد وضعیت HAProxy روی پورت `8080` قابل دسترس خواهد بود. در تنظیمات هر دو بلاک `listen production` و `listen standby`، در بخش `default-server` نیز پارامترهایی معین شده که رفتار HAProxy را در قبال سرورهای بک‌اند تعیین می‌کند:
- فاصلهٔ زمانی بین چک‌های سلامتی ۳ ثانیه است (`inter 3s`).
- اگر چک سلامتی ۳ بار پیاپی ناموفق باشد، آن نود به عنوان نود افتاده در نظر گرفته می‌شود (`fall 3`).
- اگر چک سلامتی ۲ بار پیاپی موفق باشد، آن نود مجدداً برقرار در نظر گرفته می‌شود (`rise 2`).
- اگر یک سرور به عنوان افتاده در نظر گرفته شود، HAProxy بلافاصله تمامی سشن‌های آن سرور را می‌بندد. این امر کمک می‌کند که سرورهای معیوب به سرعت کنار بروند و ترافیک به سروهای سالم هدایت شود (‍‍`on-marked-down shutdown-sessions`).

اکنون می‌توان سرویس HAProxy را به کار انداخت و صحت آن را بررسی کرد:
```bash
sudo systemctl restart haproxy 
sudo systemctl status haproxy
```
چنانچه راه‌اندازی سرویس ناموفق بود، با دستور زیر خطاهای فایل پیکربندی را بررسی کنید:
```bash
/usr/sbin/haproxy -c -V -f /etc/haproxy/haproxy.cfg
```
## ۶- ۸. نصب و پیکربندی Keepalived
برای آنکه نودهای توزیع بار بتوانند در یک حالت Highly Available کار کنند و در صورت بروز عیب در یکی از آنها، نود دیگر بتواند به کار ادامه دهد لازم است یک آی‌پی شناور بین آنها وجود داشته باشد که در هر لحظه فقط روی یکی از آن دو نود فعال باشد و ترافیک پروداکشن به سمت آن هدایت شود. برای این منظور از Keepalived می‌کنیم.
برای نصب Keepalived روی نودهای توزیع بار از فرمان زیر استفاده می‌کنیم:
```bash
sudo apt install keepalived
```
پس از نصب، هر کدام از نودها را جداگانه پیکربندی می‌کنیم. پیکربندی نود اول (`192.168.228.144`) را در مسیر `etc/keepalived/keepalived.conf/` به این صورت انجام می‌دهیم:
```
global_defs {
}
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2 
}
vrrp_instance VI_1 {
    interface ens33
    state MASTER
    priority 101          # 101 on master, 100 on backup
    virtual_router_id 51
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <some-secure-password>
    }
    virtual_ipaddress {
        192.168.228.140
    }
    unicast_src_ip 192.168.228.144  # This haproxy node
    unicast_peer {
    192.168.228.145                 # Other haproxy nodes
    }
    track_script {
        chk_haproxy
    }
}
```
به طریق مشابه پیکربندی نود دوم (`192.168.228.145`) را در همان مسیر `etc/keepalived/keepalived.conf/` به این صورت انجام می‌دهیم:
```
global_defs {
}
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2 
    weight 2 
}
vrrp_instance VI_1 {
    interface ens33
    state BACKUP
    priority 100
    virtual_router_id 51
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <some-secure-password>
    }
    virtual_ipaddress {
        192.168.228.140
    }
    unicast_src_ip 192.168.228.145  # This haproxy node
    unicast_peer {
    192.168.228.144                 # Other haproxy nodes
    }
    track_script {
        chk_haproxy
    }
}
```
در هر دو پیکربندی بالا باید این نکات توجه کرد:
- در قسمت `interface` نام کارت شبکهٔ هر ماشین را بنویسید.
- مقدار `virtual_router_id` یک عدد انتخابی است اما بین همهٔ نودهای Keepalived باید یکسان باشد.
- رمزی که برای ارتباط دو نود Keepalived به جای `<some-secure-pass>` قرار داده می‌شود باید در هر دو نود یکسان باشد.
- آی‌پی شناور (`virtual_ipaddress`) در هر دو پیکربندی یکسان و برابر با `192.168.228.140` است.
- در هر نود، `unicast_src_ip` معادل با آی‌پی همان نود و `unicast_peer` فهرستی از نودهای دیگر Keepalived است. هر کدام از آیپی‌های دیگر را باید در یک خط نوشت.

پس ذخیره کردن پیکربندی Keepalived در هر دو نود می‌توانیم سرویس آن را فعال کنیم:
```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```
درستی عملکرد Keepalived را نخست با بررسی وضعیت سرویس از طریق فرمان زیر:
```bash
sudo systemctl status keepalived
```
و سپس از طریق بررسی آی‌پی‌های کارت شبکه با فرمان `ip a` متوجه می‌شویم. چنانچه نود ۱ علاوه بر آی‌پی خودش، آی‌پی شناور را نیز دریافت کرده باشد یعنی راه‌اندازی Keepalived به درستی انجام شده و مراحل راه‌اندازی کلاستر Highly Available پستگرس با ۳ نود دیتابیس و ۲ نود توزیع‌کنندهٔ بار به موفقیت به پایان رسیده است. هم اکنون می‌توانیم از آدرس `192.168.228.140:5000` برای درخواست‌های read/write و از `192.168.228.140:5001` برای درخواست‌های read-only استفاده کنیم.

## ۶- ۹. تست عملکرد
برای تست عملیات failover، داشبورد HAProxy را که روی آدرس `192.168.228.140:8080` در دسترس است، باز می‌کنیم. دقت کنید که پورت 8080 را قبلاً خودمان در پیکربندی HAProxy تعیین کرده‌ایم و آی‌پی شناور نیز به همیشه ما را به آن نود HAProxy که ترافیک پروداکشن روی آن جریان دارد هدایت می‌کند. ذیل جدول `postgres` فهرست نودهای بک‌اند را مشاهده می‌کنید. نود اول که اکنون به عنوان primary عمل می‌کند سبز رنگ و `up` است و دو نود دیگر که در حالت streaming قرار دارند قرمز و `down` نشان داده می‌شوند.
اکنون با دستور زیر نود ۱ را از مدار خارج می‌کنیم:
```bash
sudo systemctl stop patroni
```
سپس وضعیت پاترونی را با دستور زیر چک می‌کنیم:
```bash
patronictl -c /etc/patroni/config.yml list
```
خروجی باید چنین چیزی باشد:
```
+ Cluster: postgres (7422247765142374405) --+-----------+----+-----------+
| Member        | Host            | Role    | State     | TL | Lag in MB |
+---------------+-----------------+---------+-----------+----+-----------+
| dc1-psql-db-1 | 192.168.228.141 | Replica | stopped   |    |   unknown |
| dc1-psql-db-2 | 192.168.228.142 | Replica | running   |  1 |         0 |
| dc1-psql-db-3 | 192.168.228.143 | Leader  | running   |  2 |           |
+---------------+-----------------+---------+-----------+----+-----------+
```

اکنون اگر به داشبود HAProxy بازگردید، خواهید دید که نود اول `DOWN` شده و به جای آن یکی از دو نود دوم و سوم `UP` شده است. در نهایت، زمانی که نود ۱ را دوباره به کلاستر برگردانید، به عنوان standby با نود primary همگام‌سازی خواهد شد.

## ۶- ۱۰. پیوندها
- [Streaming Replication Protocol - PostgreSQL Documentation](https://www.postgresql.org/docs/current/protocol-replication.html)
- [Patroni GitHub Repo](https://github.com/patroni/patroni/tree/master)
## ۶- ۱۱. منابع
- [High Availability PostgreSQL with Patroni and HAProxy - ATA Learning ](https://adamtheautomator.com/patroni/)
- [PostgreSQL HA Architecture - Mahmut Can Uçanefe](https://medium.com/@c.ucanefe/patroni-ha-proxy-feed1292d23f)
- [Create a Highly Available PostgreSQL Cluster Using Patroni and HAProxy - Linode Docs](https://www.linode.com/docs/guides/create-a-highly-available-postgresql-cluster-using-patroni-and-haproxy/)
- [Highly Available PostgreSQL Cluster using Patroni and HAProxy - JFrog Community](https://jfrog.com/community/devops/highly-available-postgresql-cluster-using-patroni-and-haproxy/)
