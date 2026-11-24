# 🐳 Fullstack Docker API

Projeto de estudos para aprender Docker na prática: uma API simples em **Node.js + TypeScript**, empacotada em uma imagem Docker e executada em container.

O objetivo aqui não é a API em si (ela só responde `Hello World!`), e sim **entender como funcionam imagens, containers, portas e o fluxo de build**, errando, depurando e documentando cada problema pelo caminho.

## 📚 Sumário

- [Tecnologias](#-tecnologias)
- [Estrutura do projeto](#-estrutura-do-projeto)
- [Rodando localmente (sem Docker)](#-rodando-localmente-sem-docker)
- [Rodando com Docker](#-rodando-com-docker)
- [Entendendo o Dockerfile](#-entendendo-o-dockerfile)
- [Comandos Docker que aprendi](#-comandos-docker-que-aprendi)
- [Problemas que encontrei e como resolvi](#-problemas-que-encontrei-e-como-resolvi)
- [Conceitos que fixei](#-conceitos-que-fixei)
- [Próximos passos](#-próximos-passos)

## 🛠 Tecnologias

- [Node.js](https://nodejs.org/) 20
- [TypeScript](https://www.typescriptlang.org/)
- [tsx](https://tsx.is/) para executar TypeScript em desenvolvimento
- [Docker](https://www.docker.com/) e Docker Desktop (WSL 2 no Windows)

## 📁 Estrutura do projeto

```
fullstack-docker-api-main/
├── src/
│   └── server.ts        # servidor HTTP
├── .dockerignore        # arquivos que não vão para a imagem
├── .gitignore
├── Dockerfile           # receita da imagem
├── package.json
├── package-lock.json
└── tsconfig.json
```

> ⚠️ O `Dockerfile` e o `.dockerignore` ficam na **raiz** do projeto, ao lado do `package.json`, e não dentro de `src`.

## 💻 Rodando localmente (sem Docker)

```bash
npm install
npx tsx src/server.ts
```

Acesse [http://localhost:3333](http://localhost:3333) e a resposta será `Hello World!`.

## 🐋 Rodando com Docker

Construir a imagem:

```bash
docker build -t api .
```

Subir o container em segundo plano, com nome fixo:

```bash
docker run -d --name api -p 3333:3333 api
```

Testar:

```bash
curl http://localhost:3333
```

Ver os logs e parar:

```bash
docker logs -f api
docker stop api
```

### Alterei o código, e agora?

A imagem é uma "foto" do código no momento do build. Ao alterar o código, é preciso reconstruir a imagem e recriar o container:

```bash
docker build -t api .
docker rm -f api
docker run -d --name api -p 3333:3333 api
```

## 📝 Entendendo o Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

EXPOSE 3333

CMD ["npm", "start"]
```

| Instrução | O que faz |
|---|---|
| `FROM node:20-alpine` | Imagem base: Node 20 sobre Alpine Linux (pequena, mas sem `bash`) |
| `WORKDIR /app` | Define a pasta de trabalho **dentro do container** (Linux, nunca um caminho do Windows) |
| `COPY package*.json ./` | Copia só os arquivos de dependências primeiro |
| `RUN npm ci` | Instala as dependências de forma exata, a partir do `package-lock.json` |
| `COPY . .` | Copia o restante do código |
| `RUN npm run build` | Compila o TypeScript (gera a pasta `dist`) |
| `EXPOSE 3333` | Documenta a porta usada pela aplicação |
| `CMD ["npm", "start"]` | Comando executado quando o container inicia |

### Por que copiar o `package.json` antes do código?

O Docker guarda cada instrução como uma **camada em cache**. Se só o código mudou, as camadas do `npm ci` são reaproveitadas e o build fica muito mais rápido. Se eu copiasse tudo de uma vez, qualquer alteração no código reinstalaria todas as dependências.

### `.dockerignore`

```
node_modules
dist
.git
```

Evita copiar o `node_modules` do Windows por cima do que foi instalado dentro do container.

### Scripts esperados no `package.json`

```json
"scripts": {
  "dev": "tsx watch src/server.ts",
  "build": "tsc",
  "start": "node dist/server.js"
}
```

## ⌨️ Comandos Docker que aprendi

### Imagens

| Comando | O que faz |
|---|---|
| `docker build -t api .` | Constrói a imagem `api` a partir do `Dockerfile` da pasta atual |
| `docker images` | Lista as imagens locais |
| `docker rmi api` | Remove uma imagem |

### Containers

| Comando | O que faz |
|---|---|
| `docker run -p 3333:3333 api` | Cria e roda um container (`porta-do-PC:porta-do-container`) |
| `docker run -d --name api -p 3333:3333 api` | Roda em segundo plano e com nome fixo |
| `docker run --rm api` | Apaga o container automaticamente quando ele parar |
| `docker ps` | Lista os containers em execução |
| `docker ps -a` | Lista todos, inclusive os parados |
| `docker stop api` / `docker start api` | Para / inicia um container existente |
| `docker rm -f api` | Remove o container, mesmo em execução |
| `docker container prune` | Remove todos os containers parados |

### Depuração

| Comando | O que faz |
|---|---|
| `docker logs -f api` | Acompanha os logs em tempo real |
| `docker exec -it api sh` | Abre um terminal dentro do container (Alpine usa `sh`, não `bash`) |
| `docker inspect api` | Mostra os detalhes completos do container |

## 🔧 Problemas que encontrei e como resolvi

Esta é a parte mais valiosa do estudo. Cada erro ensinou algo sobre como o Docker funciona.

### 1. `failed to connect to the docker API at npipe:////./pipe/docker_engine`

**Causa:** o Docker Desktop não estava rodando. O comando `docker` é só um cliente e precisa falar com o daemon.

**Solução:** abrir o Docker Desktop e esperar aparecer **Engine running**. Se não subir, verificar o WSL 2 (`wsl --status`, `wsl --update`) e a virtualização na BIOS.

### 2. `open Dockerfile: no such file or directory`

**Causa:** o `Dockerfile` estava dentro de `src`, e não na raiz. O `docker build -t api .` procura o arquivo na pasta onde o comando é executado.

**Solução:** mover o `Dockerfile` e o `.dockerignore` para a raiz do projeto.

### 3. `Bind for 0.0.0.0:3333 failed: port is already allocated`

**Causa:** a porta 3333 do meu computador já estava em uso, por outro container ou pelo servidor local rodando com `tsx`.

**Solução:**

```bash
docker ps                            # vê quem está usando a porta
docker stop <id>                     # para o container
netstat -ano | findstr :3333         # no Windows, descobre o processo
```

Ou usar outra porta do lado do PC: `docker run -p 3334:3333 api`.

### 4. Vários containers acumulados (`interesting_curran`, `nice_mccarthy`...)

**Causa:** cada `docker run` cria um **container novo**, com nome aleatório. As tentativas que falharam por porta ocupada deixaram containers criados, mas parados.

**Solução:** dar nome fixo (`--name api`), usar `start`/`stop` para o mesmo container, `--rm` em testes rápidos e limpar com `docker container prune`.

### 5. `exec: "bash": executable file not found in $PATH`

**Causa:** a imagem `node:20-alpine` não tem `bash`, só `sh`.

**Solução:** `docker exec -it api sh`.

### 6. `WORKDIR` com caminho do Windows

**Causa:** usei algo como `WORKDIR \Estudos\...` no Dockerfile. O `WORKDIR` é um caminho **dentro do container**, que é Linux.

**Solução:** `WORKDIR /app`.

### 7. Erro `Cannot find name 'node:http'` no VS Code (ts 2591)

**Causa:** faltava o `@types/node` ou o editor estava com cache antigo. O `npx tsc --noEmit` passou sem erros, então o código estava correto.

**Solução:**

```bash
npm install -D @types/node
```

E no VS Code: `Ctrl + Shift + P` → **TypeScript: Restart TS Server**.

## 🧠 Conceitos que fixei

- **Imagem** é o modelo (a receita pronta). **Container** é uma instância em execução dessa imagem. Posso criar vários containers da mesma imagem.
- Containers são **descartáveis**. O que importa é a imagem e o `Dockerfile`.
- O `docker run` **sempre cria** um container novo. Para reaproveitar um existente, uso `docker start`.
- O mapeamento `-p 3333:3333` conecta a porta do meu computador à porta do container. Sem ele, a aplicação roda, mas não é acessível de fora.
- Cada instrução do Dockerfile vira uma **camada com cache**. A ordem das instruções afeta a velocidade do build.
- O código dentro da imagem é uma cópia do momento do build. Alterou o código, **reconstrói a imagem**.
- No Windows, o Docker Desktop roda sobre o **WSL 2**, que executa o kernel Linux onde os containers rodam.
- `npm ci` é mais rápido e reproduzível que `npm install` em builds automatizados.
- Alpine é leve, mas minimalista: usa `sh` em vez de `bash`.

## 🚀 Próximos passos

- [ ] Usar **multi-stage build** para gerar uma imagem final menor (só com `dist` e dependências de produção)
- [ ] Rodar o container como usuário sem privilégios (`USER node`)
- [ ] Criar um `docker-compose.yml`
- [ ] Adicionar um banco de dados (PostgreSQL) em outro container
- [ ] Aprender **volumes** para persistir dados
- [ ] Entender **redes** entre containers
- [ ] Publicar a imagem no Docker Hub
- [ ] Automatizar o build com GitHub Actions

---

Feito com muita tentativa, erro e `docker logs` 🐳
