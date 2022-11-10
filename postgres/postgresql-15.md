# PostgreSQL-15 on Debian-11

## 安装脚本

```bash
#!/bin/bash

sudo mkdir -p /pgsql/postgresql
sudo ln -s /pgsql/postgresql /var/lib/postgresql

sudo mkdir -p /pgsql/log
sudo ln -s /pgsql/log /var/log/postgresql

curl -s https://mirrors.ustc.edu.cn/postgresql/repos/apt/ACCC4CF8.asc  | sudo gpg  --no-default-keyring --keyring gnupg-ring:/etc/apt/trusted.gpg.d/postgresql.gpg --import 

sudo chmod 644 /etc/apt/trusted.gpg.d/postgresql.gpg

echo "deb [arch=amd64] https://mirrors.ustc.edu.cn/postgresql/repos/apt/ bullseye-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list

sudo apt update

sudo apt install -y postgresql-15 postgresql-contrib libpq-dev

echo "listen_addresses='*'" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
echo "host    all             all             0.0.0.0/0              md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf


sudo systemctl enable postgresql.service
sudo systemctl restart postgresql.service

sudo -u postgres psql -c "alter user postgres with password 'pg1234'"

sudo chmod 700 -R /pgsql
sudo chown -R postgres:postgres /pgsql

sudo -u postgres mkdir -p /pgsql/tblspc/default/tbs_data
sudo -u postgres mkdir -p /pgsql/tblspc/default/tbs_idx
sudo -u postgres psql -c "create tablespace default_data owner postgres location '/pgsql/tblspc/default/tbs_data';"
sudo -u postgres psql -c "create tablespace default_idx owner postgres location '/pgsql/tblspc/default/tbs_idx';"

sudo -u postgres psql -c "revoke all on schema public from public;"
sudo -u postgres psql -c "revoke all on database postgres from public;"
sudo -u postgres psql -c "grant all on schema public to postgres;"
```

## 创建数据库

```sql
CREATE DATABASE demo WITH
    OWNER = postgres
    TEMPLATE = template0
    ENCODING = 'UTF8'
    LC_COLLATE = 'zh_CN.UTF-8'
    LC_CTYPE = 'zh_CN.UTF-8'
    TABLESPACE = default_data
;

-- 以postgres用户切换到新创建的数据库: \c demo;
revoke all on database demo from public;
revoke all on schema public from public;

CREATE SCHEMA core;
```

## 用户管理

- 只读用户

```sql
--创建用户: demo_ro
create user demo_ro with nosuperuser nocreatedb nocreaterole noinherit login noreplication nobypassrls password 'demo1234';
GRANT CONNECT ON DATABASE demo TO demo_ro;
GRANT TEMPORARY ON DATABASE demo TO demo_ro;

--public schema赋权
GRANT USAGE ON SCHEMA public TO demo_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO demo_ro;
ALTER DEFAULT PRIVILEGES FOR USER postgres IN SCHEMA public GRANT SELECT ON TABLES TO demo_ro;

--schema core赋权
GRANT USAGE ON SCHEMA core TO demo_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA core TO demo_ro;
ALTER DEFAULT PRIVILEGES FOR USER postgres IN SCHEMA core GRANT SELECT ON TABLES TO demo_ro;
```

- 读写用户

```sql
--创建用户: demo_rw
create user demo_rw with nosuperuser nocreatedb nocreaterole noinherit login noreplication nobypassrls password 'demo1234';
GRANT CONNECT ON DATABASE demo TO demo_rw;
GRANT TEMPORARY ON DATABASE demo TO demo_rw;

--public schema赋权
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO demo_rw;
ALTER DEFAULT PRIVILEGES FOR USER postgres IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO demo_rw;

--schema core赋权
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA core TO demo_rw;
ALTER DEFAULT PRIVILEGES FOR USER postgres IN SCHEMA core GRANT ALL PRIVILEGES ON TABLES TO demo_rw;
```

- 删除用户

```sql
REASSIGN OWNED BY demo_rw TO postgres;
DROP OWNED BY demo_rw;
DROP USER demo_rw;
```

## DDL

```sql
DROP TABLE IF EXISTS public.evt_record;
CREATE TABLE public.evt_record
(
    --ETL字段
    pkid             BIGINT     NOT NULL GENERATED ALWAYS AS IDENTITY,
    modified_time    TIMESTAMP  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    --源业务字段
    origin_id        VARCHAR    NOT NULL,
    name             VARCHAR    NOT NULL,
    id_card          VARCHAR    NOT NULL,
    origin_payload   JSONB      NULL     DEFAULT NULL,
    --源业务附加状态字段
    name_id_card_md5 VARCHAR(64) GENERATED ALWAYS AS (MD5(name::VARCHAR || id_card::VARCHAR)) STORED,
    op_code          VARCHAR(1) NOT NULL,
    op_time          TIMESTAMP  NULL     DEFAULT NULL,
    --约束
    CONSTRAINT pk_id PRIMARY KEY (pkid) USING INDEX TABLESPACE default_idx,
    CONSTRAINT unique_origin_id UNIQUE (origin_id) USING INDEX TABLESPACE default_idx
) TABLESPACE default_data;
COMMENT ON COLUMN public.evt_record.origin_id IS '源业务域聚合根,字段数量、名称、数据类型与源域一一对应';
COMMENT ON COLUMN public.evt_record.op_code IS 'I：新增,U：更新,D：删除';
CREATE INDEX idx_op_time ON public.evt_record USING BTREE (op_time) TABLESPACE default_idx;


```
