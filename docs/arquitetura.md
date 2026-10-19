# Diagrama de Arquitetura — Achados & Perdidos

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HOST (Seu Computador)                            │
│                                                                         │
│  Navegador / curl                                                       │
│        │                                                                │
│        │ HTTP                                                           │
│        ↓                                                                │
│   localhost:8080                                                        │
│        │                                                                │
│        │ (Porta publicada)                                             │
└────────┼─────────────────────────────────────────────────────────────┘
         │
         │ Docker Engine mapeia 8080 → 5000
         │
┌────────┼─────────────────────────────────────────────────────────────┐
│        ↓                DOCKER                                         │
│ ┌──────────────────────────────────────────────────────────────────┐  │
│ │              rede_interna (Bridge com DNS)                       │  │
│ │                                                                  │  │
│ │  ┌─────────────────────────┐      ┌─────────────────────────┐  │  │
│ │  │  Contêiner: achados-web │      │  Contêiner: achados-db  │  │  │
│ │  │  ─────────────────────  │      │  ─────────────────────  │  │  │
│ │  │  Imagem: app/Dockerfile │      │  Imagem: postgres:16-   │  │  │
│ │  │  (construída localmente)│      │         alpine          │  │  │
│ │  │                         │      │                         │  │  │
│ │  │  Gunicorn :5000         │──SQL─▶│  PostgreSQL :5432      │  │  │
│ │  │  Flask app              │ ◀────│  (porta NÃO publicada) │  │  │
│ │  │  Usuário: appuser       │      │                         │  │  │
│ │  │  (sem privilégios)      │      │  Healthcheck:           │  │  │
│ │  │                         │      │  pg_isready ✓           │  │  │
│ │  │  Variáveis:             │      │                         │  │  │
│ │  │  - DB_HOST=db           │      │  Variáveis (do .env):   │  │  │
│ │  │  - DB_PORT=5432         │      │  - POSTGRES_DB          │  │  │
│ │  │  - DB_NAME=${.env}      │      │  - POSTGRES_USER        │  │  │
│ │  │  - DB_USER=${.env}      │      │  - POSTGRES_PASSWORD    │  │  │
│ │  │  - DB_PASSWORD=${.env}  │      │                         │  │  │
│ │  │  - UPLOAD_DIR=/app/     │      │                         │  │  │
│ │  │              uploads    │      │                         │  │  │
│ │  │                         │      │                         │  │  │
│ │  │  Monta volumes:         │      │  Monta volumes:         │  │  │
│ │  │  ┌──────────────┐       │      │  ┌──────────────┐      │  │  │
│ │  │  │ fotos        │       │      │  │ dados_banco  │      │  │  │
│ │  │  └──────┬───────┘       │      │  └──────┬───────┘      │  │  │
│ │  └─────────┼───────────────┘      └─────────┼─────────────┘  │  │
│ │            │                                 │                 │  │
│ │  Bind mount (somente leitura):              │                 │  │
│ │  ./db/init.sql ──────────────────────────┐  │                 │  │
│ │     ↓                                     ↓  ↓                 │  │
│ │  /docker-entrypoint-initdb.d/init.sql:ro    │                 │  │
│ │  (executado apenas na 1ª inicialização)     │                 │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────┐           ┌───────────────────────────┐  │
│  │ Volume: fotos      │           │ Volume: dados_banco       │  │
│  │ ──────────────     │           │ ─────────────────         │  │
│  │ Tipo: nomeado      │           │ Tipo: nomeado             │  │
│  │ Montado em:        │           │ Montado em:               │  │
│  │ /app/uploads       │           │ /var/lib/postgresql/data  │  │
│  │                    │           │                           │  │
│  │ Conteúdo:          │           │ Conteúdo:                 │  │
│  │ - a3f8e9.jpg       │           │ - Tabela: itens           │  │
│  │ - f1a2b3.jpg       │           │ - Índices                 │  │
│  │ - ...              │           │ - WAL logs                │  │
│  │                    │           │                           │  │
│  │ Persiste após:     │           │ Persiste após:            │  │
│  │ docker compose down│           │ docker compose down       │  │
│  │ ✓                  │           │ ✓                         │  │
│  └────────────────────┘           └───────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ DESAFIO BÔNUS: Contêiner adminer (profile: debug)           ││
│  │ ──────────────────────────────────────────────────────       ││
│  │ Imagem: adminer:4.8.1                                        ││
│  │ Porta: localhost:8081 → 8080                                 ││
│  │ Sobe apenas com: docker compose --profile debug up -d        ││
│  │                                                              ││
│  │ [adminer :8080] ──────────▶ [db :5432]                      ││
│  └──────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                     FILESYSTEM DO HOST                                  │
│                                                                         │
│  ./db/init.sql ────┐ (bind mount, read-only)                           │
│                    │                                                    │
│  .env ─────────────┼──▶ Credenciais (NÃO versionado no Git)            │
│                    │   - POSTGRES_DB=achados_perdidos                   │
│                    │   - POSTGRES_USER=achados_user                     │
│                    │   - POSTGRES_PASSWORD=SenhaSegura123!              │
│                    │                                                    │
│  Volumes Docker: ──┘                                                   │
│  Windows: \\wsl$\docker-desktop-data\data\docker\volumes\              │
│  Linux:   /var/lib/docker/volumes/                                     │
│    ├─ achados-perdidos_fotos/_data/                                    │
│    └─ achados-perdidos_dados_banco/_data/                              │
└─────────────────────────────────────────────────────────────────────────┘

