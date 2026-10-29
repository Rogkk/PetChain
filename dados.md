# PetChain — Prompt de Dados, Banco e Analytics

## Instrução para o Kiro

Atue como engenheiro(a) de dados e arquiteto(a) de dados sênior. Defina e implemente a camada de persistência, histórico de eventos, consultas geográficas e indicadores do **PetChain**. Leia também `frontend.md` e `backend.md` e mantenha um único contrato de dados entre as três partes.

Antes de implementar:
1. Inspecione a stack e os recursos já existentes.
2. Identifique claramente dados operacionais, eventos históricos e agregações analíticas.
3. Não crie indicadores que não possam ser calculados a partir dos dados registrados.
4. Use dados sintéticos apenas para demonstração e identifique-os como fictícios.
5. Se o MCP de documentação AWS estiver disponível, consulte as práticas oficiais atuais de DynamoDB, S3, IAM e analytics.
6. Gere documentação do esquema e testes para as consultas/indicadores.

## Objetivo

Projetar dados que permitam:
- Consultar o estado atual de cada ocorrência.
- Reconstruir a linha do tempo completa.
- Identificar etapa, responsável e motivo de bloqueio.
- Medir duração por etapa e tempo de resposta.
- Encontrar falhas de coordenação, cancelamentos e reatribuições.
- Visualizar ocorrências georreferenciadas com filtros.
- Alimentar dashboards sem exigir que o frontend processe todo o histórico.
- Preservar privacidade e trilha de auditoria.

## Categorias de dados

Separar logicamente:

1. **Dados operacionais atuais:** estado e campos de consulta rápida da ocorrência.
2. **Eventos imutáveis:** histórico de status, atribuições, confirmações, bloqueios e alterações.
3. **Cadastros de referência:** organizações, voluntários/responsáveis, destinos e políticas de etapa.
4. **Anexos:** metadados no banco; arquivos privados no S3.
5. **Dados analíticos/agregações:** contadores e séries temporais para dashboard.
6. **Dados pessoais restritos:** contato do solicitante e outros campos privados, com acesso limitado.

## Modelo mínimo de dados

### Occurrence
- `id` — UUID/ID único.
- `createdAt`, `updatedAt` — ISO 8601 UTC.
- `reportedAt` — data/hora da ocorrência se conhecida.
- `status`, `currentStage`, `severity`, `severityReason`.
- `species`, `animalCount`, `size`, `color`, `description`.
- `visibleInjuries`, `siteRisks`, `unknownInformation`.
- `location`: latitude, longitude, endereço/referência e precisão.
- `reporterId` e contato restrito, quando necessário.
- `currentResponsibleId`, `currentOrganizationId`, `currentAssignmentId`.
- `blockReason`, `nextActionAt`, `closedAt`, `outcome`.
- `source`, `version`, `isSynthetic` quando aplicável.

### OccurrenceEvent
- `eventId`, `occurrenceId`, `eventType`, `occurredAt`.
- `actorId`, `actorRole`.
- `previousStatus`, `newStatus`, `previousStage`, `newStage`.
- `reasonCode`, `comment`, `assignmentId`, `correlationId`.
- `metadata` restrito, sem segredos nem cópia desnecessária de dados pessoais.
- `idempotencyKey` quando aplicável.

### Assignment
- `assignmentId`, `occurrenceId`, `stage`, `assigneeId`, `organizationId`.
- `createdAt`, `acceptedAt`, `declinedAt`, `startedAt`, `completedAt`.
- `dueAt`, `status`, `declineReason`, `cancelReason`, `reassignedFrom`.
- Histórico de atribuições deve permanecer acessível mesmo após reatribuição.

### Organization / Responder / Destination / StagePolicy
Definir os campos mínimos necessários para identificar organizações, papéis, disponibilidade, destinos, capacidade/vagas declaradas e prazos por etapa. Não modelar dados pessoais excessivos. Disponibilidade e vagas devem ter timestamp de atualização para não tratar dados antigos como atuais.

## Persistência AWS

Preferência para o MVP:
- DynamoDB para ocorrências, eventos, atribuições e entidades operacionais.
- S3 privado para fotos e anexos.
- Agregações no backend ou tabelas de resumo para o dashboard.
- CloudWatch para métricas técnicas e logs estruturados.

