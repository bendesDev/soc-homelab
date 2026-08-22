# Documentação técnica

Referência detalhada de cada role: comandos exatos, decisões de
arquitetura, troubleshooting e pegadinhas já resolvidas. Para uma visão
geral do projeto, veja o [README](../README.md).

Playbooks Ansible que documentam e automatizam a configuração desta máquina
(`localhost`), a partir de 2026-08-20. A ideia é simples: toda vez que algo
for instalado ou configurado manualmente aqui, isso vira uma task neste
repositório — o playbook funciona como documentação viva e como forma de
reproduzir o setup em outra máquina.

## Estrutura

- `site.yml` — playbook principal, roda todas as roles.
- `inventory/hosts.ini` — inventário (só `localhost`).
- `roles/packages` — pacotes instalados via `pacman` e AUR (`paru`).
  - Lista em `roles/packages/defaults/main.yml`.
- `roles/dotfiles` — dotfiles versionados em `roles/dotfiles/files/` e
  symlinkados para o `$HOME`.
  - Mapeamento em `roles/dotfiles/defaults/main.yml`.
- `roles/storage` — dedica o HD adicional (`/dev/sda`) como ext4 montado em
  `/mnt/storage`, para uso de alta capacidade (VMs, dados de containers,
  etc.).
- `roles/docker` — Docker com `data-root` em `/mnt/storage/docker`.
- `roles/dns` — `dnsmasq` servindo o domínio `.lab` pra rede local, mais
  roteamento de `*.lab` no `systemd-resolved` desta máquina. Registros em
  `roles/dns/defaults/main.yml` (`dns_records`).
- `roles/soc_lab` — laboratório de estudo SOC (ver Roadmap abaixo).
- `roles/wazuh_agent` — agente Wazuh nesta própria máquina (instalado via
  AUR, pacote `wazuh-agent`), apontado para `wazuh.lab`.
- `roles/wazuh_syslog` — listener de syslog (UDP 514) no manager, pra
  ingerir logs de dispositivos externos (ver FortiWiFi abaixo). Config
  fica num volume Docker nomeado, não em arquivo do host — a role edita
  via `docker exec`.
- `roles/grafana` — dashboards conectados direto no indexer (OpenSearch)
  do Wazuh, container na mesma rede Docker do stack (`single-node_default`).
- `roles/pentest_network` — rede Docker isolada (`pentest_lab`,
  `10.10.20.0/24`) pro lab de pentest (ver seção abaixo).
- `roles/kali` — container `kalilinux/kali-rolling`, CLI-only, sem GUI.
- `roles/windows_vm` — VMs Windows 11 via QEMU/KVM (`dockurr/windows`),
  parametrizada por `windows_instances` (2 instâncias por padrão).
- `roles/linux_target` — VM Debian leve (`qemux/qemu`) como alvo Linux.
- `CHANGELOG.md` — registro humano do que foi feito e por quê, em ordem
  cronológica.

## Segredos (senhas) por role

Toda role que precisa de uma senha (ex.: `grafana`, que precisa da senha
do indexer do Wazuh pra configurar o datasource) segue este padrão:

- `roles/<nome>/vars/main.yml.example` — **versionado**, mostra a
  estrutura esperada com placeholder.
- `roles/<nome>/vars/main.yml` — **gitignored** (`roles/*/vars/main.yml`
  no `.gitignore`), com as senhas reais. `vars/main.yml` carrega
  automático no Ansible (prioridade maior que `defaults/`), sem precisar
  de `vars_files` explícito.

Pra reproduzir isso numa outra máquina: copiar o `.example`, tirar o
sufixo, preencher com as senhas reais.

Novas roles podem ser adicionadas conforme surgirem outras categorias de
mudança (serviços systemd, configs do sistema, etc.) — seguir o mesmo padrão
de `roles/<nome>/{tasks,defaults}/main.yml`.

## Uso

Rodar tudo:

```bash
ansible-playbook site.yml
```

Rodar só uma role:

```bash
ansible-playbook site.yml --tags packages
```

(As tags por role só funcionam se forem adicionadas nas roles — por padrão o
playbook roda tudo.)

## Fluxo de trabalho

1. Instalou um pacote, mudou uma config, criou um dotfile? Adicione a
   entrada correspondente na role certa (ou crie uma nova role).
2. Rode `ansible-playbook site.yml` para confirmar que o playbook aplica a
   mudança de forma idempotente (rodar de novo não deve reportar `changed`).
3. Registre uma linha no `CHANGELOG.md` com a data e o motivo da mudança.
4. Commit.

## Roadmap: laboratório SOC (`roles/soc_lab`)

Gabriel começa numa vaga de analista SOC em setembro/2026 e está montando um
lab de estudo em casa, inspirado em setups completos de SOC self-hosted.
Fase atual e próximas fases (cada uma vira uma role nova quando for
implementada):

1. ✅ **Core SIEM** — Wazuh (manager + indexer + dashboard) via Docker,
   single-node.