════════════════════════════════════════════════════════════════════════════
FLUXO DE UMA REQUISIÇÃO: POST /itens (com foto)
════════════════════════════════════════════════════════════════════════════

1. curl -F "foto=@oculos.jpg" http://localhost:8080/itens
   │
2. Host (porta 8080) ──▶ Docker mapeia para achados-web:5000
   │
3. Gunicorn recebe → Flask processa → Gera UUID: "a3f8e9.jpg"
   │
4. foto.save("/app/uploads/a3f8e9.jpg") ──▶ Volume: fotos
   │
5. DNS interno resolve "db" → IP 172.18.0.2
   │
6. psycopg.connect(host="db", port=5432) ──▶ PostgreSQL
   │
7. INSERT INTO itens (...) VALUES (...) ──▶ Volume: dados_banco
   │
8. Retorna: {"id": 4, "foto": "a3f8e9.jpg"}
   │
9. Cliente acessa: http://localhost:8080/fotos/a3f8e9.jpg
   │
10. send_from_directory("/app/uploads", "a3f8e9.jpg") ──▶ Volume: fotos

════════════════════════════════════════════════════════════════════════════
ISOLAMENTO DE SEGURANÇA
════════════════════════════════════════════════════════════════════════════

✅ ACESSÍVEL DE FORA:
   - localhost:8080 → web (porta publicada)
   - localhost:8081 → adminer (apenas com --profile debug)

❌ ISOLADO (só dentro da rede_interna):
   - db:5432 (PostgreSQL)
   - Comunicação entre web ↔ db

🔒 PROTEÇÕES:
   - Banco sem porta publicada (não acessível do host/rede local)
   - Usuário appuser sem privilégios de root no contêiner web
   - Senhas em .env (não versionado no Git)
   - init.sql montado como read-only (:ro)

════════════════════════════════════════════════════════════════════════════
PERSISTÊNCIA DE DADOS
════════════════════════════════════════════════════════════════════════════

docker compose down       → Volumes MANTIDOS ✅
docker compose up -d      → Dados voltam intactos ✅
docker compose down -v    → Volumes REMOVIDOS ❌ (CUIDADO!)

════════════════════════════════════════════════════════════════════════════
ORDEM DE INICIALIZAÇÃO (depends_on + healthcheck)
════════════════════════════════════════════════════════════════════════════

1. db inicia
2. PostgreSQL carrega (5-10 segundos)
3. Healthcheck: pg_isready testa a cada 5s
4. Quando retorna "healthy" ✓
5. web inicia (garante conexão bem-sucedida)

════════════════════════════════════════════════════════════════════════════
```

## Legenda:

- **┌─┐**: Contêiner ou componente isolado
- **→**: Comunicação/fluxo de dados
- **:5000, :5432**: Portas internas dos serviços
- **localhost:8080**: Porta publicada no host
- **${.env}**: Valor lido do arquivo `.env`
- **✓**: Funcionalidade ativa/habilitada
- **❌**: Bloqueado/isolado

## Notas importantes:

1. **DNS interno**: O nome `db` é resolvido automaticamente pelo Docker na rede `rede_interna`
2. **Volumes nomeados**: Gerenciados pelo Docker, persistem entre recriações de contêineres
3. **Bind mount**: Mapeia arquivo local (`./db/init.sql`) diretamente para o contêiner
4. **Healthcheck**: Garante que o banco aceita conexões antes do web iniciar
5. **Isolamento**: Apenas `web` tem porta publicada; `db` é acessível apenas dentro do Docker
