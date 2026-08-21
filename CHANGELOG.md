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
