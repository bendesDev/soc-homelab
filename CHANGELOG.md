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
