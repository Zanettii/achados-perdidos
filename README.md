# 🔍 Achados & Perdidos

Sistema de gerenciamento de objetos perdidos do campus, empacotado em contêineres Docker para fácil implantação e escalabilidade.

## 📋 Índice

- [Objetivo](#-objetivo)
- [Pré-requisitos](#-pré-requisitos)
- [Configuração inicial](#-configuração-inicial)
- [Como subir o ambiente](#-como-subir-o-ambiente)
- [Testando a aplicação](#-testando-a-aplicação)
- [Rotas da API](#-rotas-da-api)
- [Como parar e destruir o ambiente](#-como-parar-e-destruir-o-ambiente)
- [Estrutura do projeto](#-estrutura-do-projeto)
- [Desafios bônus implementados](#-desafios-bônus-implementados)

---

## 🎯 Objetivo

Este projeto empacota o sistema **Achados & Perdidos** em contêineres Docker, permitindo que qualquer pessoa suba o ambiente completo (aplicação web + banco de dados) com um único comando, sem instalações manuais.

**Principais características:**
- ✅ Dados persistentes mesmo após recriar os contêineres
- ✅ Senhas protegidas em arquivo `.env` (não versionado)
- ✅ Banco de dados isolado da rede externa
- ✅ Usuário sem privilégios de root no contêiner web
- ✅ Healthcheck garantindo que o banco aceita conexões antes da aplicação iniciar

---

## 📦 Pré-requisitos

Você precisa ter instalado:

1. **Docker** (versão 20.10 ou superior)
2. **Docker Compose** (versão 2.0 ou superior)

### Verificar instalação:

```bash
docker --version
docker compose version
```

**Instalação do Docker:**
- **Windows/Mac**: [Docker Desktop](https://www.docker.com/products/docker-desktop)
- **Linux**: [Instruções oficiais](https://docs.docker.com/engine/install/)

---

## ⚙️ Configuração inicial

### 1. Configurar as credenciais do banco

Copie o arquivo de exemplo e edite com suas senhas:

**Windows (PowerShell):**
```powershell
Copy-Item .env.example .env
notepad .env
```

**Linux/Mac:**
```bash
cp .env.example .env
nano .env
```

**Conteúdo do `.env`:**
```env
POSTGRES_DB=achados_perdidos
POSTGRES_USER=achados_user
POSTGRES_PASSWORD=SuaSenhaForteAqui123!
```

> ⚠️ **IMPORTANTE**: O arquivo `.env` contém senhas reais e **NÃO deve ser versionado** no Git (já está no `.gitignore`).

---

## 🚀 Como subir o ambiente

### Comando único para subir tudo:

```bash
docker compose up -d --build
```

**O que acontece:**
1. ✅ Constrói a imagem da aplicação web (`achados-web`)
2. ✅ Baixa a imagem do PostgreSQL 16 Alpine
3. ✅ Cria a rede interna `rede_interna`
4. ✅ Cria os volumes `dados_banco` e `fotos`
5. ✅ Sobe o banco de dados e executa o `init.sql`
6. ✅ Aguarda o banco ficar saudável (healthcheck)
7. ✅ Sobe a aplicação web conectada ao banco

### Verificar se está rodando:

```bash
docker compose ps
```

**Saída esperada:**
```
NAME                IMAGE                    STATUS              PORTS
achados-db          postgres:16-alpine       Up (healthy)        -
achados-web         achados-perdidos-web     Up                  0.0.0.0:8080->5000/tcp
```

### Ver logs em tempo real:

```bash
docker compose logs -f
```

---

## 🧪 Testando a aplicação

### 1. Verificar saúde do sistema:

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

### 2. Listar itens cadastrados:

```bash
curl http://localhost:8080/itens
```

**Resposta:** JSON com os 3 itens de exemplo do `init.sql`.

### 3. Filtrar por local:

```bash
curl "http://localhost:8080/itens?local=biblio"
```

### 4. Cadastrar novo item com foto:

**Windows (PowerShell):**
```powershell
curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**Linux/Mac:**
```bash
curl -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**Resposta:**
```json
{
  "id": 4,
  "foto": "a3f8e9d1c2b4.jpg"
}
```

### 5. Ver estatísticas:

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

### 6. Visualizar foto no navegador:

Abra no navegador: `http://localhost:8080/fotos/a3f8e9d1c2b4.jpg`

---

## 🗺️ Rotas da API

| Método | Rota | Descrição | Parâmetros |
|--------|------|-----------|------------|
| `GET` | `/health` | Verifica status da aplicação e conexão com o banco | - |
| `GET` | `/itens` | Lista todos os objetos cadastrados | `?local=texto` (opcional) |
| `POST` | `/itens` | Cadastra novo objeto | `descricao`, `local`, `foto` (opcional) |
| `GET` | `/fotos/<nome>` | Retorna a foto armazenada | - |
| `GET` | `/estatisticas` | Mostra total de itens e fotos | - |

---

## 🛑 Como parar e destruir o ambiente

### Parar os contêineres (mantém volumes):

```bash
docker compose down
```

> Os dados do banco e as fotos **permanecem** nos volumes.

### Parar e **remover os volumes** (limpa tudo):

```bash
docker compose down -v
```

> ⚠️ **ATENÇÃO**: Isso apaga todos os dados! Use apenas para resetar o ambiente.

### Subir novamente após parar:

```bash
docker compose up -d
```

> Se não usar `-v` ao parar, os dados voltam intactos.

---

## 📁 Estrutura do projeto

```
achados-perdidos/
├── app/
│   ├── app.py              # Aplicação Flask
│   ├── requirements.txt    # Dependências Python
│   ├── Dockerfile          # Imagem do serviço web
│   └── .dockerignore       # Arquivos ignorados no build
├── db/
│   └── init.sql            # Script de inicialização do banco
├── docs/
│   ├── arquitetura.png     # Diagrama da arquitetura
│   ├── RELATORIO.md        # Relatório técnico detalhado
│   ├── REFLEXAO.md         # Questões de reflexão
│   └── evidencias/         # Prints dos testes
├── docker-compose.yml      # Orquestração dos serviços
├── .env                    # Credenciais (NÃO versionar!)
├── .env.example            # Modelo de credenciais
├── .gitignore              # Arquivos ignorados pelo Git
└── README.md               # Este arquivo
```

---

## 🎁 Desafios bônus implementados

### 1. ✅ Painel do banco (Adminer) — +0,3 pt

O painel Adminer está configurado com **profile**, subindo apenas quando solicitado:

```bash
docker compose --profile debug up -d
```

Acesse em: `http://localhost:8081`

**Credenciais (use as mesmas do `.env`):**
- Sistema: `PostgreSQL`
- Servidor: `db`
- Usuário: `achados_user`
- Senha: (do `.env`)
- Base de dados: `achados_perdidos`

---

## 🔐 Segurança

- ✅ Nenhuma senha no código ou no `docker-compose.yml`
- ✅ Porta do banco **não exposta** no host
- ✅ Aplicação roda com usuário **sem privilégios de root**
- ✅ Arquivo `.env` no `.gitignore`

---

## 📚 Documentação adicional

- **Relatório técnico**: [docs/RELATORIO.md](docs/RELATORIO.md)
- **Reflexões e equivalência na nuvem**: [docs/REFLEXAO.md](docs/REFLEXAO.md)
- **Evidências dos testes**: [docs/evidencias/](docs/evidencias/)

---

## 🐛 Troubleshooting

### Erro: "Cannot connect to Docker daemon"
```bash
# Windows: Certifique-se de que o Docker Desktop está rodando
# Linux: sudo systemctl start docker
```

### Erro: "port is already allocated"
```bash
# Algo já está usando a porta 8080
docker compose down
# Ou mude a porta no docker-compose.yml: "8081:5000"
```

### Ver logs detalhados:
```bash
docker compose logs web --tail 50
docker compose logs db --tail 50
```

---

## 👥 Autores

Projeto desenvolvido para a disciplina de **Computação em Nuvem**.

**Engenheiro de Nuvem responsável**: [Seu Nome]

---

## 📄 Licença

Projeto educacional — uso livre para fins acadêmicos.
