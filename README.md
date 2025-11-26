# Controle de estoque 

Este pequeno sistema tem 3 requisitos:
1. Permitir adicionar novos produtos ao estoque.
2. Permitir listar os produtos cadastrados no estoque.
3. Permitir atualizar a quantidade de produtos no estoque.

## Criação do projeto


Este guia descreve rapidamente os passos para criar um projeto Node.js com TypeScript e Prisma.

```bash
# 1. Cria o diretório do projeto
mkdir n-tier

# 2. Entra no diretório recém-criado
cd n-tier/

# 3. Inicializa um projeto Node.js com um package.json padrão
npm init -y

# 4. Instala TypeScript, tsx (para rodar TS diretamente) e tipos do Node como devDependencies
npm install typescript tsx @types/node --save-dev

# 5. Gera o arquivo de configuração padrão do TypeScript (tsconfig.json)
npx tsc --init

# 6. Instala Prisma e os tipos para Node e PostgreSQL como devDependencies
npm install prisma @types/node @types/pg @prisma/adapter-pg  @prisma/client --save-dev

# 7. Inicializa o Prisma, gerando a estrutura de arquivos na pasta ../generated/prisma
npx prisma init --output ../generated/prisma

```


## Abra o VS Code dentro da pasta do projeto

```bash
code .
```

## Ajuste o projeto para se comportar como um módulo ES

Desse modo, você poderá utilizar a sintaxe de import/export do ES Modules.

No arquivo `package.json`, adicione a seguinte linha para definir o tipo do módulo como ES:

```json
"type": "module",
```

Altere também a configuração do `tsconfig.json` para garantir que o TypeScript gere código compatível com ES Modules. 

Deixe seu `tsconfig.json` parecido com o exemplo abaixo:

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "node",
    "target": "ES2023",
    "strict": true,
    "esModuleInterop": true,
    "ignoreDeprecations": "6.0"
  }
}
```


## Configuração do banco de dados


### 1. Edite o arquivo prisma/schema.prisma

No arquivo `prisma/schema.prisma`, configure o datasource para usar PostgreSQL e defina o modelo `Produto`:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}


model Produto {
  id         Int      @id @default(autoincrement())
  nome       String
  quantidade Int      @default(0)

  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@map("produtos") // nome da tabela no Postgres
}
```

### 2. Configure a variável de ambiente DATABASE_URL

No arquivo `.env`, configure a variável `DATABASE_URL` com a string de conexão do seu banco de dados PostgreSQL:

```
DATABASE_URL="postgresql://usuario:senha@localhost:5432/nome_do_banco?schema=public"
```

Ajuste `usuario`, `senha`, `localhost`, `5432` e `nome_do_banco` conforme necessário.

**Na UFR, o usuário admin é `postgres` e a senha é `abc`.**

### 3. Execute a migração inicial

Execute o comando abaixo para criar a tabela `produtos` no banco de dados:

```bash
npx prisma migrate dev --name init
```

O comando irá criar o banco de dados se ainda não estiver criado e também deve criar a tabela `produtos` conforme o modelo definido.

### 4. Gere o cliente Prisma

Em seguida, gere o cliente Prisma para interagir com o banco de dados:

```bash
npx prisma generate
```

## Criação do Prisma Client

Neste passo vamos criar um arquivo para inicializar o Prisma Client. 
Vamos exportar e utilizar esse cliente em outras partes do projeto.

- Crie uma pasta chamada `lib` na raiz do seu projeto. 
- Dentro da pasta `lib`, crie um arquivo chamado `prisma.ts` com o seguinte conteúdo:

```typescript
import "dotenv/config";
import { PrismaPg } from '@prisma/adapter-pg'
import { PrismaClient } from '../generated/prisma/client'

const connectionString = `${process.env.DATABASE_URL}`

const adapter = new PrismaPg({ connectionString })
const prisma = new PrismaClient({ adapter })

export { prisma }
```

## Testando a conexão com o banco de dados

### 1 - Crie um diretório chamado `tools` e dentro dele um arquivo `criar.ts` com o seguinte conteúdo:

