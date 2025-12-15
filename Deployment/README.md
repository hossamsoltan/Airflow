# **Apache Airflow 3.1.4 Production Deployment (RHEL 9.4 + Python 3.11 + PostgreSQL + Celery Workers + LDAP + Kerberous + HTTPS + Systemd)**

This document provides a fully detailed, step-by-step guide to deploy **Apache Airflow 3.1.4** in **production mode** on:

* **OS:** Red Hat Enterprise Linux 9.4
* **Python:** 3.11
* **PostgreSQL Metadata DB:** `utility1.my.bigdata.local`
* **Airflow Server:** `airflow.my.bigdata.local`
* **Admin User:** hossam
* **Broker:** Redis
--- 

# **Prerequisites**

## Airflow®3.1.4 is tested with:

* **Python:** 3.10, 3.11, 3.12, 3.13
* **Databases:**
  * PostgreSQL: 13, 14, 15, 16, 17
  * MySQL: 8.0, Innovation
  * SQLite: 3.15.0+
* **Kubernetes:** 1.30, 1.31, 1.32, 1.33

* Ref: https://airflow.apache.org/docs/apache-airflow/3.1.4/installation/prerequisites.html
---

# **1. Prepare PostgreSQL Metadata Database (utility1.my.bigdata.local)**

Login as postgres:

```bash
sudo -i -u postgres
psql
```

Create Airflow DB + user:

```sql
CREATE USER airflow WITH PASSWORD 'airflow';
CREATE DATABASE airflow OWNER airflow;
ALTER USER airflow CREATEDB;   -- optional (not required in strict prod)
\l
\du
\q
```

> Use a **strong password** in production.
> Allowing `CREATEDB` is optional; recommended to disable later.

Ensure PostgreSQL allows Airflow host (edit `pg_hba.conf`):

```
host    airflow    airflow   airflow.my.bigdata.local/32   md5
```


# **2. Install Dependencies & Create Airflow User & Directory (airflow.my.bigdata.local)**

* Install OS Dependencies


    ```bash
    #Login with sudo user hossam
    sudo dnf install -y \
        gcc gcc-c++ make \
        python3-devel \
        krb5-devel krb5-libs krb5-workstation \
        openssl openssl-devel \
        openldap-devel openldap-clients \
        systemd-devel \
        libffi-devel \
        nfs-utils \
        python3.11 python3.11-devel python3.11-pip

    sudo dnf install -y redis # broker for Celery workers  
    ```
    ![alt text](image/image1.png)
* Create Airflow User & Directory
    ```bash
    sudo mkdir -p /opt/airflow
    sudo useradd -r -m -d /opt/airflow airflow
    sudo chown -R airflow:airflow /opt/airflow
    sudo chmod 755 /opt/airflow
    ```
    ![alt text](image/image2.png)

# **3. Configure Airflow Environment Variables & Python Virtual Environment**

```bash
sudo su - airflow
```

```bash
cd /opt/airflow
python3.11 -m venv airflow_venv
source airflow_venv/bin/activate
pip install --upgrade pip setuptools wheel
```
![alt text](image/image4.png)

```bash
# Edit `~/.bashrc`:
echo 'export AIRFLOW_HOME="/opt/airflow"' >> ~/.bashrc
echo 'export AIRFLOW_VERSION="3.1.4"' >> ~/.bashrc
echo 'export PYTHON_VERSION="3.11"' >> ~/.bashrc
echo 'export PATH="$AIRFLOW_HOME/airflow_venv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

#Note: Some times when use su - variables in bashrc coudln't be declared on the shell
```
![alt text](image/image3.png)
---

# **4. Install Apache Airflow 3.1.4 with Constraints**
irflow constraints ensure a stable, reproducible installation.

```bash
export CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
```

Install:

```bash
pip install \
  "apache-airflow[jdbc,kerberos,ldap,redis,rabbitmq,ssh,apache-hdfs,apache-spark,celery,microsoft-mssql,oracle,postgres]==${AIRFLOW_VERSION}" \
  --constraint "$CONSTRAINT_URL"

pip install apache-airflow-providers-fab #for LDAP Auth
```
![alt text](image/image5.png)



