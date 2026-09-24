# BarberSys Pro — Back-end Completo

Transformar o front-end estático (HTML/CSS/JS no [TES7.HTML](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/TES7.HTML)) em um sistema funcional com back-end Node.js + Express, banco PostgreSQL via Prisma ORM, autenticação JWT, lógica de agendamento, integração WhatsApp e QR Code.

> [!CAUTION]
> **Regra Crítica**: Nenhum código de gateway de pagamento (Stripe, Mercado Pago, Asaas) será implementado. A seção "Pagamentos" na UI continuará como registro local e links informativos.

## User Review Required

> [!IMPORTANT]
> **Decisão de Stack**: O plano usa **Node.js + Express** (API REST pura) + **PostgreSQL com Prisma ORM** + **JWT para autenticação**. Se prefere Supabase/Firebase como BaaS, por favor indique antes da execução.

> [!IMPORTANT]
> **WhatsApp**: O plano integra a **Evolution API** como provedor de WhatsApp. Se preferir usar Baileys (self-hosted, sem servidor Evolution), indique. A estrutura será preparada para trocar o provedor.

## Open Questions

1. **Domínio/URL de produção**: Para os links de agendamento e QR Code, qual será o domínio? (ex: `app.barbersyspro.com`). Por padrão usarei `process.env.APP_URL`.
2. **E-mail do Admin**: Deseja que o primeiro admin seja criado via seed no banco? (recomendado para facilitar primeiro acesso).

---

## Proposed Changes

### Estrutura do Projeto