2. ✅ **Agente Wazuh nesta própria máquina** — via AUR (`roles/wazuh_agent`),
   gerando telemetria real.
3. 🟡 **Dashboards de métricas/KPI** (`roles/grafana`) — feito. Ingestão
   adicional via Graylog fica pra quando houver fonte de log que precise
   de parsing extra (FortiWiFi já entra direto no Wazuh, ver acima).
4. SOAR — orquestração de playbooks (Shuffle) e gestão de casos
   (DFIR-IRIS), com roteamento por severidade (ex.: webhook no Discord para
   alertas de baixa severidade).
5. Threat intel — enriquecimento de IOCs (GeoIP/ASN, VirusTotal) e MISP.
6. 🟡 **Simulação de adversário e teste de detecção** — infraestrutura
   no ar (`roles/pentest_network`, `kali`, `windows_vm`, `linux_target`):
   Kali (atacante) + Windows 11 × 2 + VM Linux (alvos). Falta instalar
   Sysmon/auditd + agente Wazuh dentro de cada VM (passo manual, ver
   seção "Lab de pentest" acima) e rodar o Atomic Red Team em si.
7. DFIR — Velociraptor.

Cada fase é opcional e incremental; a ideia é ir crescendo o lab conforme
o estudo avançar, não montar tudo de uma vez.

## DNS local (`roles/dns`)

Os serviços do lab ficam acessíveis por nome (`wazuh.lab`, e o que for
adicionado depois) em vez de IP/porta, via `dnsmasq` rodando nesta
máquina.

- **Nesta máquina**: já funciona automaticamente (roteamento configurado
  no `systemd-resolved`).
- **Outros dispositivos da rede** (celular, notebook, etc.): configure o
  DNS manualmente pra `192.168.0.132` (IP desta máquina na rede local) —
  não dá pra automatizar isso a partir daqui, cada dispositivo tem sua
  própria tela de configuração de rede.
- **Atenção**: esse IP vem de DHCP e pode mudar se o Wi-Fi reconectar ou o
  roteador reiniciar. Se `wazuh.lab` parar de resolver, confira o IP atual
  (`ip -4 addr show wlan0`), atualize `dns_lan_ip` em
  `roles/dns/defaults/main.yml` e rode `ansible-playbook site.yml -K` de
  novo. O jeito definitivo de evitar isso é reservar esse IP pro
  endereço MAC desta máquina no painel do roteador (fora do alcance do
  Ansible).

## Trocar senhas do Wazuh (`~/soc-lab/wazuh-docker`)

Não é automatizado (é uma ação deliberada, não algo que o playbook deva
reaplicar sozinho). Depois do clone inicial, esses arquivos ficam fora do
controle do `git` da role `soc_lab` (`update: false`) — editar à vontade.

1. `docker-compose.yml`: trocar `INDEXER_PASSWORD` (usuário `admin`,
   indexer) e/ou `API_PASSWORD` (usuário `wazuh-wui`, API do manager) nos
   dois serviços onde aparecem.
2. Senha do indexer (`admin`): gerar hash com
   `docker run --rm wazuh/wazuh-indexer:<tag> bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh -p '<senha>'`,
   colar em `config/wazuh_indexer/internal_users.yml` (campo `hash` do
   usuário).
3. `docker compose down && docker compose up -d`.
4. Aplicar a config de segurança no indexer: dentro do container
   (`docker exec -it <indexer> bash`), rodar o `securityadmin.sh` (ver
   `CHANGELOG.md` de 2026-08-21 pro comando completo).
5. **Não esquecer**: `config/wazuh_dashboard/wazuh.yml` guarda a senha da
   API que o dashboard usa pra falar com o manager — é um arquivo
   estático, não se atualiza sozinho com a env var. Editar o campo
   `password` manualmente e `docker compose restart wazuh.dashboard`. Foi
   exatamente esse passo que faltou da primeira vez e quebrou o acesso
   (erro 401 em `/api/check-api`).

## Ingerir logs do FortiWiFi 40C (`roles/wazuh_syslog`)

O manager já tem decoders/regras nativas pra FortiGate/FortiOS (79 regras
em `0391-fortigate_rules.xml`, cobrindo tráfego, IPS/UTM, VPN, login
admin) e um listener de syslog UDP 514 já testado ponta a ponta
(pacote simulado → decoder `fortigate-firewall-v6` → regra disparada →
alerta no `alerts.json`).

Falta só apontar o FortiWiFi pra cá — isso é feito no próprio
equipamento, não dá pra automatizar por aqui. No CLI do FortiOS
(`Console`/SSH, usuário admin):

```
config log syslogd setting
    set status enable
    set server "192.168.0.132"
    set port 514
    set facility local7
end
```

Se os logs chegarem com aparência estranha (formato CEF em vez de
`date=... time=...`), rode também `set format traditional` (ou
`default`, depende da versão do FortiOS) dentro do mesmo bloco.

