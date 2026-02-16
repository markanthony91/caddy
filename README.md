# 🌐 Caddy Proxy - Marcelo's Systems

Este diretório centraliza a configuração do **Caddy Server** utilizado como Proxy Reverso para o ecossistema de sistemas locais, permitindo acesso seguro via HTTPS e MagicDNS (Tailscale).

## 🚀 Contexto: Jardim Digital & Mobilidade

O objetivo inicial foi criar um **Jardim Digital** (baseado em Markdown) acessível via iPhone/iPad. Após testar sincronização via Google Drive/Obsidian, optamos por uma arquitetura mais robusta:

1. **Motor de Notas:** [SilverBullet](https://silverbullet.md/) rodando em Docker (Porta 15050).
2. **Rede Segura:** Tailscale para acesso remoto sem exposição pública.
3. **Gateway HTTPS:** Caddy Server para gerenciar o tráfego e satisfazer a exigência de HTTPS do iPad.

## 🏗️ Arquitetura de Rede

```
[ iPad / iPhone ] 
      │
      ▼ (HTTPS:443 via Tailscale MagicDNS)
[ Tailscale Serve ] 
      │
      ▼ (Porta 80 Interna)
[ Caddy Server ]
      │
      ▼ (Porta 15050)
[ SilverBullet Docker ]
```

## 🛠️ Implementação Técnica

### 1. Instalação do Caddy (Fedora)
```bash
sudo dnf install -y 'dnf-command(copr)'
sudo dnf copr enable -y @caddy/caddy
sudo dnf install -y caddy
```

### 2. Configuração do Caddyfile
Localização: `/etc/caddy/Caddyfile`
```caddy
http://fedora.taild42ed2.ts.net {
    reverse_proxy localhost:15050
}
```

### 3. Integração com Tailscale
Para habilitar o HTTPS automático sem configurações complexas de certificado:
```bash
sudo tailscale serve --bg http://127.0.0.1:80
```

## 📝 Comandos Úteis

- **Reiniciar Caddy:** `sudo systemctl restart caddy`
- **Verificar Logs:** `sudo journalctl -u caddy -f`
- **Status do Tailscale Serve:** `tailscale serve status`

---
*Documentação gerada automaticamente pelo Gemini CLI em 15 de Fevereiro de 2026.*
