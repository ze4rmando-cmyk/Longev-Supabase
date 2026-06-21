# Longev — Backend

API multi-tenant para gestão de clínicas geriátricas (e profissionais individuais).

## Stack

- **NestJS** (TypeScript) — framework do backend
- **PostgreSQL** + **Prisma** — banco de dados e ORM
- **JWT** (access + refresh token rotativo) — autenticação
- **Argon2** — hash de senhas
- **AES-256-GCM** — criptografia de campos clínicos sensíveis

## O que já está pronto nesta fase (Fundação)

- Modelo de dados multi-tenant (`Tenant`, `User`, `RefreshToken`, `AuditLog`, `Patient`)
- Autenticação completa: cadastro de clínica (`/auth/register`), login, refresh token rotativo, logout
- Controle de acesso por perfil (RBAC): `SUPER_ADMIN`, `OWNER`, `DOCTOR`, `NURSE`, `RECEPTIONIST`, `FINANCE`
- Isolamento multi-tenant reforçado em 3 camadas: `TenantGuard` (HTTP) + filtro explícito por `tenantId` em todo service + (preparado para Row Level Security do Postgres no futuro)
- Auditoria automática (`@Audit(...)`) — registra quem acessou o quê, quando e de onde
- Criptografia de campos sensíveis (`EncryptionService`) — aplicada no módulo `Patient` como modelo de referência
- Soft delete em entidades de negócio (nunca exclusão física imediata)
- Rate limiting, Helmet, CORS restrito, validação de DTOs, filtro global de exceções
- Health check (`/api/v1/health`) para monitoramento em produção
- Documentação automática via Swagger (`/docs`, apenas fora de produção)

## O que falta (próximas fases — módulos de negócio)

Agendamentos, Prontuário Eletrônico, Sinais Vitais, Avaliações/Testes Clínicos,
Plano de Cuidado, Financeiro, Relatórios, Chat de Equipe, Portal do Paciente,
Integração com IA. Cada um desses módulos deve seguir exatamente o padrão
estabelecido pelo módulo `Patient` (tenant scoping + criptografia onde houver
dado sensível + auditoria nas rotas relevantes).

## Como rodar localmente

### 1. Pré-requisitos
- Node.js 20+
- Docker (para o PostgreSQL local) — ou um Postgres já instalado

### 2. Subir o banco de dados
```bash
docker compose up -d
```

### 3. Configurar variáveis de ambiente
```bash
cp .env.example .env
```
Gere segredos fortes para produção:
```bash
openssl rand -base64 64   # JWT_ACCESS_SECRET / JWT_REFRESH_SECRET
openssl rand -hex 32      # FIELD_ENCRYPTION_KEY
```

### 4. Instalar dependências
```bash
npm install
```

### 5. Rodar as migrações e gerar o Prisma Client
```bash
npx prisma migrate dev --name init
```

### 6. (Opcional) Popular dados iniciais
Cria um Super Admin e uma clínica de demonstração:
```bash
npm run prisma:seed
```

### 7. Subir a API
```bash
npm run start:dev
```

A API sobe em `http://localhost:3000/api/v1`.
Documentação Swagger em `http://localhost:3000/docs`.

## Testando rapidamente

```bash
# Cadastrar uma clínica + usuário owner
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "tenantName": "Clínica Exemplo",
    "document": "12345678000190",
    "tenantType": "CLINIC",
    "ownerName": "Dr. Exemplo",
    "ownerEmail": "owner@exemplo.com",
    "ownerPassword": "SenhaForte123!"
  }'

# Login
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "owner@exemplo.com", "password": "SenhaForte123!"}'

# Usar o accessToken retornado para criar um paciente
curl -X POST http://localhost:3000/api/v1/patients \
  -H "Authorization: Bearer SEU_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "fullName": "João da Silva",
    "birthDate": "1950-03-12",
    "sex": "MALE",
    "healthConditions": "Hipertensão, Diabetes tipo 2",
    "allergies": "Dipirona"
  }'
```

## Segurança e LGPD — decisões importantes

- **Senhas**: nunca armazenadas em texto puro, hash com Argon2.
- **Refresh tokens**: armazenados como hash SHA-256, nunca em texto puro; rotação a cada uso (token usado é revogado imediatamente).
- **Dados clínicos sensíveis**: campos como condições de saúde, alergias e medicações são criptografados (AES-256-GCM) antes de ir ao banco. A chave fica fora do banco, em variável de ambiente — migre para um KMS gerenciado (AWS KMS/GCP KMS) em produção.
- **Auditoria**: toda visualização/criação/edição/exclusão de dados de paciente é registrada com usuário, IP, user-agent e timestamp, e nunca é apagada.
- **Isolamento multi-tenant**: nenhuma query de negócio deve existir sem filtro explícito por `tenantId`. Isso é uma regra de revisão de código, não apenas uma proteção de guard.
- **Exclusão**: registros de negócio usam soft delete (campo `deletedAt`), preservando trilha de auditoria conforme exigido pela LGPD para dados de saúde.