Escolher e documentar uma estratégia de chaves de DynamoDB baseada nos padrões de acesso reais. Uma opção é usar tabelas separadas para `Occurrences`, `OccurrenceEvents`, `Assignments` e `Organizations`; outra é single-table design. Escolha uma estratégia consistente e explique os trade-offs, sem misturar padrões incompatíveis.

Requisitos:
- Definir chaves primárias, índices secundários e padrões de consulta.
- Suportar listar por status, gravidade, responsável, região e datas sem fazer scan completo em cada solicitação.
- Paginar listas e eventos.
- Usar transações/condições para manter estado e evento consistentes.
- Projetar idempotência e controle de concorrência.
- Definir TTL somente para dados temporários que realmente possam expirar; nunca apagar eventos de auditoria por TTL sem política aprovada.
- Documentar custos e limitações do MVP.

## Dados geográficos

Guardar latitude, longitude e precisão da localização. Para consultas geográficas, selecionar uma estratégia compatível com o volume e a stack:
- geohash/células geográficas ou índice equivalente;
- consulta por região e filtro por distância no backend;
- integração opcional com Amazon Location Service para geocodificação ou mapas.

Não assumir que DynamoDB possui consultas geoespaciais nativas completas. Documentar as limitações da solução escolhida.

Para o dashboard:
- Agregar ocorrências por região/bairro e período.
- Tratar registros sem coordenadas.
- Não inferir que uma região tem poucos animais em risco apenas porque tem poucos registros.
- Ocultar ou reduzir precisão de coordenadas quando a audiência não tiver autorização.
- Não expor endereços residenciais ou contato do denunciante em vistas públicas.

## Definições de indicadores

Implementar e documentar no mínimo:

1. `openOccurrences`: ocorrências não encerradas.
2. `highPriorityOpen`: ocorrências de gravidade crítica/alta que não estão resolvidas/canceladas.
3. `unassignedOccurrences`: ocorrências elegíveis para atendimento sem responsável ativo.
4. `overdueStages`: etapas com prazo vencido e ainda não concluídas.
5. `medianTimeToFirstAcceptance`: mediana entre criação/triagem definida e primeiro aceite válido; definir exatamente o marco de início.
6. `medianStageDuration`: mediana da duração de cada etapa, considerando apenas etapas com início e fim registrados.
7. `resolutionRate`: proporção de ocorrências encerradas com resultado documentado em uma coorte/período claramente definido.
8. `flowBreakRate`: proporção de atribuições/etapas que tiveram recusa, cancelamento, prazo vencido ou reatribuição. Definir denominador e evitar dupla contagem.
9. `reassignmentRate`: proporção de atribuições que precisaram ser substituídas.
10. `occurrencesByStatus`, `occurrencesBySeverity`, `occurrencesBySpecies`, `occurrencesByRegion`.
11. `openedVsResolvedTimeSeries`: novas ocorrências e encerramentos por dia/semana.
12. `blockReasons`: contagem e duração dos bloqueios por categoria.
13. `destinationConfirmationTime`: tempo entre chegada declarada e confirmação do recebimento, se ambas as etapas forem registradas.

Para cada indicador, documentar:
- Nome técnico e rótulo.
- Fórmula matemática ou regra.
- Campos e eventos necessários.
- Janela temporal e timezone.
- Tratamento de dados ausentes e casos ainda abertos.
- Possíveis vieses/limitações.
- Exemplo de cálculo testado.

Não calcular média/mediana usando timestamps ausentes como zero. Para indicadores de duração, separar os casos em andamento dos concluídos quando apropriado. Exibir quantidade de observações junto aos valores para evitar conclusões com amostras pequenas.

## Estratégia para dashboard

Não executar scans completos em DynamoDB a cada carregamento. Escolher uma abordagem compatível com o MVP:
- agregações calculadas pelo backend sobre volume limitado e controlado de dados de demonstração;
- contadores e resumos atualizados por eventos;
- tabelas de agregação por dia/região/status;
- pipeline analítico separado apenas se houver tempo e justificativa.

