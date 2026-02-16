# Caddy Reverse Proxy - Ecossistema Fedora

Configuracao centralizada do **Caddy Server** como proxy reverso para todos os servicos do ecossistema, com acesso seguro via HTTPS e Tailscale MagicDNS.

## Arquitetura de Rede

```
[ Dispositivo remoto (iPad/PC/etc) ]
      |
      v (HTTPS:443 via Tailscale MagicDNS)
[ Tailscale Serve ] --> https://fedora.taild42ed2.ts.net
      |
      v (HTTP porta 80)
[ Caddy Server (systemd no host) ]
      |
      v (portas locais dos containers)
[ Containers Docker na rede fedora-net ]
```

## Servicos Roteados

| Rota Caddy | Servico | Porta Local | Tipo |
|:---|:---|:---|:---|
| `/n8n/*` | n8n Editor | 15678 | `handle_path` (strip prefix) |
| `/qdrant/*` | Qdrant API + Dashboard | 16333 | `handle_path` (strip prefix) |
| `/dashboard/*` | Qdrant Dashboard assets | 16333 | `handle` (sem strip) |
| `/nocodb/*` | NocoDB | 18080 | `handle_path` (strip prefix) |
| `/supabase/*` | Supabase Kong API Gateway | 54321 | `handle_path` (strip prefix) |
| `/*` (fallback) | SilverBullet | 15050 | `handle` (catch-all) |

### URLs de Acesso

| Servico | URL |
|:---|:---|
| SilverBullet | `https://fedora.taild42ed2.ts.net/` |
| n8n | `https://fedora.taild42ed2.ts.net/n8n/` |
| Qdrant Dashboard | `https://fedora.taild42ed2.ts.net/qdrant/dashboard` |
| Qdrant API | `https://fedora.taild42ed2.ts.net/qdrant/collections` |
| NocoDB | `https://fedora.taild42ed2.ts.net/nocodb/` |
| Supabase REST API | `https://fedora.taild42ed2.ts.net/supabase/rest/v1/` |
| Supabase Studio | `http://localhost:54323` (acesso direto, sem Caddy) |

## Detalhes Tecnicos por Servico

### n8n

**Estrategia:** `handle_path` (strip prefix) + `N8N_PATH=/n8n/`

O n8n possui a variavel `N8N_PATH` que faz o frontend gerar todos os links de assets com o prefixo `/n8n/`. O Caddy usa `handle_path /n8n*` para remover o prefixo antes de encaminhar ao n8n. Assim:

1. Browser acessa `/n8n/` -> Caddy strip para `/` -> n8n serve HTML com links `/n8n/assets/...`
2. Browser pede `/n8n/assets/index.js` -> Caddy strip para `/assets/index.js` -> n8n serve o asset

Variaveis obrigatorias no `docker-compose.yml` do n8n:
```yaml
- N8N_PATH=/n8n/
- N8N_PROXY_HOPS=1
- N8N_SECURE_COOKIE=false
- WEBHOOK_URL=https://fedora.taild42ed2.ts.net/n8n/
```

Headers adicionais no Caddy:
```caddy
flush_interval -1     # necessario para Server-Sent Events
header_up X-Forwarded-Host {host}
```

### Qdrant

**Estrategia:** `handle_path` (strip prefix) + rota `/dashboard*` para assets

O Qdrant nao possui configuracao de base path. O dashboard gera links com paths absolutos (`/dashboard/assets/...`). Duas rotas sao necessarias:

1. `handle_path /qdrant*` -> strip prefix -> proxy para Qdrant (API e dashboard HTML)
2. `handle /dashboard*` -> proxy direto para Qdrant (assets do dashboard com paths absolutos)

Redirect adicional: `/qdrant` e `/qdrant/` redirecionam para `/qdrant/dashboard` (307).

**Nota:** A rota `/dashboard*` e exclusiva do Qdrant. Se outro servico precisar usar `/dashboard` no futuro, havera conflito.

### NocoDB

**Estrategia:** `handle_path` (strip prefix) + redirect na raiz

