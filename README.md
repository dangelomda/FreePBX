# ✅ Guia Definitivo v2 (Completo)  
## FreePBX 17 Bare Metal no Raspberry Pi 5 (Debian 12)

Este guia consolida o passo a passo completo e assertivo para instalar o **FreePBX 17** em um Raspberry Pi 5 rodando **Debian 12 (“Bookworm”) 64-bit**. Todos os comandos foram testados e incluem as correções necessárias para dependências e permissões.

---

## 🔧 Pré-requisitos

- Raspberry Pi 5 com instalação limpa do **Raspberry Pi OS Lite 64-bit (Debian 12)**
- Acesso à internet funcional no dispositivo

---

## 🛠️ Fase 1 – Preparação do Sistema

### Atualize completamente o sistema

```bash
sudo apt update && sudo apt upgrade -y
