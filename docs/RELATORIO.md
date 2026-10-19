# 📘 Relatório Técnico — Achados & Perdidos

Este documento explica em detalhes cada decisão técnica do projeto, desde as instruções do Dockerfile até o ciclo de vida dos volumes.

---

## Sumário

- [A) Explicação do Dockerfile](#a-explicação-do-dockerfile)
- [B) Explicação do docker-compose.yml](#b-explicação-do-docker-composeyml)
- [C) Caminho completo de uma requisição POST /itens](#c-caminho-completo-de-uma-requisição-post-itens)
- [D) Ciclo de vida dos volumes](#d-ciclo-de-vida-dos-volumes)

---

## A) Explicação do Dockerfile

O `app/Dockerfile` constrói a imagem do serviço web. Cada instrução foi escolhida por uma razão específica:

### Código completo comentado:

```dockerfile
# ============================================================
# 1. Imagem base
# ============================================================
FROM python:3.12-slim
```

**Por quê?**
- `python:3.12-slim`: Versão enxuta do Python 3.12, sem pacotes desnecessários
- **Tamanho**: ~130 MB (vs. ~900 MB da versão completa)
- **Segurança**: Menor superfície de ataque (menos pacotes = menos vulnerabilidades)

---

```dockerfile
# ============================================================
# 2. Variáveis de ambiente
# ============================================================
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
```

**Por quê?**
- `PYTHONDONTWRITEBYTECODE=1`: Não cria arquivos `.pyc` (bytecode compilado)
  - ➜ Reduz o tamanho da imagem
  - ➜ Evita problemas com cache desatualizado
- `PYTHONUNBUFFERED=1`: Saída de logs em tempo real (sem buffer)
  - ➜ Permite ver os logs imediatamente com `docker compose logs`
  - ➜ Essencial para debugging

---

```dockerfile
# ============================================================
# 3. Diretório de trabalho
# ============================================================
WORKDIR /app
```

**Por quê?**
- Define `/app` como diretório padrão para todos os comandos seguintes
- Cria o diretório automaticamente se não existir
- Todos os arquivos copiados vão para `/app`

---

```dockerfile
# ============================================================
# 4. Copiar APENAS o requirements.txt primeiro
# ============================================================
COPY requirements.txt .
```

**Por quê? (Cache de camadas)**

Esta é a **decisão mais importante** do Dockerfile. Explicação detalhada:

**❌ Se fizéssemos assim (ruim):**
```dockerfile
COPY . .
RUN pip install -r requirements.txt
```

**Problema:** Qualquer mudança no `app.py` invalida o cache e **reinstala todas as dependências** do zero (Flask, psycopg, Gunicorn), mesmo que o `requirements.txt` não tenha mudado.

**✅ Fazendo assim (bom):**
```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

**Benefício:** Mudanças no `app.py` **não** invalidam a camada de instalação das dependências. O Docker **reutiliza** a camada já construída.

**Ganho real:**
- Primeiro build: ~45 segundos
- Rebuilds após mudar `app.py`: ~3 segundos ⚡

---

```dockerfile
# ============================================================
# 5. Instalar dependências SEM guardar cache do pip
# ============================================================
RUN pip install --no-cache-dir -r requirements.txt
```

**Por quê?**
- `--no-cache-dir`: Não salva o cache do pip (~150 MB) na imagem
- **Reduz o tamanho final da imagem** em ~40%
- O cache do pip é útil localmente, mas inútil dentro de uma imagem imutável

---

```dockerfile
# ============================================================
# 6. Criar diretório de uploads
# ============================================================
RUN mkdir -p /app/uploads
```

**Por quê?**
- O código Python grava fotos em `/app/uploads` (definido em `UPLOAD_DIR`)
- Precisa existir antes de trocar para o usuário sem privilégios
- O `-p` evita erro se o diretório já existir

---

```dockerfile
# ============================================================
# 7. Criar usuário sem privilégios e atribuir propriedade
# ============================================================
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser && \
    chown -R appuser:appgroup /app/uploads
```

**Por quê? (RNF04 — Segurança)**

**Problema:** Por padrão, contêineres rodam como `root`. Se um atacante explorar uma vulnerabilidade na aplicação, ele tem **controle total** do contêiner.

**Solução:** Criar um usuário dedicado `appuser` sem privilégios de root.

**O que cada comando faz:**
1. `addgroup --system appgroup`: Cria o grupo `appgroup`
2. `adduser --system --ingroup appgroup appuser`: Cria o usuário `appuser` no grupo
   - `--system`: Usuário de sistema (sem senha, sem shell interativo)
3. `chown -R appuser:appgroup /app/uploads`: Dá propriedade do diretório de uploads ao `appuser`
   - Essencial porque só esse usuário vai gravar arquivos lá

**Verificação:**
```bash
docker compose exec web whoami
# Saída: appuser (não root!)
```

---

```dockerfile
# ============================================================
# 8. Copiar o restante do código
# ============================================================
COPY . .
```

**Por quê vem DEPOIS do pip install?**
- Aproveita o cache de camadas (explicado no item 4)
- Mudanças no código não forçam reinstalação das dependências

---

```dockerfile
# ============================================================
# 9. Trocar para o usuário sem privilégios
# ============================================================
USER appuser
```

**Por quê?**
- Todos os comandos seguintes (incluindo o `CMD`) rodam como `appuser`
- A aplicação Flask **não roda como root**
- Se houver vulnerabilidade explorada, o atacante fica limitado às permissões do `appuser`

---

```dockerfile
# ============================================================
# 10. Documentar a porta
# ============================================================
EXPOSE 5000
```

**Por quê?**
- **Documentação**: Indica que a aplicação escuta na porta 5000
- **Não publica** a porta (isso é feito no `docker-compose.yml`)
- Usado por ferramentas de análise e orquestradores

**Atenção:** O código `app.py` usa Gunicorn para escutar na porta 5000:
```python
# app.py não define porta; quem define é o Gunicorn no CMD
```

---

```dockerfile
# ============================================================
# 11. Iniciar a aplicação com Gunicorn
# ============================================================
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

**Por quê usar exec form (com colchetes)?**

**❌ Shell form (sem colchetes):**
```dockerfile
CMD gunicorn --bind 0.0.0.0:5000 app:app
```
- Cria um processo shell (`/bin/sh -c`) que inicia o Gunicorn
- O processo principal (PID 1) é o **shell**, não o Gunicorn
- Sinais (SIGTERM) não chegam corretamente ao Gunicorn
- `docker compose down` demora mais (timeout de 10s)

**✅ Exec form (com colchetes):**
```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```
- Gunicorn é o **processo principal** (PID 1)
- Recebe sinais corretamente
- Shutdown gracioso e rápido

**O que cada argumento faz:**
- `gunicorn`: Servidor de produção WSGI (melhor que o servidor dev do Flask)
- `--bind 0.0.0.0:5000`: Escuta em todas as interfaces na porta 5000
  - `0.0.0.0` é essencial (não pode ser `127.0.0.1`, senão o Docker não acessa)
- `app:app`: Formato `módulo:aplicação`
  - Primeiro `app`: arquivo `app.py`
  - Segundo `app`: variável `app = Flask(__name__)` dentro do arquivo

---

### 📊 Resumo da ordem das instruções:

```
1. FROM          → Imagem base
2. ENV           → Configurações de ambiente
3. WORKDIR       → Define diretório de trabalho
4. COPY req.txt  → Copia APENAS dependências (cache!)
5. RUN pip       → Instala dependências (camada pesada)
6. RUN mkdir     → Cria diretório de uploads
7. RUN adduser   → Cria usuário sem privilégios
8. COPY . .      → Copia código (camada leve, muda sempre)
9. USER          → Troca para usuário sem privilégios
10. EXPOSE       → Documenta porta
11. CMD          → Comando de inicialização
```

**Por que essa ordem?**
- ✅ Camadas pesadas (pip install) antes das leves (COPY código)
- ✅ Aproveita cache de camadas
- ✅ Segurança (USER no final, afetando CMD)

---

## B) Explicação do docker-compose.yml

O `docker-compose.yml` orquestra os dois serviços (web + db) e gerencia volumes e redes.

### Estrutura geral:

```yaml
services:   # Define os contêineres
  db:       # Banco PostgreSQL
  web:      # Aplicação Flask
  adminer:  # Painel (bônus)
volumes:    # Armazenamento persistente
networks:   # Comunicação entre contêineres
```

---

### 🗄️ Serviço: **db** (Banco de dados)

```yaml
db:
  image: postgres:16-alpine
```

**Por quê?**
- `postgres:16`: Versão estável e atual do PostgreSQL
- `alpine`: Variante minimalista (5 MB base vs. 120 MB da versão Debian)
- Imagem oficial, mantida pela comunidade PostgreSQL

---

```yaml
  container_name: achados-db
```

**Por quê?**
- Nome fixo do contêiner (facilita identificação em `docker ps`)
- Sem isso, o Docker geraria nomes como `achados-perdidos-db-1`

---

```yaml
  restart: unless-stopped
```

**Por quê?**
- Reinicia automaticamente se o contêiner cair (crash, erro)
- **Exceto** se o usuário parou manualmente com `docker compose stop`
- Essencial para ambientes de produção

**Outras opções:**
- `no`: Nunca reinicia (padrão)
- `always`: Sempre reinicia (mesmo após `docker compose stop`)
- `on-failure`: Reinicia apenas se o código de saída for erro

---

```yaml
  environment:
    POSTGRES_DB: ${POSTGRES_DB}
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

**Por quê?**
- Configurações lidas do arquivo `.env` (RNF03 — Segurança)
- A imagem `postgres` cria automaticamente o banco, usuário e senha na primeira execução
- **Nenhuma senha escrita diretamente no arquivo!**

**Como funciona:**
1. Docker Compose lê o `.env`
2. Substitui `${POSTGRES_DB}` pelo valor real (`achados_perdidos`)
3. Passa as variáveis para o contêiner

---

```yaml
  volumes:
    - dados_banco:/var/lib/postgresql/data
    - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
```

**Por quê dois tipos de volume?**

**1. Volume nomeado (`dados_banco`):**
- Armazena os dados do PostgreSQL (`/var/lib/postgresql/data`)
- **Persistente**: sobrevive a `docker compose down`
- **Gerenciado pelo Docker**: não precisa criar manualmente
- Localização física: `\\wsl$\docker-desktop-data\data\docker\volumes\` (Windows)

**2. Bind mount (`./db/init.sql`):**
- Mapeia arquivo local para dentro do contêiner
- **Read-only (`:ro`)**: o contêiner não pode modificar o arquivo
- **Executado apenas na primeira vez**: quando o banco é criado
- PostgreSQL procura scripts em `/docker-entrypoint-initdb.d/` e executa na ordem alfabética

**Por que init.sql não usa volume nomeado?**
- É um arquivo de **configuração**, não **dados**
- Precisa estar versionado no Git
- Mudanças no arquivo devem ser refletidas imediatamente

---

```yaml
  networks:
    - rede_interna
```

**Por quê?**
- Conecta o contêiner à rede `rede_interna` (definida no final do arquivo)
- Permite que `web` encontre `db` pelo nome (DNS interno)

---

```yaml
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s
    timeout: 5s
    retries: 5
    start_period: 10s
```

**Por quê? (RNF05)**

**Problema:** O PostgreSQL demora alguns segundos para aceitar conexões após o processo iniciar. Se a aplicação tentar conectar antes, recebe erro "connection refused".

**Solução:** Healthcheck verifica se o banco está **realmente pronto**.

**O que cada parâmetro faz:**
- `test`: Comando que verifica saúde
  - `pg_isready`: Utilitário do PostgreSQL que testa conexão
  - Retorna 0 se ok, 1 se falhou
- `interval: 5s`: Testa a cada 5 segundos
- `timeout: 5s`: Desiste após 5 segundos sem resposta
- `retries: 5`: 5 tentativas antes de marcar como "unhealthy"
- `start_period: 10s`: Período de graça (falhas não contam nos primeiros 10s)

**Estados possíveis:**
- `starting`: Ainda no período de graça
- `healthy`: pg_isready retornou 0
- `unhealthy`: 5 tentativas falharam

**Verificação:**
```bash
docker compose ps
# STATUS: Up (healthy) ✅
```

---

```yaml
  # NENHUMA porta publicada no host (RNF: banco isolado)
```

**Por quê não tem `ports:`?**

**Segurança:** O banco **não deve** ser acessível de fora do Docker.

**❌ Se fizéssemos:**
```yaml
ports:
  - "5432:5432"
```
- Qualquer um na rede local poderia tentar se conectar
- Aumenta superfície de ataque (força bruta, exploits)
- Viola princípio do menor privilégio

**✅ Como está (sem `ports`):**
- Apenas contêineres na mesma rede conseguem acessar
- Isolamento total do host

**Como `web` se conecta então?**
- Através da rede `rede_interna`
- DNS interno resolve `db` para o IP privado do contêiner

---

### 🌐 Serviço: **web** (Aplicação Flask)

```yaml
web:
  build: ./app
```

**Por quê `build` em vez de `image`?**
- `image: postgres:16-alpine` → Baixa imagem pronta do Docker Hub
- `build: ./app` → **Constrói** a imagem a partir do `app/Dockerfile`
- Necessário porque nossa aplicação é customizada

**O que acontece:**
1. Docker Compose procura `./app/Dockerfile`
2. Executa o build com contexto em `./app/`
3. Gera uma imagem chamada `achados-perdidos-web`

---

```yaml
  ports:
    - "8080:5000"
```

**Por quê?**
- **Publica** a porta do contêiner no host
- Formato: `HOST:CONTAINER`
  - `8080`: Porta no seu computador
  - `5000`: Porta dentro do contêiner (onde o Gunicorn escuta)

**Como funciona:**
```
Navegador → localhost:8080 → Docker → achados-web:5000 → Gunicorn → Flask
```

**Por que 8080 e não 5000 no host?**
- Convenção: portas < 1024 requerem root
- 8080 é porta padrão para aplicações web de desenvolvimento
- Evita conflito se você tiver outro Flask rodando localmente na 5000

---

```yaml
  environment:
    DB_HOST: db
    DB_PORT: "5432"
    DB_NAME: ${POSTGRES_DB}
    DB_USER: ${POSTGRES_USER}
    DB_PASSWORD: ${POSTGRES_PASSWORD}
    UPLOAD_DIR: /app/uploads
```

**Por quê?**

O `app.py` lê essas variáveis:
```python
DB_URL = (f"host={os.getenv('DB_HOST', 'db')} port={os.getenv('DB_PORT', '5432')} "
          f"dbname={os.getenv('DB_NAME')} user={os.getenv('DB_USER')} "
          f"password={os.getenv('DB_PASSWORD')}")
```

**Valores importantes:**
- `DB_HOST: db` → Nome do serviço PostgreSQL (DNS interno!)
- `DB_PORT: "5432"` → Porta interna (não publicada no host)
- Senhas vêm do `.env` (RNF03)

**Por que `DB_HOST` é `db` e não um IP?**
- Docker cria um **DNS interno** na rede `rede_interna`
- Resolve `db` para o IP privado do contêiner (ex: `172.18.0.2`)
- Se usar IP direto, pode mudar a cada `docker compose up`

---

```yaml
  volumes:
    - fotos:/app/uploads
```

**Por quê?**
- Volume nomeado `fotos` montado em `/app/uploads`
- **Persistência**: fotos sobrevivem a `docker compose down`
- Quando o código faz `foto.save(os.path.join(UPLOAD_DIR, nome_foto))`, grava no volume

**Diferença para bind mount:**
- Bind mount: `./local:/container` → Arquivo local específico
- Volume nomeado: `nome:/container` → Gerenciado pelo Docker

---

```yaml
  depends_on:
    db:
      condition: service_healthy
```

**Por quê? (RNF05)**

**❌ Se fosse apenas `depends_on: [db]`:**
- Docker garante que `db` **inicia antes** de `web`
- Mas não garante que `db` está **pronto para conexões**
- Resultado: `web` tenta conectar e recebe "connection refused"

**✅ Com `condition: service_healthy`:**
- Docker espera o healthcheck do `db` retornar `healthy`
- Só então inicia o `web`
- Garante que o PostgreSQL aceita conexões

**Verificação:**
```bash
docker compose logs web
# Não deve ter erro "could not connect to server"
```

---

### 🎁 Serviço: **adminer** (Desafio bônus)

```yaml
adminer:
  image: adminer:4.8.1
  container_name: achados-adminer
  restart: unless-stopped
  ports:
    - "8081:8080"
  networks:
    - rede_interna
  depends_on:
    db:
      condition: service_healthy
  profiles:
    - debug
```

**Por quê `profiles: [debug]`?**

**Problema:** Não queremos o Adminer sempre rodando (consome recursos).

**Solução:** Profile `debug` faz o serviço subir **apenas quando solicitado**.

**Uso:**
```bash
# Sem Adminer:
docker compose up -d

# Com Adminer:
docker compose --profile debug up -d
```

**Acesso:** `http://localhost:8081`
- Sistema: PostgreSQL
- Servidor: `db` (não `localhost`!)
- Credenciais: do `.env`

---

### 💾 Volumes

```yaml
volumes:
  dados_banco:
  fotos:
```

**Por quê declarar aqui?**
- Cria volumes **nomeados** (não anônimos)
- Docker gerencia o ciclo de vida
- Compartilháveis entre múltiplos contêineres

**Onde ficam fisicamente?**
- **Linux**: `/var/lib/docker/volumes/achados-perdidos_dados_banco/_data`
- **Windows**: `\\wsl$\docker-desktop-data\data\docker\volumes\`
- **Mac**: `/var/lib/docker/volumes/`

**Inspeção:**
```bash
docker volume ls
docker volume inspect achados-perdidos_fotos
```

---

### 🌐 Networks

```yaml
networks:
  rede_interna:
    driver: bridge
```

**Por quê criar rede própria?**

**❌ Rede padrão (`bridge`):**
- Contêineres **não** têm DNS interno
- Precisaria usar IP direto (`172.17.0.2`)
- IPs mudam a cada reinicialização

**✅ Rede definida pelo usuário (`rede_interna`):**
- **DNS interno automático**: `db` resolve para o IP do contêiner
- Isolamento de outros projetos Docker
- Controle de quem pode acessar

**Driver `bridge`:**
- Rede virtual dentro do Docker
- Contêineres se comunicam como se estivessem na mesma LAN
- Isolados do host (exceto portas publicadas)

---

## C) Caminho completo de uma requisição POST /itens

Vamos rastrear **exatamente** o que acontece quando você executa:

```bash
curl -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" \
     http://localhost:8080/itens
```

---

### 📍 Passo 1: Requisição sai do terminal

```
[Terminal] → curl envia requisição HTTP multipart/form-data
```

**Conteúdo:**
- Header: `Content-Type: multipart/form-data; boundary=---xyz123`
- Body:
  ```
  ---xyz123
  Content-Disposition: form-data; name="descricao"
  
  Oculos de grau
  ---xyz123
  Content-Disposition: form-data; name="local"
  
  Bloco B
  ---xyz123
  Content-Disposition: form-data; name="foto"; filename="oculos.jpg"
  Content-Type: image/jpeg
  
  [bytes da imagem]
  ---xyz123--
  ```

---

### 📍 Passo 2: Requisição chega ao host (localhost:8080)

```
[localhost:8080] → Porta do sistema operacional
```

**O que acontece:**
- SO recebe pacote TCP na porta 8080
- Verifica que está publicada pelo Docker
- Repassa para o Docker Engine

---

### 📍 Passo 3: Docker Engine roteia para o contêiner

```
[Docker Engine] → Consulta mapeamento de portas
                → Encaminha para achados-web:5000
```

**Como:**
- Docker mantém regras de iptables (Linux) ou NAT (Windows)
- Traduz `localhost:8080` → `172.18.0.3:5000` (IP interno do contêiner)
- Injeta pacote na rede `rede_interna`

---

### 📍 Passo 4: Gunicorn recebe a requisição

```
[achados-web:5000] → Gunicorn (servidor WSGI)
```

**O que acontece:**
1. Gunicorn escuta em `0.0.0.0:5000` (todas as interfaces)
2. Recebe conexão TCP
3. Lê a requisição HTTP completa
4. Identifica: `POST /itens`
5. Passa para a aplicação Flask (WSGI)

---

### 📍 Passo 5: Flask processa a rota

```python
# app.py:37
@app.post("/itens")
def cadastrar():
    descricao = request.form.get("descricao")  # "Oculos de grau"
    local = request.form.get("local")          # "Bloco B"
```

**O que acontece:**
1. Flask identifica o handler da rota `/itens` com método `POST`
2. Parseia o `multipart/form-data`
3. Extrai campos `descricao` e `local`

---

### 📍 Passo 6: Validação dos campos

```python
if not descricao or not local:
    return jsonify(erro="campos 'descricao' e 'local' sao obrigatorios"), 400
```

**Validação passa** → Continua

---

### 📍 Passo 7: Processar a foto

```python
nome_foto = None
foto = request.files.get("foto")
if foto and foto.filename:
    ext = os.path.splitext(foto.filename)[1].lower()  # ".jpg"
    nome_foto = f"{uuid.uuid4().hex}{ext}"              # "a3f8e9d1c2b4.jpg"
    foto.save(os.path.join(UPLOAD_DIR, nome_foto))
```

**O que acontece:**
1. `request.files.get("foto")` → Objeto `FileStorage` do Werkzeug
2. Extrai extensão: `.jpg`
3. Gera nome único: `uuid.uuid4().hex` → `a3f8e9d1c2b4`
4. Nome final: `a3f8e9d1c2b4.jpg`
5. **Grava no disco:**
   ```python
   # UPLOAD_DIR = "/app/uploads"
   foto.save("/app/uploads/a3f8e9d1c2b4.jpg")
   ```

---

### 📍 Passo 8: Arquivo gravado no volume Docker

```
[/app/uploads/a3f8e9d1c2b4.jpg] → Volume "fotos"
```

**Caminho real:**
- Dentro do contêiner: `/app/uploads/a3f8e9d1c2b4.jpg`
- No host (Windows): `\\wsl$\docker-desktop-data\data\docker\volumes\achados-perdidos_fotos\_data\a3f8e9d1c2b4.jpg`

**Por quê persiste?**
- Volume nomeado `fotos` é montado em `/app/uploads`
- Dados escritos lá vão para o volume, não para o filesystem efêmero do contêiner
- Sobrevive a `docker compose down`

---

### 📍 Passo 9: Conectar ao banco de dados

```python
with conectar() as c:
```

**O que `conectar()` faz:**
```python
def conectar():
    return psycopg.connect(DB_URL)

# DB_URL construído a partir de:
# host=db port=5432 dbname=achados_perdidos user=achados_user password=SenhaSegura123!
```

**Caminho de rede:**
```
[achados-web] → rede_interna → DNS resolve "db" → 172.18.0.2:5432 → [achados-db]
```

**Importante:**
- `host=db` não é um domínio da internet!
- É o **DNS interno do Docker** na rede `rede_interna`
- Docker resolve para o IP privado do contêiner `achados-db`

---

### 📍 Passo 10: Inserir no banco PostgreSQL

```python
novo_id = c.execute(
    "INSERT INTO itens (descricao, local, foto) VALUES (%s, %s, %s) RETURNING id",
    (descricao, local, nome_foto)  # ("Oculos de grau", "Bloco B", "a3f8e9d1c2b4.jpg")
).fetchone()[0]
```

**O que acontece:**
1. **Driver psycopg** envia a query SQL para o PostgreSQL
2. PostgreSQL **executa:**
   ```sql
   INSERT INTO itens (descricao, local, foto) 
   VALUES ('Oculos de grau', 'Bloco B', 'a3f8e9d1c2b4.jpg') 
   RETURNING id;
   ```
3. PostgreSQL **grava** no disco:
   - Arquivo: `/var/lib/postgresql/data/base/16384/...` (no volume `dados_banco`)
4. PostgreSQL **retorna** o ID gerado: `4`

---

### 📍 Passo 11: Responder ao cliente

```python
return jsonify(id=novo_id, foto=nome_foto), 201
```

**O que acontece:**
1. Flask serializa para JSON:
   ```json
   {
     "id": 4,
     "foto": "a3f8e9d1c2b4.jpg"
   }
   ```
2. Adiciona headers:
   ```
   HTTP/1.1 201 Created
   Content-Type: application/json
   Content-Length: 45
   ```
3. Passa para o Gunicorn

---

### 📍 Passo 12: Gunicorn envia resposta

```
[Gunicorn] → Socket TCP → Docker → Host → curl
```

**Caminho reverso:**
- Gunicorn escreve no socket
- Docker roteia de volta pela porta publicada
- Host entrega ao curl

---

### 📍 Passo 13: curl exibe a resposta

```bash
$ curl -F ... http://localhost:8080/itens
{"id":4,"foto":"a3f8e9d1c2b4.jpg"}
```

---

### 📊 Diagrama de sequência completo:

```
Terminal     Host      Docker    achados-web    rede_interna    achados-db      Volumes
   |           |          |            |              |              |              |
   |-- POST -->|          |            |              |              |              |
   |           |--------->|            |              |              |              |
   |           |          |-- roteou ->|              |              |              |
   |           |          |            |-- resolve -->|              |              |
   |           |          |            |          "db"=172.18.0.2    |              |
   |           |          |            |-- grava foto ------------------------------->|
   |           |          |            |              |              |    /app/uploads
   |           |          |            |------------- INSERT -------->|              |
   |           |          |            |              |    executa   |              |
   |           |          |            |              |      |---------------------->|
   |           |          |            |              |      grava   |   dados_banco
   |           |          |            |<------------ id=4 -----------|              |
   |           |          |<-- JSON ---|              |              |              |
   |<---------- 201 ------|            |              |              |              |
{"id":4,...}  |          |            |              |              |              |
```

---

### ⏱️ Tempo estimado de cada passo:

1. curl → host: **< 1 ms** (localhost)
2. Docker roteamento: **< 1 ms**
3. Gunicorn recebe: **< 1 ms**
4. Flask parseia: **~5 ms** (multipart parsing)
5. Grava foto: **~10 ms** (I/O de disco)
6. Resolve DNS "db": **< 1 ms** (cache interno)
7. Conecta ao PostgreSQL: **~2 ms** (conexão TCP + autenticação)
8. INSERT no banco: **~15 ms** (escrita + índice)
9. Resposta volta: **~2 ms**

**Total:** ~40 ms ⚡

---

## D) Ciclo de vida dos volumes

Baseado nas evidências da **Tarefa 5**, vamos entender o comportamento dos volumes em três situações.

---

### 📋 Situação 1: Recriação dos contêineres

**Comandos executados:**
```bash
# 1. Cadastrar dois objetos com foto
curl -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
curl -F "descricao=Mochila preta" -F "local=Cantina" -F "foto=@mochila.jpg" http://localhost:8080/itens

# 2. Verificar estatísticas
curl http://localhost:8080/estatisticas
# Resposta: {"itens_no_banco": 5, "fotos_no_volume": 2}

# 3. Derrubar contêineres
docker compose down

# 4. Subir novamente
docker compose up -d

# 5. Verificar novamente
curl http://localhost:8080/estatisticas
# Resposta: {"itens_no_banco": 5, "fotos_no_volume": 2}
```

**✅ Resultado: Itens e fotos CONTINUAM LÁ!**

**Por quê?**

1. **`docker compose down` faz:**
   - Para os contêineres
   - Remove os contêineres
   - Remove a rede `rede_interna`
   - **NÃO remove os volumes** (apenas com `-v`)

2. **Volumes são independentes dos contêineres:**
   ```
   [Contêiner achados-web] ──┐
   [Contêiner achados-db]   ─┼─> [Volume fotos] (persiste)
                             └─> [Volume dados_banco] (persiste)
   ```

3. **`docker compose up -d` faz:**
   - Cria novos contêineres
   - **Monta os mesmos volumes** existentes
   - Os dados já estão lá!

**Analogia:**
- Contêiner = Computador
- Volume = HD externo
- Desligar o computador não apaga o HD externo
- Ligar um computador novo com o mesmo HD recupera os dados

---

### 📋 Situação 2: Inspeção dos volumes

**Comandos executados:**
```bash
# 1. Listar volumes
docker volume ls
# Saída:
# DRIVER    VOLUME NAME
# local     achados-perdidos_dados_banco
# local     achados-perdidos_fotos

# 2. Inspecionar volume de fotos
docker volume inspect achados-perdidos_fotos
```

**Resposta (JSON):**
```json
[
  {
    "CreatedAt": "2026-10-19T00:50:23Z",
    "Driver": "local",
    "Labels": {
      "com.docker.compose.project": "achados-perdidos",
      "com.docker.compose.version": "5.5.1",
      "com.docker.compose.volume": "fotos"
    },
    "Mountpoint": "/var/lib/docker/volumes/achados-perdidos_fotos/_data",
    "Name": "achados-perdidos_fotos",
    "Options": null,
    "Scope": "local"
  }
]
```

**Informações importantes:**

- **`Mountpoint`**: Localização física no host
  - Linux: `/var/lib/docker/volumes/achados-perdidos_fotos/_data`
  - Windows: `\\wsl$\docker-desktop-data\data\docker\volumes\achados-perdidos_fotos\_data`
  - Mac: `/var/lib/docker/volumes/achados-perdidos_fotos/_data`

- **`Labels`**: Metadados do Docker Compose
  - Identifica que o volume pertence ao projeto `achados-perdidos`
  - Criado pela versão `5.5.1` do Compose

**Listar conteúdo do volume de fotos:**
```bash
docker compose exec web ls -l /app/uploads
```

**Saída:**
```
total 24
-rw-r--r-- 1 appuser appgroup  3821 Oct 19 01:15 a3f8e9d1c2b4.jpg
-rw-r--r-- 1 appuser appgroup 15039 Oct 19 01:17 f1a2b3c4d5e6.jpg
```

**Observações:**
- Proprietário: `appuser` (usuário sem privilégios definido no Dockerfile)
- Permissões: `-rw-r--r--` (leitura e escrita para o dono, apenas leitura para outros)

---

### 📋 Situação 3: Remoção dos volumes

**Comandos executados:**
```bash
# 1. Verificar estatísticas antes
curl http://localhost:8080/estatisticas
# Resposta: {"itens_no_banco": 5, "fotos_no_volume": 2}

# 2. Derrubar E REMOVER volumes
docker compose down -v

# 3. Subir novamente
docker compose up -d

# 4. Verificar estatísticas após
curl http://localhost:8080/estatisticas
# Resposta: {"itens_no_banco": 3, "fotos_no_volume": 0}

# 5. Listar itens
curl http://localhost:8080/itens
```

**Resposta:**
```json
[
  {"id": 1, "descricao": "Garrafa termica azul", "local": "Biblioteca", "foto": null, "criado_em": "2026-10-19T01:45:12"},
  {"id": 2, "descricao": "Carregador de notebook USB-C", "local": "Laboratorio 3", "foto": null, "criado_em": "2026-10-19T01:45:12"},
  {"id": 3, "descricao": "Casaco preto com capuz", "local": "Cantina", "foto": null, "criado_em": "2026-10-19T01:45:12"}
]
```

**✅ Resultado: Apenas os 3 itens de exemplo voltaram, SEM as fotos!**

---

### 🔍 Por quê isso aconteceu?

**1. `docker compose down -v` faz:**
```bash
# Remove contêineres
# Remove rede
# Remove volumes ← DIFERENÇA!
```

**Antes:**
```
[Volume dados_banco]     [Volume fotos]
  5 itens cadastrados      2 fotos
  (3 do init.sql + 2 novos)
```

**Depois:**
```
[Volume dados_banco]     [Volume fotos]
  APAGADO!                 APAGADO!
```

---

**2. `docker compose up -d` cria volumes vazios:**

```
[Volume dados_banco]     [Volume fotos]
  VAZIO                    VAZIO
```

---

**3. O PostgreSQL detecta volume vazio e executa inicialização:**

A imagem `postgres:16-alpine` tem esta lógica:
```bash
if [ ! -f "/var/lib/postgresql/data/PG_VERSION" ]; then
  # Banco não existe, inicializar
  initdb ...
  # Executar scripts em /docker-entrypoint-initdb.d/
  psql < /docker-entrypoint-initdb.d/init.sql
fi
```

**Resultado:** Os 3 itens do `init.sql` são inseridos novamente.

---

**4. Os 2 itens cadastrados manualmente sumiram porque:**

- Foram gravados no volume `dados_banco` **antigo**
- Esse volume foi **destruído** por `docker compose down -v`
- O volume **novo** não tem esses dados

---

**5. As fotos sumiram porque:**

- Foram gravadas no volume `fotos` **antigo**
- Esse volume foi **destruído**
- O volume **novo** está vazio (`fotos_no_volume: 0`)

---

### 📊 Comparação dos comandos:

| Comando | Contêineres | Rede | Volumes | Dados persistem? |
|---------|------------|------|---------|------------------|
| `docker compose stop` | Para | Mantém | Mantém | ✅ Sim |
| `docker compose down` | Remove | Remove | **Mantém** | ✅ Sim |
| `docker compose down -v` | Remove | Remove | **Remove** | ❌ Não |

---

### 🎯 Quando usar cada comando:

**`docker compose stop`:**
- Para os contêineres temporariamente
- Mantém tudo (inclusive RAM dos contêineres)
- Use para pausar o ambiente

**`docker compose down`:**
- Para e remove os contêineres
- **Mantém os volumes** (dados seguros!)
- Use no dia a dia (libera recursos, mas preserva dados)

**`docker compose down -v`:**
- Para, remove contêineres **e volumes**
- **APAGA TUDO!**
- Use apenas para:
  - Resetar o ambiente do zero
  - Testar o `init.sql` novamente
  - Limpar antes de entregar o trabalho

---

### 🔄 Ciclo de vida completo dos volumes:

```
1. docker compose up -d
   ↓
   Volumes criados (se não existem)
   ↓
   Contêineres montam os volumes
   ↓
   Dados gravados nos volumes
   
2. docker compose down
   ↓
   Contêineres removidos
   ↓
   Volumes MANTIDOS ✅
   
3. docker compose up -d
   ↓
   Novos contêineres criados
   ↓
   Montam os MESMOS volumes
   ↓
   Dados intactos! ✅
   
4. docker compose down -v
   ↓
   Contêineres removidos
   ↓
   Volumes REMOVIDOS ❌
   
5. docker compose up -d
   ↓
   Volumes criados VAZIOS
   ↓
   init.sql executado novamente
   ↓
   Apenas dados de exemplo ⚠️
```

---

### 💡 Conclusão sobre persistência:

✅ **Volumes são a solução para dados persistentes em Docker:**
- Sobrevivem a recriação de contêineres
- Independentes do ciclo de vida dos contêineres
- Gerenciados centralmente pelo Docker

❌ **Sem volumes, tudo seria efêmero:**
- Dados gravados no filesystem do contêiner somem ao remover
- Cada `docker compose up` começaria do zero

🎯 **Regra de ouro:**
- Use `docker compose down` no dia a dia (mantém volumes)
- Use `docker compose down -v` apenas para resetar completamente

---

## 📚 Referências

- [Dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Compose specification](https://docs.docker.com/compose/compose-file/)
- [Docker volumes](https://docs.docker.com/storage/volumes/)
- [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres)
- [Gunicorn documentation](https://docs.gunicorn.org/)

---

**Documento atualizado em:** 19/10/2026
