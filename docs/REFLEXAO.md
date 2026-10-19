# 🤔 Reflexão e Equivalência em Nuvem

Este documento responde às questões de reflexão propostas e mapeia cada componente do ambiente local para serviços gerenciados em nuvem.

---

## Índice

- [Q1: Resolução de nomes (DNS interno)](#q1-resolução-de-nomes-dns-interno)
- [Q2: Volume nomeado vs. bind mount](#q2-volume-nomeado-vs-bind-mount)
- [Q3: Alterações no init.sql](#q3-alterações-no-initsql)
- [Q4: depends_on e healthcheck](#q4-depends_on-e-healthcheck)
- [Q5: Cache de camadas do Docker](#q5-cache-de-camadas-do-docker)
- [Q6: Segurança e LGPD](#q6-segurança-e-lgpd)
- [Q7: Escalonamento horizontal](#q7-escalonamento-horizontal)
- [Equivalência na nuvem](#equivalência-na-nuvem)

---

## Q1: Resolução de nomes (DNS interno)

### Pergunta
> Por que o serviço web consegue se conectar ao banco usando o nome `db` em vez de um endereço IP? Quem resolve esse nome e o que mudaria se os contêineres fossem criados com `docker run` na rede bridge padrão?

### Resposta

O serviço `web` consegue se conectar ao banco usando o nome `db` porque o **Docker cria um servidor DNS interno** para redes bridge definidas pelo usuário (como nossa `rede_interna`).

**Como funciona:**

1. **No `app.py`, a conexão usa:**
```python
DB_URL = f"host={os.getenv('DB_HOST', 'db')} ..."
# host=db (não um IP!)
```

2. **Quando `web` tenta conectar:**
```
web → resolve "db" → DNS interno do Docker → retorna 172.18.0.2 → conexão estabelecida
```

3. **Quem resolve:**
   - O **Docker Engine** mantém um servidor DNS embutido (`127.0.0.11:53`)
   - Cada contêiner na rede recebe `/etc/resolv.conf` apontando para ele
   - O DNS mapeia **nome do serviço → IP privado do contêiner**

4. **Vantagens:**
   - IPs internos mudam a cada `docker compose up` (ex: `172.18.0.2` vira `172.18.0.5`)
   - Nomes são **estáveis e legíveis** (`db` sempre resolve corretamente)
   - Desacoplamento: não precisa saber o IP antecipadamente

**Verificação no ambiente:**
```bash
# Dentro do contêiner web:
docker compose exec web cat /etc/resolv.conf
# nameserver 127.0.0.11  ← DNS interno

docker compose exec web ping -c 1 db
# PING db (172.18.0.2) 56(84) bytes of data.
```

---

### O que mudaria com `docker run` na rede bridge padrão?

**❌ Rede bridge padrão (criada automaticamente pelo Docker):**

```bash
docker network create bridge  # Já existe por padrão
docker run --name meu-db --network bridge postgres:16-alpine
docker run --name meu-web --network bridge minha-app
```

**Problemas:**

1. **Sem DNS automático**
   - `ping db` → `ping: db: Name or service not known`
   - O Docker **não** registra nomes de contêineres no DNS da rede padrão

2. **Soluções ruins:**
   - Usar IP fixo: `DB_HOST=172.17.0.2` (quebra se o IP mudar)
   - Usar `--link` (deprecated): `docker run --link meu-db:db ...`
   - Descobrir o IP manualmente: `docker inspect meu-db | grep IPAddress`

3. **Isolamento zero:**
   - Todos os contêineres do sistema ficam na mesma rede
   - Projeto A pode acessar banco do Projeto B (risco de segurança)

---

**✅ Rede definida pelo usuário (nosso caso):**

```yaml
networks:
  rede_interna:
    driver: bridge
```

**Vantagens:**

1. **DNS automático habilitado**
   - `db` resolve automaticamente
   - `web` resolve automaticamente
   - Aliases funcionam: `aliases: [banco-principal]`

2. **Isolamento por projeto:**
   - Apenas serviços no `docker-compose.yml` compartilham essa rede
   - Outros projetos não conseguem acessar

3. **Descoberta de serviços nativa:**
   - Não precisa hardcoded de IPs
   - Funciona em qualquer ambiente (dev, staging, prod)

---

**Comparação:**

| Característica | Rede padrão (`bridge`) | Rede definida (`rede_interna`) |
|----------------|------------------------|----------------------------------|
| DNS interno | ❌ Não | ✅ Sim |
| Resolução de nomes | ❌ Apenas por IP ou `--link` | ✅ Automática pelo nome do serviço |
| Isolamento | ❌ Todos os contêineres | ✅ Apenas serviços do mesmo Compose |
| Uso recomendado | ❌ Legado | ✅ Produção |

---

**Conclusão:**

O DNS interno do Docker em redes definidas pelo usuário é essencial para aplicações multi-contêiner. Sem ele, seria necessário hardcoded de IPs ou ferramentas externas de service discovery (como Consul ou Etcd), aumentando a complexidade desnecessariamente.

---

## Q2: Volume nomeado vs. bind mount

### Pergunta
> Qual a diferença entre um volume nomeado e um bind mount? Justifique por que o `init.sql` foi montado como bind mount e os dados do banco como volume nomeado.

### Resposta

**Volume nomeado** e **bind mount** são duas formas de persistir dados, mas com propósitos diferentes.

---

### 📦 Volume nomeado

**Definição:**
- Gerenciado inteiramente pelo Docker
- Armazenado em local específico do Docker no host
- Criado automaticamente se não existir

**Exemplo no projeto:**
```yaml
volumes:
  - dados_banco:/var/lib/postgresql/data
  - fotos:/app/uploads
```

**Características:**

| Aspecto | Detalhe |
|---------|---------|
| **Localização** | `/var/lib/docker/volumes/achados-perdidos_fotos/_data` (Linux)<br>`\\wsl$\docker-desktop-data\...` (Windows) |
| **Criação** | Automática pelo Docker Compose |
| **Portabilidade** | Alto (funciona em qualquer SO) |
| **Backup** | `docker run --rm -v fotos:/source -v $(pwd):/backup alpine tar czf /backup/fotos.tar.gz -C /source .` |
| **Inspeção** | `docker volume inspect achados-perdidos_fotos` |
| **Ownership** | Gerenciado pelo Docker |
| **Versionamento** | ❌ Não vai para o Git |

**Vantagens:**
- ✅ Docker gerencia ciclo de vida
- ✅ Funciona igual em Windows, Mac, Linux
- ✅ Melhor performance (otimizado pelo Docker)
- ✅ Isolado do filesystem do host (segurança)
- ✅ Fácil de mover entre ambientes

**Desvantagens:**
- ❌ Localização "escondida" (não é óbvio onde está)
- ❌ Acesso direto do host requer comandos Docker

---

### 📁 Bind mount

**Definição:**
- Mapeia um arquivo/diretório do host diretamente para o contêiner
- Caminho absoluto ou relativo do host

**Exemplo no projeto:**
```yaml
volumes:
  - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
```

**Características:**

| Aspecto | Detalhe |
|---------|---------|
| **Localização** | `./db/init.sql` (relativo ao `docker-compose.yml`) |
| **Criação** | Manual (arquivo deve existir antes) |
| **Portabilidade** | Médio (depende da estrutura de pastas) |
| **Backup** | Git/sistema de arquivos normal |
| **Inspeção** | Abrir com editor de texto |
| **Ownership** | Arquivo local (seu usuário) |
| **Versionamento** | ✅ Vai para o Git |

**Vantagens:**
- ✅ Fácil de editar (qualquer editor)
- ✅ Versionável no Git
- ✅ Mudanças refletidas instantaneamente (ideal para desenvolvimento)
- ✅ Localização óbvia e acessível

**Desvantagens:**
- ❌ Dependente da estrutura de pastas do host
- ❌ Problemas de permissão podem ocorrer (Windows ↔ Linux)
- ❌ Performance pode ser menor (especialmente Windows/Mac)

---

### 🎯 Por que `init.sql` é bind mount?

**Decisão:** `./db/init.sql` → bind mount (`:ro` = read-only)

**Justificativas:**

1. **É um arquivo de configuração, não dados de aplicação**
   - Define a estrutura do banco (schema)
   - Insere dados de exemplo
   - Deve estar versionado no Git

2. **Precisa ser editável facilmente**
   - Desenvolvedor pode adicionar novas colunas
   - Mudar dados de exemplo
   - Sem precisar rebuildar a imagem ou recriar volumes

3. **Executa apenas uma vez na inicialização**
   - PostgreSQL só processa scripts em `/docker-entrypoint-initdb.d/` quando o banco é criado
   - Não é escrito, apenas lido (`:ro`)

4. **Portabilidade do projeto**
   - Qualquer pessoa que clonar o repositório tem o `init.sql`
   - Não precisa criar ou popular o arquivo manualmente

5. **Comparação: se fosse volume nomeado**
   ```yaml
   volumes:
     - init_scripts:/docker-entrypoint-initdb.d/
   ```
   ❌ Como copiar o `init.sql` para dentro do volume?
   ❌ Como versionar no Git?
   ❌ Como editar facilmente?

---

### 🎯 Por que `dados_banco` é volume nomeado?

**Decisão:** `dados_banco:/var/lib/postgresql/data` → volume nomeado

**Justificativas:**

1. **São dados de aplicação, não configuração**
   - Tabelas com registros reais
   - Inseridos pelos usuários durante uso
   - Mudam constantemente

2. **Não devem estar no Git**
   - Dados sensíveis (podem ter fotos de documentos pessoais)
   - Muito grandes (podem chegar a GB)
   - Específicos de cada ambiente (dev vs. prod)

3. **Gerenciados pelo PostgreSQL, não por humanos**
   - Arquivos binários (não editáveis em editor de texto)
   - Estrutura interna complexa (WAL logs, indexes, etc.)
   - Edição manual corrompe o banco

4. **Precisam persistir entre reinicializações**
   - `docker compose down` + `docker compose up` → dados voltam
   - Proteção contra perda acidental

5. **Performance otimizada pelo Docker**
   - Volume nomeado tem melhor I/O (especialmente Windows/Mac)
   - PostgreSQL faz muitas operações de leitura/escrita

---

### 📊 Comparação aplicada ao projeto:

| Componente | Tipo | Por quê? |
|------------|------|----------|
| `init.sql` | Bind mount (`:ro`) | Configuração versionada, editável, read-only |
| `dados_banco` | Volume nomeado | Dados de runtime, binários, gerenciados pelo PostgreSQL |
| `fotos` | Volume nomeado | Dados de runtime, uploads de usuários, não versionáveis |

---

### 💡 Quando usar cada tipo:

**Use volume nomeado quando:**
- ✅ Dados de aplicação (banco de dados, uploads, cache)
- ✅ Precisa persistir entre containers
- ✅ Não deve ser editado manualmente
- ✅ Performance é crítica

**Use bind mount quando:**
- ✅ Arquivos de configuração
- ✅ Código em desenvolvimento (live reload)
- ✅ Deve estar versionado no Git
- ✅ Precisa editar frequentemente

---

**Conclusão:**

A escolha entre volume nomeado e bind mount depende da **natureza do dado**: configuração/código vai para bind mount (versionável, editável), enquanto dados de aplicação vão para volume nomeado (gerenciado, persistente, otimizado). No nosso projeto, essa distinção garante que `init.sql` esteja no Git e seja facilmente editável, enquanto os dados do PostgreSQL permanecem seguros em um volume gerenciado pelo Docker.

---

## Q3: Alterações no init.sql

### Pergunta
> Se você alterar o `init.sql` (por exemplo, adicionando uma coluna) depois que o ambiente já subiu uma vez e executar `docker compose up -d` de novo, a mudança é aplicada? Por quê? Como aplicar a alteração sem perder dados?

### Resposta

**❌ Não, a mudança NÃO é aplicada automaticamente.**

---

### Por que não aplica?

O PostgreSQL executa scripts em `/docker-entrypoint-initdb.d/` **apenas uma vez**, quando o banco é criado pela primeira vez.

**Lógica interna da imagem `postgres:16-alpine`:**

```bash
#!/bin/bash
# Script de inicialização do PostgreSQL (simplificado)

if [ ! -f "$PGDATA/PG_VERSION" ]; then
  echo "Banco não existe, inicializando..."
  initdb ...
  
  # Executar scripts de inicialização
  for f in /docker-entrypoint-initdb.d/*; do
    psql < "$f"
  done
else
  echo "Banco já existe, pulando inicialização."
fi

pg_ctl start
```

**Verificação:**
```bash
# Arquivo que indica que o banco foi inicializado:
docker compose exec db ls -l /var/lib/postgresql/data/PG_VERSION
# -rw------- 1 postgres postgres 3 Oct 19 01:45 PG_VERSION
```

Se `PG_VERSION` existe, o PostgreSQL **pula a inicialização**.

---

### Teste prático:

**1. Alterar `init.sql` (adicionar coluna):**

```sql
-- db/init.sql (nova versão)
CREATE TABLE IF NOT EXISTS itens (
    id SERIAL PRIMARY KEY,
    descricao VARCHAR(200) NOT NULL,
    local VARCHAR(100) NOT NULL,
    foto VARCHAR(100),
    categoria VARCHAR(50),  -- ← NOVA COLUNA
    criado_em TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**2. Executar `docker compose up -d`:**
```bash
docker compose up -d
# achados-db    Running  (nada acontece, contêiner já existe)
```

**3. Verificar estrutura da tabela:**
```bash
docker compose exec db psql -U achados_user -d achados_perdidos -c "\d itens"
```

**Resultado:**
```
                      Table "public.itens"
  Column   |          Type          | Collation | Nullable | Default
-----------+------------------------+-----------+----------+---------
 id        | integer                |           | not null | ...
 descricao | character varying(200) |           | not null |
 local     | character varying(100) |           | not null |
 foto      | character varying(100) |           |          |
                         ← SEM a coluna "categoria"!
```

**Por quê?** O volume `dados_banco` **já existe** com a estrutura antiga. O PostgreSQL não reinicializa.

---

### Como aplicar a alteração SEM perder dados?

Existem **4 abordagens**, cada uma com trade-offs:

---

#### ✅ Abordagem 1: Migration manual (ALTER TABLE)

**Como fazer:**

```bash
# 1. Conectar no banco
docker compose exec db psql -U achados_user -d achados_perdidos

# 2. Executar ALTER TABLE
ALTER TABLE itens ADD COLUMN categoria VARCHAR(50);

# 3. Verificar
\d itens
```

**Vantagens:**
- ✅ Mantém todos os dados existentes
- ✅ Controle total sobre a migration
- ✅ Rápido (não derruba contêineres)

**Desvantagens:**
- ❌ Manual (não automatizado)
- ❌ Não fica versionado (precisa documentar)
- ❌ Em produção, precisa de ferramenta de migration (Flyway, Liquibase, Alembic)

**Uso:** Desenvolvimento, ajustes pontuais

---

#### ✅ Abordagem 2: Backup → Destruir → Recriar

**Como fazer:**

```bash
# 1. Backup dos dados (pg_dump)
docker compose exec db pg_dump -U achados_user achados_perdidos > backup.sql

# 2. Alterar init.sql (adicionar coluna)
# ... editar db/init.sql ...

# 3. Destruir volumes e recriar
docker compose down -v
docker compose up -d

# 4. Restaurar dados (exceto estrutura)
docker compose exec -T db psql -U achados_user achados_perdidos < backup.sql
```

**Vantagens:**
- ✅ Ambiente limpo, estrutura nova do zero
- ✅ Testa o `init.sql` completo

**Desvantagens:**
- ❌ Downtime (precisa derrubar o ambiente)
- ❌ Risco de perder dados se o backup falhar
- ❌ Complexo para ambientes de produção

**Uso:** Resetar ambiente de desenvolvimento, grandes refatorações

---

#### ✅ Abordagem 3: Ferramenta de migrations (produção)

**Ferramentas:**
- **Flyway** (Java/SQL)
- **Liquibase** (XML/SQL)
- **Alembic** (Python/SQLAlchemy)
- **Django Migrations** (Python/Django)
- **Rails Migrations** (Ruby)

**Exemplo com Flyway:**

**Estrutura:**
```
db/
  migrations/
    V1__initial_schema.sql        # init.sql original
    V2__add_categoria_column.sql  # Nova migration
```

**`V2__add_categoria_column.sql`:**
```sql
ALTER TABLE itens ADD COLUMN categoria VARCHAR(50);
```

**docker-compose.yml:**
```yaml
services:
  db:
    # ... configuração existente ...
  
  flyway:
    image: flyway/flyway:9
    command: migrate
    volumes:
      - ./db/migrations:/flyway/sql
    environment:
      FLYWAY_URL: jdbc:postgresql://db:5432/achados_perdidos
      FLYWAY_USER: ${POSTGRES_USER}
      FLYWAY_PASSWORD: ${POSTGRES_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
```

**Executar:**
```bash
docker compose up flyway
# Aplica todas as migrations pendentes automaticamente
```

**Vantagens:**
- ✅ Versionamento de schema (Git)
- ✅ Automático (CI/CD)
- ✅ Rollback seguro
- ✅ Histórico de alterações (tabela `flyway_schema_history`)
- ✅ Produção-ready

**Desvantagens:**
- ❌ Complexidade inicial (setup)
- ❌ Curva de aprendizado

**Uso:** Produção, times grandes, CI/CD

---

#### ✅ Abordagem 4: Script de migration customizado

**Criar `db/migrations/001_add_categoria.sql`:**

```sql
-- Migration: Adicionar coluna categoria
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns 
        WHERE table_name = 'itens' AND column_name = 'categoria'
    ) THEN
        ALTER TABLE itens ADD COLUMN categoria VARCHAR(50);
        RAISE NOTICE 'Coluna categoria adicionada';
    ELSE
        RAISE NOTICE 'Coluna categoria já existe';
    END IF;
END $$;
```

**Executar:**
```bash
docker compose exec -T db psql -U achados_user achados_perdidos < db/migrations/001_add_categoria.sql
```

**Vantagens:**
- ✅ Idempotente (pode rodar múltiplas vezes sem erro)
- ✅ Simples (sem ferramentas externas)
- ✅ Versionável

**Desvantagens:**
- ❌ Manual (precisa executar o comando)
- ❌ Sem histórico automático

**Uso:** Projetos pequenos, migrations pontuais

---

### 📊 Comparação das abordagens:

| Abordagem | Mantém dados? | Automática? | Complexidade | Uso recomendado |
|-----------|---------------|-------------|--------------|------------------|
| ALTER TABLE manual | ✅ Sim | ❌ Não | Baixa | Dev |
| Backup → Recriar | ✅ Sim | ❌ Não | Média | Reset dev |
| Flyway/Liquibase | ✅ Sim | ✅ Sim | Alta | Produção |
| Script customizado | ✅ Sim | ⚠️ Parcial | Média | Projetos pequenos |

---

### 🎯 Recomendação para este projeto:

**Desenvolvimento:**
```bash
# Opção 1: ALTER TABLE manual (rápido)
docker compose exec db psql -U achados_user achados_perdidos -c \
  "ALTER TABLE itens ADD COLUMN categoria VARCHAR(50);"
```

**Produção (futura migração para nuvem):**
- Implementar Flyway ou Alembic
- Versionamento de migrations no Git
- Pipeline CI/CD executa automaticamente

---

**Conclusão:**

O `init.sql` só roda na primeira inicialização porque o PostgreSQL detecta se o diretório de dados já existe (`PG_VERSION`). Para aplicar alterações sem perder dados, a melhor abordagem depende do contexto: ALTER TABLE manual para desenvolvimento rápido, ferramentas de migration (Flyway, Alembic) para produção com versionamento e automação, ou backup/restore para grandes refatorações.

---

## Q4: depends_on e healthcheck

### Pergunta
> Por que `depends_on` sozinho, sem `condition: service_healthy`, não garante que a aplicação consiga se conectar ao banco na primeira tentativa?

### Resposta

**`depends_on` sozinho garante apenas a ORDEM DE INICIALIZAÇÃO, não a DISPONIBILIDADE do serviço.**

---

### Demonstração do problema:

**Cenário 1: Apenas `depends_on` (sem condition)**

```yaml
services:
  db:
    image: postgres:16-alpine
    # Sem healthcheck
  
  web:
    build: ./app
    depends_on:
      - db  # ← Apenas ordem, sem condition
```

**O que acontece:**

```
Timeline:
T=0s   Docker inicia contêiner db
T=1s   PostgreSQL processo inicia
T=2s   Docker vê que processo não crashou → considera "started"
T=2s   Docker inicia contêiner web  ← AGORA!
T=3s   web tenta conectar ao banco
T=3s   ❌ ERRO: "connection refused" (PostgreSQL ainda inicializando)
T=5s   PostgreSQL finalmente pronto para aceitar conexões
```

**Por quê?**

O PostgreSQL passa por várias etapas após o processo iniciar:

1. **T=0s**: `postgres` (processo) inicia
2. **T=1s**: Carrega configurações (`postgresql.conf`)
3. **T=2s**: Inicializa buffers de memória
4. **T=3s**: Recupera transações pendentes (WAL replay)
5. **T=4s**: Abre socket TCP na porta 5432
6. **T=5s**: **Pronto para aceitar conexões** ✅

O `depends_on` verifica apenas o **passo 1** (processo iniciou), não o **passo 6** (aceita conexões).

---

### Verificação prática:

**Testar sem `condition`:**

```bash
# Remover healthcheck temporariamente
# docker-compose.yml:
depends_on:
  - db  # Sem "condition: service_healthy"

docker compose down
docker compose up
```

**Logs do `web`:**

```
achados-web | psycopg.OperationalError: connection to server at "db" (172.18.0.2), port 5432 failed: Connection refused
achados-web |     Is the server running on that host and accepting TCP/IP connections?
achados-web | Exiting with code 1
```

**O contêiner `web` crasheia!** (Porque Gunicorn não consegue importar `app.py`, que tenta conectar ao importar `psycopg`)

---

### Cenário 2: Com `condition: service_healthy` ✅

```yaml
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  web:
    build: ./app
    depends_on:
      db:
        condition: service_healthy  # ← AGUARDA SAÚDE!
```

**O que acontece:**

```
Timeline:
T=0s   Docker inicia contêiner db
T=1s   PostgreSQL processo inicia
T=2s   Healthcheck #1: pg_isready → FALHOU
T=7s   Healthcheck #2: pg_isready → FALHOU
T=12s  Healthcheck #3: pg_isready → SUCESSO ✅
T=12s  db marcado como "healthy"
T=12s  Docker inicia contêiner web  ← AGORA (12s depois!)
T=13s  web conecta ao banco → SUCESSO ✅
```

**Por quê funciona?**

O `pg_isready` é um utilitário oficial do PostgreSQL que testa se o banco **aceita conexões TCP/IP**:

```bash
pg_isready -U achados_user -d achados_perdidos
# Saída se OK:   /var/run/postgresql:5432 - accepting connections
# Código retorno: 0

# Saída se FALHA: /var/run/postgresql:5432 - no response
# Código retorno: 1
```

O Docker repete o teste até retornar `0`, e só então libera dependentes.

---

### Por que não usar `sleep` em vez de healthcheck?

**❌ Abordagem ruim:**

```yaml
web:
  build: ./app
  command: sh -c "sleep 10 && gunicorn app:app"
  depends_on:
    - db
```

**Problemas:**

1. **Tempo fixo arbitrário:**
   - Se o banco demorar 12s, ainda falha
   - Se demorar 3s, desperdiça 7s

2. **Não testa REALMENTE:**
   - E se o PostgreSQL crashear durante os 10s?
   - E se houver problema de rede?

3. **Não funciona em prod:**
   - Ambientes de nuvem têm latência variável
   - Cold start pode demorar minutos

---

### Outras `condition` disponíveis:

```yaml
depends_on:
  db:
    condition: service_started     # Apenas garante que iniciou (padrão de depends_on simples)
    condition: service_healthy     # Aguarda healthcheck retornar "healthy" ✅
    condition: service_completed_successfully  # Aguarda contêiner terminar com exit 0
```

---

### 📊 Comparação:

| Configuração | Garante ordem? | Garante disponibilidade? | Risco de falha |
|--------------|----------------|--------------------------|----------------|
| Sem `depends_on` | ❌ Não | ❌ Não | Alto |
| `depends_on: [db]` | ✅ Sim | ❌ Não | Médio-Alto |
| `depends_on` + `condition: service_healthy` | ✅ Sim | ✅ Sim | Baixo |
| `sleep` antes de conectar | ✅ Sim | ⚠️ Talvez | Médio |

---

### Implementação no projeto:

**Healthcheck do banco (`docker-compose.yml`):**

```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s       # Testa a cada 5 segundos
    timeout: 5s        # Desiste se não responder em 5s
    retries: 5         # 5 tentativas (25s no máximo)
    start_period: 10s  # Período de graça (falhas não contam)
```

**Dependência do web:**

```yaml
web:
  depends_on:
    db:
      condition: service_healthy  # ← Aguarda "healthy"!
```

---

### Verificação de que está funcionando:

```bash
docker compose up -d
docker compose ps
```

**Saída esperada:**

```
NAME            IMAGE                  STATUS              PORTS
achados-db      postgres:16-alpine     Up (healthy)        -
achados-web     achados-perdidos-web   Up                  0.0.0.0:8080->5000/tcp
```

Note: `(healthy)` no status do banco.

**Logs do web (sem erros):**

```bash
docker compose logs web --tail 10
```

```
achados-web | [2026-10-19 01:45:23] [1] [INFO] Starting gunicorn 23.0.0
achados-web | [2026-10-19 01:45:23] [1] [INFO] Listening at: http://0.0.0.0:5000
achados-web | [2026-10-19 01:45:23] [1] [INFO] Using worker: sync
```

**Nenhum erro de conexão!** ✅

---

**Conclusão:**

O `depends_on` sozinho apenas garante que o **processo do PostgreSQL iniciou**, não que ele está **pronto para aceitar conexões**. Como o banco passa por inicialização (carregamento de configs, recuperação de WAL, abertura de sockets), pode levar 5-15 segundos até aceitar conexões. O `healthcheck` com `pg_isready` testa ativamente se o banco responde, e `condition: service_healthy` bloqueia a aplicação até que o teste passe, eliminando race conditions e garantindo inicialização confiável.

---

## Q5: Cache de camadas do Docker

### Pergunta
> A ordem das instruções `COPY requirements.txt` → `RUN pip install` → `COPY app.py` influencia o tempo de build. Explique usando o conceito de cache de camadas e cite o que acontece quando só o `app.py` muda.

### Resposta

Essa ordem é uma das **otimizações mais importantes** em Dockerfiles. Vamos entender por quê com exemplos práticos.

---

### 📦 O que é cache de camadas?

Cada instrução no Dockerfile cria uma **camada** (layer) imutável:

```dockerfile
FROM python:3.12-slim       # Camada 1 (base)
COPY requirements.txt .     # Camada 2
RUN pip install ...         # Camada 3
COPY app.py .               # Camada 4
```

**Cache:** O Docker **reutiliza** camadas anteriores se nada mudou.

---

### ❌ Ordem RUIM (sem otimização):

```dockerfile
FROM python:3.12-slim
WORKDIR /app

# 1. Copiar TUDO de uma vez
COPY . .

# 2. Instalar dependências
RUN pip install --no-cache-dir -r requirements.txt

CMD ["gunicorn", "app:app"]
```

**Problema:**

```
Cenário: Mudar uma linha no app.py
```

**Build 1 (inicial):**
```
Step 1: FROM python:3.12-slim        [Cached] ✅
Step 2: COPY . .                     [Build]  ⏱️ 2s
Step 3: RUN pip install ...          [Build]  ⏱️ 45s
Total: 47s
```

**Build 2 (após mudar `app.py`):**
```
Step 1: FROM python:3.12-slim        [Cached] ✅
Step 2: COPY . .                     [Build]  ⏱️ 2s  ← app.py mudou!
Step 3: RUN pip install ...          [Build]  ⏱️ 45s ← REINSTALA TUDO! 😱
Total: 47s
```

**Por quê reinstala?**

O Docker detecta que o **conteúdo** copiado em `COPY . .` mudou (`app.py` diferente).

A camada 2 é **invalidada**, e todas as seguintes também (`RUN pip install`).

**Resultado:** Cada mudança no código reinstala Flask, psycopg, Gunicorn... ❌

---

### ✅ Ordem BOA (com otimização):

```dockerfile
FROM python:3.12-slim
WORKDIR /app

# 1. Copiar APENAS requirements.txt
COPY requirements.txt .

# 2. Instalar dependências (camada pesada)
RUN pip install --no-cache-dir -r requirements.txt

# 3. Copiar código (camada leve)
COPY . .

CMD ["gunicorn", "app:app"]
```

**Build 1 (inicial):**
```
Step 1: FROM python:3.12-slim        [Cached] ✅
Step 2: COPY requirements.txt .      [Build]  ⏱️ 0.1s
Step 3: RUN pip install ...          [Build]  ⏱️ 45s
Step 4: COPY . .                     [Build]  ⏱️ 2s
Total: 47s
```

**Build 2 (após mudar `app.py`, `requirements.txt` intacto):**
```
Step 1: FROM python:3.12-slim        [Cached] ✅
Step 2: COPY requirements.txt .      [Cached] ✅ ← Arquivo não mudou!
Step 3: RUN pip install ...          [Cached] ✅ ← Reutiliza camada!
Step 4: COPY . .                     [Build]  ⏱️ 2s  ← Só copia novo código
Total: 2s ⚡
```

**Ganho:** **47s → 2s** (23x mais rápido!)

---

### Por que funciona?

**Regra do cache:** O Docker invalida uma camada se:
1. A instrução mudou (`COPY requirements.txt` → `COPY outro.txt`)
2. O **conteúdo** dos arquivos mudou (checksum diferente)

**Ao separar `COPY requirements.txt` de `COPY app.py`:**
- Mudanças no `app.py` **não invalidam** a camada de `requirements.txt`
- A camada `RUN pip install` **permanece cacheada**
- Apenas a camada final `COPY . .` é rebuilda

---

### 📊 Demonstração prática:

**Teste 1: Mudar `app.py`**

```bash
# Editar app.py (adicionar um comentário)
echo "# Comentário de teste" >> app/app.py

# Rebuild
docker compose build web
```

**Saída:**
```
[+] Building 2.3s (10/10) FINISHED
 => [web 1/6] FROM python:3.12-slim          [CACHED]
 => [web 2/6] WORKDIR /app                   [CACHED]
 => [web 3/6] COPY requirements.txt .        [CACHED] ✅
 => [web 4/6] RUN pip install ...            [CACHED] ✅
 => [web 5/6] COPY . .                       [BUILD]  2.1s
```

**Apenas a última camada rebuilda!** ⚡

---

**Teste 2: Mudar `requirements.txt`**

```bash
# Adicionar nova dependência
echo "requests==2.31.0" >> app/requirements.txt

# Rebuild
docker compose build web
```

**Saída:**
```
[+] Building 18.7s (10/10) FINISHED
 => [web 1/6] FROM python:3.12-slim          [CACHED]
 => [web 2/6] WORKDIR /app                   [CACHED]
 => [web 3/6] COPY requirements.txt .        [BUILD]  0.1s ← Mudou!
 => [web 4/6] RUN pip install ...            [BUILD]  16.2s ← Reinstala!
 => [web 5/6] COPY . .                       [BUILD]  2.3s
```

**Camadas 3, 4 e 5 rebuildam** (porque 3 foi invalidada).

---

### 🎯 Impacto no desenvolvimento:

**Workflow típico:**

```
1. Mudar código (app.py)      → Build em 2s  ⚡
2. Mudar código (app.py)      → Build em 2s  ⚡
3. Mudar código (app.py)      → Build em 2s  ⚡
...
10. Adicionar biblioteca      → Build em 47s ⏱️
11. Mudar código (app.py)     → Build em 2s  ⚡
```

**90% das mudanças** são no código, não nas dependências.

**Cache de camadas** otimiza o caso comum (mudança de código).

---

### 🧠 Princípio geral:

**"Camadas menos frequentes primeiro, mais frequentes depois"**

```dockerfile
# RARAMENTE muda (1x por projeto)
FROM python:3.12-slim

# ÀS VEZES muda (1x por sprint)
COPY requirements.txt .
RUN pip install ...

# SEMPRE muda (10x por dia)
COPY . .
```

**Hierarquia de frequência:**
1. Imagem base: muda quase nunca
2. Dependências: muda ocasionalmente
3. Código: muda constantemente

---

### 📊 Comparação de tamanhos:

**Tamanho das camadas (nosso projeto):**

```bash
docker image history achados-perdidos-web
```

```
IMAGE          CREATED       SIZE      COMMENT
<hash>         2 min ago     5.2kB     CMD ["gunicorn" ...]
<hash>         2 min ago     8.1kB     COPY . .              ← LEVE
<hash>         10 min ago    0B        USER appuser
<hash>         10 min ago    4.5kB     RUN adduser ...
<hash>         10 min ago    142MB     RUN pip install ...   ← PESADA!
<hash>         10 min ago    1.2kB     COPY requirements.txt
<hash>         10 min ago    0B        WORKDIR /app
python:3.12... 3 weeks ago   130MB
```

**Observações:**
- `RUN pip install`: **142 MB** (psycopg, Flask, Gunicorn)
- `COPY . .`: **8.1 KB** (apenas código Python)

**Razão:** 142 MB / 8.1 KB = **17.500x maior**!

Rebuildar a camada pesada (pip) para mudar 8 KB de código seria absurdo.

---

### 💡 Outras otimizações de cache:

**1. Multi-stage builds:**
```dockerfile
# Stage 1: Build dependencies
FROM python:3.12-slim AS builder
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim
COPY --from=builder /root/.local /root/.local
COPY . .
```

**2. Cache de npm/yarn (Node.js):**
```dockerfile
COPY package.json package-lock.json .
RUN npm ci
COPY . .
```

**3. Cache de Go modules:**
```dockerfile
COPY go.mod go.sum .
RUN go mod download
COPY . .
```

**Padrão:** Sempre copiar arquivos de lock **antes** do código.

---

**Conclusão:**

A ordem `COPY requirements.txt` → `RUN pip install` → `COPY app.py` aproveita o cache de camadas do Docker para otimizar builds subsequentes. Como mudanças no código (`app.py`) são muito mais frequentes que mudanças nas dependências (`requirements.txt`), separar essas camadas reduz o tempo de build de ~47s para ~2s na maioria das mudanças, acelerando o ciclo de desenvolvimento em 23x. Esse padrão é universal e deve ser aplicado em Dockerfiles de qualquer linguagem (Python, Node.js, Go, Java, etc.).

---

## Q6: Segurança e LGPD

### Pergunta
> Por que a porta 5432 do banco não deve ser publicada no host? Considerando que as fotos podem mostrar documentos pessoais esquecidos (carteira de estudante, RG), relacione a sua configuração com a LGPD e proponha duas medidas adicionais de proteção.

### Resposta

---

### 🔒 Por que a porta 5432 NÃO deve ser publicada?

**Decisão no `docker-compose.yml`:**

```yaml
db:
  image: postgres:16-alpine
  # NENHUMA linha "ports:" ← Porta 5432 ISOLADA!
```

**Efeito:** A porta 5432 do PostgreSQL fica **acessível apenas dentro da rede `rede_interna`**, não no host.

---

### Cenário 1: ❌ Se publicássemos a porta (errado):

```yaml
db:
  ports:
    - "5432:5432"  # ← NUNCA FAÇA ISSO EM PROD!
```

**Consequências:**

1. **Exposição na rede local:**
   ```
   Qualquer computador na rede → localhost:5432 → PostgreSQL
   ```
   
   - Colega de trabalho na mesma rede pode tentar se conectar
   - Dispositivos IoT comprometidos podem fazer port scan
   - Malware no seu computador tem acesso direto

2. **Ataques de força bruta:**
   ```bash
   # Atacante tenta senhas comuns:
   psql -h 192.168.1.100 -U achados_user -d achados_perdidos
   # Senha: admin, password, 123456, postgres, ...
   ```

3. **Exploits conhecidos:**
   - CVEs do PostgreSQL (SQL injection, privilege escalation)
   - Se houver vulnerabilidade não corrigida, atacante explora diretamente

4. **Violação do princípio do menor privilégio:**
   - Banco só precisa conversar com `web`
   - Não há razão para expor ao host ou à rede

---

### Cenário 2: ✅ Porta NÃO publicada (correto):

**Como `web` se conecta?**

```
[web] → rede_interna (DNS resolve "db") → [db:5432]
```

- Comunicação **apenas dentro do Docker**
- Host não tem acesso
- Rede local não tem acesso

**Verificação:**

```bash
# Do host (falha):
psql -h localhost -p 5432 -U achados_user -d achados_perdidos
# psql: error: connection to server at "localhost" (::1), port 5432 failed: Connection refused

# De dentro do contêiner web (funciona):
docker compose exec web psql -h db -U achados_user -d achados_perdidos
# achados_perdidos=> SELECT COUNT(*) FROM itens;
```

---

### 🛡️ Relação com a LGPD:

A **Lei Geral de Proteção de Dados (LGPD)** exige medidas técnicas e administrativas para proteger dados pessoais.

**Artigo 46, LGPD:**
> "Os agentes de tratamento devem adotar medidas de segurança, técnicas e administrativas aptas a proteger os dados pessoais de acessos não autorizados..."

---

**Dados pessoais no sistema:**

1. **Fotos de objetos perdidos** podem conter:
   - Carteira de estudante (nome, matrícula, foto)
   - RG esquecido na capa de caderno (CPF, filiação)
   - Agenda com anotações pessoais
   - Receitas médicas

2. **Banco de dados `itens`:**
   - Descrição pode ter informações identificadoras
   - Localização pode expor rotina da pessoa

**Classificação LGPD:**
- **Dados pessoais**: Nome, matrícula (Art. 5º, I)
- **Dados sensíveis** (se houver receita médica): Saúde (Art. 5º, II)

---

### 🔐 Como a configuração atual protege:

| Medida | Implementação | Artigo LGPD relacionado |
|--------|---------------|-------------------------|
| **Isolamento de rede** | Porta 5432 não publicada | Art. 46, § 1º (segurança) |
| **Credenciais em arquivo `.env`** | Não versionadas no Git | Art. 49 (boas práticas) |
| **Usuário sem privilégios** | `appuser` no contêiner web | Art. 46 (minimização de impacto) |
| **Volumes isolados** | Dados não acessíveis diretamente do host | Art. 46 (controle de acesso) |

---

### ➕ Duas medidas adicionais de proteção:

---

#### 🔐 Medida 1: Criptografia de dados sensíveis em repouso

**Problema:**

Atualmente, as fotos são armazenadas **sem criptografia** no volume `fotos`.

Se alguém tiver acesso ao host (administrador, backup comprometido, disco roubado), pode ler os arquivos diretamente:

```bash
# No host (se tiver acesso root):
cd /var/lib/docker/volumes/achados-perdidos_fotos/_data
ls -l
# a3f8e9d1c2b4.jpg  ← Foto de RG visível!
```

**Solução: Criptografia em nível de aplicação**

**Implementação:**

```python
# app.py (adicionar no cadastro de foto)
from cryptography.fernet import Fernet
import os

# Chave armazenada em variável de ambiente
ENCRYPTION_KEY = os.getenv("ENCRYPTION_KEY")  # 32 bytes base64
cipher = Fernet(ENCRYPTION_KEY)

@app.post("/itens")
def cadastrar():
    # ... código existente ...
    if foto and foto.filename:
        # Ler bytes da foto
        foto_bytes = foto.read()
        
        # Criptografar
        foto_criptografada = cipher.encrypt(foto_bytes)
        
        # Salvar arquivo criptografado
        nome_foto = f"{uuid.uuid4().hex}.enc"
        caminho = os.path.join(UPLOAD_DIR, nome_foto)
        with open(caminho, 'wb') as f:
            f.write(foto_criptografada)

@app.get("/fotos/<nome>")
def foto(nome):
    # Ler arquivo criptografado
    caminho = os.path.join(UPLOAD_DIR, nome)
    with open(caminho, 'rb') as f:
        foto_criptografada = f.read()
    
    # Descriptografar
    foto_bytes = cipher.decrypt(foto_criptografada)
    
    # Retornar imagem
    return Response(foto_bytes, mimetype='image/jpeg')
```

**Configuração:**

```bash
# .env
ENCRYPTION_KEY=base64_da_chave_fernet_32bytes
```

**docker-compose.yml:**
```yaml
web:
  environment:
    ENCRYPTION_KEY: ${ENCRYPTION_KEY}
```

**Vantagens:**
- ✅ Fotos ilegíveis sem a chave
- ✅ Proteção contra acesso físico ao disco
- ✅ Atende Art. 46, § 1º (criptografia de dados sensíveis)

**Desvantagens:**
- ⚠️ Perda da chave = perda de todas as fotos
- ⚠️ Leve overhead de performance

---

#### 🔐 Medida 2: Log de auditoria e retenção de dados

**Problema:**

Atualmente, não há:
- Log de quem acessou quais fotos
- Controle de retenção (fotos ficam indefinidamente)
- Trilha de auditoria para atender Art. 37 (relatório de impacto)

**Solução: Sistema de auditoria e limpeza automática**

**Implementação:**

**1. Tabela de auditoria:**

```sql
-- db/init.sql (adicionar)
CREATE TABLE auditoria (
    id SERIAL PRIMARY KEY,
    acao VARCHAR(50) NOT NULL,       -- 'visualizar', 'cadastrar', 'deletar'
    item_id INTEGER REFERENCES itens(id),
    ip_origem VARCHAR(45),            -- IPv4/IPv6
    user_agent TEXT,
    data_hora TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**2. Middleware de auditoria:**

```python
# app.py
from flask import request
import psycopg

def registrar_auditoria(acao, item_id=None):
    ip_origem = request.headers.get('X-Forwarded-For', request.remote_addr)
    user_agent = request.headers.get('User-Agent', 'Unknown')
    
    with conectar() as c:
        c.execute(
            "INSERT INTO auditoria (acao, item_id, ip_origem, user_agent) VALUES (%s, %s, %s, %s)",
            (acao, item_id, ip_origem, user_agent)
        )

@app.get("/fotos/<nome>")
def foto(nome):
    # Buscar item_id pelo nome da foto
    with conectar() as c:
        item = c.execute("SELECT id FROM itens WHERE foto = %s", (nome,)).fetchone()
    
    if item:
        registrar_auditoria('visualizar', item[0])
    
    return send_from_directory(UPLOAD_DIR, nome)

@app.post("/itens")
def cadastrar():
    # ... código existente ...
    registrar_auditoria('cadastrar', novo_id)
    return jsonify(id=novo_id, foto=nome_foto), 201
```

**3. Limpeza automática (retenção de 90 dias):**

```python
# app.py (adicionar rota administrativa)
@app.post("/admin/limpar-antigos")
def limpar_antigos():
    # LGPD Art. 16: Dados mantidos apenas pelo tempo necessário
    with conectar() as c:
        # Itens com mais de 90 dias
        itens_antigos = c.execute(
            "SELECT id, foto FROM itens WHERE criado_em < NOW() - INTERVAL '90 days'"
        ).fetchall()
        
        for item_id, foto in itens_antigos:
            # Deletar foto do disco
            if foto:
                caminho = os.path.join(UPLOAD_DIR, foto)
                if os.path.exists(caminho):
                    os.remove(caminho)
            
            # Deletar registro do banco
            c.execute("DELETE FROM itens WHERE id = %s", (item_id,))
            registrar_auditoria('deletar_automatico', item_id)
    
    return jsonify(mensagem="Limpeza concluída"), 200
```

**4. Agendar limpeza com cron:**

```yaml
# docker-compose.yml (adicionar serviço)
services:
  limpeza-agendada:
    image: curlimages/curl:8.5.0
    command: sh -c "while true; do sleep 86400; curl -X POST http://web:5000/admin/limpar-antigos; done"
    networks:
      - rede_interna
    depends_on:
      web:
        condition: service_started
    profiles:
      - producao
```

**Vantagens:**
- ✅ Rastreabilidade completa (Art. 37 LGPD: relatório de impacto)
- ✅ Retenção limitada (Art. 16: necessidade e finalidade)
- ✅ Resposta a incidentes (Art. 48: comunicação de vazamento)
- ✅ Direito ao esquecimento (Art. 18, VI: eliminação de dados)

**Evidências para auditoria:**

```bash
# Relatório de acessos:
docker compose exec db psql -U achados_user -d achados_perdidos -c \
  "SELECT acao, COUNT(*) FROM auditoria GROUP BY acao;"

# Quem acessou uma foto específica:
docker compose exec db psql -U achados_user -d achados_perdidos -c \
  "SELECT * FROM auditoria WHERE item_id = 4 ORDER BY data_hora DESC;"
```

---

### 📊 Resumo das medidas de proteção:

| Medida | Status | Artigo LGPD | Impacto |
|--------|--------|-------------|---------|
| Porta 5432 isolada | ✅ Implementado | Art. 46 (segurança) | Alto |
| Credenciais em `.env` | ✅ Implementado | Art. 49 (boas práticas) | Médio |
| Usuário sem privilégios | ✅ Implementado | Art. 46 (minimização) | Médio |
| **Criptografia de fotos** | ➕ Proposto | Art. 46, § 1º (dados sensíveis) | Alto |
| **Log de auditoria** | ➕ Proposto | Art. 37 (relatório), Art. 48 (incidentes) | Alto |
| Retenção de 90 dias | ➕ Proposto | Art. 16 (minimização temporal) | Médio |

---

**Conclusão:**

A porta 5432 do banco não deve ser publicada no host para implementar o princípio do menor privilégio: apenas o serviço `web` precisa acessar o banco, então expô-lo à rede local ou à internet aumenta desnecessariamente a superfície de ataque, facilitando força bruta, exploits e acessos não autorizados. Considerando que as fotos podem conter dados pessoais (RG, carteirinha), a LGPD (Art. 46) exige medidas técnicas adequadas. Além do isolamento de rede já implementado, duas medidas adicionais de proteção seriam: **(1) Criptografia de fotos em repouso** usando Fernet, protegendo contra acesso físico ao disco; e **(2) Sistema de auditoria com log de acessos e limpeza automática após 90 dias**, atendendo aos requisitos de rastreabilidade (Art. 37), retenção mínima (Art. 16) e resposta a incidentes (Art. 48).

---

## Q7: Escalonamento horizontal

### Pergunta
> Tente executar `docker compose up -d --scale web=3`. O que acontece e por quê? O que seria necessário para rodar três réplicas da aplicação? As réplicas poderiam compartilhar o volume `fotos` com segurança?

### Resposta

Vou simular o comando e analisar o comportamento:

---

### ⚠️ Tentativa de escalar:

```bash
docker compose up -d --scale web=3
```

**Resultado esperado:**

```
Error response from daemon: driver failed programming external connectivity on endpoint achados-web-2: 
Bind for 0.0.0.0:8080 failed: port is already allocated
```

**❌ FALHA! O Docker não consegue criar as réplicas.**

---

### 🔍 Por que falha?

**Problema:** Conflito de porta publicada.

**No `docker-compose.yml`:**

```yaml
web:
  container_name: achados-web  # Nome fixo!
  ports:
    - "8080:5000"  # Porta do host fixa!
```

**O que acontece:**

```
Tentativa de criar 3 réplicas:

Réplica 1 (achados-web):
  - Publica 0.0.0.0:8080 → container:5000 ✅

Réplica 2 (achados-web-2):
  - Tenta publicar 0.0.0.0:8080 → container:5000 ❌
  - ERRO: Porta 8080 já está em uso!

Réplica 3 (achados-web-3):
  - Nem tenta criar (falhou antes)
```

**Dois problemas:**

1. **`container_name: achados-web`**
   - Nomes de contêineres devem ser únicos
   - `--scale` não consegue criar `achados-web-2` porque o nome é fixo

2. **`ports: "8080:5000"`**
   - Porta 8080 do host só pode ser usada por um contêiner
   - Segunda réplica não tem onde publicar

---

### ✅ O que seria necessário para rodar 3 réplicas?

Precisamos de um **proxy reverso** (load balancer) na frente das réplicas.

---

#### Solução 1: Nginx como proxy reverso

**Arquitetura:**

```
Cliente (navegador)
    ↓
    8080:80 (porta única no host)
    ↓
 [nginx]  ← Load balancer
    ↓
    ├── web-1:5000  ← Réplica 1
    ├── web-2:5000  ← Réplica 2
    └── web-3:5000  ← Réplica 3
         ↓
      [db:5432]
```

**Implementação:**

**1. Remover `container_name` e porta publicada do `web`:**

```yaml
# docker-compose.yml
services:
  web:
    build: ./app
    # container_name: achados-web  ← REMOVER!
    # ports:                        ← REMOVER!
    #   - "8080:5000"
    environment:
      # ... configurações existentes ...
    volumes:
      - fotos:/app/uploads
    networks:
      - rede_interna
    depends_on:
      db:
        condition: service_healthy
    deploy:
      replicas: 3  # ← Definir réplicas aqui
```

**2. Adicionar Nginx:**

```yaml
# docker-compose.yml (adicionar serviço)
services:
  nginx:
    image: nginx:1.25-alpine
    container_name: achados-nginx
    ports:
      - "8080:80"  # ← ÚNICA porta publicada no host
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - rede_interna
    depends_on:
      - web
```

**3. Configurar balanceamento de carga:**

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    # Pool de backends (réplicas)
    upstream web_backend {
        # Round-robin (padrão)
        server achados-perdidos-web-1:5000;
        server achados-perdidos-web-2:5000;
        server achados-perdidos-web-3:5000;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://web_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

**4. Subir com 3 réplicas:**

```bash
docker compose up -d --scale web=3
```

**Resultado:**

```
NAME                       IMAGE                    STATUS
achados-db                 postgres:16-alpine       Up (healthy)
achados-perdidos-web-1     achados-perdidos-web     Up
achados-perdidos-web-2     achados-perdidos-web     Up
achados-perdidos-web-3     achados-perdidos-web     Up
achados-nginx              nginx:1.25-alpine        Up
```

**Teste:**

```bash
curl http://localhost:8080/health
# {"status":"ok","banco":"conectado","container":"achados-perdidos-web-1"}

curl http://localhost:8080/health
# {"status":"ok","banco":"conectado","container":"achados-perdidos-web-2"}

curl http://localhost:8080/health
# {"status":"ok","banco":"conectado","container":"achados-perdidos-web-3"}
```

**Nginx distribui requisições alternadamente (round-robin)** ✅

---

#### Solução 2: Docker Swarm (nativo)

```bash
# Inicializar Swarm
docker swarm init

# Deploy com 3 réplicas
docker stack deploy -c docker-compose.yml achados
```

**docker-compose.yml (modo Swarm):**

```yaml
version: '3.8'
services:
  web:
    deploy:
      replicas: 3
      endpoint_mode: vip  # Virtual IP (load balancing interno)
```

**Vantagens:**
- ✅ Load balancing nativo do Docker
- ✅ Failover automático
- ✅ Sem nginx extra

**Desvantagens:**
- ❌ Menos flexível que Nginx
- ❌ Não funciona no Docker Compose normal

---

### 🤔 As réplicas poderiam compartilhar o volume `fotos` com segurança?

**Resposta: ⚠️ DEPENDE da implementação.**

---

#### Cenário 1: ✅ Leitura de fotos (seguro)

```python
@app.get("/fotos/<nome>")
def foto(nome):
    return send_from_directory(UPLOAD_DIR, nome)
```

**Comportamento:**

```
Requisição 1 → web-1 → Lê fotos/a3f8e9.jpg ✅
Requisição 2 → web-2 → Lê fotos/f1a2b3.jpg ✅
Requisição 3 → web-3 → Lê fotos/a3f8e9.jpg ✅
```

**✅ Seguro:** Múltiplas leituras simultâneas não causam problema.

---

#### Cenário 2: ⚠️ Escrita de fotos (race condition)

```python
@app.post("/itens")
def cadastrar():
    nome_foto = f"{uuid.uuid4().hex}.jpg"
    foto.save(os.path.join(UPLOAD_DIR, nome_foto))
```

**Problema potencial:**

```
T=0s   web-1 gera nome: a3f8e9.jpg
T=0s   web-2 gera nome: f1a2b3.jpg
T=1s   web-1 escreve: fotos/a3f8e9.jpg ✅
T=1s   web-2 escreve: fotos/f1a2b3.jpg ✅
```

**✅ Neste caso, seguro!** Porque `uuid.uuid4()` garante nomes únicos (probabilidade de colisão: ~1 em 10³⁸).

---

**Mas e se não usasse UUID?**

```python
# CÓDIGO RUIM (exemplo):
nome_foto = f"foto_{len(os.listdir(UPLOAD_DIR)) + 1}.jpg"
```

**Race condition:**

```
T=0s   web-1: lê diretório → 10 arquivos → nome será "foto_11.jpg"
T=0s   web-2: lê diretório → 10 arquivos → nome será "foto_11.jpg" (mesma!)
T=1s   web-1: escreve "foto_11.jpg" (foto do óculos)
T=1s   web-2: escreve "foto_11.jpg" (foto da mochila) ← SOBRESCREVE!
```

**❌ Perda de dados!**

---

### 📊 Análise de segurança do compartilhamento:

| Operação | Seguro? | Por quê? |
|----------|---------|----------|
| **Leitura** | ✅ Sim | Múltiplas leituras simultâneas são idempotentes |
| **Escrita com UUID** | ✅ Sim | Nomes únicos garantidos (sem colisão) |
| **Escrita com contador** | ❌ Não | Race condition (duas réplicas geram mesmo nome) |
| **Deleção** | ⚠️ Depende | Se baseada em ID do banco, OK; se baseada em varredura de diretório, race condition |
| **Renomeação** | ❌ Não | Duas réplicas podem tentar renomear simultaneamente |

---

### 🔐 Alternativa: Armazenamento externo (S3-compatible)

Para **produção em larga escala**, volume compartilhado entre réplicas não é ideal.

**Solução: Armazenamento de objetos (S3, MinIO, Azure Blob)**

**Vantagens:**

1. **Sem race conditions:**
   - S3 garante atomicidade de uploads
   - Operações PUT são thread-safe

2. **Escalabilidade ilimitada:**
   - Não depende de disco local
   - CDN pode cachear fotos globalmente

3. **Redundância:**
   - S3 replica automaticamente (99,999999999% durabilidade)

**Implementação (boto3 + MinIO):**

```python
# app.py
import boto3
from botocore.client import Config

s3 = boto3.client(
    's3',
    endpoint_url='http://minio:9000',
    aws_access_key_id=os.getenv('S3_ACCESS_KEY'),
    aws_secret_access_key=os.getenv('S3_SECRET_KEY'),
    config=Config(signature_version='s3v4')
)

@app.post("/itens")
def cadastrar():
    nome_foto = f"{uuid.uuid4().hex}.jpg"
    s3.put_object(
        Bucket='achados-fotos',
        Key=nome_foto,
        Body=foto.stream,
        ContentType='image/jpeg'
    )

@app.get("/fotos/<nome>")
def foto(nome):
    obj = s3.get_object(Bucket='achados-fotos', Key=nome)
    return Response(obj['Body'].read(), mimetype='image/jpeg')
```

**docker-compose.yml (adicionar MinIO):**

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    networks:
      - rede_interna

volumes:
  minio_data:
```

---

### 📊 Comparação de arquiteturas:

| Arquitetura | Escalabilidade | Complexidade | Custo | Uso recomendado |
|-------------|----------------|--------------|-------|------------------|
| **Sem réplicas** | ❌ Baixa | Baixa | Baixo | Dev/teste |
| **Réplicas + Nginx + Volume** | ⚠️ Média | Média | Baixo | Staging |
| **Réplicas + S3** | ✅ Alta | Alta | Médio | Produção |
| **Kubernetes + S3 + CDN** | ✅ Muito alta | Muito alta | Alto | Larga escala |

---

**Conclusão:**

Executar `docker compose up -d --scale web=3` **falha** porque o `docker-compose.yml` define uma `container_name` fixa e publica a porta 8080 diretamente, impossibilitando múltiplas réplicas (apenas uma pode ocupar a porta 8080 do host). Para rodar três réplicas, seria necessário: **(1) Remover `container_name` e `ports` do serviço `web`**; **(2) Adicionar um proxy reverso como Nginx** que publica a porta 8080 e distribui requisições entre as réplicas usando round-robin; **(3) Configurar `deploy.replicas: 3`** no Compose. Quanto ao volume `fotos`, as réplicas **podem compartilhá-lo com segurança** para leitura e para escrita usando UUID (como já implementado), mas em produção em larga escala, a melhor prática seria migrar para armazenamento de objetos (S3, MinIO, Azure Blob) para evitar gargalos de I/O no volume e garantir escalabilidade ilimitada.

---

## 🌐 Equivalência na nuvem

### Do ambiente local para a nuvem

O ambiente Docker local é um **protótipo funcional** do que será publicado na nuvem. A tabela abaixo mapeia cada componente local para o serviço gerenciado equivalente nos três principais provedores.

---

### 📊 Tabela de equivalência:

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
|------------------------|-----------------|-----|--------------|
| **Contêiner web** | Azure Container Instances (ACI) ou Azure App Service (Container) | AWS Fargate (ECS) ou AWS App Runner | Google Cloud Run ou Google Kubernetes Engine (GKE) Autopilot |
| **Contêiner db + volume `dados_banco`** | Azure Database for PostgreSQL (Flexible Server) | Amazon RDS for PostgreSQL ou Amazon Aurora PostgreSQL | Cloud SQL for PostgreSQL |
| **Volume `fotos`** | Azure Blob Storage ou Azure Files | Amazon S3 | Google Cloud Storage (GCS) |
| **rede_interna** | Azure Virtual Network (VNet) | Amazon VPC | Google Cloud VPC |
| **Arquivo `.env` (segredos)** | Azure Key Vault | AWS Secrets Manager ou AWS Systems Manager Parameter Store | Google Secret Manager |
| **Imagem construída localmente** | Azure Container Registry (ACR) | Amazon Elastic Container Registry (ECR) | Google Artifact Registry |

---

### 🔍 Detalhamento de cada migração:

---

#### 1. **Contêiner web** → Serviços de contêiner gerenciados

**Local:**
```yaml
web:
  build: ./app
  ports:
    - "8080:5000"
```

**Azure: Azure Container Instances (ACI)**
```bash
az container create \
  --resource-group achados-rg \
  --name achados-web \
  --image acrachados.azurecr.io/achados-web:latest \
  --cpu 1 --memory 1 \
  --ports 5000 \
  --environment-variables DB_HOST=achados-db.postgres.database.azure.com
```

**AWS: AWS Fargate (ECS)**
```json
{
  "family": "achados-web",
  "networkMode": "awsvpc",
  "containerDefinitions": [{
    "name": "web",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/achados-web:latest",
    "portMappings": [{"containerPort": 5000}],
    "environment": [
      {"name": "DB_HOST", "value": "achados-db.xyz.us-east-1.rds.amazonaws.com"}
    ]
  }]
}
```

**Google Cloud: Cloud Run**
```bash
gcloud run deploy achados-web \
  --image gcr.io/meu-projeto/achados-web:latest \
  --platform managed \
  --port 5000 \
  --set-env-vars DB_HOST=10.20.30.40 \
  --allow-unauthenticated
```

---

#### 2. **Banco de dados** → Bancos gerenciados (DBaaS)

**Local:**
```yaml
db:
  image: postgres:16-alpine
  volumes:
    - dados_banco:/var/lib/postgresql/data
```

**Azure: Azure Database for PostgreSQL**
- **Vantagens:**
  - Backups automáticos (7-35 dias)
  - Alta disponibilidade (99,99% SLA)
  - Patching automático de segurança
  - Escalabilidade vertical (2-64 vCores)
  
**AWS: Amazon RDS for PostgreSQL**
- **Vantagens:**
  - Multi-AZ (failover automático)
  - Read replicas (até 5)
  - Automated backups (1-35 dias)
  - Point-in-time recovery (PITR)

**Google Cloud: Cloud SQL for PostgreSQL**
- **Vantagens:**
  - Réplicas cross-region
  - Import/export automatizado
  - Insights de performance (Query Insights)
  - Proxy de conexão (Cloud SQL Proxy)

---

#### 3. **Volume `fotos`** → Object Storage

**Local:**
```yaml
volumes:
  fotos:/app/uploads
```

**Azure: Azure Blob Storage**
```python
from azure.storage.blob import BlobServiceClient

blob_service = BlobServiceClient.from_connection_string(os.getenv("AZURE_STORAGE_CONNECTION_STRING"))
container_client = blob_service.get_container_client("achados-fotos")

# Upload
with open(foto_path, "rb") as data:
    container_client.upload_blob(name=nome_foto, data=data)

# Download
blob_client = container_client.get_blob_client(nome_foto)
return blob_client.download_blob().readall()
```

**AWS: Amazon S3**
```python
import boto3

s3 = boto3.client('s3')

# Upload
s3.put_object(Bucket='achados-fotos', Key=nome_foto, Body=foto.stream)

# Download (presigned URL - mais eficiente)
url = s3.generate_presigned_url('get_object', Params={'Bucket': 'achados-fotos', 'Key': nome_foto}, ExpiresIn=3600)
```

**Google Cloud: Cloud Storage**
```python
from google.cloud import storage

storage_client = storage.Client()
bucket = storage_client.bucket('achados-fotos')

# Upload
blob = bucket.blob(nome_foto)
blob.upload_from_file(foto.stream)

# Download (public URL com assinatura)
url = blob.generate_signed_url(expiration=3600)
```

---

#### 4. **rede_interna** → VPC/VNet

**Local:**
```yaml
networks:
  rede_interna:
    driver: bridge
```

**Nuvem:**
- **Azure VNet**: Subredes privadas, Network Security Groups (NSGs)
- **AWS VPC**: Subredes públicas/privadas, Security Groups
- **Google VPC**: Subredes globais, Firewall Rules

**Isolamento:**
- Contêiner `web` em subnet pública (com IP público)
- Banco de dados em subnet privada (sem IP público)
- Comunicação entre eles pela VPC (DNS interno)

---

#### 5. **Arquivo `.env`** → Secret Managers

**Local:**
```bash
# .env
POSTGRES_PASSWORD=SenhaSegura123!
```

**Azure Key Vault:**
```bash
# Criar segredo
az keyvault secret set --vault-name achados-kv --name db-password --value "SenhaSegura123!"

# Referenciar no código (Python)
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://achados-kv.vault.azure.net/", credential=credential)
senha = client.get_secret("db-password").value
```

**AWS Secrets Manager:**
```bash
# Criar segredo
aws secretsmanager create-secret --name achados/db-password --secret-string "SenhaSegura123!"

# Referenciar no código
import boto3

client = boto3.client('secretsmanager')
response = client.get_secret_value(SecretId='achados/db-password')
senha = response['SecretString']
```

**Google Secret Manager:**
```bash
# Criar segredo
echo -n "SenhaSegura123!" | gcloud secrets create db-password --data-file=-

# Referenciar no código
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = "projects/meu-projeto/secrets/db-password/versions/latest"
response = client.access_secret_version(request={"name": name})
senha = response.payload.data.decode('UTF-8')
```

---

#### 6. **Imagem construída localmente** → Container Registries

**Local:**
```bash
docker build -t achados-web:latest ./app
```

**Azure Container Registry (ACR):**
```bash
# Login
az acr login --name acrachados

# Build remoto (não precisa do Docker localmente!)
az acr build --registry acrachados --image achados-web:latest ./app
```

**Amazon ECR:**
```bash
# Login
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Build e push
docker build -t achados-web:latest ./app
docker tag achados-web:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/achados-web:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/achados-web:latest
```

**Google Artifact Registry:**
```bash
# Build com Cloud Build (build remoto)
gcloud builds submit --tag gcr.io/meu-projeto/achados-web:latest ./app
```

---

### 🔄 Qual peça muda MAIS ao migrar?

**Resposta: Volume `fotos` → Object Storage (S3/Blob/GCS)**

---

### Por quê?

**1. Mudança de paradigma:**

**Local (volume):**
- Filesystem tradicional (POSIX)
- Operações: `open()`, `read()`, `write()`, `close()`
- Paths: `/app/uploads/foto.jpg`

**Nuvem (object storage):**
- API HTTP/REST
- Operações: `put_object()`, `get_object()`, `delete_object()`
- Keys: `s3://bucket/foto.jpg`

---

**2. Refatoração de código:**

**Local (send_from_directory):**
```python
@app.get("/fotos/<nome>")
def foto(nome):
    return send_from_directory(UPLOAD_DIR, nome)
```

**Nuvem (presigned URLs):**
```python
@app.get("/fotos/<nome>")
def foto(nome):
    url = s3.generate_presigned_url('get_object', Params={'Bucket': 'fotos', 'Key': nome}, ExpiresIn=3600)
    return redirect(url)  # Cliente baixa direto do S3 (mais eficiente!)
```

**Mudanças:**
- ✅ Adicionar biblioteca: `boto3`, `azure-storage-blob` ou `google-cloud-storage`
- ✅ Substituir `foto.save()` por `s3.put_object()`
- ✅ Substituir `send_from_directory()` por presigned URLs
- ✅ Implementar tratamento de erros (404, permissões)

---

**3. Configuração de permissões:**

**Local:** Qualquer contêiner na rede acessa o volume

**Nuvem (IAM):**

**AWS (políticas S3):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:PutObject", "s3:GetObject"],
    "Resource": "arn:aws:s3:::achados-fotos/*"
  }]
}
```

**Azure (RBAC):**
```bash
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee <managed-identity-id> \
  --scope /subscriptions/.../resourceGroups/.../providers/Microsoft.Storage/storageAccounts/achadosstorage
```

**Google (IAM):**
```bash
gcloud projects add-iam-policy-binding meu-projeto \
  --member="serviceAccount:achados-web@meu-projeto.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

---

**4. Lifecycle policies (retenção automática):**

**Local:** Manual (cron job deleta arquivos antigos)

**Nuvem:** Políticas de ciclo de vida

**S3 Lifecycle:**
```json
{
  "Rules": [{
    "Id": "delete-old-photos",
    "Status": "Enabled",
    "Expiration": {"Days": 90}
  }]
}
```

**Resultado:** Fotos com mais de 90 dias são deletadas automaticamente (LGPD Art. 16).

---

**5. Custos visíveis:**

**Local:** Custo oculto (disco do servidor)

**Nuvem:** Cobrado por GB armazenado + requisições

**Exemplo AWS S3 (us-east-1):**
- Armazenamento: $0,023/GB/mês
- PUT/POST: $0,005 por 1.000 requisições
- GET: $0,0004 por 1.000 requisições

**Otimização:** Usar presigned URLs para download direto (evita passar pelo contêiner).

---

### 📊 Comparação de impacto das mudanças:

| Componente | Mudança de código | Mudança de configuração | Complexidade |
|------------|-------------------|-------------------------|--------------|
| Contêiner web | ⚠️ Mínima (variáveis de ambiente) | Alta (registry, orquestração) | Média |
| Banco de dados | ✅ Nenhuma (mesmo PostgreSQL) | Alta (connection string, VPC) | Baixa |
| **Volume fotos** | ❌ **ALTA (refatorar todas operações de arquivo)** | Alta (IAM, bucket) | **Alta** |
| Rede | ✅ Nenhuma (DNS ainda funciona) | Alta (VPC, Security Groups) | Baixa |
| Segredos | ⚠️ Baixa (trocar `os.getenv` por SDK) | Média (Key Vault, IAM) | Média |
| Imagem | ✅ Nenhuma (mesmo Dockerfile) | Média (registry, CI/CD) | Baixa |

---

### 💡 Recomendações para migração:

**Fase 1: Lift-and-shift (mínimo esforço)**
- Contêineres → Fargate/Cloud Run
- Banco → RDS/Cloud SQL (dump/restore)
- Fotos → **Ainda em volume** (EFS/Azure Files)
- ⚠️ Funciona, mas não otimizado

**Fase 2: Otimização (aproveitar nuvem)**
- Fotos → **S3/Blob** (refatorar código)
- Segredos → Secret Manager
- CI/CD → GitHub Actions + ECR/ACR
- CDN → CloudFront/Azure CDN na frente do S3

**Fase 3: Cloud-native (escalabilidade)**
- Kubernetes (GKE/EKS/AKS)
- Autoscaling (baseado em CPU/requisições)
- Observabilidade (CloudWatch/Stackdriver)

---

**Conclusão:**

A migração para nuvem substitui cada componente Docker por um serviço gerenciado equivalente: contêineres vão para Fargate/Cloud Run/ACI, o banco vira RDS/Cloud SQL, volumes se tornam S3/Blob/GCS, a rede interna vira VPC/VNet, segredos migram para Key Vault/Secrets Manager, e imagens para ECR/ACR/Artifact Registry. A peça que **muda mais** é o volume `fotos` → object storage, porque exige refatoração de código (substituir `open()`/`write()` por APIs HTTP como `s3.put_object()`), implementação de presigned URLs para downloads diretos, configuração de permissões IAM/RBAC, e mudança de paradigma de filesystem POSIX para armazenamento de objetos baseado em chaves. Essa mudança traz benefícios (escalabilidade ilimitada, durabilidade de 11 noves, integração com CDN), mas é a mais trabalhosa em termos de refatoração de código.

---

**Documento atualizado em:** 19/10/2026
