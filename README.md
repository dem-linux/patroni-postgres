This Repo is for video setting up Patroni cluster.

On the Node 1 and Node 2 install following:

``` apt install postgresql postgresql-contrib -y ``´

Stop Postgres when it's done:
``` systemctl stop postgresql ```

Create Symlink on both nodes:
``` ln -s /usr/lib/postgresql/14/bin/* /usr/sbin/ ```

Install Patroni on Node 1 and 2.

```apt install python3-pip python3-dev libpq-dev -y
pip3 install --upgrade pip
pip install patroni
pip install python-etcd
pip install psycopg2``

Create Configuration File for Patroni:
```nano /etc/patroni.yml```
Paste in from /patroni.yml change IP.

Create DataDir on both nodes:
``` 
mkdir -p /mnt/data/patroni/
chown postgres:postgres /mnt/data/patroni/
chmod 700 /mnt/data/patroni/

```
Create service on both nodes:

``` 
nano /etc/systemd/system/patroni.service

Copy from /patroni.service

```

Setup ETCD:

``` apt install etcd -y

nano /etc/default/etcd
```
Paste in from /etcd. !! Change IP !!

Setup HA-PROXY:
```
apt install haproxy -y
nano /etc/haproxy/haproxy.cfg
Paste in from haproxy.cfg

```
