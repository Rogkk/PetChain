# PetChain

Plataforma de **coordenação e rastreabilidade de resgates de animais**.
Acompanha cada ocorrência do registro à conclusão, com trilha auditável de
eventos, responsáveis, prazos e confirmações, e um dashboard operacional.

Este repositório é um MVP de hackathon, monorepo TypeScript, com três pacotes:

| Pacote | Papel |
|---|---|
| `packages/shared` | Modelo de domínio, máquina de estados do fluxo, KPIs e validação. Contrato único entre backend e frontend. |
| `packages/backend` | API REST (Express), regras de negócio, persistência local (swappable para DynamoDB), seed sintético e rotina de atraso. |
| `packages/frontend` | App React + Vite: dashboard, fila, mapa, registro e acompanhamento do fluxo. |

> Todos os dados de demonstração são **fictícios** (`isSynthetic=true`) e **não**
> representam estatísticas reais de nenhuma cidade.

---

## Pré-requisitos

- **Node.js >= 18.17** e npm (usa workspaces do npm).
  > Observação: a máquina onde este projeto foi gerado **não** tinha Node.js
  > instalado. Instale o Node (https://nodejs.org) antes de executar. Sem o
  > Node, o código não pode ser compilado nem testado localmente.

## Instalação

```bash
npm install
```

## Executar localmente

Abra dois terminais (o `npm run dev` roda servidores de longa duração):

```bash
# Terminal 1 — API (porta 4000), carrega o seed sintético ao iniciar
npm run dev:backend

# Terminal 2 — Frontend (porta 5173)
npm run dev:frontend
```

Acesse `http://localhost:5173`. O frontend fala com a API em
`http://localhost:4000` (configurável via `VITE_API_BASE_URL`).

Copie os exemplos de ambiente (sem segredos):

```bash
cp packages/backend/.env.example packages/backend/.env
cp packages/frontend/.env.example packages/frontend/.env
```

### Seed / reseed

O backend carrega o dataset sintético ao iniciar (`SEED_ON_START=true`).
Para rodar o seed de forma isolada:

```bash
npm run seed
```

### Testes

```bash
npm test                 # shared + backend
npm run test --workspace @petchain/frontend
```

Cobrem: fórmulas de cada KPI, transições permitidas/proibidas, bloqueio/atraso,
aceite/recusa/reatribuição, confirmação de recebimento, concorrência e
idempotência, criação de evento junto à mudança de estado, privacidade de
localização, validação de formulário, estados de carregamento/erro/vazio e
timeline em ordem cronológica.

---

## Modelo de dados e KPIs

Veja `docs/DATA_DICTIONARY.md` para o dicionário de dados completo, o glossário
de KPIs (fórmula, campos, janela temporal, limitações) e as regras de
privacidade/retenção.

## Mapa (Google Maps)

A página **Mapa** usa o Google Maps JavaScript API quando há chave configurada.
Sem chave, ou se o Google recusar a chave, mostra uma dispersão simplificada; a
aba **Lista** está sempre disponível como alternativa acessível.

1. No [Google Cloud Console](https://console.cloud.google.com/), crie/selecione um
   projeto, **ative o faturamento** e habilite a **Maps JavaScript API**.
2. Crie uma chave de API e **restrinja-a**: referenciadores HTTP
   (ex.: `http://localhost:5173/*` e o domínio de produção) e apenas a Maps
   JavaScript API. A chave é de navegador e fica visível no cliente; a restrição
   é o que impede o uso indevido.
3. Em `packages/frontend/.env`, preencha `VITE_GOOGLE_MAPS_API_KEY`.
   `VITE_GOOGLE_MAPS_MAP_ID` pode ficar como `DEMO_MAP_ID` em desenvolvimento;
   em produção, crie um Map ID próprio (exigido pelos marcadores avançados).
4. Reinicie o frontend (variáveis `VITE_*` são lidas na inicialização).

Os pontos vêm de `GET /map/occurrences`, que já reduz a precisão das
coordenadas para papéis sem privilégio. Marcadores próximos são agrupados
(clique no grupo para aproximar). Ocorrências sem coordenadas aparecem só na lista.

## Endpoints

Base URL: `http://localhost:4000`. Identidade do ator (MVP) via cabeçalhos
`x-actor-id` e `x-actor-role` (`ADMIN|COORDINATOR|RESPONDER|ORGANIZATION|REPORTER`).

| Método | Rota | Descrição |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/meta/kpis` | Catálogo de definições de KPI |
| POST | `/occurrences` | Criar ocorrência (grava 1º evento) |
| GET | `/occurrences` | Listar com filtros, paginação e ordenação |
| GET | `/occurrences/{id}` | Detalhe + atribuições + ações disponíveis |
| GET | `/occurrences/{id}/events` | Timeline paginada |
| POST | `/occurrences/{id}/transitions` | Transição validada |
| POST | `/occurrences/{id}/assignments` | Atribuir/reatribuir responsável |
| POST | `/occurrences/{id}/confirmations` | Confirmação (mapeia para transição) |
| POST | `/occurrences/{id}/attachments/presign` | Presign S3 (503 se S3 não configurado) |
| GET | `/dashboard/summary` | KPIs agregados |
| GET | `/dashboard/timeseries` | Série temporal |
| GET | `/dashboard/breakdowns` | Distribuições |
| GET | `/map/occurrences` | Pontos com privacidade aplicada |
| GET | `/organizations`, `/responders`, `/destinations`, `/stage-policies` | Cadastros |
| POST | `/ops/detect-overdue` | Rotina idempotente de detecção de atraso (demo) |

### Exemplos

Criar ocorrência:

```bash
curl -X POST http://localhost:4000/occurrences \
  -H 'Content-Type: application/json' \
  -H 'x-actor-role: REPORTER' \
  -d '{"species":"cão","animalCount":1,"description":"Cão ferido.","location":{"latitude":-23.55,"longitude":-46.63,"region":"Centro"}}'
```

Executar uma transição:

```bash
curl -X POST http://localhost:4000/occurrences/<id>/transitions \
  -H 'Content-Type: application/json' \
  -H 'x-actor-role: COORDINATOR' \
  -d '{"action":"COMPLETE_TRIAGE"}'
```

Erros seguem um envelope consistente:

```json
{ "error": { "code": "OUTCOME_REQUIRED", "message": "Encerrar exige um resultado documentado (outcome).", "correlationId": "…" } }
```

---

## Arquitetura

- **Serverless-ready, mas local por padrão.** A camada de serviço depende de uma
  interface de repositório (`OccurrenceRepository`). O MVP usa
  `InMemoryRepository`. O mesmo contrato pode ser implementado sobre AWS.
- **Consistência estado + evento.** Toda transição grava a ocorrência e o evento
  juntos (`applyTransition`), com concorrência otimista por `version`.
- **Idempotência.** Transições e a rotina de atraso aceitam/usam
  `idempotencyKey` para evitar duplicação.
- **Dashboard no backend.** KPIs são calculados no servidor sobre o volume
  controlado do MVP; o frontend não varre o histórico.

### Mapeamento para DynamoDB (produção)

Estratégia sugerida: **tabelas separadas** (mais simples de demonstrar que
single-table para o MVP).

| Tabela | PK | SK | GSIs sugeridos |
|---|---|---|---|
| `Occurrences` | `id` | — | GSI por `status`+`createdAt`; GSI por `region`+`createdAt`; GSI por `currentResponsibleId` |
| `OccurrenceEvents` | `occurrenceId` | `occurredAt#eventId` | GSI por `idempotencyKey` (dedup) |
| `Assignments` | `occurrenceId` | `assignmentId` | GSI por `assigneeId` |
| `Organizations`/`Responders`/`Destinations` | `id` | — | GSI por `region` quando necessário |

- Transição = `TransactWriteItems` (update condicional da ocorrência por
  `version` + put do evento).
- Listagens por status/região/responsável usam os GSIs (sem `Scan`).
- **Sem consulta geoespacial nativa completa**: para mapa/filtros, usar geohash
  por célula (atributo adicional) ou filtro por região + distância no backend.
  Amazon Location Service é opcional para geocodificação/tiles.
- TTL: aplicar **apenas** a dados temporários; **nunca** expirar eventos de
  auditoria sem política aprovada.

Para trocar a implementação, crie `DynamoRepository implements OccurrenceRepository`
e injete-o em `createApp(repo)`.

## Configuração AWS (deploy manual — não executado aqui)

> O agente **não** cria infraestrutura, não faz deploy e não executa operações
> pagas. Os passos abaixo são para execução manual pela equipe.

- IaC sugerida: **AWS SAM** ou **CDK** (escolher uma). API Gateway + Lambda +
  DynamoDB + S3 privado + CloudWatch; Cognito para auth; IAM de menor privilégio.
- Secrets em Secrets Manager/Parameter Store. Nunca commitar `.env` reais.
- S3: upload por URL pré-assinada com expiração; validar tipo/tamanho.
- Rotina de atraso: EventBridge Scheduler + Lambda chamando a lógica de
  `detectOverdue` (idempotente).
- Notificações (SNS/e-mail): adaptador **opcional, desativado por padrão**. Não
  declarar envio sem o serviço configurado.

### Custos/limitações do MVP
- Em memória: dados vivem no processo; reiniciar reseta tudo (use o seed).
- Agregações recalculadas a cada requisição (ok para o volume de demo). Em
  escala, usar tabelas de resumo atualizadas por evento (consistência eventual).
- Auth é simulada por cabeçalhos — **apenas para desenvolvimento**.
- Presign S3 e MCP AWS não estão ativados; documentados como pré-requisitos.

---

## Segurança e privacidade
- Backend é a autoridade de permissões; esconder botões no frontend não é
  controle de acesso.
- Dados de contato do solicitante são restritos e não aparecem em vistas
  públicas; o mapa reduz precisão de coordenadas para papéis sem privilégio.
- Validação no servidor, limite de payload (256 kB), CORS configurável.
- Nenhum dado pessoal é registrado no console; erros não vazam detalhes internos.
- Aplica princípios da LGPD: finalidade, minimização, controle de acesso.

## Kiro Hooks (desenvolvimento)
Em `.kiro/hooks/` (automação de desenvolvimento, não runtime AWS):
- `schema-compat-review` — revisa compatibilidade ao salvar modelos compartilhados.
- `kpi-tests-on-change` — roda testes de KPI ao editar `kpi.ts`.
- `docs-check-on-stop` — lembra de atualizar README/dicionário ao finalizar.
- `secret-scan-on-stop` — lembrete para não versionar segredos.

---

## Decisões técnicas
1. **Monorepo com pacote `shared`**: evita duplicar o contrato entre as três
   frentes (backend/frontend/dados) e mantém os KPIs testados num só lugar.
2. **Repositório em memória com interface**: entrega um MVP executável sem AWS,
   preservando o caminho para DynamoDB sem reescrever o serviço.
3. **Máquina de estados explícita** (`flow.ts`): transições, papéis e campos
   obrigatórios são dados, não código espalhado — fáceis de auditar e testar.
4. **KPIs puros e testados**: timestamps ausentes nunca viram zero; cada valor
   carrega `count`.
5. **Google Maps sem dependências extras**: o script é carregado sob demanda e o
   agrupamento de marcadores é uma grade simples (`lib/cluster.ts`). Sem chave,
   cai para dispersão em SVG + lista acessível.
6. **Auth simplificada por cabeçalho**: suficiente para demonstrar papéis;
   claramente marcada como não-produção.

## Itens não implementados (escopo futuro)
- Persistência DynamoDB real e IaC (SAM/CDK) — mapeamento documentado acima.
- Upload de anexos (presign S3) — endpoint responde 503 até configurar.
- Autenticação real (Cognito) e refino de autorização por organização.
- Geocodificação de endereços (Google Geocoding ou Amazon Location).
- Tabelas de agregação por evento e pipeline analítico.
- Edição de `StagePolicy` pela tela de administração.
- Notificações (SNS/e-mail) como adaptador ativável.
