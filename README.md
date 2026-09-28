# API REST de Professores

**Aluno:** PREENCHER NOME COMPLETO  
**Disciplina:** PREENCHER NOME DA DISCIPLINA

API para cadastrar, consultar, filtrar, editar e excluir professores. Projeto acadêmico com Java, Spring Boot, Spring Data JPA e PostgreSQL, organizado em Controller, Service, Repository e Model.

> **Validação realizada:** compilação, 16 testes unitários/web, 9 testes de integração e seis operações HTTP aprovados. Os prints reais estão neste README. O banco executado foi PostgreSQL 18.3 em WebAssembly (PGlite 0.5.8), acessado pelo driver JDBC PostgreSQL. A configuração Docker com PostgreSQL 16 nativo permanece disponível, mas não foi executada neste ambiente. Faltam preencher a identificação e publicar o repositório. Consulte [Validação](docs/VALIDACAO.md).

## Tecnologias

- Java 17
- Spring Boot 3.5.16
- Spring Web, Spring Data JPA e Bean Validation
- PostgreSQL 16 e extensão `unaccent`
- Maven 3.6.3 ou superior
- JUnit 5, Mockito e MockMvc
- Docker Compose (opcional)
- Postman (coleção incluída)

## Como executar

### Opção 1 — Docker Desktop / Docker Compose

Com o Docker iniciado, abra um terminal na pasta que contém `pom.xml` e execute:

```bash
docker compose up --build -d
```

Esse comando cria o PostgreSQL, compila a aplicação e inicia a API com quatro professores de exemplo. A primeira inicialização requer acesso à internet para baixar as imagens e dependências.

- API: `http://localhost:8080/professores`
- Banco: `localhost:5432/professores_db`
- Usuário e senha de desenvolvimento: `postgres` / `postgres`

Para acompanhar a inicialização:

```bash
docker compose logs -f app
```

Para parar, preservando os dados:

```bash
docker compose down
```

As portas 8080 e 5432 precisam estar livres. Se já houver PostgreSQL instalado usando 5432, utilize a opção 2 ou ajuste o mapeamento de porta do serviço `db`.

### Opção 2 — Java, Maven e PostgreSQL instalados

1. Instale Java **JDK 17**, Maven e PostgreSQL 16.
2. No pgAdmin ou `psql`, execute:

```sql
CREATE DATABASE professores_db;
```

3. Configure as variáveis de ambiente se suas credenciais forem diferentes dos valores locais de desenvolvimento.

**PowerShell (Windows):**

```powershell
$env:DB_URL="jdbc:postgresql://localhost:5432/professores_db"
$env:DB_USERNAME="postgres"
$env:DB_PASSWORD="sua_senha"
mvn spring-boot:run "-Dspring-boot.run.profiles=demo"
```

**Linux/macOS:**

```bash
export DB_URL=jdbc:postgresql://localhost:5432/professores_db
export DB_USERNAME=postgres
export DB_PASSWORD=sua_senha
mvn spring-boot:run -Dspring-boot.run.profiles=demo
```

A aplicação executa `schema.sql`, criando a tabela e a extensão, e o Hibernate valida o mapeamento. O usuário do banco precisa ter permissão para criar a tabela e instalar `unaccent` no esquema `public`. Se a instalação não incluir a extensão, instale o pacote de contribuições correspondente à sua versão do PostgreSQL ou solicite ao administrador:

```sql
CREATE EXTENSION IF NOT EXISTS unaccent WITH SCHEMA public;
```

O perfil `demo` executa `data.sql`. Reiniciá-lo não duplica os registros de exemplo que ainda existem; registros de exemplo excluídos são recriados na próxima inicialização com esse perfil. Para usar o banco sem recarregar exemplos, execute `mvn spring-boot:run` sem o perfil.

## Banco de dados

A tabela possui exatamente as cinco colunas solicitadas:

```sql
CREATE TABLE IF NOT EXISTS professor (
    id BIGSERIAL PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL,
    area VARCHAR(100) NOT NULL,
    telefone VARCHAR(20) NOT NULL
);
```

- O banco gera o `id`; não o envie no POST ou PUT.
- Todos os campos de cadastro são obrigatórios; o e-mail deve ser válido.
- Telefone é texto, permitindo DDD, `+`, espaços, parênteses e hífen, dentro do limite de 20 caracteres.
- E-mail não possui restrição de unicidade, pois o enunciado não a exige.
- Os exemplos usam dados fictícios e endereços `example.com`.

