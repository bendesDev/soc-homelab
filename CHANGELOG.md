# Changelog

Registro cronológico do que foi feito nesta máquina e por quê.

## 2026-08-20

- Criado o repositório `infra` com a estrutura base de playbooks Ansible
  (`site.yml`, roles `packages` e `dotfiles`).
- Instalado o pacote `ansible` via pacman.
- Adicionado o dotfile do fish shell (`~/.config/fish/config.fish`) à role
  `dotfiles`, agora symlinkado a partir de
  `roles/dotfiles/files/config.fish`. (`fish_variables` não entrou por ser
  estado gerado — histórico, abreviações, etc. — não um dotfile editado à
  mão.)
- Adicionadas roles `storage`, `docker` e `soc_lab`, e declarados `docker`
  + `docker-compose` em `pacman_packages`. Início do laboratório de estudo
  para a vaga de analista SOC (início 2026-09-09):
  - `storage`: formata o HD adicional (465GB, `/dev/sda`, antes NTFS em
    `/mnt/Archive` com 266GB de dados — apagados a pedido explícito do
    Gabriel) como ext4, montado em `/mnt/storage`.
  - `docker`: move o `data-root` do Docker para `/mnt/storage/docker`
    (NVMe tinha só 92GB livres).
  - `soc_lab`: clona `wazuh-docker` (tag `v4.14.7`) e sobe o stack
    single-node (manager + indexer + dashboard) via Docker Compose.
  - Roadmap de fases futuras (Graylog, Grafana, Shuffle/DFIR-IRIS, MISP,
    Kali+Atomic Red Team, Velociraptor) documentado no `README.md`.

## 2026-08-21

- Fix na role `storage`: `force: true` no `community.general.filesystem`
  ignorava a checagem de idempotência e tentava reformatar a cada
  execução (falhando por o disco já estar montado). Virou variável
  `storage_format_force` (default `false`).
- Fix na role `docker`: `/etc/docker` não existia (pacote do Arch não cria
  o diretório), quebrando a task que escreve `daemon.json`.
- Trocada a senha padrão do Wazuh (`admin`/`SecretPassword` e a senha da
  API `wazuh-wui`) por senhas geradas aleatoriamente, seguindo o processo
  oficial (docker-compose.yml + hash via `hash.sh` + `internal_users.yml`
  + `securityadmin.sh`). Senhas não ficam no git — guardadas fora do
  repositório versionado (`docker-compose.yml` vive em `~/soc-lab`, fora
  do `infra`).
- Adicionada role `dns`: `dnsmasq` (declarado em `pacman_packages`)
  servindo o domínio local `.lab` pra rede inteira (`wazuh.lab` →
  `192.168.0.132`), mais um drop-in no `systemd-resolved`
  (`/etc/systemd/resolved.conf.d/lab.conf`) pra esta máquina rotear
  `*.lab` pro `dnsmasq` local sem afetar a resolução normal de DNS. IP
  local vem de DHCP — risco de mudar documentado no `README.md`.
- Corrigido acesso ao dashboard do Wazuh (erro 401 em `/api/check-api`):
  a troca de senha da API (`wazuh-wui`) tinha atualizado a env var e o
  manager (que resincroniza a cada boot), mas não o `wazuh.yml` do
  dashboard — arquivo estático, só lido uma vez na criação, guarda a
  senha usada pra falar com a API do manager. Atualizado
  `config/wazuh_dashboard/wazuh.yml` e reiniciado o container do
  dashboard.
- Fix na role `soc_lab`: `git clone` ganhou `update: false` — depois do
  clone inicial, arquivos do `wazuh-docker` (senhas, `wazuh.yml`) passam
  a ser editados manualmente, e o módulo `git` recusava (ou, com force,
  apagaria) essas mudanças a cada re-run do playbook.
- Fix na role `soc_lab`: task de verificar os certificados do indexer
  ganhou `become: true` — o diretório `wazuh_indexer_ssl_certs` é criado
  com `700` pelo container gerador (roda como root), ilegível pro usuário
  normal.
