# PetChain — Prompt de implementação do Backend

## Instrução para o Kiro

Atue como engenheiro(a) backend/cloud sênior. Implemente a API e as regras de negócio do **PetChain**, plataforma de coordenação e rastreabilidade de resgates de animais. Leia também `frontend.md` e `dados.md` para manter contratos e persistência consistentes.

Antes de codificar:
1. Inspecione o repositório, stack, infraestrutura e testes existentes.
2. Preserve código funcional e não assuma que a conta AWS ou credenciais já estão configuradas.
3. Proponha uma arquitetura MVP com custos e complexidade adequados a um hackathon.
4. Crie especificação, desenho técnico e tarefas no fluxo de Specs do Kiro quando disponível.
5. Consulte documentação AWS atual via MCP AWS Documentation se o servidor estiver configurado. Não invente ferramentas MCP, configurações ou credenciais.
6. Implemente incrementalmente, com testes e instruções de execução local.

## Objetivo

Fornecer uma API segura e auditável para registrar ocorrências, controlar o fluxo do resgate, atribuir responsáveis, registrar cada transição e alimentar um dashboard analítico. A ocorrência não pode ser considerada resolvida apenas porque alguém mudou seu status: o encerramento exige resultado e evidências/confirmadores adequados ao processo.

## Arquitetura AWS sugerida

Para um MVP serverless:
- Amazon API Gateway: API REST.
- AWS Lambda: handlers e regras de negócio.
- Amazon DynamoDB: ocorrências, eventos, atribuições e cadastros operacionais.
- Amazon S3: fotos/anexos, com acesso privado e URLs pré-assinadas quando necessário.
- Amazon CloudWatch: logs, métricas e alarmes.
- AWS IAM: permissões de menor privilégio.
- Amazon Cognito ou mecanismo de autenticação disponível no ambiente: autenticação e identidade dos usuários.
- Amazon Location Service pode ser integrado para geocodificação/mapa se estiver disponível e houver tempo.

Não obrigue o uso de todos os serviços caso o ambiente do hackathon tenha limitações. Forneça uma alternativa local para desenvolvimento, mas mantenha adaptadores e contratos claros para AWS. Não crie infraestrutura real, não faça deploy e não execute operações pagas sem aprovação explícita do usuário.

Infraestrutura como código deve ser preferencialmente AWS SAM, AWS CDK ou a ferramenta já existente no repositório. Escolha uma única opção, justifique e documente. Nunca comite credenciais ou arquivos `.env` reais.

## Domínio e entidades

Implementar, no mínimo:
- Occurrence: ocorrência principal.
- OccurrenceEvent: evento imutável da linha do tempo.
- Assignment: atribuição de uma etapa a um usuário/organização.
- Responder: voluntário ou equipe que pode assumir tarefas.
- Organization: ONG, clínica, abrigo ou órgão participante.
- Destination: destino de encaminhamento e confirmação de disponibilidade/recebimento.
- Attachment: metadados de foto/anexo; binário no S3.
- StagePolicy: prazo padrão e regras configuráveis de cada etapa, se viável.

Campos mínimos da ocorrência:
`id`, `createdAt`, `updatedAt`, `reportedAt` (se conhecido), `status`, `severity`, `severityReason`, `species`, `animalCount`, `description`, `location`, `locationAccuracy`, `reporterContact` (restrito), `currentResponsibleId`, `currentOrganizationId`, `currentStage`, `blockReason`, `nextActionAt`, `closedAt`, `outcome`, `source`, `version`.

A localização deve suportar latitude/longitude e precisão; validar faixas numéricas e tratar coordenadas ausentes. Dados pessoais devem ser minimizados e segregados/protegidos por autorização.

## Modelo de fluxo e transições

