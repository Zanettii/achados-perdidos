# 📚 Guia Completo para Leigos — Achados & Perdidos com Docker

## 🎯 Propósito deste documento

Este documento explica **de forma simples e detalhada** como funciona o sistema Achados & Perdidos empacotado em Docker. Se você nunca usou Docker ou não tem experiência técnica, este guia é para você.

---

## 📖 Índice

1. [O que é Docker e por que usar?](#1-o-que-é-docker-e-por-que-usar)
2. [Conceitos básicos explicados](#2-conceitos-básicos-explicados)
3. [Estrutura do projeto](#3-estrutura-do-projeto)
4. [Como funciona o sistema](#4-como-funciona-o-sistema)
5. [Passo a passo: Como usar](#5-passo-a-passo-como-usar)
6. [O que acontece por baixo dos panos](#6-o-que-acontece-por-baixo-dos-panos)
7. [Resolução de problemas comuns](#7-resolução-de-problemas-comuns)
8. [Glossário de termos técnicos](#8-glossário-de-termos-técnicos)

---

## 1. O que é Docker e por que usar?

### 🤔 A analogia do container de mercadoria

Imagine que você precisa enviar mercadorias para outro país. Antigamente, cada produto era embalado de forma diferente: caixas de madeira, sacos, barris... Isso tornava o transporte complicado e lento.

Então inventaram os **containers padronizados**: aquelas caixas enormes de metal que você vê em navios e caminhões. Não importa o que tem dentro — pode ser comida, eletrônicos, roupas — o container sempre tem o mesmo tamanho e encaixes. Qualquer navio consegue carregar, qualquer porto consegue descarregar.

**Docker faz a mesma coisa com software!**

---

### 💻 O problema antes do Docker

**Cenário típico:**

```
Desenvolvedor A (no notebook dele):
✅ Instala Python 3.12 manualmente
✅ Instala PostgreSQL 16
✅ Configura senha do banco
✅ Roda a aplicação
✅ Funciona perfeitamente! 🎉

Desenvolvedor B (em outro computador):
❌ Tenta rodar
❌ Erro: "Python 3.10 (versão diferente)"
❌ Erro: "PostgreSQL não instalado"
❌ Erro: "Senha do banco errada"
❌ "Mas no meu computador funciona!" 😭
```

**Resultado:** Dias perdidos configurando ambientes, documentação desatualizada, "funciona na minha máquina" virou piada interna.

---

### ✅ A solução com Docker

Com Docker, você **empacota tudo** em um "container":

```
Container = Aplicação + Python + PostgreSQL + Configurações + Dependências

┌──────────────────────────────────┐
│  Container "Achados & Perdidos"  │
│  ─────────────────────────────   │
│  ✅ Python 3.12                  │
│  ✅ PostgreSQL 16                │
│  ✅ Código da aplicação          │
│  ✅ Bibliotecas (Flask, etc.)    │
│  ✅ Configurações                │
└──────────────────────────────────┘

Qualquer computador com Docker:
docker compose up → Funciona! ✅
```

**Vantagens:**

1. **Funciona em qualquer lugar:** Windows, Mac, Linux, servidor na nuvem
2. **Instalação com 1 comando:** `docker compose up`
3. **Ambiente idêntico:** Dev, teste e produção usam exatamente o mesmo container
4. **Não polui seu computador:** Tudo fica isolado dentro do Docker
5. **Fácil de limpar:** `docker compose down -v` apaga tudo

---

## 2. Conceitos básicos explicados

Vamos desmistificar os termos técnicos:

---

### 🐳 Docker

**O que é:** Um programa que cria e gerencia "containers" (ambientes isolados).

**Analogia:** É como ter um computador virtual dentro do seu computador, mas **muito mais leve** que uma máquina virtual tradicional (VirtualBox, VMware).

**Diferença para máquina virtual:**

```
Máquina Virtual (VirtualBox):
├── Seu Windows (Sistema operacional completo)
├── VirtualBox (programa)
└── Ubuntu dentro do VirtualBox (outro sistema operacional completo)
    └── Aplicação
    
👎 Pesado: 2-4 GB de RAM, 20+ GB de disco
👎 Lento para iniciar: 1-2 minutos
👎 Desperdício de recursos

Docker:
├── Seu Windows (Sistema operacional)
├── Docker (programa)
└── Container (só a aplicação + bibliotecas necessárias)
    └── Aplicação
    
👍 Leve: 100-500 MB de RAM, 1-2 GB de disco
👍 Rápido para iniciar: 1-5 segundos
👍 Eficiente
```

---

### 📦 Container (Contêiner)

**O que é:** Um "pacote" isolado que contém uma aplicação e tudo que ela precisa para rodar.

**Analogia:** Pense em um aquário. Dentro dele há peixes, plantas, água... Tudo que os peixes precisam está ali dentro, isolado do ambiente externo. Você pode ter vários aquários na mesma sala, e o que acontece em um não afeta os outros.

**Exemplo prático:**

```
Você tem 3 projetos no seu computador:

Projeto A: Python 3.10 + PostgreSQL 14
Projeto B: Python 3.12 + PostgreSQL 16  ← Nosso projeto!
Projeto C: Node.js 20 + MySQL 8

Sem Docker:
❌ Conflito de versões do Python
❌ Dois bancos PostgreSQL na mesma porta
❌ Bagunça total

Com Docker:
✅ Container A (isolado)
✅ Container B (isolado)
✅ Container C (isolado)
✅ Convivem pacificamente!
```

---

### 🖼️ Imagem Docker

**O que é:** A "receita" para criar um container. É como uma foto de um momento específico.

**Analogia:** Pense em uma receita de bolo:

```
Receita de Bolo (Imagem Docker):
1. Pegue farinha, ovos, açúcar
2. Misture tudo
3. Asse por 30 minutos
→ Resultado: BOLO

Dockerfile (nossa receita):
1. Pegue Python 3.12
2. Instale Flask, psycopg, Gunicorn
3. Copie o código da aplicação
→ Resultado: IMAGEM DOCKER

docker run (executar a receita):
Imagem → Container (o "bolo" pronto)
```

**Importante:** 
- **Imagem = leitura apenas** (não muda)
- **Container = instância executando** (pode mudar)

Você pode ter 1 imagem e criar 10 containers idênticos a partir dela!

---

### 🐳 Docker Compose

**O que é:** Uma ferramenta para gerenciar **múltiplos containers** de uma vez.

**Analogia:** Pense em uma orquestra. Você tem violinos, trombones, bateria... Cada um é um instrumento (container), mas precisa de um maestro (Docker Compose) para coordenar todos juntos.

**Nosso projeto tem 2 "instrumentos":**

1. **Container `web`:** Aplicação Flask (Python) que recebe as requisições HTTP
2. **Container `db`:** Banco de dados PostgreSQL que guarda os objetos cadastrados

**Sem Docker Compose (manual):**

```bash
# Terminal 1:
docker run --name meu-postgres -e POSTGRES_PASSWORD=senha123 -d postgres:16

# Terminal 2:
docker run --name minha-app --link meu-postgres -p 8080:5000 -d minha-imagem

# Parar tudo:
docker stop meu-postgres
docker stop minha-app
docker rm meu-postgres
docker rm minha-app
```

👎 Complicado, muitos comandos, esquece facilmente

**Com Docker Compose:**

```bash
# Subir TUDO:
docker compose up -d

# Parar TUDO:
docker compose down
```

👍 Simples, um comando só!

---

### 💾 Volume Docker

**O que é:** Um "pen drive virtual" onde o container guarda dados que **precisam sobreviver** quando o container é destruído.

**Por que existe?**

Containers são **efêmeros** (temporários). Quando você para e remove um container, **tudo que estava dentro dele some**!

**Problema:**

```
Você cadastra 10 objetos perdidos no sistema.
Os dados ficam salvos no container.

docker compose down
docker compose up

❌ Os 10 objetos sumiram! (container foi recriado do zero)
```

**Solução: Volumes**

```
┌─────────────┐        ┌──────────────┐
│  Container  │───────▶│   Volume     │
│             │  grava │  (pen drive) │
└─────────────┘        └──────────────┘

docker compose down  → Container MORRE, volume VIVE
docker compose up    → Novo container LENDO DO MESMO VOLUME

✅ Os 10 objetos voltaram! (estavam no volume)
```

**Nosso projeto tem 2 volumes:**

1. **`dados_banco`**: Guarda as tabelas do PostgreSQL (objetos cadastrados)
2. **`fotos`**: Guarda as fotos enviadas pelos usuários

---

### 🌐 Rede Docker

**O que é:** Uma "rede local virtual" onde containers conseguem conversar entre si.

**Analogia:** Pense em uma sala de reunião com 5 pessoas. Todas conseguem se ouvir porque estão na mesma sala. Se uma pessoa está em outra sala, não consegue ouvir.

**Nosso projeto:**

```
┌─────────────────────────────────────┐
│     rede_interna (sala de reunião)  │
│                                     │
│  ┌──────────┐      ┌──────────┐   │
│  │   web    │──────│    db    │   │
│  └──────────┘      └──────────┘   │
│                                     │
└─────────────────────────────────────┘

Seu computador (fora da sala) → Só acessa "web"
                                 "db" está isolado!
```

**DNS interno:**

Dentro da rede, os containers se chamam pelo **nome**, não por IP:

```python
# No código do container "web":
conectar("host=db")  # "db" é o nome do container PostgreSQL

# Docker resolve automaticamente:
"db" → IP interno 172.18.0.2
```

---

## 3. Estrutura do projeto

Vamos ver **cada arquivo** e para que serve:

```
achados-perdidos/
├── app/                          ← Código da aplicação web
│   ├── app.py                    ← Aplicação Flask (Python)
│   ├── requirements.txt          ← Lista de bibliotecas necessárias
│   ├── Dockerfile                ← Receita para criar a imagem "web"
│   └── .dockerignore             ← Arquivos ignorados no build
│
├── db/                           ← Banco de dados
│   └── init.sql                  ← Script de inicialização (cria tabela)
│
├── docs/                         ← Documentação
│   ├── arquitetura.md            ← Diagrama do sistema
│   ├── RELATORIO.md              ← Explicação técnica detalhada
│   ├── REFLEXAO.md               ← Questões de reflexão
│   └── evidencias/               ← Prints dos testes
│
├── docker-compose.yml            ← Orquestra os containers (maestro!)
├── .env                          ← Senhas (NÃO vai para o Git!)
├── .env.example                  ← Modelo de .env (sem senhas reais)
├── .gitignore                    ← O que o Git ignora
└── README.md                     ← Instruções de uso
```

---

### 📄 Explicação de cada arquivo:

---

#### `app/app.py` — O cérebro da aplicação

**O que é:** Código Python que recebe requisições HTTP e responde.

**O que faz:**

```python
GET /health          → "Estou vivo e conectado ao banco!"
GET /itens           → Lista objetos cadastrados
POST /itens          → Cadastra novo objeto (com foto)
GET /fotos/<nome>    → Mostra a foto
GET /estatisticas    → Total de itens e fotos
```

**Você NÃO precisa editar este arquivo!** Ele já está pronto.

---

#### `app/requirements.txt` — Lista de compras

**O que é:** Lista de bibliotecas (pacotes) Python que a aplicação precisa.

**Conteúdo:**

```
flask==3.0.3        ← Framework web (cria as rotas HTTP)
psycopg[binary]==3.2.3  ← Driver do PostgreSQL (conecta ao banco)
gunicorn==23.0.0    ← Servidor de produção (serve a aplicação)
```

**Analogia:** É como uma lista de ingredientes da receita de bolo. O Docker vai "no mercado" (internet) e baixa tudo automaticamente.

---

#### `app/Dockerfile` — Receita da imagem

**O que é:** Instruções para o Docker construir a imagem do container `web`.

**Resumo do que faz:**

```dockerfile
1. Pega Python 3.12 (base)
2. Instala as bibliotecas do requirements.txt
3. Cria um usuário sem privilégios de root (segurança)
4. Copia o código da aplicação
5. Define o comando para iniciar: Gunicorn
```

**Por que essa ordem?** Para aproveitar o **cache de camadas** (ver seção 6.5).

---

#### `db/init.sql` — Criação da tabela

**O que é:** Script SQL que cria a tabela `itens` e insere 3 exemplos.

**Conteúdo simplificado:**

```sql
-- Cria a tabela se não existir
CREATE TABLE IF NOT EXISTS itens (
    id SERIAL PRIMARY KEY,         ← Número único automático
    descricao VARCHAR(200),        ← "Garrafa térmica azul"
    local VARCHAR(100),            ← "Biblioteca"
    foto VARCHAR(100),             ← "a3f8e9d1c2b4.jpg"
    criado_em TIMESTAMP            ← Data/hora do cadastro
);

-- Insere 3 objetos de exemplo
INSERT INTO itens (descricao, local) VALUES
    ('Garrafa termica azul', 'Biblioteca'),
    ('Carregador de notebook USB-C', 'Laboratorio 3'),
    ('Casaco preto com capuz', 'Cantina');
```

**Quando roda?** Apenas na **primeira vez** que o banco é criado. Se você mudar este arquivo depois, a mudança NÃO é aplicada automaticamente (ver seção 6.3).

---

#### `docker-compose.yml` — O maestro

**O que é:** Arquivo de configuração que define os 2 containers e como eles se conectam.

**Estrutura simplificada:**

```yaml
services:
  db:                              # Container do PostgreSQL
    image: postgres:16-alpine      # Imagem oficial (baixa automaticamente)
    volumes:
      - dados_banco:/var/lib/postgresql/data  # Volume para dados
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  # Script inicial
    healthcheck:                   # Verifica se o banco está pronto
      test: pg_isready             # Comando de teste
    # SEM PORTA PUBLICADA (segurança)
  
  web:                             # Container da aplicação Flask
    build: ./app                   # Constrói a partir do Dockerfile
    ports:
      - "8080:5000"                # Publica porta 8080 no seu PC
    environment:                   # Variáveis de ambiente
      DB_HOST: db                  # Nome do container do banco
      DB_PASSWORD: ${POSTGRES_PASSWORD}  # Lê do arquivo .env
    volumes:
      - fotos:/app/uploads         # Volume para fotos
    depends_on:
      db:
        condition: service_healthy # Só inicia quando o banco estiver pronto

volumes:
  dados_banco:                     # Volume nomeado (gerenciado pelo Docker)
  fotos:                           # Volume nomeado

networks:
  rede_interna:                    # Rede privada entre containers
    driver: bridge
```

---

#### `.env` — Cofre de senhas

**O que é:** Arquivo com **credenciais reais** do banco de dados.

**Conteúdo:**

```env
POSTGRES_DB=achados_perdidos
POSTGRES_USER=achados_user
POSTGRES_PASSWORD=SenhaSegura123!
```

**⚠️ MUITO IMPORTANTE:**

- ❌ **NUNCA envie este arquivo para o Git!** (senhas devem ser secretas)
- ✅ Está no `.gitignore` (Git ignora automaticamente)
- ✅ Cada pessoa cria seu próprio `.env` localmente
- ✅ No servidor de produção, usa senhas diferentes

---

#### `.env.example` — Modelo sem senhas

**O que é:** Modelo do `.env` com valores **fictícios**.

**Conteúdo:**

```env
POSTGRES_DB=nome_do_banco
POSTGRES_USER=usuario
POSTGRES_PASSWORD=sua_senha_aqui
```

**Para que serve?** 

Quando outra pessoa clonar o projeto do Git, ela:
1. Copia o `.env.example` para `.env`
2. Substitui os valores fictícios por senhas reais
3. Pronto para usar!

---

## 4. Como funciona o sistema

Vamos entender o **fluxo completo** de uma requisição:

---

### 🌊 Exemplo: Cadastrar um objeto com foto

**Você executa:**

```bash
curl -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**O que acontece (passo a passo):**

---

#### **Passo 1: Navegador → Seu computador**

```
Seu navegador/terminal
    ↓
   HTTP POST com a foto (multipart/form-data)
    ↓
localhost:8080 (porta do SEU Windows)
```

**O que é `localhost`?** É um apelido para "este computador". É como se você ligasse para o seu próprio telefone.

---

#### **Passo 2: Windows → Docker**

```
Windows detecta: "Alguém está tentando acessar a porta 8080"
    ↓
Consulta: "Quem está usando essa porta?"
    ↓
Resposta: "O container achados-web"
    ↓
Docker roteia a requisição para dentro do container web
```

**Mapeamento de porta:**

```
localhost:8080 (seu computador)
     ↓
  Docker faz a ponte
     ↓
achados-web:5000 (dentro do container)
```

É como se o Docker fosse um **porteiro de prédio**: ele recebe a visita na portaria (porta 8080) e encaminha para o apartamento correto (container web, porta 5000).

---

#### **Passo 3: Gunicorn recebe a requisição**

```
Container web (achados-web):
    ↓
Gunicorn (servidor web) escutando na porta 5000
    ↓
"Recebi uma requisição POST /itens com dados de formulário"
    ↓
Repassa para o Flask (aplicação)
```

**O que é Gunicorn?** É um "garçom" que atende as requisições HTTP e entrega para o código Python processar.

---

#### **Passo 4: Flask processa a requisição**

```python
# app.py, linha 37:
@app.post("/itens")
def cadastrar():
    # 1. Extrair dados do formulário
    descricao = "Oculos de grau"
    local = "Bloco B"
    foto = [bytes da imagem]
    
    # 2. Gerar nome único para a foto
    nome_foto = "a3f8e9d1c2b4.jpg"  # UUID aleatório
    
    # 3. Salvar foto no disco
    foto.save("/app/uploads/a3f8e9d1c2b4.jpg")
```

**O que é UUID?** É um identificador único universal. É como o CPF da foto — garante que duas fotos nunca terão o mesmo nome, mesmo se forem enviadas no mesmo segundo.

---

#### **Passo 5: Gravar foto no volume**

```
foto.save("/app/uploads/a3f8e9d1c2b4.jpg")
    ↓
Docker intercepta: "/app/uploads está montado no volume 'fotos'"
    ↓
Grava no disco do seu computador:
    \\wsl$\docker-desktop-data\...\achados-perdidos_fotos\_data\a3f8e9d1c2b4.jpg
```

**Por que no volume?** Para que a foto **sobreviva** se o container for recriado!

---

#### **Passo 6: Conectar ao banco de dados**

```python
# app.py:
conectar("host=db, port=5432, user=achados_user, password=SenhaSegura123!")
    ↓
"db" (nome do container PostgreSQL)
    ↓
DNS interno do Docker resolve:
    "db" → 172.18.0.2 (IP privado dentro da rede_interna)
    ↓
Conexão TCP estabelecida: web (172.18.0.3) → db (172.18.0.2)
```

**Por que funciona?** Porque os dois containers estão na mesma **rede Docker** (`rede_interna`), que tem um **DNS automático**.

É como se fosse uma agenda telefônica interna: você disca "db" e o Docker sabe para qual número ligar.

---

#### **Passo 7: Inserir registro no banco**

```python
# app.py:
cursor.execute(
    "INSERT INTO itens (descricao, local, foto) VALUES (%s, %s, %s)",
    ("Oculos de grau", "Bloco B", "a3f8e9d1c2b4.jpg")
)
    ↓
PostgreSQL recebe o comando SQL
    ↓
Grava no disco (volume dados_banco)
    ↓
Retorna: id=4 (ID do novo registro)
```

**O que aconteceu?**

```
Tabela "itens" ANTES:
id | descricao                      | local          | foto
---+--------------------------------+----------------+------
1  | Garrafa termica azul           | Biblioteca     | null
2  | Carregador de notebook USB-C   | Laboratorio 3  | null
3  | Casaco preto com capuz         | Cantina        | null

Tabela "itens" DEPOIS:
id | descricao                      | local          | foto
---+--------------------------------+----------------+------------------
1  | Garrafa termica azul           | Biblioteca     | null
2  | Carregador de notebook USB-C   | Laboratorio 3  | null
3  | Casaco preto com capuz         | Cantina        | null
4  | Oculos de grau                 | Bloco B        | a3f8e9d1c2b4.jpg  ← NOVO!
```

---

#### **Passo 8: Responder ao cliente**

```python
return jsonify({"id": 4, "foto": "a3f8e9d1c2b4.jpg"}), 201
    ↓
Flask serializa para JSON
    ↓
Gunicorn envia resposta HTTP
    ↓
Docker roteia de volta para o host
    ↓
Seu terminal recebe:
```

```json
{
  "id": 4,
  "foto": "a3f8e9d1c2b4.jpg"
}
```

---

### 📊 Resumo visual do fluxo:

```
Seu computador (HOST)
    │
    │ 1. POST /itens (HTTP)
    ↓
localhost:8080
    │
    │ 2. Docker mapeia
    ↓
┌───────────────────────────────────────────────────┐
│             DOCKER                                │
│                                                   │
│  ┌──────────────┐         ┌──────────────┐       │
│  │  web:5000    │   SQL   │   db:5432    │       │
│  │ (Flask)      │────────▶│ (PostgreSQL) │       │
│  │              │         │              │       │
│  │ 3. Processa  │         │ 6. INSERT    │       │
│  │ 4. Grava foto│         │ 7. Retorna ID│       │
│  └──────┬───────┘         └──────┬───────┘       │
│         │                        │               │
│         ↓                        ↓               │
│  ┌────────────┐          ┌────────────┐         │
│  │   fotos    │          │dados_banco │         │
│  │ (volume)   │          │  (volume)  │         │
│  └────────────┘          └────────────┘         │
│  5. a3f8e9.jpg           8. Registro salvo      │
└───────────────────────────────────────────────────┘
    │
    │ 9. Resposta JSON
    ↓
Seu terminal: {"id": 4, "foto": "a3f8e9d1c2b4.jpg"}
```

---

## 5. Passo a passo: Como usar

Agora que você entendeu a teoria, vamos **usar o sistema**!

---

### ✅ Pré-requisitos

**Antes de começar, você precisa ter:**

1. **Docker Desktop** instalado
   - **Windows/Mac**: Baixe em [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
   - **Linux**: [docs.docker.com/engine/install](https://docs.docker.com/engine/install/)

2. **Terminal/PowerShell** aberto

3. **Projeto baixado** (clonar do Git ou extrair ZIP)

---

### 📥 Passo 1: Baixar o projeto

**Opção A: Via Git**

```bash
git clone https://github.com/seu-usuario/achados-perdidos.git
cd achados-perdidos
```

**Opção B: Via ZIP**

1. Baixe o arquivo ZIP
2. Extraia para uma pasta (ex: `C:\Users\Kerem\achados-perdidos`)
3. Abra o terminal na pasta

---

### 🔐 Passo 2: Configurar as credenciais

**Windows (PowerShell):**

```powershell
# Copiar o modelo
Copy-Item .env.example .env

# Editar com o Bloco de Notas
notepad .env
```

**Linux/Mac:**

```bash
# Copiar o modelo
cp .env.example .env

# Editar com nano
nano .env
```

**Conteúdo do `.env`** (substitua os valores):

```env
POSTGRES_DB=achados_perdidos
POSTGRES_USER=achados_user
POSTGRES_PASSWORD=MinhaSenh@Forte123
```

💡 **Dica:** Use uma senha forte (letras, números, símbolos).

**Salve e feche o arquivo.**

---

### 🚀 Passo 3: Subir o ambiente

**Comando mágico:**

```bash
docker compose up -d --build
```

**O que cada parte significa:**

- `docker compose`: Chama o Docker Compose (o maestro)
- `up`: "Suba o ambiente"
- `-d`: "Detached" (roda em background, não trava o terminal)
- `--build`: "Construa as imagens antes de subir"

---

**O que vai acontecer (leva ~1-2 minutos na primeira vez):**

```
[+] Building 47.3s (10/10) FINISHED
 => [web 1/6] FROM python:3.12-slim           ← Baixa Python
 => [web 4/6] RUN pip install ...             ← Instala bibliotecas
 => [web 6/6] COPY . .                        ← Copia código
[+] Running 5/5
 ✔ Network achados-perdidos_rede_interna      ← Cria rede
 ✔ Volume achados-perdidos_dados_banco        ← Cria volume do banco
 ✔ Volume achados-perdidos_fotos              ← Cria volume de fotos
 ✔ Container achados-db                       ← Sobe PostgreSQL
 ✔ Container achados-web                      ← Sobe Flask
```

---

### ✅ Passo 4: Verificar se está rodando

```bash
docker compose ps
```

**Saída esperada:**

```
NAME          IMAGE                    STATUS              PORTS
achados-db    postgres:16-alpine       Up (healthy)        -
achados-web   achados-perdidos-web     Up                  0.0.0.0:8080->5000/tcp
```

**O que significa:**

- `Up (healthy)`: Container rodando e saudável ✅
- `0.0.0.0:8080->5000/tcp`: Porta 8080 publicada ✅

---

### 🧪 Passo 5: Testar a aplicação

Agora vamos fazer requisições para o sistema!

---

#### **Teste 1: Verificar saúde**

```bash
curl http://localhost:8080/health
```

**Resposta esperada:**

```json
{
  "status": "ok",
  "banco": "conectado",
  "container": "achados-web"
}
```

✅ **Se viu isso, está funcionando!**

---

#### **Teste 2: Listar itens**

```bash
curl http://localhost:8080/itens
```

**Resposta esperada (3 itens de exemplo do `init.sql`):**

```json
[
  {
    "id": 1,
    "descricao": "Garrafa termica azul",
    "local": "Biblioteca",
    "foto": null,
    "criado_em": "2026-10-19T04:10:23"
  },
  {
    "id": 2,
    "descricao": "Carregador de notebook USB-C",
    "local": "Laboratorio 3",
    "foto": null,
    "criado_em": "2026-10-19T04:10:23"
  },
  {
    "id": 3,
    "descricao": "Casaco preto com capuz",
    "local": "Cantina",
    "foto": null,
    "criado_em": "2026-10-19T04:10:23"
  }
]
```

---

#### **Teste 3: Filtrar por local**

```bash
curl "http://localhost:8080/itens?local=biblio"
```

**Resposta (apenas itens da biblioteca):**

```json
[
  {
    "id": 1,
    "descricao": "Garrafa termica azul",
    "local": "Biblioteca",
    "foto": null,
    "criado_em": "2026-10-19T04:10:23"
  }
]
```

💡 **Note:** A busca é case-insensitive (`biblio` encontra `Biblioteca`).

---

#### **Teste 4: Cadastrar objeto COM foto**

Primeiro, crie uma imagem de teste:

**Windows (PowerShell):**

```powershell
# Criar uma imagem PNG simples
python -c "import struct,zlib;w=h=10;raw=b'';[raw:=raw+b'\x00'+bytes([255,0,0]*w) for _ in range(h)];idat=zlib.compress(raw);png=b'\x89PNG\r\n\x1a\n';ihdr=struct.pack('>IIBBBBB',w,h,8,2,0,0,0);png+=struct.pack('>I',13)+b'IHDR'+ihdr+struct.pack('>I',zlib.crc32(b'IHDR'+ihdr)&0xffffffff);png+=struct.pack('>I',len(idat))+b'IDAT'+idat+struct.pack('>I',zlib.crc32(b'IDAT'+idat)&0xffffffff);png+=struct.pack('>I',0)+b'IEND'+struct.pack('>I',zlib.crc32(b'IEND')&0xffffffff);open('oculos.jpg','wb').write(png)"
```

**Agora cadastre:**

```powershell
curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**Linux/Mac:**

```bash
# Criar imagem de teste
echo "fake image" > oculos.jpg

# Cadastrar
curl -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**Resposta:**

```json
{
  "id": 4,
  "foto": "a3f8e9d1c2b4.jpg"
}
```

✅ **Cadastrado com sucesso!**

---

#### **Teste 5: Ver estatísticas**

```bash
curl http://localhost:8080/estatisticas
```

**Resposta:**

```json
{
  "itens_no_banco": 4,
  "fotos_no_volume": 1
}
```

**Interpretação:**
- 4 itens cadastrados (3 de exemplo + 1 que você cadastrou)
- 1 foto armazenada (a que você enviou)

---

#### **Teste 6: Visualizar a foto no navegador**

Abra no navegador:

```
http://localhost:8080/fotos/a3f8e9d1c2b4.jpg
```

(Substitua `a3f8e9d1c2b4.jpg` pelo nome da foto que a resposta do POST retornou)

✅ **Você deve ver a imagem renderizada!**

---

### 🛑 Passo 6: Parar o ambiente

**Parar sem apagar dados:**

```bash
docker compose down
```

**O que acontece:**
- ✅ Para os containers
- ✅ Remove os containers
- ✅ **Mantém os volumes** (dados salvos!)

**Parar E apagar tudo (incluindo volumes):**

```bash
docker compose down -v
```

⚠️ **CUIDADO:** Isso **apaga todos os dados**! (objetos cadastrados, fotos)

---

### 🔄 Passo 7: Subir novamente

```bash
docker compose up -d
```

**Teste:**

```bash
curl http://localhost:8080/estatisticas
```

**Se você usou `down` (sem `-v`):**

```json
{
  "itens_no_banco": 4,
  "fotos_no_volume": 1
}
```

✅ **Os dados voltaram!** (estavam no volume)

**Se você usou `down -v`:**

```json
{
  "itens_no_banco": 3,
  "fotos_no_volume": 0
}
```

⚠️ **Apenas os 3 itens de exemplo voltaram** (do `init.sql`). O que você cadastrou foi apagado.

---

## 6. O que acontece por baixo dos panos

Vamos entender **conceitos avançados** de forma simples:

---

### 6.1. Por que o `web` consegue acessar o `db` usando o nome?

**Pergunta:** No código Python, usamos `host=db`. Por que funciona? `db` não é um endereço de internet!

**Resposta:** DNS interno do Docker.

---

**O que é DNS?**

DNS (Domain Name System) é como uma "agenda telefônica da internet":

```
Você digita: google.com
DNS traduz: 142.250.185.46 (endereço IP)
```

**Docker tem um DNS próprio** dentro de redes definidas pelo usuário (`rede_interna`).

---

**Como funciona:**

1. **Você cria a rede:**

```yaml
networks:
  rede_interna:
    driver: bridge
```

2. **Docker cria um servidor DNS interno** em `127.0.0.11:53`

3. **Cada container nessa rede recebe `/etc/resolv.conf` apontando para esse DNS:**

```bash
# Dentro do container web:
$ cat /etc/resolv.conf
nameserver 127.0.0.11
```

4. **Quando o código faz `host=db`:**

```
web → resolve "db" → 127.0.0.11 (DNS interno)
                   → retorna 172.18.0.2 (IP do container db)
web → conecta em 172.18.0.2:5432
```

---

**Verificação prática:**

```bash
# Dentro do container web:
docker compose exec web ping -c 1 db
```

**Saída:**

```
PING db (172.18.0.2) 56(84) bytes of data.
64 bytes from achados-db.rede_interna (172.18.0.2): icmp_seq=1 ttl=64 time=0.089 ms
```

✅ **`db` resolveu para `172.18.0.2`!**

---

**E se não tivesse rede definida?**

```yaml
# SEM "networks:"
services:
  db:
    ...
  web:
    ...
```

Docker usaria a rede `bridge` padrão, que **NÃO tem DNS automático**. O código falharia:

```python
psycopg.OperationalError: could not translate host name "db" to address: Name or service not known
```

---

### 6.2. Por que volumes são necessários?

**Experimento mental:**

Imagine que containers são **post-its** (aqueles papéis adesivos amarelos):

```
Post-it A (container web v1):
  - Código Python
  - Foto "oculos.jpg" colada nele

Você remove o post-it e cola um novo (recrear o container):

Post-it B (container web v2):
  - Código Python
  - SEM a foto (era no post-it anterior!)
```

❌ **A foto sumiu!**

---

**Solução: Volume (quadro branco compartilhado)**

```
Post-it A (container v1) → Escreve no quadro branco (volume)
Você remove o post-it
Post-it B (container v2) → Lê do mesmo quadro branco

✅ A foto ainda está lá!
```

---

**Onde ficam os volumes fisicamente?**

```bash
docker volume inspect achados-perdidos_fotos
```

**Resposta:**

```json
{
  "Mountpoint": "/var/lib/docker/volumes/achados-perdidos_fotos/_data"
}
```

**Windows:** `\\wsl$\docker-desktop-data\data\docker\volumes\achados-perdidos_fotos\_data`

Você pode acessar essa pasta e ver os arquivos `.jpg` diretamente!

---

### 6.3. Por que `init.sql` só roda uma vez?

**Comportamento da imagem `postgres`:**

```bash
# Script de inicialização do PostgreSQL (simplificado)

if [ ! -f "$PGDATA/PG_VERSION" ]; then
  echo "Banco não existe, criando..."
  initdb
  
  # Executar scripts em /docker-entrypoint-initdb.d/
  for script in /docker-entrypoint-initdb.d/*.sql; do
    psql < "$script"
  done
else
  echo "Banco já existe, pulando inicialização"
fi
```

**Marcador:** Arquivo `PG_VERSION` no volume `dados_banco`.

---

**Timeline:**

```
Primeira execução (volume vazio):
  1. PostgreSQL não encontra PG_VERSION
  2. Inicializa o banco do zero
  3. Executa init.sql (cria tabela + insere 3 itens)
  4. Cria arquivo PG_VERSION
  
Segunda execução (volume já existe):
  1. PostgreSQL encontra PG_VERSION
  2. "Ah, banco já existe!"
  3. Pula inicialização (init.sql NÃO roda)
```

---

**E se eu quiser aplicar mudanças no `init.sql`?**

**Opção 1: Destruir e recriar** (apaga dados!)

```bash
docker compose down -v  # Remove volumes
docker compose up -d    # Cria do zero
```

**Opção 2: ALTER TABLE manual** (mantém dados)

```bash
docker compose exec db psql -U achados_user -d achados_perdidos -c "ALTER TABLE itens ADD COLUMN nova_coluna VARCHAR(50);"
```

---

### 6.4. O que é healthcheck e por que é importante?

**Problema:**

```
T=0s   db inicia (processo postgres)
T=1s   web tenta conectar
       ❌ ERRO: "Connection refused"
       
Por quê? PostgreSQL ainda está inicializando (carregando configs, WAL replay...)
```

---

**Solução: Healthcheck**

```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U achados_user -d achados_perdidos"]
    interval: 5s
    timeout: 5s
    retries: 5
```

**O que faz:**

1. A cada 5 segundos (`interval`), roda o comando `pg_isready`
2. Se retornar `0` (sucesso): marca como `healthy`
3. Se retornar `1` (falha): tenta novamente (até 5 vezes)
4. Se falhar 5 vezes: marca como `unhealthy`

---

**Timeline com healthcheck:**

```
T=0s   db inicia
T=5s   Healthcheck #1: pg_isready → FALHA (ainda inicializando)
T=10s  Healthcheck #2: pg_isready → FALHA
T=15s  Healthcheck #3: pg_isready → SUCESSO ✅
       db marcado como "healthy"
       
T=15s  web AGORA pode iniciar (depends_on: condition: service_healthy)
T=16s  web conecta ao banco → SUCESSO ✅
```

---

**Verificação:**

```bash
docker compose ps
```

```
NAME          STATUS
achados-db    Up (healthy)    ← Note o "(healthy)"!
achados-web   Up
```

---

### 6.5. Cache de camadas do Dockerfile

**Por que importa a ordem das instruções?**

Cada linha do Dockerfile cria uma **camada** (layer):

```dockerfile
FROM python:3.12-slim       # Camada 1: 130 MB
COPY requirements.txt .     # Camada 2: 1 KB
RUN pip install ...         # Camada 3: 142 MB ← PESADA!
COPY app.py .               # Camada 4: 8 KB
```

---

**Regra do cache:**

Docker **reutiliza** uma camada se:
- A instrução não mudou
- O **conteúdo** não mudou (checksum dos arquivos)

---

**❌ Ordem ruim:**

```dockerfile
COPY . .                 # Copia TUDO (incluindo app.py)
RUN pip install ...      # Instala dependências
```

**Problema:**

```
Você muda app.py (1 linha):
  → Camada COPY . . muda (app.py diferente)
  → Camada RUN pip install... é INVALIDADA
  → Reinstala Flask, psycopg, Gunicorn (45 segundos) 😭
```

---

**✅ Ordem boa:**

```dockerfile
COPY requirements.txt .  # Copia APENAS dependências
RUN pip install ...      # Instala (camada pesada)
COPY app.py .            # Copia código (camada leve)
```

**Vantagem:**

```
Você muda app.py:
  → Camada COPY requirements.txt . NÃO muda (mesmo arquivo)
  → Camada RUN pip install... é REAPROVEITADA ✅ (cache!)
  → Camada COPY app.py . é reconstruída (rápido: 2 segundos)
  
Total: 2s em vez de 47s ⚡
```

---

**Verificação:**

```bash
docker compose build web
```

**Primeira vez:**

```
[+] Building 47.3s (10/10) FINISHED
 => [web 4/6] RUN pip install ...   45.2s
```

**Segunda vez (após mudar `app.py`):**

```
[+] Building 2.1s (10/10) FINISHED
 => [web 4/6] RUN pip install ...   CACHED ✅
 => [web 6/6] COPY app.py .         1.8s
```

---

### 6.6. Por que a porta 5432 do banco não é publicada?

**Segurança: Princípio do menor privilégio**

```yaml
db:
  # SEM "ports:" ← Porta 5432 ISOLADA!
```

**Efeito:**

```
┌───────────────────────────────────┐
│       Seu computador (host)       │
│                                   │
│  ❌ Não consegue acessar db:5432 │
└───────────────────────────────────┘
         │
         │ Porta 8080 publicada
         ↓
┌───────────────────────────────────┐
│           DOCKER                  │
│  ┌────────┐       ┌────────┐     │
│  │  web   │───────│   db   │     │
│  └────────┘       └────────┘     │
│     ✅              ✅           │
│  Acessa db     Porta 5432       │
│               apenas interna     │
└───────────────────────────────────┘
```

**Por quê?**

1. **Ataque de força bruta:** Se a porta estivesse publicada, qualquer um na rede poderia tentar senhas:

```bash
# Atacante na mesma rede:
psql -h 192.168.1.100 -U achados_user
# Tenta: admin, password, 123456, postgres...
```

2. **Exploits:** Se houver vulnerabilidade no PostgreSQL, atacante pode explorar diretamente.

3. **Desnecessário:** Apenas o `web` precisa acessar o banco, então por que expor?

---

**Como acessar o banco para debugging?**

**Opção 1: Via container web**

```bash
docker compose exec web psql -h db -U achados_user -d achados_perdidos
```

**Opção 2: Via container db**

```bash
docker compose exec db psql -U achados_user -d achados_perdidos
```

**Opção 3: Adminer (desafio bônus)**

```bash
docker compose --profile debug up -d
```

Acesse: `http://localhost:8081`

---

## 7. Resolução de problemas comuns

### ❌ Erro: "Cannot connect to Docker daemon"

**Sintoma:**

```
Error response from daemon: Bad response from Docker engine
```

**Causa:** Docker Desktop não está rodando.

**Solução:**

1. **Windows/Mac:** Abra o Docker Desktop (ícone da baleia)
2. Aguarde até aparecer "Docker Desktop is running"
3. Tente novamente

---

### ❌ Erro: "port is already allocated"

**Sintoma:**

```
Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**Causa:** Outra aplicação já está usando a porta 8080.

**Solução 1: Descobrir quem está usando**

**Windows:**

```powershell
netstat -ano | findstr :8080
```

**Linux/Mac:**

```bash
lsof -i :8080
```

Mate o processo ou...

**Solução 2: Mudar a porta no `docker-compose.yml`**

```yaml
web:
  ports:
    - "8081:5000"  # Mudou para 8081
```

Agora acesse: `http://localhost:8081`

---

### ❌ Erro: "no configuration file provided"

**Sintoma:**

```
no configuration file provided: not found
```

**Causa:** Você não está na pasta do projeto.

**Solução:**

```bash
cd caminho/para/achados-perdidos
docker compose up -d
```

---

### ❌ Aplicação não responde (timeout)

**Sintoma:**

```bash
curl http://localhost:8080/health
# (fica travado, não responde)
```

**Diagnóstico:**

```bash
docker compose ps
```

**Se `achados-web` está `Exited`:**

```bash
docker compose logs web
```

Procure por erros (geralmente problema de conexão com o banco).

**Solução comum:** Aguarde 10-15 segundos (banco ainda inicializando).

---

### ❌ Volumes não persistem (dados somem)

**Sintoma:** Você cadastra objetos, para o ambiente, sobe novamente e tudo sumiu.

**Causa:** Usou `docker compose down -v` (removeu volumes).

**Solução:** Use apenas `docker compose down` (sem `-v`).

---

### ❌ Mudança no `init.sql` não aplica

**Sintoma:** Você editou `db/init.sql`, mas as mudanças não aparecem.

**Causa:** `init.sql` só roda na **primeira** inicialização.

**Solução:**

```bash
# Apaga volumes (CUIDADO: perde dados!)
docker compose down -v

# Recria do zero
docker compose up -d
```

Ou aplique a mudança manualmente:

```bash
docker compose exec db psql -U achados_user -d achados_perdidos -c "ALTER TABLE itens ADD COLUMN nova_coluna VARCHAR(50);"
```

---

## 8. Glossário de termos técnicos

**API (Application Programming Interface):**
Interface que permite que programas conversem entre si. Nosso sistema tem uma API HTTP (rotas `/itens`, `/health`, etc.).

**Bind mount:**
Mapear uma pasta/arquivo do seu computador diretamente para dentro do container. Exemplo: `./db/init.sql:/docker-entrypoint-initdb.d/init.sql`.

**Bridge (rede):**
Tipo de rede Docker que conecta containers. É como uma "ponte" entre eles.

**Cache de camadas:**
Mecanismo do Docker para reutilizar partes já construídas de uma imagem, acelerando builds subsequentes.

**Container:**
Pacote isolado que contém uma aplicação e suas dependências. Criado a partir de uma imagem.

**Detached mode (`-d`):**
Rodar container em background (não trava o terminal).

**Dockerfile:**
Arquivo de texto com instruções para construir uma imagem Docker.

**DNS (Domain Name System):**
Sistema que traduz nomes (como `db`) para endereços IP (como `172.18.0.2`).

**Docker Compose:**
Ferramenta para orquestrar múltiplos containers definidos em um arquivo YAML.

**Docker Engine:**
O "motor" do Docker que gerencia containers, imagens, volumes e redes.

**Efêmero:**
Temporário, que não persiste. Containers são efêmeros (quando removidos, dados dentro deles somem).

**Exec form:**
Sintaxe de comando Docker com colchetes: `CMD ["gunicorn", "app:app"]`. Melhor que shell form.

**Flask:**
Framework web Python usado neste projeto para criar a API HTTP.

**Gunicorn:**
Servidor de produção WSGI para aplicações Python. Serve nossa aplicação Flask.

**Healthcheck:**
Teste automático que verifica se um container está funcionando corretamente.

**Imagem:**
Modelo imutável usado para criar containers. É como uma "receita".

**Isolamento:**
Containers não interferem uns nos outros (cada um tem seu próprio filesystem, rede, processos).

**PostgreSQL:**
Sistema de banco de dados relacional (SQL) usado para armazenar os objetos cadastrados.

**Porta publicada:**
Porta do host (seu computador) mapeada para uma porta do container. Exemplo: `8080:5000`.

**psycopg:**
Biblioteca Python para conectar ao PostgreSQL.

**UUID (Universally Unique Identifier):**
Identificador único gerado aleatoriamente. Garante que dois objetos nunca tenham o mesmo ID.

**Volume:**
Armazenamento persistente gerenciado pelo Docker. Dados sobrevivem à remoção de containers.

**WSGI (Web Server Gateway Interface):**
Padrão Python para comunicação entre servidores web (Gunicorn) e aplicações (Flask).

---

## 🎓 Conclusão

Parabéns por chegar até aqui! Agora você entende:

✅ **O que é Docker** e por que ele resolve o problema "funciona na minha máquina"  
✅ **Como containers funcionam** (isolamento, efêmero, leve)  
✅ **O que são volumes** (persistência de dados)  
✅ **Como funciona a rede Docker** (DNS interno, isolamento)  
✅ **O fluxo completo de uma requisição** (do seu navegador até o banco de dados)  
✅ **Como usar o sistema** (subir, testar, parar)  
✅ **Como resolver problemas comuns** (porta ocupada, volumes, logs)

---

## 📚 Próximos passos

Quer aprofundar? Sugestões:

1. **Adicione uma nova rota** (ex: `DELETE /itens/<id>` para apagar objetos)
2. **Experimente escalonar** (tente `docker compose up -d --scale web=3` e veja o erro)
3. **Migre para a nuvem** (AWS Fargate, Google Cloud Run)
4. **Adicione testes automatizados** (pytest)
5. **Implemente CI/CD** (GitHub Actions que roda testes e faz deploy)

---

**Documento criado para a disciplina de Computação em Nuvem**  
**Data:** 19/10/2026  
**Autor:** [Seu Nome]