O back-end será criado em `c:\Users\USUARIO-TDC\Desktop\tes7\Babil-ria\backend\`. O front-end original será mantido intacto, servido como assets estáticos pelo Express em produção.

```
backend/
├── prisma/
│   ├── schema.prisma          # Modelagem do banco
│   └── seed.js                # Dados iniciais (admin, serviços)
├── src/
│   ├── server.js              # Entry point Express
│   ├── config/
│   │   └── env.js             # Validação de variáveis de ambiente
│   ├── middleware/
│   │   ├── auth.js            # JWT verification middleware
│   │   └── roles.js           # Controle de permissões (admin/barber/client)
│   ├── routes/
│   │   ├── auth.routes.js     # Login, Cadastro, Refresh
│   │   ├── barbers.routes.js  # CRUD barbeiros
│   │   ├── services.routes.js # CRUD serviços
│   │   ├── appointments.routes.js  # CRUD agendamentos + disponibilidade
│   │   ├── clients.routes.js  # CRUD clientes
│   │   ├── qrcode.routes.js   # Geração de QR Code
│   │   └── whatsapp.routes.js # Webhook e status WhatsApp
│   ├── services/
│   │   ├── availability.service.js   # Cálculo de horários livres
│   │   ├── whatsapp.service.js       # Integração Evolution API
│   │   └── qrcode.service.js         # Geração de QR Code
│   ├── jobs/
│   │   ├── scheduler.js               # CRON setup (node-cron)
│   │   ├── feedback-reminder.job.js   # Agente de opinião (2h após conclusão)
│   │   └── appointment-reminder.job.js # Lembretes 24h e 2h antes
│   └── utils/
│       ├── password.js        # bcrypt hash/compare
│       └── jwt.js             # Geração e verificação de tokens
├── .env.example               # Template de variáveis de ambiente
├── Dockerfile                 # Container para deploy
├── docker-compose.yml         # PostgreSQL + App
├── package.json
└── README.md                  # Instruções de setup e deploy
```

---

### 1. Banco de Dados — Prisma Schema

#### [NEW] [schema.prisma](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/prisma/schema.prisma)

Entidades principais:

| Tabela | Campos Chave |
|--------|-------------|
| **User** | `id`, `name`, `phone`, `email`, `passwordHash`, `role` (ADMIN/BARBER/CLIENT) |
| **Barber** | `id`, `userId` (FK), `specialty`, `instagram`, `startTime`, `endTime`, `intervalMinutes`, `workDays` (int[]), `isActive` |
| **Service** | `id`, `name`, `price` (Decimal), `durationMinutes`, `isActive` |
| **Appointment** | `id`, `clientId` (FK→User), `barberId` (FK→Barber), `serviceId` (FK→Service), `date`, `startTime`, `endTime`, `status` (PENDING/CONFIRMED/CANCELLED/COMPLETED), `feedbackSent` |
| **Barbershop** | `id`, `name`, `address`, `phone`, `instagram`, `pixKey` |
| **BlockedSlot** | `id`, `barberId`, `date`, `startTime`, `endTime`, `reason` — para barbeiros bloquearem horários |

#### [NEW] [seed.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/prisma/seed.js)

Criará: 1 admin padrão, 3 barbeiros, 5 serviços (mesmo dados do front-end), barbearia padrão.

---

### 2. Autenticação e Autorização

#### [NEW] [auth.routes.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/routes/auth.routes.js)

| Rota | Método | Descrição |
|------|--------|-----------|
| `/api/auth/register` | POST | Cadastro (nome, telefone, senha, role) |
| `/api/auth/login` | POST | Login por telefone+senha, retorna JWT |
| `/api/auth/me` | GET | Dados do usuário logado (via token) |

#### [NEW] [auth.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/middleware/auth.js)

- Middleware `authenticate`: extrai e valida JWT do header `Authorization: Bearer <token>`
- Middleware `authorize(...roles)`: restringe acesso por role (ADMIN, BARBER, CLIENT)

**Regras de autorização**:
- **CLIENT**: só vê/gerencia seus próprios agendamentos
- **BARBER**: vê agenda completa do seu perfil, bloqueia horários, vê métricas
- **ADMIN**: acesso total (todos barbeiros, clientes, configurações)

---

### 3. Lógica de Agendamento (Core)

#### [NEW] [availability.service.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/services/availability.service.js)

**Algoritmo de disponibilidade**:
1. Recebe `barberId` + `date` + `serviceDurationMinutes`
2. Busca jornada do barbeiro (startTime, endTime, intervalMinutes, workDays)
3. Verifica se o dia da semana está nos `workDays`
4. Gera todos os slots possíveis baseado no intervalo
5. Subtrai agendamentos existentes (status ≠ CANCELLED) e slots bloqueados
6. Retorna array de horários livres `[{ time: "09:00", available: true }, ...]`

#### [NEW] [appointments.routes.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/routes/appointments.routes.js)

| Rota | Método | Auth | Descrição |
|------|--------|------|-----------|
| `/api/appointments` | GET | ADMIN/BARBER | Lista agendamentos (filtros: data, barbeiro, status) |
| `/api/appointments/my` | GET | CLIENT | Meus agendamentos |
| `/api/appointments` | POST | ALL | Criar agendamento (valida disponibilidade) |
| `/api/appointments/:id` | PUT | ADMIN/BARBER | Atualizar status |
| `/api/appointments/:id` | DELETE | ADMIN | Cancelar/deletar |
| `/api/appointments/availability/:barberId` | GET | PUBLIC | Horários disponíveis para uma data |
| `/api/appointments/block` | POST | BARBER/ADMIN | Bloquear horário |

---

### 4. Integração WhatsApp + Agente de Opinião

#### [NEW] [whatsapp.service.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/services/whatsapp.service.js)

Interface para a **Evolution API**:
- `sendMessage(phone, message)` — envia mensagem de texto
- `sendTemplateMessage(phone, template, params)` — para mensagens estruturadas
- Configuração via `EVOLUTION_API_URL` e `EVOLUTION_API_KEY`

#### [NEW] [scheduler.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/jobs/scheduler.js)

Usa `node-cron` para agendar os jobs:

#### [NEW] [feedback-reminder.job.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/jobs/feedback-reminder.job.js)

- **Executa**: a cada 30 minutos
- **Lógica**: busca agendamentos com `status = COMPLETED` e `completedAt <= NOW() - 2 hours` e `feedbackSent = false`
- **Ação**: dispara mensagem WhatsApp pedindo avaliação, marca `feedbackSent = true`

#### [NEW] [appointment-reminder.job.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/jobs/appointment-reminder.job.js)

- **Executa**: a cada 15 minutos
- **Lógica**: busca agendamentos confirmados com início em ~24h e ~2h
- **Ação**: envia lembrete via WhatsApp com dados do agendamento

---

### 5. QR Code Dinâmico

#### [NEW] [qrcode.service.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/services/qrcode.service.js)

Usa biblioteca `qrcode` (npm) para gerar QR Code server-side.

#### [NEW] [qrcode.routes.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/routes/qrcode.routes.js)

| Rota | Método | Descrição |
|------|--------|-----------|
| `/api/qrcode/barber/:barberId` | GET | Retorna imagem QR Code (base64 ou PNG) apontando para `APP_URL/agendar/:barberId` |
| `/api/qrcode/barbershop` | GET | Retorna QR Code para link geral da barbearia |

---

### 6. CRUDs Complementares

#### [NEW] [barbers.routes.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/routes/barbers.routes.js)

CRUD completo de barbeiros (ADMIN only para create/update/delete, PUBLIC para list).

#### [NEW] [services.routes.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/routes/services.routes.js)

CRUD completo de serviços (ADMIN only para create/update/delete, PUBLIC para list).

#### [NEW] [clients.routes.js](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/src/routes/clients.routes.js)

CRUD de clientes (ADMIN/BARBER para list, CLIENT para update próprio perfil).

---

### 7. Preparação para Deploy

#### [NEW] [.env.example](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/.env.example)

```env
DATABASE_URL=postgresql://user:password@localhost:5432/barbersys
JWT_SECRET=your-secret-key-here
JWT_EXPIRES_IN=7d
APP_URL=http://localhost:3000
PORT=3000
EVOLUTION_API_URL=http://localhost:8080
EVOLUTION_API_KEY=your-evolution-api-key
EVOLUTION_INSTANCE=barbersys
```

#### [NEW] [Dockerfile](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/Dockerfile)

Multi-stage build: Node 20 Alpine, `prisma generate`, `npm start`.

#### [NEW] [docker-compose.yml](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/docker-compose.yml)

Serviços: `app` (Node), `db` (PostgreSQL 16), volumes persistentes.

#### [NEW] [README.md](file:///c:/Users/USUARIO-TDC/Desktop/tes7/Babil-ria/backend/README.md)

Instruções completas de setup local, migração, seed e deploy (Render, Railway, AWS).

---

## Dependências (package.json)

| Pacote | Uso |
|--------|-----|
| `express` | Framework HTTP |
| `@prisma/client` + `prisma` | ORM PostgreSQL |
| `bcryptjs` | Hash de senhas |
| `jsonwebtoken` | Tokens JWT |
| `cors` | Cross-origin requests |
| `helmet` | Headers de segurança |
| `morgan` | Logging HTTP |
| `node-cron` | Jobs agendados (CRON) |
| `qrcode` | Geração de QR Code |
| `axios` | HTTP client (Evolution API) |
| `dotenv` | Variáveis de ambiente |
| `express-validator` | Validação de inputs |

---

## Verificação

### Automated Tests
```bash
# Verificar se compila e inicia sem erros
cd backend && npm install && npx prisma generate && npm start

# Testar rotas via curl/Postman
# POST /api/auth/register
# POST /api/auth/login
# GET /api/appointments/availability/:barberId?date=2026-09-12
# GET /api/qrcode/barber/:barberId
```

### Manual Verification
- Iniciar o servidor e verificar que todas as rotas respondem corretamente
- Verificar que o schema do Prisma gera as tabelas esperadas
- Testar o fluxo: cadastro → login → criar agendamento → verificar disponibilidade
- Verificar geração de QR Code retornando PNG/base64 válido