# **5. Generate Default airflow.cfg**

```bash
mkdir -p $AIRFLOW_HOME/{dags,logs,plugins,security}
airflow config list --defaults > $AIRFLOW_HOME/airflow.cfg
```

![alt text](image/image6.png)

# **6. Configure airflow.cfg for Postgressql Metadata, CeleryExecuter, Redis Broker, LDAP Auth, HTTPS, Kerberous**

Edit:

```bash
vim $AIRFLOW_HOME/airflow.cfg
```

Update:

```ini
[core]
dags_folder = /opt/airflow/dags
default_timezone = Africa/Cairo   # utc
executor = CeleryExecutor         # ``LocalExecutor``, ``CeleryExecutor``,``KubernetesExecutor``
auth_manager = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager  # for LDAP Auth
parallelism = 32  # you can change it 
max_active_tasks_per_dag = 16 # you can change it
max_active_runs_per_dag = 16 # you can change it
load_examples = False
plugins_folder = /opt/airflow/plugins
fernet_key = 6zKKUcsY-SnfdqzJ9agK0MG1h4inCaR3VNHJbrZO4cQ= # i will mention how to create this key
security = kerberos
test_connection = Enabled
execution_api_server_url = https://gateway1.my.bigdata.local:8080/execution/

[database]
# sql_alchemy_conn = postgresql+psycopg2://<user>:<password>@<host>/<db>
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@utility1.my.bigdata.local/airflow

[api]
instance_name = Production Airflow
secret_key = 0aa871b1a70d59a2a09c911982dcec54caff5f0dbe79f517809ad26d0557f383 # i will mention how to create it
expose_config = True
base_url = https://gateway1.my.bigdata.local:8080
workers = 4 # you can tune it
ssl_cert = /opt/airflow/security/airflow_cert.pem  # i will mention how to create it
ssl_key = /opt/airflow/security/airflow_key.pem

[api_auth]
jwt_secret = 0aa871b1a70d59a2a09c911982dcec54caff5f0dbe79f517809ad26d0557f383 # i will mention how to create it

[kerberos]
# ccache = /tmp/airflow_krb5_ccache you can use different location
ccache = /opt/airflow/security/krb5cc_airflow
principal = airflow@BIGDATA.LOCAL
keytab = /opt/airflow/security/airflow.keytab # i will mention how to create it from principle

[celery]
broker_url = redis://localhost:6379/0
result_backend = db+postgresql://airflow:airflow@utility1.my.bigdata.local:5432/airflow
#ssl_cacert = /opt/airflow/agent-cert/rootca.pem

```
Fernet_key
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```
![alt text](image/image7.png)

Secret_key and jwt_secret
```bash
python - << 'EOF'
import secrets
print(secrets.token_hex(32))
EOF
```
![alt text](image/image8.png)

CASE A: Generate the CSR and key to be singed from specific CA
```bash
openssl req -newkey rsa:4096 -sha256 \
-keyout /opt/airflow/security/airflow_key.pem \
-out /opt/airflow/security/airflow.csr \
-nodes \
-batch \
-subj '/CN=airflow.my.bigdata.local, O=Airflow Prod, C=EG' \
-extensions san -config <( echo '[req]'; echo 'distinguished_name=req'; echo 'req_extensions=san'; echo '[san]'; echo 'subjectAltName=DNS:airflow.my.bigdata.local,DNS:airflow,DNS:localhost,IP:192.168.1.70,IP:127.0.0.1'; echo 'extendedKeyUsage=serverAuth, clientAuth')
```
then send the airflow.csr to your CA administrator to sign it and send back the cert .pem or .cer airflow_cert.perm copy it to /opt/airflow/security/
![alt text](image/image9.png)

CASE B: Generate self-signed certitifcate 

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 \
-keyout /opt/airflow/security/airflow_key.pem \
-out /opt/airflow/security/airflow_cert.pem \
-nodes \
-batch \
-subj '/CN=airflow.my.bigdata.local, O=Airflow Prod, C=EG' \
-extensions san -config <( echo '[req]'; echo 'distinguished_name=req'; echo 'x509_extensions=san'; echo '[san]'; echo 'subjectAltName=DNS:airflow.my.bigdata.local,DNS:airflow,DNS:localhost,IP:192.168.1.70,IP:127.0.0.1'; echo 'extendedKeyUsage=serverAuth, clientAuth')

```

