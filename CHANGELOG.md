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
