
Integração do TheHive com o Wazuh para triagem de alertas e criação de casos.

# Guia: Integração TheHive 5 + Wazuh (Servidor Partilhado, 2 cores / 6 GB RAM)

## Nota sobre Recursos

Com 6 GB de RAM a partilhar entre Wazuh Manager, Cassandra, Elasticsearch e TheHive, tenho margem suficiente para um ambiente de lab funcional e estável.
Instalado num servidor Ubuntu 24.04, onde já está instalado e configurado o Wazuh Server.
Actualizado para 2026 e com nova versão do Wazuh. 


## Parte 1 — Instalar TheHive 5 e Dependências

### 1.1 Pré-requisitos do sistema

```bash
# Atualizar o sistema
sudo apt update && sudo apt upgrade -y

# Instalar dependências base
sudo apt install -y wget gnupg apt-transport-https git ca-certificates \
  ca-certificates-java curl software-properties-common python3-pip lsb-release jq
```

### 1.2 Instalar Java 11 (Amazon Corretto)

```bash
# Adicionar repositório Corretto
wget -O - https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | \
  sudo tee /etc/apt/sources.list.d/corretto.list

sudo apt update
sudo apt install -y java-11-amazon-corretto-jdk

# Verificar
java -version
```

No caso de ter múltiplas versões de Java, defini a 11 como default:
```bash
sudo update-alternatives --config java
# Selecionei a opção do Java 11
```

### 1.3 Instalar Apache Cassandra 4.x

```bash
# Adicionar repositório Cassandra
wget -qO - https://downloads.apache.org/cassandra/KEYS | \
  sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg

echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 40x main" | \
  sudo tee /etc/apt/sources.list.d/cassandra.sources.list

sudo apt update
sudo apt install -y cassandra
```

#### Configurar Cassandra

```bash
# Parar o serviço primeiro
sudo systemctl stop cassandra
```

Editei o `/etc/cassandra/cassandra.yaml`:
```bash
sudo nano /etc/cassandra/cassandra.yaml
```

Verifiquei e alterei estes valores:
```yaml
cluster_name: 'thp'
listen_address: 127.0.0.1
rpc_address: 127.0.0.1
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "127.0.0.1:7000"
```

**Otimizar memória do Cassandra** — editei o `/etc/cassandra/jvm-server.options` (ou `jvm.options` dependendo da versão):
```bash
sudo nano /etc/cassandra/jvm-server.options
```
Procurei e alterei o heap:
```
-Xms1G
-Xmx1G
```

> **Nota:** Com 6 GB de RAM total, 1 GB para o Cassandra oferece boa estabilidade para um ambiente de lab.

```bash
# Limpar dados antigos (importante na primeira instalação)
sudo rm -rf /var/lib/cassandra/data/system/*

# Iniciar e ativar
sudo systemctl start cassandra
sudo systemctl enable cassandra

# Verificar que está a correr (pode demorar 1-2 minutos)
sudo systemctl status cassandra
nodetool status
```

#### Criar keyspace para o TheHive

O TheHive precisa de um "keyspace" no Cassandra — é o equivalente a criar uma base de dados onde vai guardar todos os dados (alertas, casos, utilizadores, etc.).

O `cqlsh` que vem com o Cassandra 4.x **não funciona com Python 3.12+** (Ubuntu 24.04). Instalei a versão standalone:
```bash
pip3 install cqlsh --break-system-packages
```

Depois liguei-me ao Cassandra e criei o keyspace:
```bash
~/.local/bin/cqlsh localhost 9042
```

Dentro da consola `cqlsh>`, executei:
```sql
CREATE KEYSPACE thehive WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'} AND durable_writes = 'true';
EXIT;
```

> **Nota:** Num ambiente de lab com tudo no mesmo servidor, não é necessário configurar autenticação no Cassandra. O TheHive liga-se diretamente sem username/password.

### 1.4 Instalar Elasticsearch 7.x

```bash
# Adicionar repositório Elasticsearch
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install -y elasticsearch
```

#### Configurar Elasticsearch

> **Porquê a porta 9201?** O Wazuh já traz o seu próprio motor de indexação (Wazuh Indexer, baseado em OpenSearch) que ocupa a porta 9200. Como estou a instalar o TheHive no mesmo servidor, o Elasticsearch do TheHive precisa de usar portas diferentes para evitar o conflito.

