# Raízes do Nordeste — API Backend

API REST para gestão de uma rede de lanchonetes. Cobre autenticação multi-perfil, cardápio, estoque, pedidos com pagamento mock, programa de fidelidade e auditoria de logs.

## Stack

- **Runtime:** Node.js + Express
- **ORM / Banco:** Prisma + PostgreSQL
- **Auth:** JWT
- **Validação:** Zod
- **Docs:** Swagger UI (`/api-docs`)
- **Testes:** Jest + Supertest

---

## Pré-requisitos

- Node.js ≥ 18
- PostgreSQL ≥ 14 (rodando localmente ou via container)
- npm

---

## Instalação e configuração

```bash
# 1. Instalar dependências
npm install

# 2. Criar o arquivo de variáveis de ambiente
cp .env.example .env
```

Edite o `.env` com os valores reais:

| Variável | Descrição | Exemplo |
|---|---|---|
| `DATABASE_URL` | String de conexão PostgreSQL | `postgresql://postgres:senha@localhost:5432/raizes_db` |
| `PORT` | Porta do servidor | `3000` |
| `NODE_ENV` | Ambiente | `development` |
| `JWT_SECRET` | Chave secreta para assinar tokens | string longa e aleatória |
| `JWT_EXPIRES_IN` | Validade do access token | `1h` |
| `JWT_REFRESH_EXPIRES_IN` | Validade do refresh token | `7d` |
| `PAYMENT_MOCK_MODE` | Comportamento do mock de pagamento | `always_approve` \| `always_reject` \| `random` |
| `DATA_RETENTION_DAYS` | Retenção de dados para LGPD (dias) | `365` |

> Para testes manuais com resultado determinístico, use `PAYMENT_MOCK_MODE="always_approve"`.

```bash
# 3. Criar o banco e rodar as migrations
npx prisma migrate dev

# 4. Popular o banco com dados de seed
npm run seed
```

O seed imprime no terminal as credenciais de todos os usuários criados (cliente de teste, colaboradores por perfil e admin central). Guarde-as para os primeiros logins.

---

## Subindo o servidor

```bash
# Produção / demonstração
npm start

# Desenvolvimento (hot-reload com nodemon)
npm run dev
```

A API sobe em `http://localhost:3000`.
Documentação interativa: `http://localhost:3000/api-docs`.

---

## Scripts disponíveis

| Comando | O que faz |
|---|---|
| `npm start` | Inicia o servidor (`node src/server.js`) |
| `npm run dev` | Inicia com nodemon (reload automático) |
| `npm run seed` | Popula o banco com dados iniciais |
| `npm run migrate` | Cria/aplica migrations em desenvolvimento |
| `npm run migrate:prod` | Aplica migrations sem prompt (CI/produção) |
| `npm run generate` | Regenera o Prisma Client após mudança no schema |
| `npm run studio` | Abre o Prisma Studio (GUI do banco) |
| `npm test` | Executa todos os testes |
| `npm run test:coverage` | Testes com relatório de cobertura |
| `npm run test:watch` | Testes em modo watch |

---

## Variáveis de ambiente para testes

Crie um `.env.test` a partir do exemplo:

```bash
cp .env.test.example .env.test
```

O arquivo de teste usa um banco separado (`raizes_db_test`) e `PAYMENT_MOCK_MODE="always_approve"` para resultados determinísticos.

---

## Rotas da API

| Prefixo | Descrição |
|---|---|
| `POST /auth/login` | Login (cliente, colaborador ou admin central) |
| `POST /auth/cadastro` | Cadastro de novo cliente |
| `/usuarios` | Gestão de usuários admin centrais |
| `/unidades` | CRUD de unidades da rede |
| `/unidades/:unidadeId/estoque` | Consulta e movimentações de estoque |
| `/produtos` | Cardápio global + sugestões de itens por unidade |
| `/pedidos` | Criação, listagem e atualização de status de pedidos |
| `/pagamentos` | Consulta de pagamentos |
| `/fidelidade` | Saldo de pontos, histórico e resgate |
| `/api-docs` | Swagger UI com todos os endpoints documentados |

---

## Fluxo principal (demo rápida via curl)

### 1. Login

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"cliente@teste.com","senha":"Cliente@123","tipo":"cliente"}'
```

Guarde o `accessToken` retornado.

### 2. Consultar cardápio

```bash
curl http://localhost:3000/produtos?unidadeId=1 \
  -H "Authorization: Bearer <accessToken>"
```

Anote o `id` de um item e o `id` de uma variação dentro de `variacoes`.

### 3. Criar pedido

```bash
curl -X POST http://localhost:3000/pedidos \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <accessToken>" \
  -d '{
    "unidadeId": 1,
    "canalPedido": "APP",
    "itens": [{ "itemId": 1, "variacaoId": 1, "quantidade": 1 }],
    "formaPagamento": "PIX"
  }'
```

A resposta `201` já traz o resultado do pagamento mock (`pagamentoResultado`) e o status final do pedido (`EM_PREPARO` ou `CANCELADO`).

### 4. Avançar status (perfil COZINHA)

```bash
# Faça login com as credenciais de cozinha impressas pelo seed (tipo: "colaborador")
curl -X PATCH http://localhost:3000/pedidos/<pedidoId>/status \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <tokenCozinha>" \
  -d '{"status":"PRONTO"}'
```

Transições permitidas: `AGUARDANDO_PAGAMENTO → EM_PREPARO → PRONTO → ENTREGUE`. Qualquer estado pode ir para `CANCELADO`. `ENTREGUE` e `CANCELADO` são terminais.

---

## Postman

A coleção `raizes-backend.postman_collection.json` na raiz do projeto contém todos os requests organizados por módulo. Os scripts de teste preenchem automaticamente as variáveis de coleção (`tokenCliente`, `pedidoId` etc.) conforme os requests são executados em ordem.

---

## Perfis de acesso

| Perfil | Tipo | Permissões principais |
|---|---|---|
| `ADMIN` | Central | Acesso total |
| `FINANCEIRO` | Central | Leitura de pedidos e pagamentos |
| `MARKETING` | Central | Cardápio, sugestões |
| `RH_CENTRAL` | Central | Leitura geral |
| `SUPORTE` | Central | Leitura geral |
| `GERENTE` | Colaborador | Gestão da própria unidade (estoque, cardápio, pedidos) |
| `ATENDENTE` | Colaborador | Criar e listar pedidos |
| `COZINHA` | Colaborador | Atualizar status de pedidos |
| `LIMPEZA` | Colaborador | Acesso restrito |
| `CLIENTE` | App/Web | Cadastro, pedidos, fidelidade |
