# PetChain — Prompt de implementação do Frontend

## Instrução para o Kiro

Atue como engenheiro(a) frontend sênior e implemente o frontend funcional do **PetChain**, uma plataforma de coordenação e rastreabilidade de resgates de animais. Este arquivo define o escopo do frontend; consulte também `backend.md` e `dados.md` para manter os contratos e os modelos consistentes.

Não entregue somente uma tela estática ou um protótipo com números hardcoded. Construa uma aplicação utilizável, conectada à API quando disponível, com estados de carregamento, erro, vazio e sucesso. Se o backend ainda não estiver implementado, crie uma camada de serviço tipada e dados mock claramente identificados, de modo que seja simples trocar o mock pela API real sem reescrever os componentes.

Antes de implementar:
1. Inspecione o repositório e identifique framework, convenções, scripts e dependências existentes.
2. Preserve decisões e código já funcionais. Não substitua a stack sem justificativa.
3. Leia `backend.md` e `dados.md`; se houver divergência, proponha uma resolução antes de duplicar contratos.
4. Gere uma especificação/planejamento no Kiro para as tarefas maiores e implemente em etapas pequenas.
5. Não invente integração AWS já configurada. Documente pré-requisitos e configurações ausentes.

## Objetivo do produto

Acompanhar cada ocorrência desde o registro inicial até a conclusão, mantendo uma trilha auditável de eventos, responsáveis, prazos e confirmações. A interface deve ajudar a responder:
- Quais ocorrências exigem atenção agora?
- Em que etapa cada resgate está?
- Quem é o responsável atual?
- Qual etapa está atrasada ou bloqueada, e por quê?
- Quais são os gargalos da operação por período, região e tipo de ocorrência?
- Onde estão as ocorrências registradas?

## Stack e diretrizes

- Utilize a stack existente no repositório. Se o projeto estiver vazio, prefira React + TypeScript + Vite.
- Interface responsiva, acessível, em português brasileiro (`pt-BR`).
- Use componentes reutilizáveis, validação de formulários e tipagem forte.
- Evite adicionar dependências pesadas sem necessidade.
- Separe componentes visuais, páginas, tipos, configuração e camada de acesso à API.
- Não exponha chaves AWS, credenciais ou segredos no frontend.
- Não simule dados reais: identifique explicitamente todos os dados fictícios.
- Não use IA para declarar automaticamente diagnóstico veterinário ou definir a gravidade final. A interface deve apresentar triagem como apoio à avaliação humana.

## Páginas e funcionalidades obrigatórias

### 1. Dashboard operacional

Criar uma página inicial com filtros por intervalo de datas, status, gravidade, tipo de ocorrência e região. Exibir:
- Total de ocorrências abertas.
- Ocorrências de alta prioridade ainda não resolvidas.
- Ocorrências sem responsável designado.
- Ocorrências com etapas atrasadas ou bloqueadas.
- Tempo mediano entre registro e primeiro aceite.
- Tempo mediano por etapa.
- Taxa de conclusão.
- Taxa de quebra de fluxo: etapas canceladas, recusadas, vencidas ou reatribuídas.
- Distribuição de ocorrências por status, gravidade, região e tipo.
- Tendência temporal de entradas e conclusões.

Todo indicador deve mostrar período, unidade, definição acessível e estado sem dados. Não invente resultados; os números devem vir da API ou de dados mock identificados.

### 2. Mapa de ocorrências

Implementar um mapa interativo com marcadores geográficos, agrupamento quando houver muitos pontos e filtros por status, gravidade, tipo e data.
- Ao selecionar um marcador, mostrar resumo, status, gravidade, horário de abertura e responsável atual.
- Oferecer alternativa em lista/tabela para acessibilidade e para quando o mapa não carregar.
- Tratar coordenadas ausentes ou imprecisas.
- Não mostrar publicamente endereço residencial, telefone ou dados pessoais do solicitante.
- Não publicar coordenadas exatas de animais em risco em uma vista pública. O acesso detalhado deve respeitar permissões.
- Usar um provedor de mapas configurável; não assumir que uma chave de mapa existe. Se não houver chave, mostrar uma vista alternativa útil e instruções de configuração.

### 3. Registro de ocorrência

Criar formulário dividido em seções:
- Localização: endereço/referência, latitude/longitude quando disponível e precisão da localização.
- Animal: espécie, quantidade, porte aproximado, cor/características, foto opcional.
- Situação: descrição, ferimentos aparentes, riscos do local e informações desconhecidas.
- Solicitante: nome e canal de contato, apenas quando necessários; não expor esses dados em painéis públicos.
- Triagem inicial: gravidade proposta, justificativa e indicação de que precisa de validação humana.
- Metadados: data/hora de ocorrência se conhecida, fonte da informação e consentimentos aplicáveis.

Regras:
- Campos obrigatórios e opcionais devem estar explícitos.
- Validar formatos, limites de texto e tipos de arquivo.
- Não exigir dados que o solicitante não tem como saber.
- Permitir salvar com informações incompletas, marcando os dados pendentes para triagem.
- Ao salvar, apresentar o identificador da ocorrência e navegar para a página de detalhes.
- Prevenir envio duplicado por clique repetido; tratar idempotência conforme suporte da API.
- Mostrar erros de validação da API de forma compreensível.

