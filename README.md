# Full Cycle Clean Architecture

Projeto TypeScript implementando Clean Architecture com casos de uso para as entidades Customer e Product.

## Requisitos

- Node.js >= 18
- npm

## Instalação

```bash
npm install
```

## Validação Desacoplada — Entidade Product

A entidade `Product` utiliza o **Notification Pattern** combinado com o **Validator Pattern**: as regras de validação são delegadas a uma classe especializada (`ProductYupValidator`), criada via `ProductValidatorFactory`. A entidade não contém mais lógica de validação direta (sem if/else soltos).

**Fluxo de validação:**

```
Product.validate()
  └─ ProductValidatorFactory.create()   → ProductYupValidator
       └─ Yup.validateSync(abortEarly: false)
            └─ entity.notification.addError(...)  ← acumula todos os erros
  └─ notification.hasErrors() → throw NotificationError
```

**Exemplo de comportamento:**

```typescript
// Todos os erros são acumulados e lançados de uma vez
new Product("123", "", -1);
// throw NotificationError([
//   { context: "product", message: "Name is required" },
//   { context: "product", message: "Price must be greater than zero" },
// ])
```

**Arquivos relacionados:**

| Arquivo | Descrição |
|---|---|
| `src/domain/product/entity/product.ts` | Entidade — delega validação ao factory |
| `src/domain/product/factory/product.validator.factory.ts` | Factory que instancia o validador concreto |
| `src/domain/product/validator/product.yup.validator.ts` | `ProductYupValidator` — valida com Yup |
| `src/domain/@shared/notification/notification.ts` | Container acumulador de erros |
| `src/domain/@shared/notification/notification.error.ts` | Erro lançado com todos os erros acumulados |

### Rodando os testes do Validator Pattern

```bash
./node_modules/.bin/jest src/domain/product/entity/product.spec.ts
```

O teste obrigatório de múltiplos erros simultâneos está em `product.spec.ts`:

```
✓ should accumulate multiple validation errors simultaneously
```

Ele força `name` vazio e `price` negativo ao mesmo tempo e verifica que a `NotificationError` contém ambos os erros.

## Executando os Testes

### Todos os testes

```bash
npm test
```

> `npm test` executa a verificação de tipos do TypeScript (`tsc --noEmit`) seguida da suíte completa do Jest.

### Apenas testes de unidade

```bash
./node_modules/.bin/jest --testPathPattern="unit.spec"
```

### Apenas testes de integração

```bash
./node_modules/.bin/jest --testPathPattern="integration.spec"
```

### Apenas casos de uso de Product

```bash
./node_modules/.bin/jest --testPathPattern="src/usecase/product"
```

### Apenas casos de uso de Customer

```bash
./node_modules/.bin/jest --testPathPattern="src/usecase/customer"
```

## API

### Executando a API

```bash
npm run dev
```

A API sobe por padrão na porta `3000` (configurável via variável de ambiente `PORT` no arquivo `.env`).

### Endpoints disponíveis

#### Customer

| Método | Rota        | Descrição              |
|--------|-------------|------------------------|
| POST   | `/customer` | Cria um novo customer  |
| GET    | `/customer` | Lista todos os customers |

#### Product

| Método | Rota       | Descrição             |
|--------|------------|-----------------------|
| GET    | `/product` | Lista todos os products |

A rota `GET /product` suporta negociação de conteúdo:
- `Accept: application/json` (padrão) — retorna JSON
- `Accept: application/xml` — retorna XML

**Exemplo de resposta JSON:**
```json
{
  "products": [
    { "id": "uuid", "name": "Product 1", "price": 10.0 }
  ]
}
```

**Exemplo de resposta XML:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<products>
  <product>
    <id>uuid</id>
    <name>Product 1</name>
    <price>10</price>
  </product>
</products>
```

### Testes E2E

```bash
./node_modules/.bin/jest --testPathPattern="e2e.spec"
```

Os testes E2E simulam requisições HTTP reais contra a API usando **supertest** com banco SQLite em memória.

| Arquivo de Teste                          | Cobertura                                         |
|-------------------------------------------|---------------------------------------------------|
| `customer.e2e.spec.ts`                    | POST `/customer`, GET `/customer` (JSON e XML)    |
| `product.e2e.spec.ts`                     | GET `/product` (JSON, XML, lista vazia)           |

## Cobertura de Testes

Cada caso de uso possui testes de **unidade** e de **integração**:

| Caso de Uso | Teste de Unidade | Teste de Integração |
|---|---|---|
| CreateProduct | `create.product.unit.spec.ts` | `create.product.integration.spec.ts` |
| FindProduct | `find.product.unit.spec.ts` | `find.product.integration.spec.ts` |
| ListProduct | `list.product.unit.spec.ts` | `list.product.integration.spec.ts` |
| UpdateProduct | `update.product.unit.spec.ts` | `update.product.integration.spec.ts` |
| CreateCustomer | `create.customer.unit.spec.ts` | — |
| FindCustomer | `find.customer.unit.spec.ts` | `find.customer.integration.spec.ts` |
| ListCustomer | `list.customer.unit.spec.ts` | — |
| UpdateCustomer | `update.customer.unit.spec.ts` | — |

**Testes de unidade** utilizam repositórios mockados e validam a lógica de negócio de forma isolada.

**Testes de integração** utilizam um banco de dados SQLite em memória via Sequelize e validam o fluxo completo de ponta a ponta.