Editei o `/etc/elasticsearch/elasticsearch.yml`:
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yaml
cluster.name: thehive
node.name: node-1
network.host: 127.0.0.1
http.port: 9201
transport.port: 9301
cluster.initial_master_nodes: ["node-1"]
xpack.security.enabled: false
```

**Otimizar memória** — criei o `/etc/elasticsearch/jvm.options.d/thehive.options`:
```bash
sudo tee /etc/elasticsearch/jvm.options.d/thehive.options << 'EOF'
-Xms1g
-Xmx1g
-Dlog4j2.formatMsgNoLookups=true
EOF
```

```bash
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

# Verificar (pode demorar uns segundos)
curl -s http://127.0.0.1:9201 | jq .
```

### 1.5 Instalar TheHive 5

> **Nota:** Os repositórios APT antigos da StrangeBee (`archives.strangebee.com` e `deb.strangebee.com`) já não resolvem. O método atual é descarregar o pacote `.deb` diretamente do site de downloads.

```bash
# Descarregar o pacote .deb do TheHive 5.5
wget -O /tmp/thehive_5.5.14-2_all.deb https://thehive.download.strangebee.com/5.5/deb/thehive_5.5.14-2_all.deb

# Instalar
sudo apt-get install -y /tmp/thehive_5.5.14-2_all.deb
```

#### Configurar permissões do diretório de ficheiros

> **Porquê este passo?** O TheHive precisa de um local no disco para guardar ficheiros anexados a casos e alertas (amostras de malware, capturas de ecrã, logs, relatórios, etc.). O `mkdir -p` cria essa pasta e todos os diretórios intermédios. O `chown -R` muda o dono para o utilizador `thehive` (criado automaticamente na instalação do pacote), porque o serviço corre com esse utilizador por segurança — sem isto, o TheHive não conseguiria escrever ficheiros e daria erro de permissão.

```bash
sudo mkdir -p /opt/thp/thehive/files
sudo chown -R thehive:thehive /opt/thp
```

#### Configurar TheHive

Editei o `/etc/thehive/application.conf`:
```bash
sudo nano /etc/thehive/application.conf
```

```hocon
# Base de dados — Cassandra (sem autenticação em ambiente lab)
db.janusgraph {
  storage {
    backend = cql
    hostname = ["127.0.0.1"]
    cql {
      cluster-name = "thp"
      keyspace = thehive
    }
  }
  index.search {
    backend = elasticsearch
    hostname = ["127.0.0.1:9201"]
    index-name = thehive
  }
}

# Armazenamento de ficheiros local
storage {
  provider = localfs
  localfs.location = /opt/thp/thehive/files
}

# Configuração do serviço
application.baseUrl = "http://172.30.80.99:9000"
play.http.context = "/"
```

```bash
# Iniciar TheHive
sudo systemctl start thehive
sudo systemctl enable thehive

# Verificar
sudo systemctl status thehive
```

#### Primeiro acesso

Abri o browser em `http://172.30.80.99:9000`

Credenciais default:
- **Email:** admin@thehive.local
- **Password:** secret

> **IMPORTANTE:** Alterei a password do admin imediatamente após o primeiro login!
<br/>
<br/>  
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-13-44-14" src="https://github.com/user-attachments/assets/cfa096bf-3b1a-4aeb-93d9-0281289a9837" />
<br/>
<br/>  

---

## Parte 2 — Configurar TheHive para Receber Alertas

### 2.1 Criar organização e utilizadores

No interface web do TheHive (logado como admin@thehive.local / secret):

1. **Criar uma Organização** — fui a "Organizations" (ícone do edifício na barra lateral) > botão "+" > preenchi:
   - Name: `SOC-Lab`
   - Description: `Laboratório Projeto Final`
   - Cliquei em **Confirm**
<br/>
<br/>  
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-13-44-06" src="https://github.com/user-attachments/assets/d4c8d507-08e4-4a41-9e6e-3241e79c97ae" />
<br/>
  <br/>

2. **Criar um utilizador org-admin** — dentro da organização SOC-Lab, cliquei em "Add user":
   - Type: **Normal**
   - Login: `labmanager@thehive.local` (tem de ser formato email)
   - Name: O meu nome
   - Profile: **org-admin**
   - Cliquei em **Confirm**
   - **Definir password:** voltei à lista de utilizadores, cliquei no nome `labmanager@thehive.local` para abrir os detalhes, e cliquei em **Set password** para definir uma password. Sem este passo, não é possível fazer login com este utilizador.