I will use CASE A

![alt text](image/image10.png)

Trust the certificate for the Virtual Environment 
```bash
cat /opt/airflow/security/airflow_cert.pem /opt/airflow/security/ca-chain.pem  >> /opt/airflow/airflow_venv/lib64/python3.11/site-packages/certifi/cacert.pem

#if self signed airflow_cert.pem is enough 
```

LDAP Configuration file
```bash
vim /opt/airflow/webserver_config.py
```

```ini
from airflow.configuration import conf
from flask_appbuilder.security.manager import AUTH_LDAP

SQLALCHEMY_DATABASE_URI = conf.get("database", "sql_alchemy_conn")
AUTH_TYPE = AUTH_LDAP

AUTH_ROLE_ADMIN = "Admin"
AUTH_ROLE_PUBLIC = "Public"  # or None to block anonymous access completely

AUTH_USER_REGISTRATION = True
AUTH_USER_REGISTRATION_ROLE = "User"  # default role for new LDAP users

AUTH_LDAP_SERVER = "ldap://dc.bigdata.local:389"  # <-- change to your DC

# If you use ldaps://, usually keep TLS off (TLS is for startTLS on 389)
AUTH_LDAP_USE_TLS = False

# Base DN where users are located
AUTH_LDAP_SEARCH = "CN=Users,DC=bigdata,DC=local"  

AUTH_LDAP_BIND_USER = "CN=super,CN=Users,DC=bigdata,DC=local"  
AUTH_LDAP_BIND_PASSWORD = "Admin&P@ssw0rd"  

AUTH_LDAP_UID_FIELD = "sAMAccountName"  # e.g. hossam.sultan

AUTH_LDAP_GROUP_FIELD = "memberOf"
AUTH_ROLES_MAPPING = {
    "CN=Domain Admins,CN=Users,DC=bigdata,DC=local": ["Admin"],
    "CN=Domain Users,CN=Users,DC=bigdata,DC=local": ["User"],
}
AUTH_ROLES_SYNC_AT_LOGIN = True
```
---

# **7. Run airflow services Manaully in background**

```bash
airflow db migrate
```
![alt text](image/image11.png)


```bash
airflow api-server \
  -D \
  --stdout /opt/airflow/logs/airflow_apiserver.out \
  --stderr /opt/airflow/logs/airflow_apiserver.err \
  -l /opt/airflow/logs/airflow_apiserver.log
```

![alt text](image/image12.png)

![alt text](image/image13.png)

![alt text](image/image14.png)

We will run schedule, trigger, dag processor, celery, kerberous

```bash
airflow scheduler \
  -D \
  --stdout /opt/airflow/logs/airflow_scheduler.out \
  --stderr /opt/airflow/logs/airflow_scheduler.err \
  -l /opt/airflow/logs/airflow_scheduler.log
```
```bash
airflow triggerer \
  -D \
  --stdout /opt/airflow/logs/airflow_triggerer.out \
  --stderr /opt/airflow/logs/airflow_triggerer.err \
  -l /opt/airflow/logs/airflow_triggerer.log
```
Airflow 3.x has a known bug: airflow dag-processor -D crashes with:

OSError: [Errno 22] Invalid argument
Solution: run WITHOUT -D:

```bash
nohup airflow dag-processor \
  -l /opt/airflow/logs/airflow_dag_processor.log \
  > /opt/airflow/logs/airflow_dag_processor.nohup 2>&1 &
```

```bash
airflow celery worker \
    -D \
    --stdout /opt/airflow/logs/airflow_celery_worker.out \
    --stderr /opt/airflow/logs/airflow_celery_worker.err \
    -l /opt/airflow/logs/airflow_celery_worker.log
```

