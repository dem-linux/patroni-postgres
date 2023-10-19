In the video I used 2 servers for patroni, 1 for etcd and 1 for proxy. 

This Repo is for video setting up Patroni cluster.

On the Node 1 and Node 2 install postgres:

` apt install postgresql postgresql-contrib -y `

Stop Postgres when it's done:
` systemctl stop postgresql `

Create Symlink on both nodes: (change 14 to your version)
` ln -s /usr/lib/postgresql/14/bin/* /usr/sbin/ `

Install python and patroni on Node 1 and 2.

```
apt install python3-pip python3-dev libpq-dev -y 
pip3 install --upgrade pip 
pip install patroni 
pip install python-etcd 
pip install psycopg2
```

Create Configuration File for Patroni:
```nano /etc/patroni.yml```
Change:
IP
name
data_dir
```
scope: postgres
namespace: /db/
name: node1

restapi:
    listen: 10.10.0.181:8008 
    connect_address: 10.10.0.181:8008

etcd:
    host: 10.10.0.184:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true

    initdb:
    - encoding: UTF8
    - data-checksums

    pg_hba:
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 10.10.0.181/0 md5
    - host replication replicator 10.10.0.182/0 md5
    - host all all 0.0.0.0/0 md5

    users:
        admin:
            password: admin
            options:
                - createrole
                - createdb

postgresql:
    listen: 10.10.0.181:5432
    connect_address: 10.10.0.181:5432
    data_dir: /mnt/data/patroni/
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: password
        superuser:
            username: postgres
            password: password
    parameters:
        unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false

```

Create DataDir on both nodes: ( I used /mnt/ as example but pick other location)
``` 
mkdir -p /mnt/data/patroni/ 
chown postgres:postgres /mnt/data/patroni/
chmod 700 /mnt/data/patroni/
```
Create the service on both nodes:

``` 
nano /etc/systemd/system/patroni.service
```
``` 
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple

User=postgres
Group=postgres

ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.targ
```

Setup ETCD:

```
apt install etcd -y
nano /etc/default/etcd
```
```
ETCD_LISTEN_PEER_URLS="http://10.10.0.184:2380,http://127.0.0.1:7001"
ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379, http://10.10.0.184:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.0.181:2380"
ETCD_INITIAL_CLUSTER="etcd0=http://10.10.0.184:2380,"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.0.184:2379"
ETCD_INITIAL_CLUSTER_TOKEN="node1"
ETCD_INITIAL_CLUSTER_STATE="new
```

Setup HA-PROXY:
```
apt install haproxy -y
nano /etc/haproxy/haproxy.cfg
```
```
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.10.0.181:5432 maxconn 100 check port 8008
    server node2 10.10.0.182:5432 maxconn 100 check port 8008
```

To setup HA on HAProxy you need to have 2 HA proxy servers that points to same backend
 
Use same configuration file on both servers:
```
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.10.0.181:5432 maxconn 100 check port 8008
    server node2 10.10.0.182:5432 maxconn 100 check port 8008
```
Now install Keepalived on both servers (proxys)
```
apt install keepalived
nano /etc/keepalived/keepalived.conf
```
Add the following to the keepalived.conf on the ha proxy server 1 and on the other server change the STATE to SLAVE
Make sure you pick a free IP for the virtual IP!
```
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    interface eth0
    virtual_router_id 51


    state MASTER
    priority 101

    virtual_ipaddress {
      10.10.0.218
  }
  track_script {
    chk_haproxy
  }
}
```
After adding the configuration to both severs start the keepalived service and check the status.
```
systemctl start keepalived
systemctl status keepalived
```
The "MASTER" server HA proxy 1 should get 1 extra IP, the his the virtual IP that keepalived will use, to check that run "ip a" on the master ha proxy sevrer.
The result should be somthing like this:
```

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 0e:fd:d7:5e:26:da brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    altname ens18
    inet 10.10.0.217/24 brd 10.10.0.255 scope global dynamic eth0
       valid_lft 84885sec preferred_lft 84885sec
    inet 10.10.0.218/32 scope global eth0   < THIS IS MY VIRTUAL IP I ADDED IN THE CONFIG!
       valid_lft forever preferred_lft forever
    inet6 fe80::cfd:d7ff:fe5e:26da/64 scope link
       valid_lft forever preferred_lft forever
```

Now when you try to access the HA proxy you will use the Virtual IP that IP will always point to the master HA proxy and manage the failover in case master dies the slave will take over as a master role.
In my case I would use 10.10.0.218 to access the Patroni cluster!