Depois de configurar, verificar no dashboard do Wazuh
(`https://wazuh.lab` → *Threat Intelligence* / *Discover*, filtrar por
`rule.groups: fortigate` ou `location: 192.168.0.132`) ou via linha de
comando:

```
docker exec single-node-wazuh.manager-1 tail -f /var/ossec/logs/alerts/alerts.json | grep fortigate
```

`wazuh_syslog_allowed_ips` (`roles/wazuh_syslog/defaults/main.yml`) está
restrito a `192.168.0.0/24` — ajuste pro `/32` do FortiWiFi se quiser
travar mais.

## Grafana (`roles/grafana`)

Dashboards em `http://localhost:3000` (login `admin`, senha em
`roles/grafana/vars/main.yml`, campo `grafana_admin_password`).
Datasource "Wazuh - OpenSearch" já provisionado automaticamente, lendo
direto do índice `wazuh-alerts-*` do indexer — nenhum dado próprio, é só
uma janela pros mesmos alertas do dashboard do Wazuh, em formato de
dashboard/KPI.

Pegadinhas encontradas:
- **`GF_INSTALL_PLUGINS` está deprecado** nessa versão do Grafana (warning
  no log), mas ainda funciona pra instalar o
  `grafana-opensearch-datasource`. Se parar de funcionar numa atualização
  futura, trocar para `GF_PLUGINS_PREINSTALL_SYNC`.
- **`jsonData.version` do datasource precisa ser a versão real do
  OpenSearch** (ex.: `"2.19.5"`), não um wildcard tipo `"2.11+"` — senão
  o health-check falha com "No version set".
- **O botão "Save & Test" do datasource falha com "Index not found:
  wazuh-alerts-\*"** mesmo com tudo funcionando — é o health-check do
  plugin não lidando bem com índice por wildcard/data (`wazuh-alerts-4.x-
  AAAA.MM.DD`). Confirmado que a query de verdade funciona (contagem de
  documentos bate) apesar do aviso vermelho. Não é um problema real.
- Grafana conecta no indexer via rede Docker (`wazuh.indexer:9200`,
  mesma rede do stack do Wazuh) — não depende do DNS `.lab` de dentro do
  container.

## Lab de pentest (`roles/pentest_network`, `kali`, `windows_vm`, `linux_target`)

Fase 6 do roadmap. Kali = atacante, Windows 11 × 2 e a VM Linux = alvos
(vão rodar Atomic Red Team + Sysmon/auditd + agente Wazuh). Tudo numa
rede Docker isolada (`pentest_lab`, `10.10.20.0/24`) — **não** exposta
pro resto da rede doméstica (essa máquina só tem Wi-Fi, `macvlan` não
funcionaria de forma confiável, e isolar é mais seguro de qualquer jeito
pra alvos deliberadamente vulneráveis). O manager do Wazuh está
conectado nessa rede também, então os agentes apontam pra
`wazuh.manager` (DNS interno do Docker), sem depender do domínio `.lab`.

Acesso:
- Kali: `docker exec -it kali bash` (sem ferramentas pré-instaladas de
  propósito — instalar via `apt install <pacote>` conforme a
  necessidade).
- `win11-a`: `http://localhost:8006` (VNC web, só em loopback) e RDP na
  porta `3389` depois de instalado.
- `win11-b`: `http://localhost:8007` (VNC) / RDP na porta `3390`.
- `linux-target`: `http://localhost:8008` (VNC).

**Passos manuais depois do primeiro boot** (não dá pra automatizar 100%
nesta primeira passada — documentado aqui, não esquecido):
1. Windows: completar a instalação pela tela VNC (Windows já vem
   descompactando o ISO sozinho, primeira vez demora). Depois, instalar
   Sysmon e o agente Wazuh (MSI oficial, `msiexec /i wazuh-agent... 
   WAZUH_MANAGER='wazuh.manager'`).
2. Linux target: primeiro boot do Debian, depois `apt install auditd` e
   instalar o agente Wazuh apontando pro mesmo `WAZUH_MANAGER=wazuh.manager`
   (mesmo pacote/processo do `roles/wazuh_agent`, mas dentro da VM).

Por que a VM Linux não é só um container Debian comum: `auditd` depende
do subsistema de auditoria do kernel via netlink, que é *system-wide* —
num container ele enxergaria o kernel do host, não teria visibilidade
isolada e realista do "alvo". Por isso é uma VM de verdade (QEMU/KVM),
igual ao Windows, só que bem mais leve (`RAM_SIZE: 2G`).

Orçamento de recursos (22GB RAM total): Windows 11 × 2 @ 4GB = 8GB,
Linux target @ 2GB, Kali/Grafana/Wazuh ~6GB — folga de uns 6GB mesmo com
tudo rodando ao mesmo tempo. Disco: `/mnt/storage` tem 424GB livres,
VMs usam thin-provisioning (só ocupam o que realmente é escrito).