```bash
airflow kerberos  \
  -D \
  --stdout /opt/airflow/logs/airflow_kerberos.out \
  --stderr /opt/airflow/logs/airflow_kerberos.err \
  -l /opt/airflow/logs/airflow_kerberos.log
```

![alt text](image/image15.png)

![alt text](image/image16.png)

As you see It is healthy now

# **8. Test Airflow DAG**


```bash
vim /opt/airflow/dags/testing.py
```

```ini
from datetime import datetime
from airflow import DAG
from airflow.providers.standard.operators.bash import BashOperator

with DAG("test_bash", start_date=datetime(2025,1,1), schedule=None, catchup=False):
    BashOperator(task_id="hello", bash_command="echo hi")

```
It will appear in Dags tab. trigger it manual for testing
![alt text](image/image17.png)

![alt text](image/image18.png)

Now Kill all airflow services to Create Production Systemd Services

```bash
ps -u airflow -o pid= | grep -v "$$" | xargs -r kill -9
```

---
# **9. Create Production Systemd Services**

Switch to admin user with sudo 
```bash
su - hossam
```

## Environment
- Airflow Home: /opt/airflow
- Airflow Config: /opt/airflow/airflow.cfg
- PostgreSQL: Remote server
- Redis: Local server
- Kerberos ccache: /opt/airflow/krb5cc/krb5cc_airflow
- systemd pattern: Based on official Apache Airflow scripts/systemd
https://github.com/apache/airflow/tree/main/scripts/systemd

---

## 1. Create Airflow Environment File

File: /etc/sysconfig/airflow

```bash
sudo tee /etc/sysconfig/airflow > /dev/null <<'EOF'
# ==============================
# Airflow Core
# ==============================
AIRFLOW_HOME=/opt/airflow
AIRFLOW_CONFIG=/opt/airflow/airflow.cfg

# ==============================
# Python Virtualenv
# ==============================
AIRFLOW_VENV=/opt/airflow/airflow_venv
PATH=/opt/airflow/airflow_venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# ==============================
# Logs
# ==============================
AIRFLOW_LOG_DIR=/opt/airflow/logs

# ==============================
# Hadoop / YARN
# ==============================
HADOOP_CONF_DIR=/etc/hadoop/conf
YARN_CONF_DIR=/etc/hadoop/conf

# ==============================
# Kerberos
# ==============================
KRB5CCNAME=/opt/airflow/security/krb5cc_airflow
KRB5_KTNAME=/opt/airflow/security/airflow.keytab
EOF
```

---

---

## 2. systemd Service Units

All files go under: `/etc/systemd/system/`

---

### 3.1 Airflow API Server

```bash
sudo tee /etc/systemd/system/airflow-api-server.service > /dev/null <<'EOF'
[Unit]
Description=Airflow API Server
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
Group=airflow
WorkingDirectory=/opt/airflow
Type=simple

ExecStart=/bin/bash -lc 'source "${AIRFLOW_VENV}/bin/activate" && exec airflow api-server -l /opt/airflow/logs/airflow_apiserver.log'

Restart=always
RestartSec=5

StandardOutput=append:/opt/airflow/logs/airflow_apiserver.out
StandardError=append:/opt/airflow/logs/airflow_apiserver.err

[Install]
WantedBy=multi-user.target
EOF
```

---

### 3.2 Airflow Scheduler

```bash
sudo tee /etc/systemd/system/airflow-scheduler.service > /dev/null <<'EOF'
[Unit]
Description=Airflow Scheduler
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
Group=airflow
WorkingDirectory=/opt/airflow
Type=simple

ExecStart=/bin/bash -lc 'source "${AIRFLOW_VENV}/bin/activate" && exec airflow scheduler -l /opt/airflow/logs/airflow_scheduler.log'

Restart=always
RestartSec=5

StandardOutput=append:/opt/airflow/logs/airflow_scheduler.out
StandardError=append:/opt/airflow/logs/airflow_scheduler.err

[Install]
WantedBy=multi-user.target
EOF
```

---

### 3.3 Airflow Triggerer