Definir estados e transições explicitamente, por exemplo:
- `RECEIVED` — registrada.
- `TRIAGE_PENDING` — aguardando triagem.
- `READY_FOR_ASSIGNMENT` — triada, aguardando responsável.
- `ASSIGNED` — responsável designado, aguardando aceite.
- `ACCEPTED` — aceite confirmado.
- `IN_PROGRESS` — resgate em andamento.
- `TRANSPORT_PENDING` — aguardando transporte.
- `IN_TRANSIT` — transporte em andamento.
- `DESTINATION_PENDING` — aguardando vaga/confirmação de destino.
- `RECEIVED_BY_DESTINATION` — recebimento confirmado.
- `FOLLOW_UP` — acompanhamento posterior necessário.
- `RESOLVED` — resultado documentado e encerrado.
- `BLOCKED` — bloqueada, com motivo obrigatório.
- `CANCELLED` — cancelada, com motivo obrigatório.

Revise os estados com base no domínio e mantenha o fluxo configurável sem criar complexidade excessiva. Não permita transições arbitrárias. A API deve validar:
- estado atual;
- transição permitida;
- papel/permissão do ator;
- campos obrigatórios da transição;
- motivo obrigatório para recusa, cancelamento, bloqueio, reatribuição ou encerramento excepcional;
- confirmação de recebimento pelo destino;
- existência de responsável quando a etapa exige um;
- regras de concorrência para impedir atualizações conflitantes.

A gravidade e o estado operacional são conceitos separados. Gravidade inicial pode ser `CRITICAL`, `HIGH`, `MODERATE`, `LOW` ou `NEEDS_TRIAGE`, mas documente a semântica e não substitua avaliação profissional. Guardar quem definiu/alterou a gravidade, quando e por quê.

## Trilha de eventos e auditoria

Cada alteração relevante deve criar um `OccurrenceEvent` imutável com:
- `eventId`, `occurrenceId`, `eventType`, `occurredAt`, `actorId`, `actorRole`;
- `previousStatus`, `newStatus`, `previousStage`, `newStage`;
- `comment`, `reasonCode`, `assignmentId`;
- `metadata` sem segredos ou dados pessoais desnecessários;
- `correlationId` e chave de idempotência, quando aplicável.

Não sobrescrever a história anterior. Guardar o estado atual da ocorrência para consulta rápida e eventos para a timeline/auditoria. Garantir consistência entre atualizar a ocorrência e registrar o evento, usando transações do DynamoDB quando apropriado. Use versionamento otimista ou condição equivalente para evitar que duas pessoas sobrescrevam mudanças simultâneas.

## Endpoints REST sugeridos

Implementar e documentar:
- `POST /occurrences` — criar ocorrência.
- `GET /occurrences` — listar com filtros, paginação e ordenação segura.
- `GET /occurrences/{id}` — detalhes.
- `GET /occurrences/{id}/events` — timeline paginada.
- `POST /occurrences/{id}/transitions` — executar transição validada.
- `POST /occurrences/{id}/assignments` — atribuir/reatribuir responsável.
- `POST /occurrences/{id}/attachments/presign` — gerar URL de upload privada, se S3 estiver habilitado.
- `POST /occurrences/{id}/confirmations` — confirmação de etapa/recebimento, ou incorporar ao endpoint de transição com contrato documentado.
- `GET /dashboard/summary` — KPIs agregados.
- `GET /dashboard/timeseries` — série temporal por período.
- `GET /dashboard/breakdowns` — agrupamentos por status, gravidade, tipo e região.
- `GET /map/occurrences` — pontos autorizados para mapa.
- `GET /organizations` e `GET /responders` — listas conforme permissões.
- `GET /health` — health check sem expor detalhes internos.

Use respostas de erro consistentes com código, mensagem segura e identificador de correlação. Validar entradas no servidor, limitar tamanho de payloads, aplicar rate limiting quando possível e usar paginação. Não confiar na validação frontend.

## Regras para atribuição e confirmação

- Uma atribuição tem responsável, etapa, status, hora de criação, prazo e histórico.
- Aceite e recusa são ações explícitas.
- Recusa exige motivo selecionável e/ou comentário.
- Cancelamento deve permitir reatribuição e manter o histórico.
- Prazo vencido deve sinalizar a ocorrência como atrasada ou gerar um evento de atraso; não mudar silenciosamente para resolvida.
- Confirmações devem identificar quem confirmou e quando.
- A confirmação de recebimento por clínica/abrigo deve ser distinta da confirmação de chegada pelo motorista, quando aplicável.
- Evitar notificar automaticamente terceiros sem consentimento/configuração.

