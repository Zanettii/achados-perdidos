# 🌸 Achados & Perdidos 🌸

Um sisteminha super fofo de gerenciamento de objetos perdidos do campus, empacotadinho em contêineres Docker para facilitar a nossa vida e escalar como mágica! ✨☁️

## 🎀 Índice Mágico

- [Nosso Objetivo](#-nosso-objetivo-cor-de-rosa-)
- [Pré-requisitos](#-o-que-você-precisa-ter-pré-requisitos-)
- [Preparando o terreno](#-preparando-o-terreno-)
- [Voando para a Nuvem](#-voando-para-a-nuvem-️)
- [Hora de Testar a Mágica](#-hora-de-testar-a-mágica-)
- [Caminhos da nossa API](#-caminhos-da-nossa-api-)
- [Hora de dar Tchau](#-hora-de-dar-tchau-parar-e-destruir-)
- [Como organizamos nossa casinha](#-como-organizamos-nossa-casinha-)
- [Presentinhos Bônus](#-presentinhos-bônus-implementados-)

---

## 💖 Nosso Objetivo Cor-de-Rosa 

Este projeto empacota o sistema **Achados & Perdidos** em contêineres Docker, permitindo que qualquer pessoa suba o ambiente completo (aplicação web + banco de dados) com um único comando, sem instalações chatas e manuais. Tudo perfeitinho e automatizado! 🌷

**Principais características:**
- 🎀 Dados persistentes (nada se perde, tudo se guarda!)
- 🎀 Senhas super protegidas no arquivo `.env` (escondidas do mundo)
- 🎀 Banco de dados isolado da rede externa, bem seguro
- 🎀 Usuário sem privilégios de root no contêiner web, porque segurança é essencial
- 🎀 Healthcheck garantindo que o banco está prontinho antes da aplicação iniciar

---

## 🧸 O que você precisa ter (Pré-requisitos)

Você precisa ter instalado no seu computador:

1. **Docker** (versão 20.10 ou superior) 🐳
2. **Docker Compose** (versão 2.0 ou superior) ✨

### Para verificar se já tem tudo certinho:

```bash
docker --version
docker compose version
```

**Onde baixar:**
- **Windows/Mac**: [Docker Desktop](https://www.docker.com/products/docker-desktop)
- **Linux**: [Instruções oficiais](https://docs.docker.com/engine/install/)

---

## ⚙️ Preparando o Terreno ✨

### 1. Configurar as credenciais do banco

Copie o nosso arquivo de exemplo e edite com as suas senhas secretas:

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

**Conteúdo do seu `.env`:**
```env
POSTGRES_DB=achados_perdidos
POSTGRES_USER=achados_user
POSTGRES_PASSWORD=SuaSenhaFofaEForteAqui123!
```

> ⚠️ **ATENÇÃO CUIDADOSA**: O arquivo `.env` contém senhas reais e **NÃO deve ser enviado** para o GitHub de jeito nenhum (ele já está no nosso `.gitignore`, ufa!). 🙈

---

## 🚀 Voando para a Nuvem ☁️✨

### O comando mágico para subir tudo de uma vez:

```bash
docker compose up -d --build
```

**O que essa mágica faz:**
1. 🌸 Constrói a imagem da nossa aplicação web (`achados-web`)
2. 🌸 Baixa a imagem do PostgreSQL 16 Alpine
3. 🌸 Cria a nossa rede interna toda conectadinha (`rede_interna`)
4. 🌸 Cria os volumes `dados_banco` e `fotos` para guardar tudo com carinho
5. 🌸 Sobe o banco de dados e executa o `init.sql`
6. 🌸 Espera o banco ficar saudável (healthcheck)
7. 🌸 Sobe a aplicação web conectada ao banco

### Para ver se está tudo rodando lindamente:

```bash
docker compose ps
```

**O que vai aparecer:**
```
NAME                IMAGE                    STATUS             PORTS
achados-db          postgres:16-alpine       Up (healthy)       -
achados-web         achados-perdidos-web     Up                 0.0.0.0:8080->5000/tcp
```

### Para ver os logs passando em tempo real (como em um filme):

```bash
docker compose logs -f
```

---

## 🍧 Hora de Testar a Mágica 🪄

### 1. Verificar a saúde do nosso sistema:

```bash
curl http://localhost:8080/health
```

**Resposta fofa esperada:**
```json
{
  "status": "ok",
  "banco": "conectado",
  "container": "achados-web"
}
```

### 2. Listar os itens que já guardamos:

```bash
curl http://localhost:8080/itens
```

### 3. Filtrar por local específico:

```bash
curl "http://localhost:8080/itens?local=biblio"
```

### 4. Cadastrar um item novo com foto:

**Windows (PowerShell):**
```powershell
curl.exe -F "descricao=Oculos Rosa" -F "local=Cantina" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**Linux/Mac:**
```bash
curl -F "descricao=Oculos Rosa" -F "local=Cantina" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

### 5. Ver as nossas estatísticas:

```bash
curl http://localhost:8080/estatisticas
```

### 6. Ver a fotinha no navegador:

Basta abrir: `http://localhost:8080/fotos/NOME_DA_FOTO.jpg` 📸

---

## 🗺️ Caminhos da nossa API 🌸

| Método | Rota | O que faz? | O que precisa enviar? |
|--------|------|-----------|------------|
| `GET` | `/health` | Vê se o sistema e o banco estão bem | - |
| `GET` | `/itens` | Mostra todos os achados | `?local=texto` (se quiser) |
| `POST` | `/itens` | Guarda um objeto novo | `descricao`, `local`, `foto` (opcional) |
| `GET` | `/fotos/<nome>` | Mostra a imagem guardada | - |
| `GET` | `/estatisticas` | Conta quantos itens e fotos temos | - |

---

## 🛑 Hora de dar Tchau (Parar e Destruir) 🧸

### Para os contêineres dormirem (mantém os dados a salvo):

```bash
docker compose down
```

> Suas fotos e itens do banco continuam guardadinhos em segurança nos volumes! 💖

### Para fazer uma faxina completa (limpa TUDO):

```bash
docker compose down -v
```

> ⚠️ **CUIDADO**: Essa opção apaga tudinho! Use só quando quiser começar do zero.

---

## 📁 Como organizamos nossa casinha 🎀

```
achados-perdidos/
├── app/                  # O coração do sistema
│   ├── app.py            # Nossa aplicação Flask
│   ├── requirements.txt  # As dependências
│   ├── Dockerfile        # A receita da imagem web
│   └── .dockerignore     # O que o Docker não deve olhar
├── db/                   
│   └── init.sql          # Arquivo que prepara o banco logo de cara
├── docs/                 
│   ├── arquitetura.png   # O desenho lindo do projeto
│   ├── RELATORIO.md      # Nosso relatório técnico super caprichado
│   ├── REFLEXAO.md       # Nossos pensamentos sobre a nuvem
│   └── evidencias/       # Os prints provando que tudo funciona
├── docker-compose.yml    # O maestro da nossa orquestra
├── .env                  # Segredos (shhh! 🤫)
├── .env.example          # Molde para os segredos
├── .gitignore            # O que o Git não deve levar
└── README.md             # Este arquivo maravilhoso!
```

---

## 🎁 Presentinhos Bônus Implementados 🦄

### 1. Painel do Banco (Adminer) ✨

Criamos um painel super prático que só sobe quando a gente chama:

```bash
docker compose --profile debug up -d
```

Acesse em: `http://localhost:8081`

**Credenciais (use as mesmas do seu `.env`):**
- Sistema: `PostgreSQL`
- Servidor: `db`
- Usuário: `achados_user`
- Senha: (do `.env`)
- Base de dados: `achados_perdidos`

---

## 🔐 Segurança (Porque proteger os dados é super importante!) 💖

- 🎀 Zero senhas expostas no código!
- 🎀 Porta do banco trancadinha, sem acesso externo.
- 🎀 Aplicação rodando como usuário simples (sem root), bem seguro.
- 🎀 Arquivo `.env` ignorado no `.gitignore` com sucesso!

---

## 👩‍💻 Autora & Informações Acadêmicas

Feito com muito carinho e dedicação para a nossa entrega final! 🎓✨

- **Engenheira de Nuvem**: Ana Clara (@Zanettii)
- **Disciplina**: Computação em Nuvem
- **Professor Incrível**: Fábio Castro

---

> *"Nas nuvens, a gente guarda o que encontra e não perde mais nada!"* ☁️💖