- Fix na role `dns`: `wazuh.lab` não resolvia mesmo com tudo `ok` no
  playbook — os handlers de restart (`dnsmasq`, `systemd-resolved`) só
  rodam no fim do play inteiro, e como a role `soc_lab` falhava logo
  depois (nos dois problemas acima), o play abortava antes de flushar os
  handlers: o `dnsmasq` chegou a subir certo (primeira ativação já lê a
  config nova), mas o `systemd-resolved` nunca foi reiniciado pra
  carregar o drop-in de roteamento do domínio `.lab`. Adicionado
  `meta: flush_handlers` no fim da role `dns`, pra não depender do resto
  do playbook terminar sem erro.
- Tentativa inicial de instalar o agente Wazuh via `.deb` + `dpkg`
  diretamente falhou (Arch não resolve dependências Debian por esse
  caminho — `libc6`, `lsb-release`, `debconf`, `adduser`, `procps` "não
  instalados", apesar dos equivalentes Arch existirem). Revertido com
  `dpkg -P wazuh-agent` e reinstalado corretamente via AUR
  (`paru -S wazuh-agent`, pacote mantido pela comunidade, mesma versão
  4.14.7 do manager).
- Corrigido bug pré-existente na role `packages`: a task de AUR chamava
  `yay`, que não está instalado nesta máquina (só `paru`). Trocado para
  `paru -S --needed --noconfirm`.
- Adicionada role `wazuh_agent`: aponta `/var/ossec/etc/ossec.conf` para
  `wazuh.lab` e garante o serviço `wazuh-agent` habilitado/rodando.
  `wazuh-agent` declarado em `aur_packages`. Agente já apareceu como
  ativo no dashboard. Fase 2 do roadmap do lab concluída.
- Adicionada role `wazuh_syslog`: listener de syslog (UDP 514) no
  manager, pra ingerir logs do FortiWiFi 40C do Gabriel (Wazuh já traz
  decoders/regras nativas pra FortiGate/FortiOS). `ossec.conf` do manager
  fica num volume Docker nomeado (`wazuh_etc`), não em bind-mount no
  host — a role edita via `docker exec` + Python (idempotente),
  diferente do padrão `template` usado nas outras roles.
  - Testado ponta a ponta com pacote UDP simulado: decoder
    `fortigate-firewall-v6` reconheceu o log, regra `81606` ("Fortigate:
    Login failed") disparou, alerta indexado.
  - `allowed-ips` restrito a `192.168.0.0/24` — motivo real de um teste
    inicial ter falhado: pacote enviado via `127.0.0.1` chega ao manager
    como `172.18.0.1` (NAT do Docker pro localhost), fora da faixa
    permitida. Não é bug — confirma que o filtro de IP está funcionando;
    o FortiWiFi de verdade manda da rede local e passa normalmente.
  - Falta o passo manual no próprio FortiWiFi (`config log syslogd
    setting`, ver `README.md`) — não automatizável a partir daqui.
- Pré-baixadas as imagens `grafana/grafana`, `graylog/graylog:6.1` e
  `mongo:6`, adiantando a fase 3 do roadmap sem ainda decidir a
  arquitetura (aguardando decisão: Grafana direto no indexer do Wazuh
  primeiro, Graylog só quando houver fonte de log adicional pra ele).
- Decidido: Grafana primeiro (dado real disponível, zero sudo), Graylog
  fica pra quando o FortiWiFi estiver mandando log de verdade. Adicionada
  role `grafana`: container na rede Docker do stack do Wazuh
  (`single-node_default`), datasource "Wazuh - OpenSearch" provisionado
  automático contra o indexer.
  - Introduzido o padrão de segredos por role: `roles/<nome>/vars/main.yml`
    (gitignored) + `.example` versionado. `roles/grafana/vars/main.yml`
    guarda `wazuh_indexer_password` (reaproveitada da senha já trocada) e
    `grafana_admin_password` (gerada nova).
  - Dois bugs de configuração corrigidos ao testar: `jsonData.version`
    tinha que ser a versão real do OpenSearch (`2.19.5`), não `"2.11+"`
    ("No version set"); e o health-check do plugin falha com "Index not
    found" pra índice por wildcard mesmo funcionando (confirmado via
    query direta: contagem de documentos bate, 353). Detalhes no
    `README.md`.
  - Aplicado manualmente via `docker compose` (sem sudo, grupo `docker`
    já bastou) — não passou pelo `ansible-playbook site.yml -K` ainda,
    então falta confirmar idempotência na próxima rodada completa.
- Início do "shape" da fase 6 (simulação de adversário): Kali (atacante)
  + Windows 11 × 2 + VM Linux (alvos, vão rodar Atomic Red Team). Roles
  novas: `pentest_network`, `kali`, `windows_vm`, `linux_target`.
  - Decisão de rede: esta máquina só tem Wi-Fi, `macvlan` não é confiável
    nesse cenário (roteador filtra MAC spoofing) — criada rede Docker
    isolada `pentest_lab` (`10.10.20.0/24`), manager do Wazuh conectado
    nela também. Isola os alvos vulneráveis do resto da rede doméstica
    (bônus de segurança, não só contorno de limitação).
  - Decisão técnica: VM Linux alvo é uma VM de verdade (QEMU/KVM via
    `qemux/qemu`, `RAM_SIZE: 2G`), não um container Debian comum —
    `auditd` depende do subsistema de auditoria do kernel via netlink
    (system-wide), não teria visibilidade isolada/realista num container
    compartilhando o kernel do host.
  - Windows 11 × 2 via `dockurr/windows` (QEMU/KVM, instalação
    automatizada do ISO), `RAM_SIZE: 4G` cada, VNC web só em loopback
    (portas 8006/8007), RDP publicado (3389/3390).
  - Aplicado manualmente via `docker compose` (rede + 4 serviços,
    ~8GB RAM em uso no total, dentro da folga esperada). Ainda falta
    passar pelo `ansible-playbook site.yml -K` pra confirmar
    idempotência, e os passos manuais dentro de cada VM (Sysmon/auditd +
    agente Wazuh) documentados no `README.md`.

## 2026-08-24

- Adicionado `ani-cli` (AUR) à role `packages`.
- Nova role `media`: stack de mídia (Jellyfin + Prowlarr + Sonarr + Radarr
  + qBittorrent + Bazarr + Jellyseerr) via `docker compose`, dados em
  `/mnt/storage/media-stack`. Sem VPN na frente do qBittorrent (decisão
  consciente do gabriel, tráfego sai direto).
  - Aplicado via `ansible-playbook site.yml -K`. Diretório raiz precisou
    de `owner`/`group` explícitos nas tasks porque `/mnt/storage` é
    `root:root` (mount point criado pela role `storage`) — sem isso o
    usuário `gabriel` não conseguia criar nada lá dentro.
  - Descoberto durante o setup: a imagem `linuxserver/qbittorrent` usa
    `/downloads` como save path padrão, que não bate com o volume
    montado (`/data/downloads`) — corrigido manualmente via API
    (`app/setPreferences`). Ainda não voltou pra dentro da role — próxima
    vez que o container for recriado do zero, esse ajuste (e a senha
    permanente do WebUI) precisa ser refeito manualmente.
  - Login/contas configuradas manualmente via API de cada serviço
    (`admin`/`251718` em qBittorrent, Sonarr, Radarr, Prowlarr e
    Jellyfin), Prowlarr conectado ao Sonarr/Radarr, Jellyfin com
    bibliotecas "Filmes" (`/data/movies`) e "Séries" (`/data/tv`).
    Nenhuma dessas contas/integrações está na role Ansible — só existem
    no estado runtime dos containers, então não sobrevivem a um volume
    novo. Fica como TODO se algum dia isso for reproduzido do zero.
  - Bazarr e Jellyseerr não deram pra terminar via API (settings do
    Bazarr não persiste via API sem entender o formato exato; Jellyseerr
    rejeita o admin do Jellyfin com `NO_ADMIN_USER` mesmo com
    `IsAdministrator: true` — não investigado a fundo). Ficaram como
    passos manuais na UI.
- `site.yml`: roles convertidas pra `{ role: ..., tags: ... }`, uma tag
  por role (mesmo nome da role). Permite `ansible-playbook site.yml -K
  --tags <role>` pra aplicar só uma parte, sem rodar o resto (ex.: sem
  isso, qualquer mudança pequena reaplicava até o stack do Wazuh/Kali/VMs
  inteiro).
- Adicionado `flaresolverr` à role `media` (container extra no mesmo
  `docker-compose.yml`) e conectado ao Prowlarr como Indexer Proxy
  (`Settings > Indexer Proxies`, tag `flaresolverr`).
  - **Indexer Proxy no Prowlarr só se aplica a indexadores com a mesma
    tag** — campo `tags` vazio no proxy não significa "aplica a tudo".
    Criada a tag `flaresolverr` via API e associada ao proxy; qualquer
    indexador que precise do FlareSolverr precisa da mesma tag nele.
  - Pegadinha feia: o campo "Host" do Indexer Proxy tem que ser
    `http://flaresolverr:8191/` (nome do container, resolvido de dentro
    do container do Prowlarr) — `localhost:8191` (o que se vê no
    navegador) dá "connection refused" porque ali "localhost" é o
    próprio Prowlarr, não o FlareSolverr. Aconteceu duas vezes (uma
    porque um `Settings > General > Proxy` do próprio Prowlarr também
    tinha sido setado erroneamente pra `localhost:8191`).
  - Testado de ponta a ponta contra `nowsecure.nl` (site padrão de teste
    de bypass de Cloudflare) — resolve desafio real, não só responde
    "ready".
- Nova role `dashboard`: [Homepage](https://gethomepage.dev) agregando
  tudo (mídia + SOC lab) num painel só, `docker-compose` na rede `media`
  (só essa rede — status de container vem do socket do Docker, não
  precisa estar na mesma rede pra isso). Login/senha de cada serviço nas
  descrições dos cards, de propósito (uso pessoal, ver decisão no
  histórico da conversa) — ciente que isso fica visível pra qualquer um
  na rede Wi-Fi de casa que acessar o painel.
  - Segredos (senhas dos serviços de mídia + API keys do Sonarr/Radarr/
    Prowlarr) em `roles/dashboard/vars/main.yml` (gitignored, `.example`
    versionado), seguindo o padrão já usado pela role `grafana`. As
    senhas do Grafana e do Wazuh não foram duplicadas — o template só
    referencia `grafana_admin_password`/`wazuh_indexer_password`
    (definidas em `roles/grafana/vars/main.yml`), que já ficam
    disponíveis no escopo da play porque a role `grafana` roda antes da
    `dashboard` no `site.yml`.
  - Três bugs até funcionar de verdade:
    1. `HOMEPAGE_LISTEN_PORT` não é respeitada nessa imagem — ela sempre
       escuta 3000 por dentro. Corrigido mapeando `3001:3000` em vez de
       trocar a porta interna.
    2. Next.js valida o header `Host` da requisição contra uma allowlist
       — sem `HOMEPAGE_ALLOWED_HOSTS`, até `localhost:3001` vindo de fora
       do container toma "Host validation failed".
    3. Os `href` dos cards estavam todos como `localhost:PORTA` — funciona
       abrindo do navegador desta máquina (onde "localhost" = ela mesma),
       mas não de outro dispositivo na rede (celular, etc.), onde
       "localhost" aponta pro próprio dispositivo. Trocado pro IP da LAN
       (`dns_lan_ip`, `192.168.0.132`) em todos os serviços expostos ao
       resto da rede. Os 3 de VNC (Windows A/B, Linux target) continuam
       em `localhost` de propósito — essas portas são loopback-only
       (bind `127.0.0.1`), só acessíveis mesmo a partir desta máquina.
  - Idioma: pedido pra deixar em pt-BR, mas essa imagem trava
    `defaultLocale`/`locales` em `["en"]` direto no build
    (`next-i18next.config.js`) — `settings.yaml` não tem como contornar
    isso, apesar de existirem arquivos de tradução pt-BR empacotados
    (parecem vestígio de uma feature de i18n ainda não habilitada nessa
    versão). Decisão: deixar em inglês (menus/telas do app), já que os
    nomes de grupo/serviço/descrição (o que mais se vê no dia a dia) são
    texto nosso, não tradução do app.
- `roles/network` (nova): fixa o IP desta máquina (`192.168.0.132`) via
  `nmcli` (a conexão Wi-Fi `Lucca5G` já existente, só os campos `ipv4.*`
  — nada de recriar a conexão, risco alto numa Wi-Fi ativa). Decisão
  consciente do gabriel: fixar no cliente em vez de reservar por MAC no
  roteador, ciente do risco de conflito de IP se o DHCP do roteador
  também distribuir esse endereço um dia.
- **`roles/media/tasks/configure.yml` (novo)**: tudo que tinha sido feito
  manualmente via curl numa sessão de troubleshooting (contas do
  qBittorrent/Sonarr/Radarr/Prowlarr/Jellyfin, integrações entre eles,
  bibliotecas do Jellyfin, proxy do FlareSolverr) virou task idempotente
  de Ansible, usando `ansible.builtin.uri`. Motivo: gabriel vai migrar
  esse setup pra um Proxmox depois, sem se preocupar com backup dos
  dados — só precisa que `ansible-playbook site.yml -K` reproduza tudo
  sozinho numa VM nova.
  - API keys do Sonarr/Radarr/Prowlarr não são segredo fixo — são
    geradas por cada app no primeiro boot e lidas do `config.xml` de
    cada um via `grep` (fact `set_fact`), não hardcoded.
  - Testado de verdade: rodado contra o ambiente já configurado (o
    melhor teste de idempotência possível) várias vezes até sair 100%
    limpo, sem `changed` nem erro. Alguns detalhes que só apareceram
    testando de verdade (não dava pra saber só lendo a doc da API):
    Sonarr/Radarr/Prowlarr retornam **202**, não 200, ao salvar
    `config/host` com o método de autenticação mudando; o login do
    qBittorrent retorna **204**, não 200; e o payload de `config/host`
    do Sonarr/Radarr/Prowlarr precisa de *todos* os campos do resource
    (incluindo os que parecem irrelevantes, tipo `logSizeLimit` e
    `backupInterval`) — mandar só um subconjunto derruba a API com
    `NullReferenceException` (500) em vez de validar e rejeitar (400).
  - `roles/dashboard` também foi ajustado: em vez de duplicar a senha e
    as API keys em `vars/main.yml` próprio, agora lê a senha de
    `media_admin_password` (definida em `roles/media/vars/main.yml`) e
    as API keys direto do `config.xml` de cada app — do jeito que
    estava antes, `--tags dashboard` sozinho quebraria se `--tags media`
    não tivesse rodado antes na mesma invocação (facts não sobrevivem
    entre invocações separadas do `ansible-playbook`).
  - Bazarr continua de fora (ver decisão de 2026-08-24 mais acima) —
    ainda não tem uma forma confiável de automatizar.
- **Jellyseerr resolvido** — o `NO_ADMIN_USER` que travava o setup desde
  2026-08-24 não tinha nada a ver com o Jellyfin: lendo o código-fonte
  compilado dentro do próprio container (`/app/dist/routes/auth.js`),
  achei que `POST /api/v1/auth/jellyfin` exige um campo `serverType: 2`
  (Jellyfin) no corpo — sem ele, cai direto nesse erro genérico. Formas
  não documentadas descobertas na mesma investigação:
  - Se `settings.jellyfin.ip` já estiver preenchido (rerun depois de
    setup parcial), mandar `hostname` de novo quebra com "Jellyfin
    hostname already configured" — o corpo do request muda dependendo
    do que já foi salvo.
  - `initialized: true` não é automático depois do login — precisa de
    um `POST /api/v1/settings/initialize` autenticado à parte.
  - Sincronizar bibliotecas (`?sync=true`) sempre zera `enabled` de
    todas (só fica true se vier no mesmo request via `?enable=`) — não
    dá pra só "atualizar a lista" sem reafirmar quais ficam ligadas.
  - Sonarr exige o campo `enableSeasonFolders` que não aparece em nenhum
    outro app da Servarr; sem ele, 400.
  - Tudo isso virou task no `configure.yml`: setup inicial, sincronização
    de bibliotecas, e conexão com Sonarr/Radarr (perfil de qualidade
    HD-1080p, descoberto dinamicamente via `/settings/{radarr,sonarr}/test`
    em vez de hardcoded — o ID desse perfil não é garantido ser o mesmo
    em toda instalação nova, só o nome é estável).
- Achado pelo gabriel: um filme já importado pelo Radarr (arquivo no
  disco certinho) não aparecia no Jellyfin — o Jellyfin não descobre
  arquivo novo sozinho sem scan manual/agendado ou monitoramento em
  tempo real (nenhum dos dois configurado). Corrigido de duas formas:
  1. Scan manual disparado uma vez pra resolver o backlog na hora
     (`POST /Library/Refresh`).
  2. **Conexão Sonarr/Radarr -> Jellyfin** (tipo "Emby/Jellyfin" nas
     notificações de cada um, `updateLibrary: true`) adicionada ao
     `configure.yml` — a partir de agora, todo import novo já dispara um
     refresh direcionado no Jellyfin sozinho, sem precisar de scan
     manual ou agendado. Usa uma API key do Jellyfin gerada e guardada
     via `/Auth/Keys?App=Ansible` (idempotente: só cria se ainda não
     existir uma chamada "Ansible").
- **Bazarr resolvido** — a settings API que "não persistia" (ver decisão
  de mais cedo hoje) na verdade funcionava, só não do jeito que eu tinha
  tentado. Lendo `/app/bazarr/bin/bazarr/api/system/settings.py` de
  dentro do próprio container achei a causa real: o endpoint só lê
  `request.form` (nunca JSON, por isso as tentativas em JSON eram
  ignoradas em silêncio) e as chaves têm que ser literalmente
  `settings-<seção>-<campo>` (ex.: `settings-sonarr-apikey`), com
  booleano como string minúscula `"true"`/`"false"` — `"True"` (Python-
  style, meu primeiro palpite de manhã) é rejeitado.
  - Formalizado em `configure.yml`: idiomas (pt-BR + inglês) habilitados,
    perfil "pt-BR + EN" criado, conectado ao Sonarr/Radarr como perfil
    padrão de série/filme.
  - Pegadinha só reproduzida testando via Ansible (não apareceu nos
    testes manuais de manhã, por causa da ordem diferente das chamadas):
    o item de cada idioma no perfil precisa do campo `audio_only_include`
    — sem ele, `list_missing_subtitles_movies()` quebra com `KeyError`
    (500) **só se o Sonarr/Radarr já estiverem conectados** no momento
    de salvar o perfil (senão essa função nem roda). Mais um motivo pra
    sempre testar contra o ambiente real, não só validar sintaxe.
  - Provedor de legenda: só `napiprojekt` habilitado por enquanto (não
    precisa de conta, mas fraco pra lançamentos novos/pt-BR). Gabriel vai
    criar conta grátis no opensubtitles.com e passar as credenciais pra
    eu conectar — anotado como próximo passo em
    `roles/media/tasks/configure.yml`.
- **opensubtitles.com conectado.** Pegadinha: a API deles rejeita login
  com e-mail, exige o *username* da conta especificamente (confirmado
  testando direto contra `api.opensubtitles.com/api/v1/login`, fora do
  Bazarr, pra isolar se o problema era credencial ou o Bazarr — foi
  credencial). Credenciais em `roles/media/vars/main.yml` (gitignored,
  `bazarr_opensubtitlescom_username`/`_password`), task nova em
  `configure.yml`. Testado de verdade: legenda em inglês da "Avatar
  Aang" baixada automaticamente pelo Bazarr assim que conectado. pt-BR
  não achou pra essa release específica ainda (lançamento
  recente/nicho, não é erro — o Bazarr continua tentando sozinho).
  - Outra pegadinha só vista testando: depois de um erro de credencial
    (`ConfigurationError`), o Bazarr **throttla o provedor por 12h**
    (fica em memória, não é sobre tempo real passado) — reiniciar o
    container limpa esse estado. Sem isso, mesmo corrigindo a credencial
    certa, as buscas seguintes pareciam "não fazer nada" porque nem
    chegavam a tentar de novo.
- Mesmo problema do filme "sumido" (ver entrada de hoje sobre o Jellyfin
  não escanear sozinho) aconteceu de novo com a legenda que o Bazarr
  baixou — e dessa vez não tem como resolver com uma notificação tipo a
  do Sonarr/Radarr, porque o Bazarr não tem esse recurso de avisar o
  Jellyfin. Solução definitiva: `EnableRealtimeMonitor: true` nas duas
  bibliotecas do Jellyfin (monitoramento de sistema de arquivos, pega
  qualquer mudança sozinho — import novo, legenda nova, o que for).
  Aplicado nas bibliotecas já existentes e também no `configure.yml` pra
  bibliotecas criadas do zero numa migração futura.

## 2026-08-26

- **Grafana em crash loop, achado e corrigido.** 4474 restarts —
  `roles/grafana/tasks/main.yml` escrevia `wazuh.yml` (datasource
  provisionado) com `mode: "0600"`, ilegível pro UID não-root do
  container. Trocado pra `0644`. Sem isso, todo `ansible-playbook
  --tags grafana` reintroduzia o loop.
- **Jellyfin: transcodificação por hardware (VAAPI/Intel QuickSync).**
  Host tem GPU Intel integrada (CometLake UHD) com `/dev/dri/renderD128`
  world-accessible (0666) mas o container não tinha o device passado.
  Adicionado `devices:` no compose (`roles/media/templates/
  docker-compose.yml.j2`) + config `HardwareAccelerationType: vaapi` e
  `HardwareDecodingCodecs` incluindo `hevc` via `configure.yml`
  (`System/Configuration/encoding`, GET-then-compare-then-POST porque
  esse endpoint não distingue criar/atualizar).
- **Readarr + Prowlarr conectados; Audiobookshelf instalado.**
  - Readarr (`lscr.io/linuxserver/readarr`) não publica tag `latest`
    (projeto beta) — e a tag rolling `develop` veio com o manifest
    quebrado no registry (Docker Hub marca como "inactive", nem
    `linux/amd64` resolve). Usando tag versionada fixa
    `develop-0.4.18.2805-ls157`.
  - **Achado real, não corrigido:** o cliente de download qBittorrent
    não consegue ser cadastrado no Readarr — "Authentication failure"
    mesmo com credencial certa. Isolado testando: login direto de
    dentro do container do Readarr contra o qBittorrent funciona (204),
    e o mesmo teste no Sonarr funciona também (200) — só o código do
    Readarr (herdado, não atualizado desde meados de 2025) não sabe
    interpretar a resposta 204-sem-corpo que o qBittorrent 5.2.3 passou
    a mandar (versões antigas do qBittorrent respondiam 200 com corpo
    "Ok."). Testado tanto na tag `develop` quanto `nightly` — mesmo bug
    nas duas. Documentado em `configure.yml` com comentário; sem
    automação até o projeto corrigir isso ou aparecer outra solução.
  - Pasta raiz `/data/library/books` criada com perfil "eBook" (quality)
    + "Standard" (metadata) — ids descobertos em runtime por nome, não
    hardcoded (podem mudar entre instâncias).
  - Prowlarr → Readarr conectado com `syncCategories` de livros (Books/
    Mags/EBook/Comics: 7000/7010/7020/7030).
  - Audiobookshelf (`ghcr.io/advplyr/audiobookshelf`) não é um app
    Servarr — sem `config.xml`/API key. Usuário root criado via
    `POST /init` (só funciona uma vez, `GET /status.isInit` vira `true`
    depois — descoberto testando ao vivo). Biblioteca "Audiobooks"
    apontando pra `/data/library/audiobooks`.

- **Readarr trocado pelo Chaptarr.** Investigando o bug do qBittorrent
  documentado acima, achei um problema bem maior: o Readarr foi
  **oficialmente aposentado** pelo time do Servarr em 2026 — o backend
  de metadados (`api.bookinfo.club`) morreu de vez, confirmado como
  NXDOMAIN até em DNS público (8.8.8.8). Ou seja, nem a busca de autor/
  livro funcionava mais, independente do bug do download client.
  - Substituído pelo **Chaptarr** (`chaptarr/chaptarr:latest` no Docker
    Hub — não confundir com o mirror pessoal `robertlordhood/chaptarr`),
    fork ativo que assumiu o projeto. Junta ebooks e audiobooks numa
    instância só.
    - Metadados funcionando de verdade (Goodreads + Hardcover),
      confirmado buscando "Arthur Conan Doyle" e recebendo overview,
      links, capas.
    - qBittorrent como cliente de download funciona de primeira — sem o
      bug do Readarr. Confirmado cadastrando ao vivo antes de escrever a
      task no `configure.yml`.
    - Volumes mudaram: `/ebooks` e `/audiobooks` (pastas separadas,
      diferente do root folder único do Readarr) + `/downloads` (sem o
      prefixo `/data` que os outros *arr usam).
    - Schema da API mudou em relação ao Readarr — achado testando, não
      só lendo doc: `config/host` ganhou campos OIDC novos que causam
      500 genérico se o PUT não vier com o objeto inteiro (por isso
      GET-then-modify-then-PUT em vez de montar o corpo na mão, diferente
      do padrão usado pro Sonarr/Radarr/Prowlarr); perfis de qualidade/
      metadata são separados por tipo de mídia; o campo de categoria do
      qBittorrent virou `audiobookCategory`/`ebookCategory` em vez de um
      `category` único; e o endpoint de adicionar autor (`POST /author`)
      exige um campo `path` explícito além do `rootFolderPath`, e
      `audiobookQualityProfileId`/`ebookQualityProfileId`/
      `audiobookMetadataProfileId`/`ebookMetadataProfileId` em vez de um
      único par — sem eles, a mensagem de erro nem mostra o valor
      inválido ("Invalid Path: '{path}'", literal, parece um bug de
      template de mensagem no próprio Chaptarr).
  - Prowlarr não tem uma implementação nativa "Chaptarr" nesta versão —
    continua cadastrado como aplicação tipo "Readarr" (mesmo contrato de
    API v1, funciona igual), agora com a categoria 3030 (Audio/
    Audiobook) somada às de livro.
  - Removido o workaround "Torrent Blackhole" que tinha sido montado pra
    contornar o bug do Readarr (pasta `/watch` no qBittorrent, `scan_dirs`
    configurado) — não é mais necessário com o Chaptarr.
  - Config antigo do Readarr arquivado em
    `config/readarr.bak-dead-project` (não apagado, só fora do
    `docker-compose.yml`) — pode ser removido quando não fizer mais
    falta.
  - Request de livro/audiobook agora é pela própria UI do Chaptarr em
    `http://192.168.0.132:8789` (sem app tipo Jellyseerr pra livros).
  - **Achado pelo gabriel usando a UI de verdade** (não só testando por
    API): as duas pastas raiz tinham sido criadas com
    `defaultQualityProfileId`/`defaultMetadataProfileId` (nomes do
    Readarr) — o `POST /rootfolder` aceita esses campos com 201 mas
    **ignora silenciosamente**, e a UI mostrava "Selected root folder
    '/ebooks' does not have ebook defaults configured" ao tentar
    adicionar um livro. Só um `PUT /rootfolder/{id}` com os campos
    corretos (`ebookQualityProfileId`/`ebookMetadataProfileId` pra
    ebooks, `audiobookQualityProfileId`/`audiobookMetadataProfileId`
    pra audiobooks) realmente preenche o objeto aninhado `ebook`/
    `audiobook` que a UI lê. Corrigido nas duas pastas ao vivo e no
    `configure.yml` (sempre reaplica os PUTs, não só no create — cobre
    quem já tinha passado por esse bug numa aplicação anterior da
    role). De brinde, liguei `audiobookWriteAudioBookShelfMetadataJson`
    e `audiobookWriteAudioBookShelfCover` na pasta de audiobooks — o
    Chaptarr escreve metadata/capa no formato que o Audiobookshelf lê
    direto, sem precisar rematch por conta própria.