```typescript
import { prisma } from '../lib/prisma.js'
async function main() {
  const novoProduto = await prisma.produto.create({
    data: {
      nome: 'Produto Exemplo',
      quantidade: 10,
    },
  })
  console.log(novoProduto)
}
main()
  .catch((e) => {
    console.error(e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

Execute o arquivo `criar.ts` usando o comando abaixo:

```bash
npx tsx tools/criar.ts
```

### 2 - Dentro do diretório `tools` crie um arquivo chamado `ler.ts` com o seguinte conteúdo:

```typescript
import { prisma } from '../lib/prisma.js'

async function main() {
  const produtos = await prisma.produto.findMany()
  console.log(produtos)
}
main()
  .catch((e) => {
    console.error(e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

Execute o arquivo `ler.ts` usando o comando abaixo:

```bash
npx tsx tools/ler.ts
```

## Criação das Entidades de Produto

Crie uma pasta chamada `entities` na raiz do projeto.
Dentro da pasta `entities`, crie um arquivo chamado `Produto.ts` com o seguinte conteúdo:

```typescript
export class Produto {
  constructor(
    public id: number,
    public nome: string,
    public quantidade: number,
    public createdAt?: Date,
    public updatedAt?: Date
  ) {}
}

O objetivo desse arquivo é definir a estrutura da entidade Produto que será utilizada em todo o sistema. 
Já temos uma estrutura parecida no Prisma. Porém, essa estrutura criada aqui é independente do Prisma e pode ser utilizada em qualquer camada do sistema.
Ela não carrega nenhuma dependência do Prisma, o que facilita a manutenção e evolução do código ao longo do tempo. 

É evidente que se sua aplicação for muito pequena, ou mesmo só conter cadastros, você pode optar por não criar essa camada de entidades. 

```

## Criação do Repositório de Produtos

Crie uma pasta chamada `repos` na raiz do projeto.

Dentro da pasta `repos`, crie um arquivo chamado `ProdutoRepo.ts` com o seguinte conteúdo:

```typescript
import { prisma } from '../lib/prisma.js'
import { Produto } from '../entities/Produto.js'

export class ProdutoRepo {
  async adicionarProduto(nome: string, quantidade: number): Promise<Produto> {
    const produto = await prisma.produto.create({
      data: { nome, quantidade },
    })
    return new Produto(
      produto.id,
      produto.nome,
      produto.quantidade,
      produto.createdAt,
      produto.updatedAt
    )
  }
  async listarProdutos(): Promise<Produto[]> {
    const produtos = await prisma.produto.findMany()
    return produtos.map(
      (produto) =>
        new Produto(
          produto.id,
          produto.nome,
          produto.quantidade,
          produto.createdAt,
          produto.updatedAt
        )
    )
  }
  async atualizarQuantidade(
    id: number,
    quantidade: number
  ): Promise<Produto | null> {
    const produto = await prisma.produto.update({
      where: { id },
      data: { quantidade },
    })
    return new Produto(
      produto.id,
      produto.nome,
      produto.quantidade,
      produto.createdAt,
      produto.updatedAt
    )
  }


  async obterProdutoPorId(id: number): Promise<Produto | null> {
    const produto = await prisma.produto.findUnique({
      where: { id },
    })
    if (!produto) {
      return null
    }
    return new Produto(
      produto.id,
      produto.nome,
      produto.quantidade,
      produto.createdAt,
      produto.updatedAt
    )
  }
}
```

## Criação dos Casos de Uso / Services

Crie uma pasta chamada `services` na raiz do projeto.

Dentro da pasta `services`, crie um arquivo chamado `ProdutoService.ts` com o seguinte conteúdo:

```typescript
import { Produto } from '../entities/Produto.js'
import { ProdutoRepo } from '../repos/ProdutoRepo.js'

export class ProdutoService {
  constructor(private produtoRepo: ProdutoRepo) {}

  async adicionarProduto(
    nome: string,
    quantidade: number
  ): Promise<Produto> {
    return this.produtoRepo.adicionarProduto(nome, quantidade)
  }

  async listarProdutos(): Promise<Produto[]> {
    return this.produtoRepo.listarProdutos();
  }

  async incrementarQuantidade(
    id: number,
    quantidade: number
  ): Promise<Produto | null> {

    if (quantidade <= 0) {
      throw new Error('A quantidade a ser incrementada deve ser maior que zero.')
    }

    let produto = await this.produtoRepo.obterProdutoPorId(id);

    if (produto === null) {
      throw new Error(`Produto com ID ${id} não encontrado.`)
    }

    produto = await this.produtoRepo.atualizarQuantidade(id, produto.quantidade + quantidade)
    return produto;    
  }

  async decrementarQuantidade(
    id: number,
    quantidade: number
  ): Promise<Produto | null> {

    if (quantidade <= 0) {
      throw new Error('A quantidade a ser decrementada deve ser maior que zero.')
    }

    let produto = await this.produtoRepo.obterProdutoPorId(id);

    if (!produto) {
      throw new Error(`Produto com ID ${id} não encontrado.`)
    }

    if (produto.quantidade < quantidade) {
      throw new Error(`Quantidade insuficiente em estoque para o produto com ID ${id}.`)
    }

    const novaQuantidade = produto.quantidade - quantidade;
    produto = await this.produtoRepo.atualizarQuantidade(id, novaQuantidade)
    return produto;    
  }
}
```

O Objetivo do service é implementar as regras de negócio da aplicação. Ela utiliza o repositório para acessar os dados e aplicar as regras necessárias antes de retornar os resultados ou fazer alterações no banco de dados.


## Criação do Adaptador de Linha de Comando (CLI)

Instale o pacote `commander` para facilitar a criação de uma interface de linha de comando:

```bash
npm install commander
```

Crie uma pasta chamada `cli` na raiz do projeto. 

Dentro da pasta `cli`, crie um arquivo chamado `index.ts` com o seguinte conteúdo:

```typescript
import { Command } from 'commander'
import { ProdutoRepo } from '../repos/ProdutoRepo.js'
import { ProdutoService } from '../services/ProdutoService.js'
const program = new Command()
const produtoRepo = new ProdutoRepo()
const produtoService = new ProdutoService(produtoRepo)

program
  .name('estoque-cli')
  .description('CLI para controle de estoque')
  .version('1.0.0');

program
  .command('adicionar')
  .description('Adiciona um novo produto ao estoque')
  .requiredOption('-n, --nome <string>', 'Nome do produto')
  .requiredOption('-q, --quantidade <number>', 'Quantidade do produto', parseInt)
  .action(async (options) => {
    const produto = await produtoService.adicionarProduto(
      options.nome,
      options.quantidade
    )
    console.log('Produto adicionado:', produto)
  });

program
  .command('listar')
  .description('Lista todos os produtos no estoque')
  .action(async () => {
    const produtos = await produtoService.listarProdutos()
    console.log('Produtos no estoque:', produtos)
  });

program
  .command('incrementar')
  .description('Incrementa a quantidade de um produto no estoque')
  .requiredOption('-i, --id <number>', 'ID do produto', parseInt)
  .requiredOption('-q, --quantidade <number>', 'Quantidade a ser incrementada', parseInt)
  .action(async (options) => {
    try {
      const produto = await produtoService.incrementarQuantidade(
        options.id,
        options.quantidade
      )
      console.log('Quantidade incrementada:', produto)
    } catch (error) {
      console.error('Erro ao incrementar quantidade:', error.message)
    }
  });

program
  .command('decrementar')
  .description('Decrementa a quantidade de um produto no estoque')
  .requiredOption('-i, --id <number>', 'ID do produto', parseInt)
  .requiredOption('-q, --quantidade <number>', 'Quantidade a ser decrementada', parseInt)
  .action(async (options) => {
    try {
      const produto = await produtoService.decrementarQuantidade(
        options.id,
        options.quantidade
      )
      console.log('Quantidade decrementada:', produto);
    } catch (error) {
      console.error('Erro ao decrementar quantidade:', error.message)
    }
  });

program.parse(process.argv);
```

Para executar a CLI, use o comando abaixo, seguido do comando desejado e suas opções:

```bash
npx tsx cli/index.ts <comando> [opções]
```

### Exemplos de uso da CLI:

- Adicionar um novo produto:

```bash
npx tsx cli/index.ts adicionar -n "Produto A" -q 100
```

- Listar todos os produtos:

```bash
npx tsx cli/index.ts listar
```

- Incrementar a quantidade de um produto:

```bash
npx tsx cli/index.ts incrementar -i 1 -q 50
```

- Decrementar a quantidade de um produto:

```bash
npx tsx cli/index.ts decrementar -i 1 -q 30
```

## Adaptador Web utilizando Express.js

Instale o Express.js e os tipos para TypeScript:

```bash
npm install express
npm install @types/express --save-dev
```

Crie uma pasta chamada `web` na raiz do projeto.
Dentro da pasta `web`, crie um arquivo chamado `server.ts` com o seguinte conteúdo:

```typescript

import express from 'express'
import { ProdutoRepo } from '../repos/ProdutoRepo.js'
import { ProdutoService } from '../services/ProdutoService.js'

const app = express()
const port = 3000
const produtoRepo = new ProdutoRepo()
const produtoService = new ProdutoService(produtoRepo)

app.use(express.json())

app.post('/produtos', async (req, res) => {
  const { nome, quantidade } = req.body
  try {
    const produto = await produtoService.adicionarProduto(nome, quantidade)
    res.status(201).json(produto)
  } catch (error: any) {
    res.status(400).json({ error: error?.message })
  }
})
app.get('/produtos', async (req, res) => {
  const produtos = await produtoService.listarProdutos()
  res.json(produtos)
})

app.patch('/produtos/:id/incrementar', async (req, res) => {
  const id = parseInt(req.params.id)
  const { quantidade } = req.body
  try {
    const produto = await produtoService.incrementarQuantidade(id, quantidade)
    res.json(produto)
  } catch (error: any) {
    res.status(400).json({ error: error?.message })
  }
})

app.patch('/produtos/:id/decrementar', async (req, res) => {
  const id = parseInt(req.params.id)
  const { quantidade } = req.body
  try {
    const produto = await produtoService.decrementarQuantidade(id, quantidade)
    res.json(produto)
  } catch (error: any) {
    res.status(400).json({ error: error?.message })
  }
})

app.listen(port, () => {
  console.log(`Servidor rodando em http://localhost:${port}`)
})

```

Para iniciar o servidor web, use o comando abaixo:

```bash
npx tsx web/server.ts
```

O servidor estará rodando em `http://localhost:3000`.

Para testar, podemos utilizar a ferramenta `curl`. Se caso estiver no Windows, você pode usar o PowerShell ou instalar o Git Bash para ter acesso ao `curl`. No Linux e MacOS, se caso não já tiver instalado, você pode instalar via gerenciador de pacotes.

Você deve abrir outro terminal para executar os comandos abaixo:

### Adicionar um novo produto

```bash
curl -X POST http://localhost:3000/produtos -H "Content-Type: application/json" -d '{"nome": "Produto B", "quantidade": 50}'
```

### Listar todos os produtos

```bash
curl http://localhost:3000/produtos
```

### Incrementar a quantidade de um produto

```bash
curl -X PATCH http://localhost:3000/produtos/1/incrementar -H "Content-Type: application/json" -d '{"quantidade": 20}'
```

### Decrementar a quantidade de um produto

```bash
curl -X PATCH http://localhost:3000/produtos/1/decrementar -H "Content-Type: application/json" -d '{"quantidade": 10}'
```


## Conclusão

Neste guia, você criou um sistema de controle de estoque utilizando Node.js, TypeScript e Prisma.
O sistema foi estruturado em camadas, incluindo entidades, repositórios, serviços e adaptadores para linha de comando e web.
Toda a lógica de negócio foi encapsulada na camada de serviços e foi utilizada pelos dois adaptadores. 


**Uma única implementação de regras de negócio para múltiplas interfaces.**

