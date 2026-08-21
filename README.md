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
