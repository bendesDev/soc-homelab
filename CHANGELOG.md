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