## Estrutura

```text
api-professores/
├── src/main/java/br/edu/professores/
│   ├── controller/ProfessorController.java
│   ├── service/ProfessorService.java
│   ├── repository/ProfessorRepository.java
│   ├── model/Professor.java
│   ├── dto/ProfessorRequest.java
│   ├── exception/
│   └── ProfessoresApplication.java
├── src/main/resources/
│   ├── application.properties
│   ├── application-demo.properties
│   ├── schema.sql
│   └── data.sql
├── src/test/
├── database/criar_bancos.sql
├── postman/Professores.postman_collection.json
├── scripts/testar_e_gerar_evidencias.py
├── docs/
├── .github/workflows/testes.yml
├── compose.yaml
├── Dockerfile
├── pom.xml
└── README.md
```

O Controller recebe as requisições e devolve as respostas HTTP; o Service aplica as operações e controla as transações; o Repository faz o acesso ao banco; a entidade Professor representa a tabela. O DTO valida a entrada, e o tratamento centralizado de exceções devolve erros JSON.

## Endpoints

URL base: `http://localhost:8080`

| Método | Endpoint | Descrição | Sucesso |
|---|---|---|---|
| GET | `/professores` | Lista todos, ordenados por id | 200 |
| GET | `/professores/nome/{nome}` | Busca parcial por nome, ignorando caixa e acentos | 200 |
| GET | `/professores/area/{area}` | Busca pela área completa, ignorando caixa | 200 |
| POST | `/professores` | Cadastra professor | 201 |
| PUT | `/professores/{id}` | Atualiza todos os campos do professor | 200 |
| DELETE | `/professores/{id}` | Exclui professor | 204, sem corpo |
| GET | `/professores/{id}` | Consulta individual adicional | 200 |

Filtros sem correspondência retornam `200` com `[]`. Identificador inexistente retorna `404`; dados inválidos, JSON malformado ou id não numérico retornam `400`. O POST devolve o objeto criado e o cabeçalho `Location` para consultar esse recurso.

### Consultas derivadas e acentos

O Repository utiliza, sem `@Query`:

```java
List<Professor> findByNomeBuscaContainingIgnoreCaseOrderByIdAsc(String nome);
List<Professor> findByAreaIgnoreCaseOrderByIdAsc(String area);
```

`nomeBusca` é um atributo calculado de leitura com `@Formula("public.unaccent(nome)")`. Ele não acrescenta uma coluna à tabela, não aparece no JSON e preserva a grafia original do nome. O Service também remove os acentos do parâmetro recebido; a consulta continua sendo derivada pelo Spring Data JPA.

Isso é necessário porque `IgnoreCase` ignora diferenças entre maiúsculas e minúsculas, mas não remove acentos. Assim:

| Busca | Resultado com os dados iniciais |
|---|---|
| `/professores/nome/joao` | João da Silva; João Pedro |
| `/professores/nome/JOA` | João da Silva; João Pedro; MARIA JOANA |
| `/professores/nome/João` | João da Silva; João Pedro |
| `/professores/area/desenvolvimento` | João da Silva; João Pedro |
| `/professores/area/Desenv` | Lista vazia: o filtro de área é exato |

**Observação sobre o enunciado:** “MARIA JOANA” não contém “joao”, mesmo após remover acentos. Para encontrar os três nomes por correspondência parcial, utilize “joa”.

### Exemplos de requisições

No Postman, escolha o método e a URL. Para POST/PUT, selecione **Body → raw → JSON**.

**POST `/professores`**

```json
{
  "nome": "Maria Silva",
  "email": "maria@example.com",
  "area": "Desenvolvimento",
  "telefone": "86999999999"
}
```

**PUT `/professores/{id}`**, substituindo `{id}` pelo valor retornado no cadastro:

```json
{
  "nome": "Maria Silva Santos",
  "email": "mariasantos@example.com",
  "area": "Engenharia de Software",
  "telefone": "86988888888"
}
```

**DELETE `/professores/{id}`** usa esse mesmo id e não envia corpo. Depois, **GET `/professores/{id}`** deve retornar `404`, confirmando a remoção.

Exemplos com curl (no PowerShell, use `curl.exe`):

