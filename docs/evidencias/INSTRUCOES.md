# 📸 Evidências dos Testes — Instruções

Este arquivo contém os **comandos que você deve executar** quando o Docker Desktop estiver instalado e rodando, para gerar as evidências das Tarefas 4 e 5.

---

## ⚠️ Pré-requisito

**Antes de executar os comandos abaixo:**

1. ✅ Instale o Docker Desktop ([docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop))
2. ✅ Inicie o Docker Desktop (ícone da baleia deve aparecer na bandeja)
3. ✅ Aguarde até o status "Docker Desktop is running"
4. ✅ Abra o PowerShell na pasta do projeto: `C:\Users\Kerem-1\Documents\Faculdade\Computação em nuvem\achados-perdidos`

---

## 📋 TAREFA 4 — Testes funcionais

Execute os comandos abaixo **na ordem** e salve a saída de cada um (print de tela ou copiar o texto).

---

### 1. Subir o ambiente

```powershell
docker compose up -d --build
```

**Aguarde cerca de 1-2 minutos (primeira vez demora mais).**

**Evidência esperada:**
```
[+] Building 47.3s (10/10) FINISHED
 => [web 1/6] FROM python:3.12-slim
 => [web 4/6] RUN pip install ...
[+] Running 5/5
 ✔ Network achados-perdidos_rede_interna
 ✔ Volume achados-perdidos_dados_banco
 ✔ Volume achados-perdidos_fotos
 ✔ Container achados-db
 ✔ Container achados-web
```

💾 **Salve esta saída como:** `docs/evidencias/01_docker_compose_up.txt`

---

### 2. Verificar status dos contêineres

```powershell
docker compose ps
```

**Evidência esperada:**
```
NAME          IMAGE                    STATUS              PORTS
achados-db    postgres:16-alpine       Up (healthy)        -
achados-web   achados-perdidos-web     Up                  0.0.0.0:8080->5000/tcp
```

💾 **Salve como:** `docs/evidencias/02_docker_compose_ps.txt`

**🔍 Importante:** Note o `(healthy)` no status do banco!

---

### 3. Teste de saúde

```powershell
curl.exe http://localhost:8080/health
```

**Evidência esperada:**
```json
{"status":"ok","banco":"conectado","container":"achados-web"}
```

💾 **Salve como:** `docs/evidencias/03_health_check.txt`

---

### 4. Listar itens

```powershell
curl.exe http://localhost:8080/itens
```

**Evidência esperada:** JSON com 3 itens de exemplo do `init.sql`

💾 **Salve como:** `docs/evidencias/04_listar_itens.txt`

---

### 5. Filtrar por local

```powershell
curl.exe "http://localhost:8080/itens?local=biblio"
```

**Evidência esperada:** Apenas item da Biblioteca

💾 **Salve como:** `docs/evidencias/05_filtrar_local.txt`

---

### 6. Criar imagem de teste

```powershell
python -c "import struct,zlib;w=h=10;raw=b'';exec('[raw:=raw+b''\x00''+bytes([255,0,0]*w) for _ in range(h)]');idat=zlib.compress(raw);png=b'\x89PNG\r\n\x1a\n';ihdr=struct.pack('>IIBBBBB',w,h,8,2,0,0,0);png+=struct.pack('>I',13)+b'IHDR'+ihdr+struct.pack('>I',zlib.crc32(b'IHDR'+ihdr)&0xffffffff);png+=struct.pack('>I',len(idat))+b'IDAT'+idat+struct.pack('>I',zlib.crc32(b'IDAT'+idat)&0xffffffff);png+=struct.pack('>I',0)+b'IEND'+struct.pack('>I',zlib.crc32(b'IEND')&0xffffffff);open('oculos.jpg','wb').write(png)"
```

Se der erro, crie manualmente um arquivo qualquer:
```powershell
echo "fake image" > oculos.jpg
```

---

### 7. Cadastrar objeto com foto

