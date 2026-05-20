<div align="center">

# Acauã

### Sistema de Monitoramento e Gestão de Ocorrências de Perturbação do Sossego

[![Java](https://img.shields.io/badge/Java-21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=for-the-badge&logo=spring-boot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-PostGIS-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://postgis.net/)
[![JWT](https://img.shields.io/badge/Auth-JWT%20%2F%20RBAC-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white)](https://jwt.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

*Tornando o trabalho das forças de segurança pública do primeiro aviso à apreensão com respaldo legal e rastreabilidade total.*

</div>

---

## 📋 Sumário

- [O Problema](#-o-problema)
- [A Solução — Acauã](#-a-solução--acauã)
- [Funcionalidades Principais](#-funcionalidades-principais)
- [Arquitetura Técnica](#-arquitetura-técnica)
- [Diagrama de Entidade-Relacionamento](#-diagrama-de-entidade-relacionamento)
- [Stack Tecnológica](#-stack-tecnológica)
- [Instalação e Execução Local](#-instalação-e-execução-local)
- [Referência de Endpoints](#-referência-de-endpoints)
- [Decisões de Engenharia](#-decisões-de-engenharia)
- [Roadmap](#-roadmap)
- [Licença](#-licença)

---

## 🔴 O Problema

O atendimento de ocorrências de perturbação do sossego (poluição sonora, festas, shows irregulares) pelas polícias militares e guardas municipais sofre de uma falha estrutural crítica: **cada viatura atende cada ocorrência como se fosse a primeira**.

Esse modelo é inerentemente **stateless**:

```
Viatura A → Atende ocorrência → Emite advertência → Encerra chamado
                                                         ↓
                                              Estado perdido para sempre

Viatura B → Atende reincidência no MESMO local → Emite outra advertência
            (sem saber que já houve notificação anterior)
```

As consequências práticas são sérias:

| Problema | Impacto |
|---|---|
| Ausência de histórico por infrator/endereço | Reincidentes recebem sempre a mesma advertência verbal, sem progressão legal |
| Nenhum rastreamento espacial | Impossível identificar "pontos quentes" de perturbação na cidade |
| Processo manual e não auditável | Abertura para inconsistências, favorecimentos e fraudes |
| Multas e apreensões dependem da memória da guarnição | Perda de receita para o município e impunidade para o infrator |

---

## ✅ A Solução — Acauã

O **Acauã** é uma API REST que resolve a falha de persistência de estado do processo de fiscalização, transformando cada atendimento em um evento rastreável, auditável e juridicamente válido.

O sistema vincula toda ocorrência a **três âncoras de responsabilidade**:

1. **A pessoa física** → identificada pelo CPF do infrator
2. **O endereço** → com coordenadas geoespaciais no momento do flagrante
3. **O agente de segurança** → registrado via autenticação JWT, garantindo cadeia de custódia

Na prática, quando uma viatura é despachada para um endereço, o agente consulta o CPF do responsável e, em milissegundos, o Acauã retorna o histórico completo: quantas vezes aquele cidadão foi notificado, em quais endereços, por quais agentes e em quais datas — habilitando, automaticamente, a aplicação da penalidade correta conforme a legislação municipal.

> **O nome**: O Acauã (Herpetotheres cachinnans) é uma ave de rapina imponente e típica do Brasil, famosa por sua máscara preta e por ser uma exímia caçadora de serpentes. No folclore do Nordeste, seu canto forte e estridente carrega um peso místico: ele age como um presságio, um alerta inescapável que ecoa pelo sertão para avisar que algo está para acontecer. A escolha desse nome para o software nasce exatamente dessa força simbólica de aviso e precisão. Assim como o pássaro emite seu chamado para alertar o sertanejo, o sistema Acauã atua como um rastreador implacável e um alerta inteligente para a segurança pública. Ele "canta" para a corporação policial assim que identifica um infrator reincidente, servindo como um presságio de que a tolerância acabou e a autuação é iminente. É a tecnologia agindo com a mesma precisão de uma ave de rapina para devolver o sossego à população.

---

## ⚙️ Funcionalidades Principais

### 1. 🪪 Registro por CPF e Endereço
O núcleo do sistema vincula cada ocorrência à pessoa física responsável pelo local (proprietário ou responsável legal). Isso garante validade legal ao processo administrativo e possibilita o acúmulo de notificações por indivíduo, não apenas por endereço — o que é decisivo quando o mesmo infrator muda de local.

### 2. 🔁 Motor de Reincidência Automática
Ao consultar um CPF, o sistema executa uma query parametrizada que varre o histórico de ocorrências e retorna imediatamente:
- Total de notificações anteriores
- Datas e locais de cada ocorrência
- Status atual (ativo, invalidado, convertido em multa)
- **Nível de penalidade recomendado** com base nas regras configuradas

Toda essa lógica é centralizada em uma camada de serviço desacoplada (`ReincidenciaService`), facilitando a customização das regras por município.

### 3. 🗺️ Geolocalização com PostGIS
A API recebe a `latitude` e `longitude` da viatura no momento do registro — dados captados pelo GPS do dispositivo móvel do agente. Essas coordenadas são persistidas como tipo `GEOMETRY(POINT, 4674)` (SIRGAS 2000, o datum oficial brasileiro), habilitando:
- **Busca por raio**: "Liste todas as ocorrências num raio de 500m deste ponto"
- **Mapeamento de zonas críticas** para relatórios do Comando
- **Prova georreferenciada** para fins jurídicos

### 4. 🔐 Controle de Acesso por Papéis (RBAC)
O sistema implementa controle de acesso baseado em papéis com os seguintes perfis:

| Papel | Permissões |
|---|---|
| `ROLE_AGENTE` | Registrar ocorrências, consultar CPF, visualizar histórico de ocorrências |
| `ROLE_SUPERVISOR` | Tudo do Agente + Invalidar ocorrências com justificativa |
| `ROLE_COMANDO` | Tudo do Supervisor + Emitir relatórios gerenciais, converter ocorrências em multas, gerenciar usuários |
| `ROLE_ADMIN` | Acesso total ao sistema, incluindo configuração de regras de reincidência |

### 5. 🔒 Logs de Auditoria Imutáveis
Ocorrências registradas **não podem ser deletadas**. O sistema segue o princípio de **append-only** para registros de ocorrência. Toda alteração de estado (ex: invalidação) gera um novo registro de auditoria com:
- `timestamp` da operação
- `id` do usuário executor
- Justificativa obrigatória
- Estado anterior e novo estado

Isso cria uma trilha de auditoria completa e à prova de adulteração, protegendo tanto a instituição quanto o cidadão.

---

## 🏛️ Arquitetura Técnica

O Acauã segue uma arquitetura em camadas clássica do ecossistema Spring, com separação clara de responsabilidades:

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENTE (App Mobile / Web)           │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS + JWT Bearer Token
┌──────────────────────────▼──────────────────────────────┐
│                  CAMADA DE SEGURANÇA                    │
│          Spring Security · JWT Filter · RBAC            │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│               CAMADA DE APRESENTAÇÃO (API)              │
│              Controllers REST (@RestController)         │
│         OcorrenciaController · UsuarioController        │
│         RelatorioController · AuthController            │
└──────────────────────────┬──────────────────────────────┘
                           │ DTOs (Request/Response)
┌──────────────────────────▼──────────────────────────────┐
│                 CAMADA DE NEGÓCIO (Services)            │
│       OcorrenciaService · ReincidenciaService           │
│       MultaService · AuditoriaService                   │
└──────────────────┬───────────────────────┬──────────────┘
                   │ JPA/Hibernate Spatial │
┌──────────────────▼──────────┐  ┌─────────▼──────────────┐
│      CAMADA DE DADOS        │  │   CAMADA DE AUDITORIA  │
│  Repositories (Spring Data) │  │   Logs Imutáveis (DB)  │
└──────────────────┬──────────┘  └─────────┬──────────────┘
                   │                       │
┌──────────────────▼───────────────────────▼────────────────┐
│              PostgreSQL 15 + PostGIS 3.x                  │
│     Dados Relacionais · Geometria Espacial (SIRGAS 2000)  │
└───────────────────────────────────────────────────────────┘
```

### Fluxo de uma Ocorrência

```
Agente no campo
     │
     ├─ 1. Autentica (POST /auth/login) → recebe JWT
     │
     ├─ 2. Consulta CPF do infrator (GET /ocorrencias/cpf/{cpf})
     │       └─ Sistema retorna histórico + alerta de reincidência
     │
     ├─ 3. Registra a ocorrência (POST /ocorrencias)
     │       └─ Payload inclui: CPF, endereço, coordenadas GPS, tipo de perturbação
     │
     └─ 4. Sistema persiste, calcula penalidade e retorna protocolo
```

---

## 🗄️ Diagrama de Entidade-Relacionamento

```
┌──────────────────┐       ┌───────────────────────┐       ┌───────────────────┐
│    tb_usuario    │       │    tb_ocorrencia      │       │    tb_infrator    │
├──────────────────┤       ├───────────────────────┤       ├───────────────────┤
│ PK id            │ 1   N │ PK id                 │ N   1 │ PK id             │
│    nome          ├───────┤ FK id_usuario_agente  ├───────┤    cpf (UNIQUE)   │
│    matricula     │       │ FK id_infrator        │       │    nome_completo  │
│    email         │       │    tipo_perturbacao   │       │    data_nascimento│
│    senha_hash    │       │    descricao          │       │    telefone       │
│    role          │       │    nivel_som_db       │       │    email          │
│    ativo         │       │    localizacao (GEO)  │       └───────────────────┘
└──────────────────┘       │    endereco_logradouro│
                           │    endereco_numero    │       ┌───────────────────┐
┌──────────────────┐       │    endereco_bairro    │       │   tb_auditoria    │
│  tb_penalidade   │       │    endereco_cidade    │       ├───────────────────┤
├──────────────────┤       │    status             │ 1   N │ PK id             │
│ PK id            │ 1   N │    numero_notificacao ├───────┤ FK id_ocorrencia  │
│ FK id_ocorrencia ├───────┤    criado_em          │       │ FK id_usuario_op  │
│    tipo_pena     │       │    atualizado_em      │       │    acao_realizada │
│    valor_multa   │       └───────────────────────┘       │    estado_anterior│
│    data_emissao  │                                       │    estado_novo    │
│    data_pagamento│                                       │    justificativa  │
│    status_multa  │                                       │    criado_em      │
└──────────────────┘                                       └───────────────────┘
```

**Descrição das entidades:**

- **`tb_usuario`** — Agentes, supervisores e membros do Comando cadastrados no sistema. O campo `role` mapeia diretamente para os papéis do Spring Security.
- **`tb_infrator`** — Cadastro da pessoa física. O CPF é o identificador único e imutável que garante a rastreabilidade mesmo que o infrator mude de endereço.
- **`tb_ocorrencia`** — Entidade central. O campo `localizacao` é do tipo PostGIS `GEOMETRY(POINT, 4674)`, armazenando latitude e longitude com projeção SIRGAS 2000. O campo `status` segue uma máquina de estados: `ATIVA → INVALIDADA | CONVERTIDA_EM_MULTA`.
- **`tb_penalidade`** — Registro financeiro/administrativo gerado a partir de uma ocorrência. Separado da ocorrência para permitir que uma notificação exista sem necessariamente gerar multa.
- **`tb_auditoria`** — Log imutável de toda operação de escrita no sistema. Nenhum registro é deletado — apenas novos estados são inseridos aqui.

---

## 🛠️ Stack Tecnológica

| Camada | Tecnologia | Justificativa |
|---|---|---|
| **Linguagem** | Java 21 | LTS com Virtual Threads (Project Loom) para alta concorrência em operações de I/O |
| **Framework** | Spring Boot 3.x | Ecossistema maduro, autoconfiguration e integração nativa com todo o stack |
| **Segurança** | Spring Security + JJWT | Implementação de RBAC com filtro JWT stateless, sem sessão no servidor |
| **Persistência** | Spring Data JPA + Hibernate 6 | ORM com suporte nativo a Hibernate Spatial via JTS |
| **Banco de Dados** | PostgreSQL 15 | Banco relacional robusto com suporte à extensão PostGIS |
| **Extensão Espacial** | PostGIS 3.x | Indexação espacial com GiST, funções `ST_Distance`, `ST_DWithin`, `ST_AsGeoJSON` |
| **Geometria Java** | JTS Topology Suite | Biblioteca padrão para manipulação de tipos geométricos no lado Java |
| **Migração de Schema** | Flyway | Versionamento e migração de banco de dados de forma declarativa e auditável |
| **Build** | Maven | Gerenciamento de dependências e ciclo de build padrão no ecossistema Spring |
| **Documentação** | SpringDoc OpenAPI 3 | Geração automática do Swagger UI a partir das anotações dos Controllers |

---

## 🚀 Instalação e Execução Local

### Pré-requisitos

- **JDK 21** ou superior
- **Maven 3.9+**
- **Docker** e **Docker Compose** (para subir o PostgreSQL + PostGIS)

### 1. Clonar o Repositório

```bash
git clone https://github.com/seu-usuario/acaua.git
cd acaua
```

### 2. Subir o Banco de Dados com Docker

O projeto inclui um `docker-compose.yml` que provisiona o PostgreSQL com a extensão PostGIS já habilitada:

```bash
docker compose up -d
```

Isso irá subir:
- **PostgreSQL 15** na porta `5432`
- **Banco**: `acaua_db`
- **Usuário/Senha**: definidos no arquivo `.env` (ver próximo passo)

### 3. Configurar as Variáveis de Ambiente

Copie o arquivo de exemplo e preencha com suas configurações:

```bash
cp .env.example .env
```

```dotenv
# .env
DB_URL=jdbc:postgresql://localhost:5432/acaua_db
DB_USERNAME=acaua_user
DB_PASSWORD=sua_senha_aqui

JWT_SECRET=sua_chave_secreta_jwt_com_pelo_menos_256_bits
JWT_EXPIRATION_MS=86400000

ADMIN_EMAIL=admin@acaua.gov.br
ADMIN_PASSWORD=senha_inicial_admin
```

> ⚠️ **Nunca comite o arquivo `.env`** em repositórios públicos. Ele já está listado no `.gitignore`.

### 4. Executar a Aplicação

```bash
./mvnw spring-boot:run
```

O Flyway irá aplicar automaticamente todas as migrations, criar as tabelas e habilitar a extensão `postgis` no banco.

A API estará disponível em: **`http://localhost:8080`**

A documentação interativa (Swagger UI) estará em: **`http://localhost:8080/swagger-ui.html`**

---

## 📡 Referência de Endpoints

### Autenticação

```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "email": "agente@pm.gov.br",
  "senha": "senha123"
}
```

*Resposta: Bearer token JWT para uso nos demais endpoints.*

---

### Ocorrências

```http
# Registrar nova ocorrência
POST /api/v1/ocorrencias
Authorization: Bearer {token}
Roles: AGENTE, SUPERVISOR, COMANDO

{
  "cpfInfrator": "123.456.789-00",
  "nomeInfrator": "João da Silva",
  "tipoPerturbacao": "FESTA_RESIDENCIAL",
  "descricao": "Som alto com subwoofer externo. Reclamações de vizinhos.",
  "nivelSomDb": 85.5,
  "latitude": -7.2195,
  "longitude": -35.9083,
  "enderecoLogradouro": "Rua das Palmeiras",
  "enderecoNumero": "123",
  "enderecoBairro": "Centro",
  "enderecoCidade": "Campina Grande"
}
```

```http
# Consultar histórico por CPF (com alerta de reincidência)
GET /api/v1/ocorrencias/cpf/{cpf}
Authorization: Bearer {token}
Roles: AGENTE, SUPERVISOR, COMANDO
```

```http
# Buscar ocorrências por raio geográfico (em metros)
GET /api/v1/ocorrencias/proximidade?lat=-7.2195&lng=-35.9083&raioMetros=500
Authorization: Bearer {token}
Roles: SUPERVISOR, COMANDO
```

```http
# Listar todas as ocorrências (com filtros opcionais)
GET /api/v1/ocorrencias?status=ATIVA&cidade=Campina+Grande&page=0&size=20
Authorization: Bearer {token}
Roles: SUPERVISOR, COMANDO
```

```http
# Invalidar uma ocorrência (com justificativa obrigatória)
PATCH /api/v1/ocorrencias/{id}/invalidar
Authorization: Bearer {token}
Roles: SUPERVISOR, COMANDO

{
  "justificativa": "Ocorrência registrada com CPF incorreto. Corrigida no protocolo 2024/0042."
}
```

---

### Penalidades e Multas

```http
# Converter ocorrência em multa formal
POST /api/v1/penalidades
Authorization: Bearer {token}
Roles: COMANDO

{
  "idOcorrencia": "a3f8d2c1-...",
  "tipoPenalidade": "MULTA_ADMINISTRATIVA",
  "valorMulta": 850.00
}
```

---

### Relatórios (Comando)

```http
# Relatório de reincidentes por período
GET /api/v1/relatorios/reincidentes?dataInicio=2024-01-01&dataFim=2024-12-31
Authorization: Bearer {token}
Roles: COMANDO

# Mapa de calor (retorna GeoJSON para plotagem)
GET /api/v1/relatorios/mapa-calor?cidade=Campina+Grande
Authorization: Bearer {token}
Roles: COMANDO
```

---

## 💡 Decisões de Engenharia

### Por que PostGIS em vez de calcular distâncias na aplicação?

Calcular distâncias geográficas via fórmula de Haversine na camada de serviço exigiria trazer **todos os registros do banco para a memória** e filtrar em Java. O PostGIS resolve isso com índices espaciais do tipo **GiST (Generalized Search Tree)**, executando queries como `ST_DWithin(localizacao, ST_MakePoint(:lng, :lat), :raio)` diretamente no banco, com complexidade próxima a O(log n). Para uma cidade com dezenas de milhares de ocorrências, a diferença é de segundos vs. milissegundos.

### Por que auditoria append-only e não soft delete?

Um soft delete (`deleted = true`) ainda permite que o registro seja fisicamente removido por alguém com acesso direto ao banco. A abordagem append-only com tabela de auditoria separada cria uma **cadeia de custódia imutável**: cada mudança de estado é um novo registro com `id` próprio, `timestamp` e operador responsável. Isso segue o princípio de **Event Sourcing** parcial e é a abordagem exigida em sistemas com validade jurídica.

### Por que CPF como âncora em vez de apenas endereço?

Vincular a reincidência apenas ao endereço criaria uma brecha óbvia: o infrator aluga outro imóvel na mesma cidade e começa do zero. Ao vincular ao CPF, o histórico do **indivíduo** é preservado independentemente do local, o que está alinhado com a legislação de contravenções penais brasileiras.

---

## 🗺️ Roadmap

- [ ] **App Mobile** (Android/iOS) para os agentes de campo com integração GPS nativa
- [ ] **Módulo de Notificações** via SMS/WhatsApp ao infrator após registro da ocorrência
- [ ] **Painel do Comando** com mapa de calor interativo (integração Leaflet.js / GeoJSON)
- [ ] **Integração com e-SIC / sistemas municipais** para emissão de multas oficiais
- [ ] **Exportação de relatórios** em PDF e XLSX
- [ ] **Multi-tenancy** — suporte a múltiplos municípios em uma única instância

---

## 📄 Licença

Distribuído sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.

---

<div align="center">

Desenvolvido com ☕ e Java para fortalecer a segurança pública brasileira.

*"O acauã não perdoa quem perturba o sossego da mata."*

</div>