```bash
sudo tee /etc/systemd/system/airflow-triggerer.service > /dev/null <<'EOF'
[Unit]
Description=Airflow Triggerer
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
Group=airflow
WorkingDirectory=/opt/airflow
Type=simple

ExecStart=/bin/bash -lc 'source "${AIRFLOW_VENV}/bin/activate" && exec airflow triggerer -l /opt/airflow/logs/airflow_triggerer.log'

Restart=always
RestartSec=5

StandardOutput=append:/opt/airflow/logs/airflow_triggerer.out
StandardError=append:/opt/airflow/logs/airflow_triggerer.err

[Install]
WantedBy=multi-user.target
EOF
```

---

### 3.4 Airflow DAG Processor (NO `-D`)

> Airflow 3.x bug: `airflow dag-processor -D` crashes
> systemd runs it in foreground safely

```bash
sudo tee /etc/systemd/system/airflow-dag-processor.service > /dev/null <<'EOF'
[Unit]
Description=Airflow DAG Processor
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
Group=airflow
WorkingDirectory=/opt/airflow
Type=simple

ExecStart=/bin/bash -lc 'source "${AIRFLOW_VENV}/bin/activate" && exec airflow dag-processor -l /opt/airflow/logs/airflow_dag_processor.log'

Restart=always
RestartSec=5

StandardOutput=append:/opt/airflow/logs/airflow_dag_processor.nohup
StandardError=append:/opt/airflow/logs/airflow_dag_processor.nohup

[Install]
WantedBy=multi-user.target
EOF
```

---

### 3.5 Airflow Celery Worker

```bash
sudo tee /etc/systemd/system/airflow-celery-worker.service > /dev/null <<'EOF'
[Unit]
Description=Airflow Celery Worker
After=network-online.target redis.service
Wants=network-online.target redis.service

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
Group=airflow
WorkingDirectory=/opt/airflow
Type=simple

ExecStart=/bin/bash -lc 'source "${AIRFLOW_VENV}/bin/activate" && exec airflow celery worker -l /opt/airflow/logs/airflow_celery_worker.log'

Restart=always
RestartSec=5

StandardOutput=append:/opt/airflow/logs/airflow_celery_worker.out
StandardError=append:/opt/airflow/logs/airflow_celery_worker.err

[Install]
WantedBy=multi-user.target
EOF
```

---

### 3.6 Airflow Kerberos Ticket Renewer

```bash
sudo tee /etc/systemd/system/airflow-kerberos.service > /dev/null <<'EOF'
[Unit]
Description=Airflow Kerberos Ticket Renewer
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/sysconfig/airflow
User=airflow
Group=airflow
WorkingDirectory=/opt/airflow
Type=simple

ExecStart=/bin/bash -lc 'source "${AIRFLOW_VENV}/bin/activate" && exec airflow kerberos -l /opt/airflow/logs/airflow_kerberos.log'

Restart=always
RestartSec=10

StandardOutput=append:/opt/airflow/logs/airflow_kerberos.out
StandardError=append:/opt/airflow/logs/airflow_kerberos.err

[Install]
WantedBy=multi-user.target
EOF
```

---

## 4. Reload, Enable & Start Services

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable \
  airflow-api-server \
  airflow-scheduler \
  airflow-triggerer \
  airflow-dag-processor \
  airflow-celery-worker \
  airflow-kerberos
```

```bash
sudo systemctl start airflow-kerberos
sudo systemctl start airflow-api-server
sudo systemctl start airflow-scheduler
sudo systemctl start airflow-triggerer
sudo systemctl start airflow-dag-processor
sudo systemctl start airflow-celery-worker
```

---

## 5. Verification

```bash
systemctl status airflow-api-server airflow-scheduler airflow-triggerer airflow-dag-processor airflow-celery-worker airflow-kerberos --no-pager
```

```bash
klist -c /opt/airflow/krb5cc/krb5cc_airflow
```

---

## Notes

* PostgreSQL is remote → no dependency in systemd
* Redis is local → services wait for redis
* Kerberos ccache is shared across **all Airflow components**
* No env duplication inside DAGs or tasks
* Fully compliant with Airflow 3.x systemd model
* You can do the same test on step 8

---