## Hooks e automações em runtime AWS

Não confundir **Kiro Agent Hooks** (automação durante o desenvolvimento) com eventos do backend em produção. No runtime, implementar automações apenas quando forem necessárias e demonstráveis.

Para o MVP:
- Criar uma rotina que detecte etapas vencidas, via EventBridge Scheduler + Lambda, ou alternativa local testável.
- Registrar evento `STAGE_OVERDUE` e atualizar indicador de atraso sem apagar o status de negócio.
- Criar notificações como adaptador opcional (por exemplo, SNS ou e-mail), desativado por padrão até configuração.
- Evitar duplicidade com idempotência.
- Registrar erros e métricas no CloudWatch.
- Não declarar que notificações foram enviadas se o serviço não estiver configurado.

## Segurança

- IAM de menor privilégio.
- Autenticação e autorização por papel, como `ADMIN`, `COORDINATOR`, `RESPONDER`, `ORGANIZATION`, `REPORTER` conforme necessário.
- Restringir dados de contato e localização exata conforme papel.
- S3 privado, validação de tipo/tamanho e URLs pré-assinadas com expiração.
- Não logar tokens, dados de contato ou detalhes pessoais desnecessários.
- Sanitizar entradas e prevenir injeção.
- Secrets em AWS Secrets Manager/Parameter Store ou ambiente seguro; nunca em código.
- Auditar alterações de gravidade, responsáveis, destino e encerramento.
- Aplicar princípios da LGPD: finalidade, minimização, controle de acesso e retenção adequada.
- Tratar localização pública com precisão reduzida ou ocultação, quando necessário.

## MCP e documentação AWS

Se o MCP de documentação AWS estiver configurado no Kiro:
- Consultar documentação oficial atual para serviços e padrões utilizados.
- Verificar sintaxe e recursos de SAM/CDK, IAM, DynamoDB e API Gateway antes de fixar decisões.
- Não presumir que um MCP de AWS pode fazer deploy ou acessar a conta. Documentação MCP é contexto para o agente; acesso à infraestrutura exige ferramenta e permissão apropriadas.
- Não inventar nome de servidor, ferramenta ou arquivo de configuração. Se MCP não estiver disponível, seguir a documentação local do repositório e registrar a limitação.

## Testes obrigatórios

Criar testes unitários e/ou de integração para:
- Criação e validação de ocorrência.
- Transições permitidas e proibidas.
- Bloqueio/atraso com motivo.
- Aceite, recusa e reatribuição.
- Confirmação de recebimento.
- Concorrência e idempotência.
- Criação de evento junto à alteração de estado.
- Autorização e proteção de dados pessoais.
- Agregações do dashboard e filtros por período.
- Coordenadas inválidas ou ausentes.
- Rotina de detecção de prazo vencido sem eventos duplicados.

## Critérios de aceite

1. Criar ocorrência grava os dados e o primeiro evento.
2. Toda transição válida atualiza estado e acrescenta evento auditável.
3. Transições inválidas retornam erro claro e não alteram os dados.
4. O histórico da ocorrência pode ser consultado cronologicamente.
5. Atribuições, aceites, recusas e confirmações são rastreáveis.
6. O dashboard obtém dados agregados do backend, sem varrer todos os registros no frontend.
7. Ocorrências georreferenciadas podem ser consultadas com controles de privacidade.
8. A rotina de atraso é idempotente e testada.
9. O projeto roda localmente com instruções reproduzíveis.
10. A infraestrutura AWS pode ser implantada seguindo instruções, mas o agente não deve fazer deploy sem aprovação.

## Entregáveis

- API funcional e documentada.
- Modelo de domínio e validações.
- Testes automatizados.
- Infraestrutura como código, se viável.
- `.env.example` sem segredos.
- README com execução local, configuração AWS, deploy manual, permissões e custos/limitações.
- Exemplos de requisições/respostas e lista de endpoints.