```powershell
curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

**Evidência esperada:**
```json
{"id":4,"foto":"a3f8e9d1c2b4.jpg"}
```

💾 **Salve como:** `docs/evidencias/06_cadastrar_com_foto.txt`

**📝 Anote o nome da foto retornado** (você vai usar no próximo passo)

---

### 8. Ver estatísticas

```powershell
curl.exe http://localhost:8080/estatisticas
```

**Evidência esperada:**
```json
{"itens_no_banco":4,"fotos_no_volume":1}
```

💾 **Salve como:** `docs/evidencias/07_estatisticas.txt`

---

### 9. Acessar foto no navegador

Abra no navegador (substitua pelo nome da foto que você anotou):

```
http://localhost:8080/fotos/a3f8e9d1c2b4.jpg
```

📸 **Tire um print de tela mostrando a imagem renderizada**

💾 **Salve como:** `docs/evidencias/08_foto_navegador.png`

---

### 10. Ver logs da aplicação

```powershell
docker compose logs web --tail 20
```

**Evidência esperada:**
```
achados-web | [2026-10-19 04:10:23] [1] [INFO] Starting gunicorn 23.0.0
achados-web | [2026-10-19 04:10:23] [1] [INFO] Listening at: http://0.0.0.0:5000
achados-web | [2026-10-19 04:10:23] [1] [INFO] Using worker: sync
achados-web | [2026-10-19 04:15:45] [1] [INFO] POST /itens
```

💾 **Salve como:** `docs/evidencias/09_logs_web.txt`

---

## 📋 TAREFA 5 — Prova de persistência

---

### Situação 1: Recriação dos contêineres

**Objetivo:** Provar que dados sobrevivem a `docker compose down` (sem `-v`)

---

#### 1.1. Anotar estatísticas antes

```powershell
curl.exe http://localhost:8080/estatisticas
```

💾 **Salve como:** `docs/evidencias/10_estatisticas_antes_down.txt`

---

#### 1.2. Cadastrar mais itens (opcional, para ter mais dados)

```powershell
curl.exe -F "descricao=Mochila preta" -F "local=Cantina" -F "foto=@oculos.jpg" http://localhost:8080/itens
```

---

#### 1.3. Derrubar contêineres (SEM remover volumes)

```powershell
docker compose down
```

**Evidência esperada:**
```
[+] Running 3/3
 ✔ Container achados-web  Removed
 ✔ Container achados-db   Removed
 ✔ Network achados-perdidos_rede_interna  Removed
```

💾 **Salve como:** `docs/evidencias/11_docker_compose_down.txt`

**🔍 Note:** NÃO menciona volumes (eles ficaram intactos)

---

#### 1.4. Subir novamente

```powershell
docker compose up -d
```

---

#### 1.5. Verificar estatísticas após

```powershell
curl.exe http://localhost:8080/estatisticas
```

**Evidência esperada:** Mesmos números de antes!

💾 **Salve como:** `docs/evidencias/12_estatisticas_apos_down.txt`

**✅ Conclusão:** Dados persistiram!

---

### Situação 2: Inspeção dos volumes

---

#### 2.1. Listar volumes

```powershell
docker volume ls
```

**Evidência esperada:**
```
DRIVER    VOLUME NAME
local     achados-perdidos_dados_banco
local     achados-perdidos_fotos
```

💾 **Salve como:** `docs/evidencias/13_docker_volume_ls.txt`

---

#### 2.2. Inspecionar volume de fotos

```powershell
docker volume inspect achados-perdidos_fotos
```

**Evidência esperada:** JSON com `Mountpoint`, `CreatedAt`, `Labels`

💾 **Salve como:** `docs/evidencias/14_inspect_volume_fotos.txt`

---

#### 2.3. Listar conteúdo do volume de fotos

```powershell
docker compose exec web ls -l /app/uploads
```

**Evidência esperada:**
```
total 24
-rw-r--r-- 1 appuser appgroup  3821 Oct 19 04:15 a3f8e9d1c2b4.jpg
-rw-r--r-- 1 appuser appgroup 15039 Oct 19 04:17 f1a2b3c4d5e6.jpg
```

💾 **Salve como:** `docs/evidencias/15_listar_fotos_volume.txt`

**🔍 Note:** Proprietário é `appuser` (não root!)

---

### Situação 3: Remoção dos volumes

**Objetivo:** Provar que `docker compose down -v` apaga tudo

---

#### 3.1. Anotar estatísticas antes

```powershell
curl.exe http://localhost:8080/estatisticas
```

💾 **Salve como:** `docs/evidencias/16_estatisticas_antes_down_v.txt`

---

#### 3.2. Derrubar E REMOVER volumes

```powershell
docker compose down -v
```

**Evidência esperada:**
```
[+] Running 5/5
 ✔ Container achados-web  Removed
 ✔ Container achados-db   Removed
 ✔ Volume achados-perdidos_fotos  Removed  ← REMOVEU!
 ✔ Volume achados-perdidos_dados_banco  Removed  ← REMOVEU!
 ✔ Network achados-perdidos_rede_interna  Removed