```bash
curl -i http://localhost:8080/professores
curl -i http://localhost:8080/professores/nome/joao
curl -i http://localhost:8080/professores/area/desenvolvimento
```

## Testes

### Postman

Importe [a coleção](postman/Professores.postman_collection.json) pelo menu **Import**. Com a aplicação no perfil `demo`, execute as requisições em ordem ou use o Collection Runner. A variável `baseUrl` vem configurada para `http://localhost:8080`. O POST salva automaticamente o `professorId` usado pelo PUT, DELETE e pela consulta de confirmação. Cada requisição verifica o status esperado.

### Testes locais sem banco

```bash
mvn test
```

Executa os testes unitários de Service e os testes web com MockMvc. Nessa etapa, o Repository/Service é substituído por mock; o resultado **não comprova a integração com PostgreSQL**.

### Integração real com PostgreSQL

Crie um banco exclusivo para testes:

```sql
CREATE DATABASE professores_test;
```

Ou, com o banco do Compose já ativo:

```bash
docker compose exec db createdb -U postgres professores_test
```

Então execute:

```bash
mvn verify
```

Por padrão, a integração acessa `jdbc:postgresql://localhost:5432/professores_test` com usuário/senha `postgres`. Para alterar, configure `TEST_DB_URL`, `TEST_DB_USERNAME` e `TEST_DB_PASSWORD` no mesmo formato das variáveis de execução. A integração só permite bancos com sufixo `_test` e limpa a tabela antes de cada teste. Use exclusivamente um banco descartável para testes.

Os testes de integração exercitam a aplicação HTTP em uma porta aleatória, o JPA e o PostgreSQL, verificando persistência, edição, exclusão, filtros, acentos, id inexistente e validação. Não utilizam H2.

O workflow [Testes da API](.github/workflows/testes.yml) configura PostgreSQL 16 no GitHub Actions, executa `mvn verify` e gera um artefato com os resultados e os prints. É preciso publicar os arquivos para que ele execute.

## Evidências de execução

Execução realizada em **28/09/2026**, contra a aplicação Spring Boot conectada ao **PostgreSQL 18.3 (PGlite 0.5.8 em WebAssembly)**, com persistência em disco. O Hibernate e o driver JDBC PostgreSQL foram usados sem mocks nessa etapa. O pool ficou limitado a uma conexão, conforme o modo de execução do PGlite.

As imagens são capturas no Chromium de relatórios produzidos por chamadas HTTP reais. Não são capturas da interface do Postman. Cada registro original `.json` e relatório `.html` acompanha o PNG em [docs/evidencias](docs/evidencias/).

### Caso 1 — Listar professores

GET `/professores` → **200 OK**.

![Listagem de professores](docs/evidencias/01-listar.png)

### Caso 2 — Filtrar por nome

GET `/professores/nome/joao` → **200 OK**. A busca sem acento encontra João da Silva e João Pedro.

![Filtro por nome](docs/evidencias/02-nome.png)

### Caso 3 — Filtrar por área

GET `/professores/area/desenvolvimento` → **200 OK**.

![Filtro por área](docs/evidencias/03-area.png)

### Caso 4 — Cadastrar professor

POST `/professores` → **201 Created**, com id gerado pelo banco.

![Cadastro de professor](docs/evidencias/04-cadastrar.png)

### Caso 5 — Editar professor

PUT `/professores/{id}` → **200 OK**. O script consulta o registro depois da alteração para conferir os valores persistidos.

![Edição de professor](docs/evidencias/05-editar.png)

### Caso 6 — Excluir professor

DELETE `/professores/{id}` → **204 No Content**. GET do mesmo id → **404 Not Found**, confirmando a exclusão.

![Exclusão e confirmação](docs/evidencias/06-excluir.png)

### Banco e resultado dos testes

- [Consultas SQL após reabrir o banco persistido](docs/evidencias/07-banco.json): versão, cinco colunas, chave primária, extensão `unaccent` e quatro registros iniciais mantidos em disco.
- [Saída real do Maven](docs/evidencias/maven-verificacao.txt): 16 testes unitários/web e 9 de integração, todos aprovados.
- [Detalhes e limites da validação](docs/VALIDACAO.md).

### Reproduzir as capturas

Com a API em execução e Python 3.10 ou superior:

```bash
python -m pip install -r scripts/requirements.txt
python -m playwright install chromium
python scripts/testar_e_gerar_evidencias.py
```