<br/>
  <br/>
  
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-13-50-39" src="https://github.com/user-attachments/assets/f7c4a571-b00b-49f9-a07b-8792292f97c2" />
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-13-52-34" src="https://github.com/user-attachments/assets/d33e6279-c3d2-4588-9d37-6d21ef83420c" />
<br/>
<br/>  

3. **Criar um utilizador para API (Wazuh)** — ainda dentro da organização, cliquei novamente em "Add user":
   - Type: **Service** (utilizadores de serviço são para bots/integrações via API)
   - Login: `wazuh-api@thehive.local`
   - Name: `Wazuh API`
   - Profile: **analyst**
   - Cliquei em **Confirm**

4. **Gerar API Key** — na lista de utilizadores, cliquei no nome "Wazuh API" para abrir os detalhes do utilizador. Na secção "API Key", cliquei em **Create API Key**, depois em **Reveal** para ver e copiar a chave
<br/>
<br/>  
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-13-55-29" src="https://github.com/user-attachments/assets/47fb3e6a-6ca4-4faa-b47e-e7c580c5abf2" />
<br/>
<br/>  

> Guardei esta API Key — é necessária no próximo passo.

---

## Parte 3 — Integrar o Wazuh Manager com TheHive

### 3.1 Instalar a biblioteca thehive4py

```bash
# Instalar usando o Python interno do Wazuh
sudo /var/ossec/framework/python/bin/pip3 install thehive4py==1.8.1
```

### 3.2 Criar o script Python de integração

```bash
sudo nano /var/ossec/integrations/custom-w2thive.py
```

Colei o seguinte conteúdo:

```python
#!/var/ossec/framework/python/bin/python3
import json
import sys
import os
import re
import logging
import uuid
from thehive4py.api import TheHiveApi
from thehive4py.models import Alert, AlertArtifact

# ======== CONFIGURAÇÃO DO UTILIZADOR ========
# Nível mínimo de alerta Wazuh para enviar ao TheHive (0-15)
# Recomendação: 5 ou superior para evitar ruído excessivo
lvl_threshold = 5

# Nível mínimo para alertas Suricata
suricata_lvl_threshold = 3

# Ativar debug (True) ou apenas info (False)
debug_enabled = False
info_enabled = True
# =============================================

# Caminhos e logging
pwd = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))
log_file = '{0}/logs/integrations.log'.format(pwd)
logger = logging.getLogger(__name__)
logger.setLevel(logging.WARNING)
if info_enabled:
    logger.setLevel(logging.INFO)
if debug_enabled:
    logger.setLevel(logging.DEBUG)

fh = logging.FileHandler(log_file)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
fh.setFormatter(formatter)
logger.addHandler(fh)


def main(args):
    logger.debug('#start main')
    alert_file_location = args[1]
    thive_api_key = args[2]
    thive = args[3]
    thive_api = TheHiveApi(thive, thive_api_key)

    w_alert = json.load(open(alert_file_location))
    logger.debug('#alert data: %s', str(w_alert))

    # Ignorar alertas de serviço/agente (ruído)
    rule_id = w_alert.get('rule', {}).get('id', '')
    if rule_id in ['502', '503', '504', '505']:
        logger.info('Skip Wazuh service state/agent alert (rule %s)', rule_id)
        return

    alt = pr(w_alert, '', [])
    format_alt = md_format(alt)
    artifacts_dict = artifact_detect(format_alt)
    alert = generate_alert(format_alt, artifacts_dict, w_alert)

    # Filtro por threshold
    if w_alert['rule'].get('groups') == ['ids', 'suricata']:
        if 'data' in w_alert and 'alert' in w_alert.get('data', {}):
            if int(w_alert['data']['alert']['severity']) <= suricata_lvl_threshold:
                send_alert(alert, thive_api)
    elif int(w_alert['rule']['level']) >= lvl_threshold:
        send_alert(alert, thive_api)


def pr(data, prefix, alt):
    for key, value in data.items():
        if hasattr(value, 'keys'):
            pr(value, prefix + '.' + str(key), alt=alt)
        else:
            alt.append((prefix + '.' + str(key) + '|||' + str(value)))
    return alt


def md_format(alt, format_alt=''):
    md_title_dict = {}
    for now in alt:
        now = now[1:]
        dot = now.split('|||')[0].find('.')
        if dot == -1:
            md_title_dict[now.split('|||')[0]] = [now]
        else:
            if now[0:dot] in md_title_dict:
                md_title_dict[now[0:dot]].append(now)
            else:
                md_title_dict[now[0:dot]] = [now]
    for now in md_title_dict:
        format_alt += '### ' + now.capitalize() + '\n'
        format_alt += '| key | val |\n| ------ | ------ |\n'
        for let in md_title_dict[now]:
            key, val = let.split('|||')[0], let.split('|||')[1]
            format_alt += '| **' + key + '** | ' + val + ' |\n'
    return format_alt


def artifact_detect(format_alt):
    artifacts_dict = {}
    artifacts_dict['ip'] = re.findall(r'\d+\.\d+\.\d+\.\d+', format_alt)
    artifacts_dict['url'] = re.findall(
        r'http[s]?://(?:[a-zA-Z]|[0-9]|[$-_@.&+]|[!*\(\),]|(?:%[0-9a-fA-F][0-9a-fA-F]))+',
        format_alt
    )
    artifacts_dict['domain'] = []
    for now in artifacts_dict['url']:
        artifacts_dict['domain'].append(now.split('//')[1].split('/')[0])
    return artifacts_dict


def generate_alert(format_alt, artifacts_dict, w_alert):
    sourceRef = str(uuid.uuid4())[0:6]
    artifacts = []

    if 'agent' in w_alert:
        if 'ip' not in w_alert['agent']:
            w_alert['agent']['ip'] = 'no agent ip'
    else:
        w_alert['agent'] = {'id': 'no agent id', 'name': 'no agent name', 'ip': 'no agent ip'}

    for key, value in artifacts_dict.items():
        for val in value:
            artifacts.append(AlertArtifact(dataType=key, data=val))

    alert = Alert(
        title=w_alert['rule']['description'],
        tlp=2,
        tags=[
            'wazuh',
            'rule=' + w_alert['rule']['id'],
            'agent_name=' + w_alert['agent']['name'],
            'agent_id=' + w_alert['agent']['id'],
            'agent_ip=' + w_alert['agent']['ip'],
        ],
        description=format_alt,
        type='wazuh_alert',
        source='wazuh',
        sourceRef=sourceRef,
        artifacts=artifacts,
        severity=wazuh_to_thehive_severity(int(w_alert['rule']['level']))
    )
    return alert


def wazuh_to_thehive_severity(level):
    """Mapear severidade Wazuh (0-15) para TheHive (1-4)"""
    if level <= 4:
        return 1  # Low
    elif level <= 7:
        return 2  # Medium
    elif level <= 11:
        return 3  # High
    else:
        return 4  # Critical


def send_alert(alert, thive_api):
    response = thive_api.create_alert(alert)
    if response.status_code == 201:
        logger.info('TheHive alert created: %s', str(response.json()['id']))
    else:
        logger.error('Error creating TheHive alert: %s/%s', response.status_code, response.text)


if __name__ == "__main__":
    try:
        logger.debug('debug mode')
        main(sys.argv)
    except Exception:
        logger.exception('Error in custom-w2thive')
```

### 3.3 Criar o script Bash wrapper

```bash
sudo nano /var/ossec/integrations/custom-w2thive
```

```bash
#!/bin/sh
WPYTHON_BIN="framework/python/bin/python3"
SCRIPT_PATH_NAME="$0"
DIR_NAME="$(cd $(dirname ${SCRIPT_PATH_NAME}); pwd -P)"
SCRIPT_NAME="$(basename ${SCRIPT_PATH_NAME})"

case ${DIR_NAME} in
    */active-response/bin | */wodles*)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/../..; pwd)"
        fi
        PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
        ;;
    */bin)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd)"
        fi
        PYTHON_SCRIPT="${WAZUH_PATH}/framework/scripts/${SCRIPT_NAME}.py"
        ;;
    */integrations)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd)"
        fi
        PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
        ;;
esac

${WAZUH_PATH}/${WPYTHON_BIN} ${PYTHON_SCRIPT} "$@"
```

