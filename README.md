### :open_file_folder: Estrutura: 
```bash
.
├── data
│   ├── db
│   ├── env_vars
│   │   ├── db.env
│   │   ├── grafana.env
│   │   ├── srv.env
│   │   └── web.env
│   ├── grafana
│   └── zabbix
│       ├── alertscripts
│       ├── mibs
│       └── snmptraps
└── docker-compose.yml
```
### Observação:
Aplicar o comando...
```bash
chown -R 472:472 data/grafana
```
Onde ```grafana``` é a pasta base para os dados gerados pelo Grafana.
### Arquivos de variáveis e senhas:
```db.env```

```grafana.env```

```srv.env```

```web.env```
### Conteúdo do arquivo ```docker-compose.yml```:
```yml
version: "3.2"

networks:
  zbx_net:
    driver: bridge
    driver_opts:
      com.docker.network.enable_ipv6: "false"
    ipam:
      driver: default
      config:
      - subnet: 172.16.238.0/24

services:
  mariadb:
    container_name: mariadb-zabbix
    image: mariadb:10.8
    networks:
      - zbx_net
    restart: unless-stopped
    ports:
      - "3406:3306"
    volumes:
      - ./data/db:/var/lib/mysql
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --default-authentication-plugin=mysql_native_password
    env_file:
      - ./data/env_vars/db.env

  zabbix-server:
    container_name: zabbix-server
    image: zabbix/zabbix-server-mysql:ubuntu-6.2-latest
    networks:
      - zbx_net
    links:
      - mariadb
    restart: unless-stopped
    ports:
      - "10051:10051"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./data/zabbix/alertscripts:/usr/lib/zabbix/alertscripts:ro
      - ./data/zabbix/mibs:/var/lib/zabbix/mibs:ro
      - ./data/zabbix/snmptraps:/var/lib/zabbix/snmptraps:rw
    env_file:
      - ./data/env_vars/db.env
      - ./data/env_vars/srv.env
    depends_on:
      - mariadb
    sysctls:
      - net.ipv4.ip_local_port_range=1024 65000
      - net.ipv4.conf.all.accept_redirects=0
      - net.ipv4.conf.all.secure_redirects=0
      - net.ipv4.conf.all.send_redirects=0

  zabbix-frontend:
    container_name: zabbix-frontend
    image: zabbix/zabbix-web-apache-mysql:ubuntu-6.2-latest
    networks:
      - zbx_net
    links:
      - mariadb
    restart: unless-stopped
    ports:
      - "8082:8080"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - ./data/env_vars/db.env
      - ./data/env_vars/web.env
    depends_on:
      - mariadb
      - zabbix-server
    sysctls:
      - net.core.somaxconn=65535

  grafana:
    container_name: grafana
    image: grafana/grafana:9.0.7-ubuntu
    networks:
      - zbx_net
    links:
      - mariadb
      - zabbix-server
    restart: unless-stopped
    user: "472"
    ports:
      - "3000:3000"
    volumes:
      - ./data/grafana:/var/lib/grafana
    env_file:
      - ./data/env_vars/grafana.env
    depends_on:
      - mariadb
      - zabbix-server

  zabbix-agent:
    container_name: zabbix-agent
    image: zabbix/zabbix-agent2:alpine-6.2-latest
    networks:
      - zbx_net
    links:
      - zabbix-server
    restart: unless-stopped
    privileged: true
    pid: "host"
    user: 0:0
    ports:
      - "10050:10050"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ZBX_HOSTNAME=zabbix-server
    depends_on:
      - zabbix-server

  zabbix-snmptraps:
    container_name: zabbix-snmptraps
    image: zabbix/zabbix-snmptraps:ubuntu-6.2-latest
    networks:
      - zbx_net
    restart: unless-stopped
    ports:
      - "162:1162/udp"
    volumes:
      - ./data/zabbix/snmptraps:/var/lib/zabbix/snmptraps:rw
    stop_grace_period: 5s
    labels:
      com.zabbix.description: "Zabbix snmptraps"
      com.zabbix.component: "snmptraps"
```
