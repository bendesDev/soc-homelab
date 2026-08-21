# infra

Playbooks Ansible que documentam e automatizam a configuração desta máquina
(`localhost`), a partir de 2026-08-20. A ideia é simples: toda vez que algo
for instalado ou configurado manualmente aqui, isso vira uma task neste
repositório — o playbook funciona como documentação viva e como forma de
reproduzir o setup em outra máquina.

## Estrutura

- `site.yml` — playbook principal, roda todas as roles.
- `inventory/hosts.ini` — inventário (só `localhost`).
- `roles/packages` — pacotes instalados via `pacman` e AUR (`yay`).
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
- `CHANGELOG.md` — registro humano do que foi feito e por quê, em ordem
  cronológica.

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

1. **Core SIEM (atual)** — Wazuh (manager + indexer + dashboard) via Docker,
   single-node.
2. Agente Wazuh nesta própria máquina, pra gerar telemetria real.
3. Ingestão/parsing adicional (Graylog), dashboards de métricas/KPI
   (Grafana).
4. SOAR — orquestração de playbooks (Shuffle) e gestão de casos
   (DFIR-IRIS), com roteamento por severidade (ex.: webhook no Discord para
   alertas de baixa severidade).
5. Threat intel — enriquecimento de IOCs (GeoIP/ASN, VirusTotal) e MISP.
6. Simulação de adversário e teste de detecção — Kali Linux + Atomic Red
   Team, endpoints Windows (Sysmon) e Linux (auditd) como alvo.
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