O NocoDB usa paths **relativos** para assets (`_nuxt/entry.css` em vez de `/_nuxt/entry.css`). Isso faz com que o `handle_path` funcione naturalmente:

1. Browser acessa `/nocodb/dashboard/` -> Caddy strip para `/dashboard/` -> NocoDB serve HTML
2. HTML referencia `_nuxt/entry.css` (relativo) -> browser resolve para `/nocodb/dashboard/_nuxt/entry.css`
3. Caddy strip para `/dashboard/_nuxt/entry.css` -> NocoDB serve o asset

Redirect adicional: `/nocodb` e `/nocodb/` redirecionam para `/nocodb/dashboard/` (307), pois o NocoDB internamente redireciona `/` para `/dashboard` com path absoluto, perdendo o prefixo.

### Supabase

**Estrategia:** `handle_path` (strip prefix) -> Kong API Gateway (porta 54321)

O Kong e o API gateway do Supabase e roteia para os servicos backend:

| Endpoint Kong | Servico Backend |
|:---|:---|
| `/rest/v1/` | PostgREST (API REST do Postgres) |
| `/auth/v1/` | GoTrue (autenticacao) |
| `/storage/v1/` | Storage API |
| `/functions/v1/` | Edge Functions |
| `/realtime/v1/` | Realtime (WebSocket) |
| `/graphql/v1` | GraphQL |
| `/pg/` | pg-meta |
| `/analytics/v1/` | Logflare |

**Supabase Studio** (porta 54323) nao e acessivel via Caddy subpath. E uma aplicacao Next.js que nao suporta configuracao de `basePath` em runtime. Deve ser acessado diretamente na porta 54323.

#### Multi-projeto com Schemas

Uma unica instancia self-hosted do Supabase comporta multiplos projetos logicos usando schemas do Postgres:

```sql
CREATE SCHEMA projeto_a;
CREATE SCHEMA projeto_b;
```

Acesso via API com header `Accept-Profile`:
```bash
curl /supabase/rest/v1/tabela \
  -H "apikey: <ANON_KEY>" \
  -H "Accept-Profile: projeto_a"
```

### SilverBullet

Fallback catch-all. Qualquer rota que nao case com os handlers acima e direcionada ao SilverBullet (porta 15050).

## Instalacao e Configuracao

### 1. Instalar Caddy (Fedora)
```bash
sudo dnf install -y 'dnf-command(copr)'
sudo dnf copr enable -y @caddy/caddy
sudo dnf install -y caddy
```

### 2. Copiar Caddyfile
```bash
sudo cp Caddyfile /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

### 3. Integrar com Tailscale
```bash
sudo tailscale serve --bg http://127.0.0.1:80
```

## Comandos Uteis

| Comando | Descricao |
|:---|:---|
| `sudo systemctl reload caddy` | Recarregar configuracao sem downtime |
| `sudo systemctl restart caddy` | Reiniciar o servico |
| `sudo journalctl -u caddy -f` | Acompanhar logs em tempo real |
| `sudo caddy validate --config /etc/caddy/Caddyfile` | Validar configuracao |
| `tailscale serve status` | Ver status do Tailscale Serve |

## Troubleshooting

### Assets nao carregam (CSS/JS quebrado)
O problema mais comum em subpath reverse proxy. Cada app lida com base path de forma diferente:
- **Paths absolutos** (`/assets/...`): precisam de configuracao de base path no app OU rota extra no Caddy
- **Paths relativos** (`assets/...`): funcionam naturalmente com `handle_path`
- **Variavel de base path** (ex: `N8N_PATH`): ideal quando disponivel

### Redirect perde o prefixo
Quando um app redireciona internamente (ex: `/` -> `/dashboard`), o redirect usa path absoluto e perde o prefixo do Caddy. Solucao: adicionar `redir` no Caddy para a rota raiz do servico.

### Debug
O Caddyfile inclui `debug` no bloco global. Para ver todos os requests roteados:
```bash
sudo journalctl -u caddy -f | grep "upstream"
```
