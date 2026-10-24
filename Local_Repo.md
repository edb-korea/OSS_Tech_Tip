# local 레포지토리 구성

별도 서버에 local 레포지토리(예, WHPG)구성하여, 폐쇄망 서버에서도 레포지토리를 사용할 수 있도록 설정합니다.

---

### 필수 패키지 설치

```bash
sudo dnf install -y httpd createrepo
```

---

### WHPG 레포지토리 디렉터리 생성

```bash
sudo mkdir -p /data/whpg-repo /data/whpg-repo/ha /data/whpg-repo/tpc-ds
sudo chown -R apache: /data/whpg-repo
```

---

### Apache Alias 설정 및 서비스 활성화

```bash
sudo tee /etc/httpd/conf.d/whpg-repo.conf > /dev/null <<EOF
Alias /whpg-repo "/data/whpg-repo"
<Directory "/data/whpg-repo">
    Options Indexes FollowSymLinks
    Require all granted
</Directory>
EOF
```

```bash
### 서비스 활성화
sudo systemctl enable --now httpd
sudo systemctl status httpd

### 설정 테스트
sudo apachectl configtest

### repomd.xml 접근 확인 (createrepo 실행 후 확인)
# (주의) rpm 파일 변경이나 추가시 수행 필요.
sudo createrepo /data/whpg-repo
sudo curl -s -I http://localhost/whpg-repo/repodata/repomd.xml
```

---

### 회사 웹프록시 제외 처리

```bash
sudo cat << 'EOF' >> /etc/profile.d/repo_no_proxy.sh
export no_proxy="localhost,127.0.0.1,<MY IP>"
EOF
```

---

### 대상 서버 레포지토리 등록

```bash
sudo tee /etc/yum.repos.d/whpg-local.repo > /dev/null <<'EOF'
[whpg-local]
name=WarehousePG Local Repository
baseurl=http://<MY IP>/whpg-repo/
enabled=1
gpgcheck=0
priority=1
EOF
```

```bash
sudo dnf clean all
sudo dnf makecache
sudo dnf repolist
```

### ex) update new rpm at whpg local repo 
```bash
sudo cd /data/whpg-repo
sudo dnf install -y dnf-plugins-core && sudo dnf download --resolve --alldeps apr apr-util bash bzip2 curl iproute krb5-devel \
libcgroup-tools libcurl libevent libuuid libuv libxml2 libyaml libzstd openldap openssh openssh-clients openssh-server openssl \
openssl-libs perl python3 python3-psycopg2 python3-psutil python3-pyyaml python3.11 python3.11-devel \
readline rsync sed tar which zip zlib --setopt=strict=0

sudo dnf download edb-clickhouse-client edb-clickhouse-server edb-clickhouse-common-static edb-jdbc edb-odbc \
edb-whpg7-anonymizer edb-whpg7-user_profile edb-whpg7-pgaudit edb-whpg7-pg_partman edb-whpg7-whpg-cron \
edb-whpg7-pljava edb-whpg7-plr edb-whpg7-data-science-python311 edb-whpg7-data-science-r edb-whpg7-madlib \
edb-whpg7-diskquota whpg-backup whpg-backup-ddboost-plugin whpg7-flow-server edb-whpg-copy edb-whpg-dr \
edb-whpg7-pgaa edb-whpg7-pgfs edb-whpg7-pgvector edb-whpg7-vectorplus \
edb-whpg7-postgis edb-whpg7-hll edb-whpg7-q3c edb-whpg7-pgsphere edb-whpg7-utl-raw edb-whpg7-whpg_fdw \
warehouse-pg-7 warehouse-pg-7-clients edb-whpg7-pxf \
whpg-enterprise-manager edb-acp-host-agent edb-otelcol 
# edb-whpg7-orafce-ext edb-whpg7-cjk 

sudo createrepo /data/whpg-repo
```
