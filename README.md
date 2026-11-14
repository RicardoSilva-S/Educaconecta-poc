# EducaConecta — PoC (AEP 2026.2 — 6S — Engenharia de Software)

> **Versão atual: 2.0.0 — Entrega 2 de 2 (versão final).**
> Tags no GitHub: [`v1.0.0`](https://github.com/RicardoSilva-S/educaconecta-poc/tree/v1.0.0) = 1ª entrega · [`v2.0.0`](https://github.com/RicardoSilva-S/educaconecta-poc/tree/v2.0.0) = 2ª entrega (final).

Sistema para ONGs que oferecem **reforço escolar gratuito** a crianças e adolescentes em situação de vulnerabilidade. Ele gerencia alunos, voluntários, turmas, matrículas, frequência e vagas por turno.

**Equipe:** Ricardo Henrique Silva de Souza · Victor _(sobrenome)_ · _(integrante 3, se houver)_
**Quadro de tarefas:** _(cole aqui o link do GitHub Projects/Trello)_ — resumo em [docs/METODOLOGIA.md](docs/METODOLOGIA.md)
**Vídeo de demonstração:** _(cole aqui o link do vídeo)_

---

## 1. Problema e ODS

ONGs de reforço escolar costumam controlar inscrições em planilhas soltas ou em papel. Isso causa cadastros duplicados, perda dos contatos dos responsáveis e **dificuldade para saber quantas vagas ainda restam em cada turma e turno**.

**ODS 4 — Educação de Qualidade.** O sistema facilita a gestão de vagas e o acompanhamento da frequência dos alunos, o que ajuda a ONG a manter mais crianças em atividades educacionais.

## 2. Evolução em relação à 1ª entrega

| Aspecto | Entrega 1 (`v1.0.0`) | Entrega 2 (`v2.0.0`) |
|---|---|---|
| Coleções | 1 (`alunos`) | **4** (`alunos`, `voluntarios`, `turmas`, `matriculas`) |
| Relacionamentos | — | `turmas → voluntarios` (1:N) e `matriculas` ligando `alunos ↔ turmas` (N:N) |
| Objetos complexos | Documento plano | `endereco` (documento aninhado), `responsaveis[]`, `horarios[]` e `frequencias[]` (listas de subdocumentos) |
| Regras de negócio | Validação de campos | Vagas por turma, compatibilidade de turno, matrícula duplicada, frequência, integridade ao remover |
| Consultas | `find` simples | Agregações com `$lookup`, `$filter` e `$size` (relatório de vagas, alunos da turma) |
| Banco | Sem índices | Índices (incluindo um **único parcial**) e validação `$jsonSchema` |
| POO | Classes por camada | **Herança e polimorfismo**: `Entidade`, `BaseRepository`, `CrudService`, `BaseController`, hierarquia de erros |
| Testes | 53 testes, 98,5% | **160 testes, 99,8%** + testes de integração com MongoDB real |
| Endpoints | 6 (incluindo `/health`) | 24 (incluindo `/health`) |

Detalhes em [CHANGELOG.md](CHANGELOG.md).

## 3. Tecnologias

| Camada | Tecnologia |
|---|---|
| Linguagem / paradigma | JavaScript (Node.js 18+), Programação Orientada a Objetos |
| Banco de dados NoSQL | MongoDB 7 (driver oficial `mongodb` 6.x) — orientado a documentos |
| API HTTP | Express 4 |
| Testes | Jest (unitários e de fluxo), Supertest (HTTP), mingo (banco em memória nos testes de fluxo), mongodb-memory-server (integração) |
| Ambiente | Docker Compose (MongoDB local), dotenv |

## 4. Arquitetura (resumo)

```
src/
  models/         Entidade (abstrata), Aluno, Endereco, Responsavel, Voluntario, Turma, Matricula
  validation/     Validador (regras reutilizadas por todas as entidades)
  errors/         DomainError → ValidationError (400), NotFoundError (404), BusinessRuleError (409)
  repositories/   BaseRepository (CRUD genérico) → Aluno/Voluntario/Turma/MatriculaRepository
  services/       CrudService (Template Method) → Aluno/Voluntario/TurmaService; MatriculaService
  controllers/    BaseController → CrudController → Aluno/Voluntario/TurmaController; MatriculaController
  routes/         crudRoutes (genérica) + rotas específicas
  db/             database (conexão), colecoes (nomes), indices (índices + $jsonSchema)
  container.js    composition root: liga coleções → repositórios → services
  app.js          monta o Express recebendo os services (injeção de dependência)
  server.js       ponto de entrada
```

Fluxo de uma requisição: **rota → controller → service (regras) → repository → MongoDB**. Cada camada recebe a de baixo pelo construtor (injeção de dependência), então tudo é testável sem banco real.
Diagrama de classes e decisões de projeto em [docs/ARQUITETURA.md](docs/ARQUITETURA.md).

## 5. Estrutura do banco (resumo)

```
voluntarios ──1:N──> turmas ──1:N──> matriculas <──N:1── alunos
                    (voluntarioId)   (turmaId, alunoId)
```

Exemplo de documento da coleção `alunos` (objeto complexo):

```json
{
  "_id": "ObjectId(...)",
  "nome": "Maria Silva",
  "idade": 10,
  "escola": "EM Joao Paulo II",
  "serie": "5o ano",
  "turno": "tarde",
  "endereco": { "rua": "Rua A", "numero": "100", "bairro": "Centro", "cidade": "Maringa", "estado": "PR", "cep": "87000000" },
  "responsaveis": [
    { "nome": "Ana Silva", "parentesco": "mae", "telefone": "44999998888", "principal": true }
  ],
  "ativo": true
}
```

A modelagem completa (todas as coleções, índices, validação, agregações e justificativa de embutir ou referenciar) está em **[docs/BANCO_DE_DADOS.md](docs/BANCO_DE_DADOS.md)**.

## 6. Como instalar e executar

**Pré-requisitos:** Node.js 18+ e um MongoDB acessível (Docker, instalação local ou MongoDB Atlas).

```bash
# 1. dependências
npm install

# 2. MongoDB (opção com Docker; ou use um MongoDB local/Atlas)
docker compose up -d

# 3. configuração
cp .env.example .env        # no Windows: copy .env.example .env  (ajuste MONGODB_URI se precisar)

# 4. (opcional) dados de exemplo: 2 voluntários, 3 turmas, 5 alunos, 5 matrículas
npm run seed

# 5. subir a API
npm start                   # http://localhost:3000
```

Variáveis do `.env`: `MONGODB_URI` (padrão `mongodb://localhost:27017`), `DB_NAME` (padrão `educaconecta`), `PORT` (padrão `3000`).

Se você tiver dados da 1ª entrega no banco, rode `npm run migrar` para converter os alunos antigos (`responsavel`/`telefone`) para o novo formato (`responsaveis[]`).

### Testando a API rapidamente

O arquivo [`requests.http`](requests.http) tem todas as requisições prontas para a extensão *REST Client* do VS Code. Exemplo com curl:

```bash
curl http://localhost:3000/turmas/vagas
```

### Endpoints

| Método | Rota | Descrição |
|---|---|---|
| GET | `/health` | Status da API |
| POST/GET | `/alunos` | Cadastra / lista (`?ativos=true`, `?cidade=Maringa`) |
| GET/PUT/DELETE | `/alunos/:id` | Busca / atualiza (parcial) / remove |
| GET | `/alunos/:id/matriculas` | Matrículas do aluno |
| POST/GET | `/voluntarios` | Cadastra / lista (`?disciplina=matematica`) |
| GET/PUT/DELETE | `/voluntarios/:id` | Busca / atualiza / remove |
| POST/GET | `/turmas` | Cadastra / lista (`?turno=tarde&disciplina=matematica`) |
| GET | `/turmas/vagas` | **Relatório de vagas por turma** (`?turno=tarde`) |
| GET/PUT/DELETE | `/turmas/:id` | Busca / atualiza / remove |
| GET | `/turmas/:id/alunos` | Alunos matriculados, com resumo de frequência |
| POST/GET | `/matriculas` | Matricula aluno em turma / lista (`?alunoId=&turmaId=&status=`) |
| GET | `/matriculas/:id` | Busca matrícula (inclui `percentualFrequencia`) |
| PATCH | `/matriculas/:id/cancelar` | Cancela a matrícula (libera a vaga) |
| POST | `/matriculas/:id/frequencias` | Registra presença/falta `{ "data": "2026-11-02", "presente": true }` |

Corpos de requisição, respostas e códigos de erro estão em [docs/API.md](docs/API.md).

## 7. Testes e cobertura (evidência reproduzível)

```bash
npm test                    # testes unitários + fluxo completo, com relatório de cobertura
npm run test:integration    # (opcional) integração com MongoDB real em memória; baixa o MongoDB na 1ª vez
```

O `npm test` roda o Jest com `--coverage`. O `package.json` define `coverageThreshold` de **70%** (statements, branches, functions e lines), e o comando **falha** se a cobertura ficar abaixo disso. O relatório HTML fica em `coverage/lcov-report/index.html`.

**Resultado da versão 2.0.0:** 160 testes em 9 suítes, todos passando.

| Statements | Branches | Functions | Lines |
|---|---|---|---|
| 99,81% | 97,95% | 99,38% | 99,79% |

Estratégia de testes, o que cada suíte cobre e como ler o relatório: [docs/TESTES.md](docs/TESTES.md).

## 8. Documentação

| Documento | Conteúdo |
|---|---|
| [docs/ARQUITETURA.md](docs/ARQUITETURA.md) | Camadas, diagrama de classes, padrões de POO usados e decisões |
| [docs/BANCO_DE_DADOS.md](docs/BANCO_DE_DADOS.md) | Coleções, exemplos de documentos, relacionamentos, índices, `$jsonSchema`, agregações |
| [docs/API.md](docs/API.md) | Referência dos endpoints, corpos de requisição e erros |
| [docs/TESTES.md](docs/TESTES.md) | Estratégia de testes e cobertura |
| [docs/METODOLOGIA.md](docs/METODOLOGIA.md) | Metodologia (Kanban) e quadro de tarefas |
| [CHANGELOG.md](CHANGELOG.md) | Histórico de versões |

## 9. Versionamento

- `v1.0.0`: 1ª entrega (coleção única `alunos`, CRUD básico).
- `v2.0.0`: 2ª entrega, **versão final** (múltiplas coleções, relacionamentos, objetos complexos).

Os commits seguem o padrão *Conventional Commits* (`feat:`, `test:`, `docs:`, `refactor:`, `chore:`).