Para o hackathon, priorizar consultas simples, previsíveis e demonstráveis. Se usar agregações pré-calculadas, explicar consistência eventual e como os números são atualizados.

Filtros suportados:
- intervalo de datas;
- status;
- gravidade;
- espécie/tipo;
- região;
- organização/responsável;
- etapas bloqueadas/atrasadas.

## Dataset de demonstração

Criar seed script com dados sintéticos coerentes para demonstrar:
- ocorrências em diferentes regiões;
- diversos níveis de gravidade;
- fluxo concluído e fluxo em andamento;
- aceite, recusa, cancelamento e reatribuição;
- etapas vencidas e bloqueadas por causas distintas;
- dados sem coordenadas;
- confirmação de recebimento;
- ocorrências duplicadas vinculadas ou marcadas para revisão.

Regras para dados sintéticos:
- Marcar registros como `isSynthetic=true` ou usar ambiente claramente separado.
- Não usar dados pessoais reais.
- Não apresentar os números do seed como estatísticas reais da cidade.
- Gerar timestamps consistentes para que os cálculos de duração façam sentido.
- Permitir reset/reseed no ambiente local.
- Criar dados suficientes para que gráficos e filtros demonstrem comportamento realista, sem exagerar os números.

## MCP e ferramentas de desenvolvimento

Se MCP estiver configurado:
- Usar o servidor oficial de documentação AWS para consultar práticas atuais.
- Usar MCP de banco ou observabilidade apenas se explicitamente instalado e autorizado.
- Não presumir que MCP tem acesso ao banco, à conta AWS ou a dados de produção.
- Não inserir credenciais, dados pessoais ou tokens em prompts de MCP.
- Não inventar configuração de servidor; documentar pré-requisitos e oferecer configuração manual.

## Kiro Hooks — automação de desenvolvimento

Propor hooks de agente do Kiro, em arquivos de configuração válidos para a versão/ambiente em uso, para automatizar tarefas de desenvolvimento:
1. Após edição de arquivos de modelo/schema: solicitar revisão de compatibilidade com `backend.md` e `frontend.md`.
2. Após edição de consultas/indicadores: executar testes de unidade/lint aplicáveis ou pedir ao agente para revisar fórmulas.
3. Ao finalizar uma tarefa: verificar se documentação de esquema e README foram atualizados.
4. Antes de concluir mudanças: verificar que não foram adicionados segredos ou dados reais ao repositório.

Antes de criar hooks, consulte a documentação atual do Kiro ou os exemplos instalados. Não invente nomes de eventos, campos JSON ou comandos de hook. Diferencie hooks do agente Kiro (desenvolvimento) das automações de runtime AWS.

## Qualidade e testes

Criar testes para:
- Fórmulas de cada indicador.
- Eventos fora de ordem e timestamps inválidos.
- Etapas abertas versus concluídas.
- Reatribuições e cancelamentos sem dupla contagem.
- Filtros de período e timezone.
- Dados sem coordenadas.
- Paginação e ordenação.
- Privacidade de localização e contato.
- Atualização de agregações e idempotência.
- Consistência entre estado da ocorrência e histórico de eventos.

## Critérios de aceite

1. O modelo permite reconstruir a linha do tempo completa.
2. Os padrões de consulta do dashboard estão documentados e não dependem de scans descontrolados.
3. Cada KPI possui definição verificável e testes.
4. A camada geográfica suporta mapa e filtros com tratamento de coordenadas ausentes.
5. Dados sintéticos são claramente marcados e reproduzíveis.
6. Dados pessoais e coordenadas sensíveis têm regras de acesso.
7. A estratégia de agregação é documentada, incluindo limitações de consistência.
8. O frontend consegue consumir os endpoints e campos definidos em `backend.md`.
9. As decisões de persistência e índices são documentadas.

## Entregáveis

- Esquema/modelos de dados.
- Estratégia de chaves e índices.
- Migrações/infraestrutura ou definições de tabelas.
- Seed script sintético.
- Implementação das agregações/consultas necessárias.
- Testes automatizados de indicadores.
- Dicionário de dados e glossário dos KPIs.
- Documentação de geolocalização, privacidade, retenção e limitações.
