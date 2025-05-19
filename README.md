# Documentação: Configuração do Galera Cluster com MariaDB em 3 Nós

## Visão Geral
Foi configurado um cluster de alta disponibilidade utilizando **MariaDB com Galera Cluster** em **três servidores**:

- **server01** - 192.168.15.172
- **server02** - 192.168.15.76
- **server03** - 192.168.15.127

Cada nó é um servidor ativo com leitura e escrita sincronizadas em tempo real, utilizando o protocolo de replicação síncrona do Galera.

---

## Requisitos

- openSUSE Leap / SLES
- MariaDB com suporte ao Galera (`galera-4`, `mariadb`, `mariadb-errormessages`)
- `rsync` instalado em todos os nós
- Portas abertas no firewall:
  - TCP: 3306, 4444, 4567, 4568
  - UDP: 4567

---

## Configuração do Arquivo `/etc/my.cnf.d/galera.cnf`

```ini
[mysqld]
bind-address = 0.0.0.0
binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2

wsrep_on = ON
wsrep_provider = /usr/lib64/libgalera_smm.so
wsrep_cluster_name = "cluster_xpert"
wsrep_cluster_address = "gcomm://192.168.15.172,192.168.15.76,192.168.15.127"

# Personalizar para cada nó:
wsrep_node_address = "<IP_DESTE_NO>"
wsrep_node_name = "<NOME_DESTE_NO>"

wsrep_sst_method = rsync
```

Exemplo no `server02`:
```ini
wsrep_node_address = "192.168.15.76"
wsrep_node_name = "server02"
```

---

## Inicialização do Cluster

### No primeiro nó (`server01`):
1. Editar `wsrep_cluster_address = "gcomm://"`
2. Iniciar com:
   ```bash
   sudo galera_new_cluster
   # ou
   sudo systemctl start mariadb@bootstrap.service
   ```
3. Verificar:
   ```bash
   mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
   # Esperado: 1
   ```

### Nos demais nós:
1. Configurar `galera.cnf` com todos os IPs no `gcomm://`
2. Iniciar normalmente:
   ```bash
   sudo systemctl start mariadb
   ```
3. Verificar se o nó entrou no cluster:
   ```bash
   mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
   # Esperado: 3
   ```

---

## Verificação e Monitoramento

- Ver membros conectados:
  ```bash
  mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'wsrep_incoming_addresses';"
  ```

- Ver estado local do nó:
  ```bash
  mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
  # Esperado: Synced
  ```

- Verificar UUID compartilhado:
  ```bash
  mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'wsrep_cluster_state_uuid';"
  ```

---

## Complementos Recomendados

### Balanceamento de Carga com HAProxy

Instale o HAProxy em um servidor intermediário:

```bash
sudo zypper install haproxy
```

Configuração simples para encaminhar conexões ao nó preferido:

```haproxy
frontend mysql_front
    bind *:3307
    default_backend mysql_cluster

backend mysql_cluster
    balance roundrobin
    server node1 192.168.15.172:3306 check
    server node2 192.168.15.76:3306 check backup
    server node3 192.168.15.127:3306 check backup
```

Reinicie o HAProxy:
```bash
sudo systemctl restart haproxy
```

### Backup Seguro com MariaDB

Em um dos nós (de preferência um com baixa carga), execute:

```bash
mysqldump -u root -p --all-databases --single-transaction --flush-logs --master-data=2 > backup.sql
```

### Monitoramento com comandos manuais:

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"
```

### Automar verificação de saúde (exemplo com script Bash):

```bash
#!/bin/bash
mysql -u root -pSEUSENHA -e "SHOW STATUS LIKE 'wsrep_cluster_size';" | grep -q "3" || echo "[ALERTA] Cluster fora do ar"
```

---

## Considerações Finais

- Todos os nós estão com `wsrep_cluster_size = 3`
- O cluster está operacional, replicando com sucesso e pronto para produção
- Recomenda-se configurar balanceador de carga (HAProxy, MaxScale) e monitoramento (Zabbix, Prometheus) para ambientes de produção.

---

**Responsável pela configuração:** Emílio Sarges  
**Data:** 19/05/2025