```

💾 **Salve como:** `docs/evidencias/17_docker_compose_down_v.txt`

---

#### 3.3. Subir novamente

```powershell
docker compose up -d
```

---

#### 3.4. Verificar estatísticas após

```powershell
curl.exe http://localhost:8080/estatisticas
```

**Evidência esperada:**
```json
{"itens_no_banco":3,"fotos_no_volume":0}
```

💾 **Salve como:** `docs/evidencias/18_estatisticas_apos_down_v.txt`

**✅ Conclusão:** Apenas os 3 itens de exemplo voltaram (do `init.sql`). Fotos e itens cadastrados manualmente sumiram!

---

#### 3.5. Listar itens (prova que só voltaram os de exemplo)

```powershell
curl.exe http://localhost:8080/itens
```

💾 **Salve como:** `docs/evidencias/19_listar_itens_apos_reset.txt`

---

## 📋 Evidências adicionais

---

### Inspecionar a rede

```powershell
docker network inspect achados-perdidos_rede_interna
```

💾 **Salve como:** `docs/evidencias/20_inspect_network.txt`

**🔍 Procure por:** IPs dos containers, DNS habilitado (`"EnableIPv6": false`)

---

### Tamanho da imagem gerada

```powershell
docker image ls
```

**Evidência esperada:**
```
REPOSITORY               TAG       SIZE
achados-perdidos-web     latest    285MB
postgres                 16-alpine 240MB
```

💾 **Salve como:** `docs/evidencias/21_docker_image_ls.txt`

---

### Verificar usuário do container web

```powershell
docker compose exec web whoami
```

**Evidência esperada:**
```
appuser
```

💾 **Salve como:** `docs/evidencias/22_whoami.txt`

**✅ Confirmação:** Container NÃO roda como root!

---

## 📊 Checklist de evidências

Ao final, você deve ter **22 arquivos** na pasta `docs/evidencias/`:

- [ ] `01_docker_compose_up.txt`
- [ ] `02_docker_compose_ps.txt`
- [ ] `03_health_check.txt`
- [ ] `04_listar_itens.txt`
- [ ] `05_filtrar_local.txt`
- [ ] `06_cadastrar_com_foto.txt`
- [ ] `07_estatisticas.txt`
- [ ] `08_foto_navegador.png` (screenshot)
- [ ] `09_logs_web.txt`
- [ ] `10_estatisticas_antes_down.txt`
- [ ] `11_docker_compose_down.txt`
- [ ] `12_estatisticas_apos_down.txt`
- [ ] `13_docker_volume_ls.txt`
- [ ] `14_inspect_volume_fotos.txt`
- [ ] `15_listar_fotos_volume.txt`
- [ ] `16_estatisticas_antes_down_v.txt`
- [ ] `17_docker_compose_down_v.txt`
- [ ] `18_estatisticas_apos_down_v.txt`
- [ ] `19_listar_itens_apos_reset.txt`
- [ ] `20_inspect_network.txt`
- [ ] `21_docker_image_ls.txt`
- [ ] `22_whoami.txt`

---

## 💡 Dicas para capturar evidências

**PowerShell:**

```powershell
# Salvar saída diretamente em arquivo:
docker compose ps > docs\evidencias\02_docker_compose_ps.txt

# Ou copiar do terminal:
# 1. Executar comando
# 2. Selecionar saída com o mouse
# 3. Copiar (Ctrl+C)
# 4. Abrir notepad.exe e colar
# 5. Salvar como .txt
```

**Prints de tela:**

- **Windows:** Ferramenta de Captura (Win + Shift + S)
- **Mac:** Cmd + Shift + 4
- **Linux:** Print Screen ou Flameshot

---

## 🎯 Depois de coletar as evidências

Crie um arquivo `docs/evidencias/README.md` com legendas curtas para cada evidência:

```markdown
# Evidências dos Testes

## Tarefa 4 — Testes funcionais

- **01**: Subida do ambiente com `docker compose up -d --build`
- **02**: Status dos contêineres (ambos `Up`, db `healthy`)
- **03**: Health check retornando status OK e banco conectado
- **04**: Listagem dos 3 itens de exemplo do init.sql
- **05**: Filtro por local (biblioteca)
- **06**: Cadastro de objeto com foto (retorna id=4)
- **07**: Estatísticas: 4 itens, 1 foto
- **08**: Foto renderizada no navegador
- **09**: Logs do Gunicorn (aplicação rodando)

## Tarefa 5 — Persistência

- **10-12**: Dados persistem após `docker compose down`
- **13-15**: Inspeção dos volumes (localização, conteúdo)
- **16-19**: Volumes removidos com `-v`, apenas init.sql volta

## Evidências adicionais

- **20**: Estrutura da rede interna (IPs, DNS)
- **21**: Tamanho das imagens (~285 MB para web)
- **22**: Usuário do container web (appuser, não root)
```

---

**Pronto!** Execute os comandos quando tiver o Docker instalado e funcionando.