### 4. Detalhes e acompanhamento do fluxo

Esta é a funcionalidade central. Cada ocorrência deve ter:
- Cabeçalho com ID, status, gravidade, horário de abertura e responsável atual.
- Linha do tempo cronológica de eventos, com data/hora, ator, ação, etapa anterior/nova etapa, comentário e confirmação.
- Visualização das etapas do fluxo com concluída, atual, pendente, bloqueada ou atrasada.
- Registro do histórico de atribuições, aceites, recusas, cancelamentos e reatribuições.
- Indicação explícita do motivo de bloqueio e da próxima ação necessária.
- Botões de ação exibidos conforme o papel e o estado: atribuir responsável, aceitar, recusar com motivo, iniciar deslocamento, confirmar chegada, registrar encaminhamento, confirmar recebimento, bloquear/desbloquear, reatribuir e encerrar.
- Formulários de confirmação com observações e anexos quando aplicável.
- Aviso de que encerrar exige resultado documentado e que casos não concluídos devem ter motivo.
- Atualização após ações com confirmação visual; não mostrar sucesso antes de a API confirmar.

Não permitir que a interface altere livremente o status sem validar transições no backend. A interface deve refletir as transições autorizadas pela API.

### 5. Fila de trabalho

Criar uma tabela ou lista com busca e filtros para:
- Sem responsável.
- Aguardando aceite.
- Etapas próximas do prazo.
- Atrasadas.
- Bloqueadas por transporte, vaga, clínica ou falta de informação.
- Alta gravidade.
- Aguardando confirmação de recebimento.

Ordenação padrão deve favorecer gravidade e risco de atraso, mas explicar os critérios. Não usar uma pontuação opaca para substituir decisão humana.

### 6. Administração básica

Para o MVP, oferecer telas simples para consultar responsáveis/organizações e configurar prazos padrão por etapa, se houver permissão. Se isso exceder o escopo, priorizar um painel de configuração mínima e documentar o restante.

## Design e experiência

- Visual limpo, confiável e operacional; evitar estética infantilizada.
- Hierarquia visual clara para gravidade e atrasos, sem depender apenas de cores.
- Usar ícones e rótulos de texto em conjunto.
- Layout responsivo para desktop e celular, já que voluntários podem acessar no campo.
- Exibir data/hora em formato local brasileiro e armazenar/receber timestamps ISO 8601 com timezone.
- Mensagens de erro úteis, estados de carregamento e telas vazias com orientação.
- Confirmar ações críticas e exigir motivo para recusa, cancelamento, bloqueio ou encerramento sem resgate.
- Garantir navegação por teclado, rótulos de formulário e contraste adequado.

## Contrato de integração

Criar `src/services/api` (ou equivalente) e centralizar chamadas HTTP. Base URL via variável de ambiente, por exemplo `VITE_API_BASE_URL`. Não chamar `fetch` diretamente em todos os componentes.

Preparar integração para estes recursos REST, ajustando os nomes finais conforme `backend.md`:
- `POST /occurrences`
- `GET /occurrences`
- `GET /occurrences/{id}`
- `GET /occurrences/{id}/events`
- `POST /occurrences/{id}/transitions`
- `POST /occurrences/{id}/assignments`
- `GET /dashboard/summary`
- `GET /dashboard/timeseries`
- `GET /dashboard/breakdowns`
- `GET /map/occurrences`
- `GET /organizations`
- `GET /responders`

Usar paginação, filtros e respostas de erro consistentes. Não carregar todas as ocorrências para calcular o dashboard no navegador.

## Segurança e privacidade

- Não armazenar tokens sensíveis em localStorage se houver alternativa de sessão mais segura definida pelo backend.
- Não registrar dados pessoais no console.
- Respeitar controle de acesso retornado pelo backend.
- Não confiar em permissões apenas por esconder botões; o backend é a autoridade.
- Evitar mostrar telefone, endereço residencial ou dados pessoais em mapas e tabelas sem autorização.
- Tratar uploads de imagens com limite de tamanho e tipos permitidos.

## Testes e critérios de aceite

Criar testes para:
- Validação do formulário de ocorrência.
- Exibição de estados de carregamento, vazio e erro.
- Filtros do dashboard e da fila.
- Renderização da timeline em ordem cronológica.
- Botões disponíveis de acordo com status/permissão.
- Falha de API sem perda silenciosa de dados.
- Responsividade básica e acessibilidade dos formulários.

O frontend estará pronto para o MVP quando:
1. Uma ocorrência puder ser registrada e persistida pela API.
2. O usuário conseguir consultar detalhes e acompanhar a linha do tempo.
3. Uma transição autorizada atualizar o estado e gerar evento visível.
4. O dashboard consultar indicadores da API.
5. O mapa ou sua alternativa em lista mostrar ocorrências georreferenciadas.
6. A interface identificar atrasos e bloqueios sem inventar dados.
7. Houver instruções claras para executar localmente e configurar variáveis de ambiente.

## Entregáveis

- Código frontend funcional.
- Componentes e páginas organizados.
- Tipos TypeScript compartilháveis ou alinhados ao contrato de API.
- Testes essenciais.
- `.env.example` sem segredos.
- README com execução, configuração e limitações conhecidas.
- Lista de decisões técnicas e itens não implementados.