Em Linux sem dependências gráficas, use `python -m playwright install --with-deps chromium`. O script cria professores fictícios, valida os status e os dados retornados, captura os relatórios e remove somente os registros que criou. Use uma base local de demonstração. Não utiliza respostas simuladas; interrompe a execução em caso de divergência.

O workflow GitHub Actions também gera essas evidências em um artefato chamado `evidencias-e-testes`; a execução desse workflow fica pendente até publicar o repositório.

### Alternativa de desenvolvimento usada nesta verificação

Para reproduzir o PostgreSQL WebAssembly, instale Node.js 22 ou superior e execute, em outro terminal:

```bash
cd dev-postgres
npm ci
npm start
```

O script cria os bancos `professores_db` na porta 5432 e `professores_test` na porta 5433, com armazenamento em `dev-postgres/data/`. Deixe esse terminal aberto. Não execute o Docker e essa alternativa nas mesmas portas ao mesmo tempo.

Na raiz do projeto, configure o pool para uma conexão e execute a API:

**PowerShell:**

```powershell
$env:DB_URL="jdbc:postgresql://127.0.0.1:5432/professores_db?sslmode=disable"
$env:SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE="1"
mvn spring-boot:run "-Dspring-boot.run.profiles=demo"
```

**Linux/macOS:**

```bash
export DB_URL='jdbc:postgresql://127.0.0.1:5432/professores_db?sslmode=disable'
export SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=1
mvn spring-boot:run -Dspring-boot.run.profiles=demo
```

Para os testes, mantendo o banco iniciado, configure `TEST_DB_URL` para `jdbc:postgresql://127.0.0.1:5433/professores_test?sslmode=disable` e mantenha `SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=1`; então execute `mvn verify`.

O PGlite utiliza o motor PostgreSQL compilado para WebAssembly. Seu servidor TCP e modelo de conexão diferem da instalação nativa; estes testes não comprovam o funcionamento do Docker nem equivalência para todos os cenários de concorrência. A alternativa é local e de desenvolvimento. A aplicação principal continua configurada para PostgreSQL convencional e não depende de Node.js.

## Publicar no GitHub e entregar

1. Preencha o nome completo e a disciplina no início deste README.
2. Confira os testes e os seis prints já incluídos. Caso execute novamente no seu computador, atualize as evidências.
3. No GitHub, crie um repositório chamado `api-professores`, marque **Public** e não inicialize com README, licença ou `.gitignore`.
4. Na pasta do projeto:

```bash
git init
git add .
git commit -m "Implementa API REST de professores"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/api-professores.git
git push -u origin main
```

Substitua `SEU_USUARIO` pelo login da conta escolhida. Se preferir, publique a pasta pelo GitHub Desktop. O `.gitignore` exclui `target/`, configurações de IDE, logs e `.env`; envie `src/`, `pom.xml`, README, scripts, documentação e prints.

5. Confira a execução em **Actions** e abra o repositório em uma janela anônima para confirmar que está público.
6. Na Plataforma A, envie o link do repositório público, no formato `https://github.com/SEU_USUARIO/api-professores`.

### Checklist final

- [ ] Nome completo e disciplina preenchidos.
- [x] Código do CRUD e filtros implementado.
- [x] Tabela, registros iniciais e configuração PostgreSQL incluídos.
- [x] README com execução e endpoints.
- [x] Coleção Postman e testes automatizados incluídos.
- [x] `mvn verify` aprovado com PostgreSQL 18.3 em PGlite (25 testes).
- [x] Seis prints reais inseridos no README.
- [ ] Repositório público publicado e acessível.
- [ ] Link enviado à Plataforma A.

## Referências

- [Requisitos do Spring Boot 3.5](https://docs.spring.io/spring-boot/3.5/system-requirements.html)
- [Consultas derivadas do Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/3.5/jpa/query-methods.html)
- [Atributos calculados com Hibernate Formula](https://docs.hibernate.org/orm/6.6/javadocs/org/hibernate/annotations/Formula.html)
- [Extensão unaccent do PostgreSQL](https://www.postgresql.org/docs/16/unaccent.html)

- [PostgreSQL em WebAssembly: PGlite](https://pglite.dev/docs/about)
- [Servidor TCP do PGlite](https://pglite.dev/docs/pglite-socket)