### 3.4 Definir permissões

```bash
sudo chmod 755 /var/ossec/integrations/custom-w2thive.py
sudo chmod 755 /var/ossec/integrations/custom-w2thive
sudo chown root:wazuh /var/ossec/integrations/custom-w2thive.py
sudo chown root:wazuh /var/ossec/integrations/custom-w2thive
```

> **Nota:** Em Wazuh 4.7+, o grupo é `wazuh`. Em versões mais antigas (< 4.3), o grupo era `ossec`.

### 3.5 Configurar a integração no ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Adicionei o seguinte bloco **dentro** de `<ossec_config>`, logo após o bloco `</global>`:

```xml
<integration>
  <name>custom-w2thive</name>
  <hook_url>http://127.0.0.1:9000</hook_url>
  <api_key>COLAR_AQUI_A_API_KEY</api_key>
  <alert_format>json</alert_format>
</integration>
```

> **Importante:** Como o TheHive está no mesmo servidor, usei `127.0.0.1`. Substituí `COLA_AQUI_A_API_KEY` pela API key gerada no passo 2.1.

### 3.6 Reiniciar o Wazuh Manager

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

---

## Parte 4 — Verificação e Troubleshooting

### 4.1 Verificar que os serviços estão todos ativos

```bash
sudo systemctl status cassandra
sudo systemctl status elasticsearch
sudo systemctl status thehive
sudo systemctl status wazuh-manager
```

### 4.2 Verificar os logs de integração

```bash
# Log principal de integração
sudo tail -f /var/ossec/logs/integrations.log
```

Confirmei que apareciam linhas como:
```
TheHive alert created: ~123456
```

### 4.3 Testar manualmente a API do TheHive

```bash
curl -s -H "Authorization: Bearer A_MINHA_API_KEY" \
  http://127.0.0.1:9000/api/v1/alert?range=0-5 | jq .
```

Dentro do TheHive já começam a chegar alertas configurados anteriormente, neste caso com integração com VirusTotal para remover ameaças automaticamente usando as capacidades active response do Wazuh configuradas por mim noutro laboratório. Abrindo o alerta, é possível obter mais informações sobre o mesmo, a sua origem, elementos importantes para investigação e triagem e, por fim, importante, criar cases.
<br/>
<br/>  
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-14-19-22" src="https://github.com/user-attachments/assets/2b888d2a-01ef-447a-937f-b96ee1ec882c" />
<br/>
<br/>  
<img width="1704" height="878" alt="Windows 10 Marte-2026-03-28-14-19-41" src="https://github.com/user-attachments/assets/aea69d0d-398f-4deb-82ff-8be1a7975e8a" />
<br/>
<br/>  

### 4.4 Problemas comuns e soluções

| Problema | Causa provável | Solução |
|----------|----------------|---------|
| `ModuleNotFoundError: No module named 'thehive4py.api'` | Instalei thehive4py v2, mas o script usa v1 | `pip3 install thehive4py==1.8.1` |
| `Invalid element 'integration'` no log do Wazuh | Bloco `<integration>` fora de `<ossec_config>` | Mover o bloco para dentro de `<ossec_config>` |
| TheHive não arranca | Cassandra/Elasticsearch não estão prontos | Verificar com `nodetool status` e `curl localhost:9201` |
| Demasiados alertas no TheHive | `lvl_threshold` está a 0 | Aumentar para 5+ no script Python |
| `java.lang.OutOfMemoryError` | Heap demasiado pequeno | Verificar que Cassandra tem 1G e ES tem 1g nos ficheiros de config |
| TheHive diz "database not ready" | Cassandra keyspace não existe | Executar o `CREATE KEYSPACE` no cqlsh |



## Parte 5 — Fluxo de Trabalho no TheHive

Assim que os alertas começaram a chegar:

1. **Triagem** — abri o separador "Alerts" no TheHive. Cada alerta mostra a descrição da regra Wazuh, severidade, agente, e IPs envolvidos
2. **Promover a Caso** — se o alerta merece investigação, clico em "Create Case" para o transformar num caso formal
3. **Atribuir Tarefas** — dentro do caso, crio tarefas e atribuo-as a membros da equipa
4. **Observables** — o script extrai automaticamente IPs, URLs e domínios como observables para investigação
5. **Encerrar** — documento as conclusões e fecho o caso
