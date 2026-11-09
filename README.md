# Media server caseiro

Servidor de mídia pessoal no Windows: **Jellyfin** + **qBittorrent** em **Docker** (Docker Desktop + WSL2), com autocura e acesso por qualquer aparelho da casa.
Organiza a minha coleção de filmes, séries e músicas com capas, sinopses e legendas em português, e toca na TV, no celular e no navegador.

> Inspirado em [patrickdeangelis/mediaserver-setup](https://github.com/patrickdeangelis/mediaserver-setup), adaptado para Windows e simplificado:
> sem indexadores automáticos, com download manual apenas de conteúdo liberado (domínio público, Creative Commons, software livre).

## Arquitetura

```
                                Docker Desktop (WSL2)
                    ┌─────────────────────────────────────────────┐
 .torrent/magnet ──▶│ qbittorrent :8085                           │
 (conteúdo livre)   │  baixa em downloads\incompletos             │
                    │  ao terminar move para media\<categoria> ───┼──┐
                    └─────────────────────────────────────────────┘  │
                                                                     ▼
                                                     D:\mediaserver\media\{filmes,series,musica}
                                                                     │ varredura a cada 15 min
                    ┌────────────────────────────────────────────────▼┐   :8096   ┌──────────────┐
                    │ jellyfin                                        │ ────────▶ │ Navegador    │
                    │  biblioteca + metadados pt-BR + legendas + player│           │ TV / Celular │
                    └────────────────────────────────────────────────▲┘           └──────────────┘
                    ┌────────────────────────────────────────────────┴┐
                    │ autoheal: reinicia quem ficar "unhealthy"        │
                    └─────────────────────────────────────────────────┘
```

| Serviço | Imagem | Porta | Pra quê |
|---|---|---|---|
| Jellyfin | `lscr.io/linuxserver/jellyfin` | 8096 (web), 7359/udp (descoberta) | Biblioteca, metadados, legendas e player |
| qBittorrent | `lscr.io/linuxserver/qbittorrent` | 8085 (web), 6881 tcp/udp (peers) | Downloads de conteúdo liberado |
| Autoheal | `willfarrell/autoheal` | — | Reinicia containers que falham no healthcheck |

## Endereços

| App | No próprio PC | Outros aparelhos da rede |
|---|---|---|
| Jellyfin | http://localhost:8096 | `http://<IP do PC>:8096` ou app **Jellyfin** (Android, iOS, Android TV, Fire TV, LG, Samsung) |
| qBittorrent | http://localhost:8085 | `http://<IP do PC>:8085` |

## Bibliotecas

| Biblioteca | Pasta no Windows | Pasta no container | Metadados |
|---|---|---|---|
| Filmes | `D:\mediaserver\media\filmes` | `/data/media/filmes` | pt-BR / Brasil |
| Séries | `D:\mediaserver\media\series` | `/data/media/series` | pt-BR / Brasil |
| Músicas | `D:\mediaserver\media\musica` | `/data/media/musica` | pt-BR / Brasil |

Arquivos novos aparecem em até **15 minutos** (varredura agendada). Para ver na hora: **Painel → Bibliotecas → Escanear todas as bibliotecas**.

> Por que não em segundos? O monitoramento em tempo real do Jellyfin usa *inotify* (Linux), e pastas do Windows montadas no Docker Desktop
> não repassam esses eventos para o container. Por isso a tarefa *Escanear biblioteca* foi trocada de 12 h para 15 min,
> e *Baixar legendas ausentes* de 24 h para 1 h.

Nomes que o Jellyfin reconhece:

```
filmes\Nome do Filme (2010)\Nome do Filme (2010).mkv
filmes\Nome do Filme (2010)\Nome do Filme (2010).pt-BR.srt     legenda carregada automaticamente
series\Nome da Série\Season 01\Nome da Série S01E01.mkv
```

Se um download chegar com nome bagunçado, corrija no Jellyfin em **⋯ → Identificar**.

## Downloads (qBittorrent)

1. Abra http://localhost:8085 e adicione o `.torrent` ou link magnet.
2. Escolha a **categoria** `filmes`, `series` ou `musica`.
3. Enquanto baixa, o arquivo fica em `downloads\incompletos` (o Jellyfin não vê arquivo pela metade).
4. Ao terminar, o qBittorrent move para `media\<categoria>` e o Jellyfin adiciona sozinho na próxima varredura (até 15 min).

Fontes de conteúdo liberado: [Internet Archive](https://archive.org/details/feature_films) (filmes em domínio público, com opção de torrent),
[Blender Studio](https://studio.blender.org/films/) (Big Buck Bunny, Sintel, Tears of Steel, Spring) e distribuições Linux.

> O BitTorrent **envia** pedaços para outras pessoas enquanto baixa, e o IP fica visível no swarm.
> Baixar ou compartilhar obras protegidas sem autorização viola a Lei de Direitos Autorais (Lei 9.610/98).

## Legendas

Plugin **Open Subtitles** do Jellyfin (catálogo oficial), com conta grátis em [opensubtitles.com](https://www.opensubtitles.com):

- Bibliotecas Filmes e Séries buscam legendas em **português** e **inglês** automaticamente ao adicionar um vídeo (e na tarefa agendada *Baixar legendas ausentes*).
- Pula a busca se o vídeo já tem legenda embutida ou áudio no idioma.
- A legenda é salva ao lado do vídeo (`Filme (2010).pt.srt`), então sobrevive a reinstalar o servidor.
- Busca manual: no item, **⋯ → Editar legendas → Pesquisar**.

## Estrutura

```
D:\mediaserver\
├─ docker-compose.yml        definição dos serviços
├─ README.md
├─ LEIA-ME.txt               guia rápido
├─ config\qbittorrent\       configuração do qBittorrent (pastas e categorias)
├─ downloads\incompletos\    downloads em andamento
└─ media\{filmes,series,musica}
```

A configuração do Jellyfin (banco SQLite, usuários, metadados) fica no volume Docker `jellyfin-config`, e não numa pasta do Windows:
o SQLite sobre NTFS montado no Docker é lento e sujeito a erros de lock, enquanto o volume roda no sistema de arquivos Linux do WSL2.

## Persistência e autocura

- Todo o estado fica fora dos containers (volume `jellyfin-config`, `config\` e `media\`). Apagar ou recriar os containers não perde nada.
- `restart: unless-stopped` religa os serviços quando o Docker Desktop abre (com *Start Docker Desktop when you sign in* ligado).
- Healthcheck em todos os serviços a cada 30s. Se falhar 3 vezes, o `autoheal` reinicia o container.

## Comandos

```powershell
cd D:\mediaserver
docker compose up -d                          # liga
docker compose down                           # desliga
docker compose pull; docker compose up -d     # atualiza as imagens
docker ps                                     # status e saúde dos containers
docker logs -f jellyfin                       # logs
```

Backup da configuração do Jellyfin:

```powershell
docker run --rm -v mediaserver_jellyfin-config:/config -v ${PWD}:/backup alpine tar czf /backup/jellyfin-config.tgz -C /config .
```

## Instalação do zero

Requisitos: Windows 10/11 com WSL2 e Docker Desktop.

```powershell
winget install -e --id Docker.DockerDesktop
# abra o Docker Desktop uma vez e aceite os termos
git clone <este repositório> D:\mediaserver
cd D:\mediaserver
docker compose up -d
```

Depois:

1. **Jellyfin** (http://localhost:8096): assistente inicial em pt-BR, usuário administrador **com senha**, bibliotecas em
   `/data/media/filmes`, `/data/media/series` e `/data/media/musica` com metadados em Português (Brasil).
   Em Plugins, instale **Open Subtitles** e entre com a conta do opensubtitles.com.
2. **qBittorrent** (http://localhost:8085): no primeiro start ele gera uma senha temporária para o usuário `admin`
   (veja com `docker logs qbittorrent`). Entre e troque em **Ferramentas → Opções → Interface Web**.
   As categorias já vêm em `config\qbittorrent\qBittorrent\categories.json`. Em **Opções → Downloads**, deixe o caminho padrão
   `/data/media/filmes` e ligue **Manter torrents incompletos em** `/data/downloads/incompletos`.

## Segurança

- Diferente do projeto original, Jellyfin e qBittorrent têm **usuário e senha**.
- Só fica exposto na rede local. Para outros aparelhos, libere as portas no Firewall do Windows apenas em redes **privadas**.
- Não há porta aberta para a internet. Para acesso fora de casa, prefira uma VPN (ex.: Tailscale) a abrir porta no roteador.
- Sem indexadores automáticos: só baixa o que eu adiciono manualmente, de fontes legais.

## O que aprendi

- Docker Compose com healthchecks e reinício automático (`autoheal`).
- Diferenças entre bind mount (NTFS) e volume nomeado (ext4 no WSL2) para bancos SQLite.
- Configuração do Jellyfin pela API REST (bibliotecas, idioma e país dos metadados, plugin de legendas).
- Fluxo de download → biblioteca: pasta temporária e categorias.
- Por que o *inotify* não atravessa bind mounts do Windows no Docker Desktop, e como contornar com tarefas agendadas.
- Adaptar um projeto feito para macOS (OrbStack, launchd, Bonjour) para Windows.
