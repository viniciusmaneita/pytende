# PRD — Pytende

Versão: 1.0 — primeira versão funcional. Data de elaboração e consulta: 07/11/2026.
Responsável pelo produto: administrador da ViniTech.
Status: PRD concluído para planejamento e implementação; decisões pendentes explicitadas, sem implementação da aplicação nesta entrega.

Convenções deste documento:

- **Confirmado:** requisito informado pelo responsável pelo produto.
- **Proposta técnica (PT):** solução recomendada para atender ao requisito; não representa uma nova funcionalidade solicitada.
- **Hipótese (H):** premissa de planejamento que precisa ser validada.
- **Decisão pendente (DP):** definição ainda necessária, com impacto e momento de resolução na seção 15.

Identificadores de código são escritos em inglês; textos, mensagens e interface serão em português brasileiro. As tarefas e verificações futuras deste documento não foram executadas nesta entrega.

## 1. Visão geral, problema e objetivo

O Pytende será o sistema interno de controle dos atendimentos da ViniTech. Os serviços abrangem formatação de computadores, configuração de sistemas, suporte técnico e desenvolvimento de sites.

O problema a resolver é a necessidade de centralizar clientes e serviços, acompanhar pendências e consultar o histórico dos atendimentos. O sistema deverá permitir que o responsável identifique o cliente, abra um atendimento, registre o andamento, conclua ou cancele e preserve o registro para consulta.

O sucesso da primeira versão será verificado pela execução integral dessa jornada, pela correspondência entre painel e registros persistidos e pela impossibilidade de contornar permissões e transições através de requisições diretas. Não foram definidas metas comerciais, economia de tempo ou volume de atendimentos.

## 2. Escopo e exclusões

### 2.1 Escopo confirmado

Autenticação administrativa; cadastro, consulta e atualização de clientes; abertura, consulta e atualização de atendimentos; pesquisa e filtros; controle de estados; observações acrescentadas ao histórico; histórico das transições; painel simples; valores estimado e final opcionais; interface responsiva; testes; documentação e plano de deploy.

O registro de valores é apenas informativo e não cria um módulo financeiro. Não haverá exclusão de atendimentos pela interface. Registros abertos por engano serão cancelados com justificativa.

### 2.2 Fora do escopo

- Portal e cadastro público de clientes, funcionários, equipes, setores, múltiplos perfis e aprovação de usuários.
- Agendamento, calendário, integrações com WhatsApp, e-mail ou SMS e notificações automáticas.
- Anexos, upload e download de arquivos de clientes.
- Pagamentos, cobranças, notas fiscais, controle financeiro, estoque e peças.
- Relatórios avançados, exportações, aplicativo móvel, SPA e API pública.
- Multiempresa, multi-tenant, Celery, RabbitMQ e agentes de IA.
- Reabertura de atendimentos e gerenciamento de tipos de serviço.
- Coleta de CPF, RG, endereço ou senhas de equipamentos, contas e sistemas dos clientes.

Não há fluxo de exclusão de clientes confirmado; não será acrescentado à interface desta versão. A política operacional para dados pessoais permanece pendente e não implica retenção indefinida.

### 2.3 Limites da entrega documental

Somente este PRD é criado nesta etapa. Código, configurações existentes, dependências, README, banco e infraestrutura não são alterados. Nenhuma migration, instalação, publicação de imagem, commit, push ou implantação faz parte desta entrega.

## 3. Usuários e matriz de permissões

Há um único perfil de negócio: **administrador**, utilizado pelo responsável pela ViniTech, com acesso a todos os registros.

**PT-01:** utilizar o modelo `User` nativo do Django e criar a conta por `python manage.py createsuperuser`, com entrada interativa segura. A autorização da aplicação exigirá usuário autenticado, ativo e `is_superuser=True`. `is_staff` isoladamente não autoriza o Pytende. Essa opção evita modelo de usuário e papéis personalizados desnecessários; não cria outros perfis de negócio.

| Ação | Administrador ativo | Visitante sem sessão | Conta inativa ou sem autorização administrativa |
| --- | --- | --- | --- |
| Acessar login | Permitido; sessão válida pode ir ao painel | Permitido | Não concede acesso privado |
| Entrar | Com credenciais válidas | Pode apresentar credenciais | Negado para acesso ao sistema |
| Sair | POST protegido por CSRF | Sem sessão privada a encerrar | Sessão eventualmente existente pode ser encerrada |
| Consultar painel, clientes e atendimentos | Todos os registros | Redirecionado ao login | Negado |
| Cadastrar/editar cliente | Permitido | Negado | Negado |
| Abrir/editar atendimento | Permitido, respeitando campos e regras | Negado | Negado |
| Alterar estado | Apenas transições permitidas | Negado | Negado |
| Acrescentar observação | Permitido | Negado | Negado |
| Editar/excluir observação ou transição | Não disponível pela interface | Negado | Negado |
| Excluir atendimento | Não disponível pela interface | Negado | Negado |
| Reabrir estado final | Negado | Negado | Negado |

A proteção deve alcançar GET, POST, URLs diretas e qualquer superfície administrativa habilitada. **PT-02:** não registrar modelos de negócio no Django Admin nesta versão, evitando caminhos alternativos de exclusão, edição de histórico ou alteração arbitrária de estado. O admin nativo, se mantido, ficará restrito às necessidades técnicas de autenticação. A conta é provisionada por comando, sem tela de cadastro público e sem senha padrão.

## 4. Jornadas e exceções

### 4.1 Jornada principal

1. Administrador entra com credenciais válidas e vê o painel.
2. Pesquisa cliente por nome ou telefone; consulta os resultados ou cadastra um novo.
3. Abre atendimento vinculado ao cliente, seleciona tipo e descreve a solicitação. Equipamento e valores podem ficar vazios.
4. O sistema gera protocolo e abertura, com estado Aberto.
5. Administrador inicia o atendimento, registra observações e, quando necessário, indica espera pelo cliente.
6. Retoma atendimento em espera, conclui o atendimento em andamento ou cancela com justificativa quando permitido.
7. O registro final permanece consultável com observações e transições. Retorno posterior gera outro atendimento.

### 4.2 Exceções e recuperação

| Situação | Comportamento esperado |
| --- | --- |
| Credencial inválida ou conta não autorizada | Mensagem genérica, sem revelar existência da conta; nenhuma informação privada |
| Sessão ausente/expirada | Login para navegação; escrita negada sem alteração; reapresentar ação após autenticação |
| Pesquisa sem resultado | Estado vazio claro e acesso ao cadastro correspondente |
| Clientes com nome/telefone iguais | Mostrar registros distintos; permitir conferir detalhes antes da seleção |
| Campos inválidos | Erros associados aos campos, preservar entradas seguras e não persistir alteração parcial |
| Protocolo ou registro inexistente | Página 404 após controle de acesso, sem erro técnico exposto |
| Cancelamento sem justificativa | Recusar; estado, encerramento e histórico permanecem iguais |
| Estado final ou transição ilegal | Recusar mesmo por POST forjado; não criar histórico |
| Duas abas alteram o mesmo estado | Revalidar estado persistido; rejeitar ação incompatível com mensagem para recarregar |
| Falha ao gravar histórico | Reverter também a mudança de estado e a data de encerramento |
| Observação incorreta | Acrescentar nova observação com correção; preservar a anterior |
| Banco indisponível | Resposta de indisponibilidade sem detalhes sensíveis; nenhuma confirmação falsa de salvamento |

## 5. Requisitos funcionais

Todos os requisitos são confirmados quanto à capacidade de negócio. Detalhes identificados como PT são propostas para torná-los implementáveis. Em todos os RF, o perfil autorizado é o administrador definido na seção 3, salvo a apresentação pública da tela de login. Regras RN são definidas na seção 6; testes T e sprints S constam das seções 12 e 14.

### RF-001 — Entrar e sair

- **Descrição:** autenticar e encerrar a sessão com recursos nativos do Django.
- **Perfil autorizado:** administrador ativo; visitante pode acessar apenas o formulário de entrada.
- **Fluxo:** informar usuário e senha; validar; criar sessão; abrir painel. A ação Sair encerra a sessão e retorna ao login.
- **Exceções:** credenciais inválidas, conta inativa/não administrativa, CSRF inválido e destino de retorno externo.
- **Regras:** RN-001 e RN-014.
- **Aceite:** dado administrador válido, quando entra, então acessa o painel; credenciais inválidas não criam sessão autorizada; POST de saída invalida o acesso seguinte; GET de saída não altera sessão; sem CSRF válido, POST é recusado; retorno informado não pode redirecionar para domínio externo.
- **Rastreabilidade:** T-001, T-010; S1.

### RF-002 — Cadastrar e corrigir cliente

- **Descrição:** registrar e atualizar nome, telefone, e-mail e observações cadastrais.
- **Perfil autorizado:** administrador.
- **Fluxo:** abrir formulário, preencher, validar no servidor e salvar; permitir corrigir os mesmos dados depois.
- **Exceções:** obrigatórios em branco, telefone inválido e e-mail malformado; duplicidade de nome ou telefone não é erro.
- **Regras:** RN-002, RN-003, RN-012 e RN-013.
- **Aceite:** nome/telefone vazios ou somente espaços impedem gravação; e-mail vazio é aceito e e-mail inválido é recusado; dois clientes com mesmo nome e telefone podem ser salvos; formatos válidos definidos em PT-04 geram pesquisa equivalente; `created_at` não muda na edição e `updated_at` é atualizado; formulário não contém CPF, RG nem endereço.
- **Rastreabilidade:** T-002, T-010; S2.

### RF-003 — Pesquisar clientes e consultar seu histórico

- **Descrição:** listar clientes, pesquisar por nome ou telefone e consultar detalhes e atendimentos relacionados.
- **Perfil autorizado:** administrador.
- **Fluxo:** pesquisar, escolher um registro, abrir seus dados e histórico, seguir para um atendimento ou iniciar outro para esse cliente.
- **Exceções:** lista vazia, homônimos e registro inexistente.
- **Regras:** RN-002, RN-003 e RN-014.
- **Aceite:** busca por trecho do nome sem distinguir maiúsculas encontra o cliente; busca por telefone com e sem pontuação encontra os mesmos registros normalizados; duplicados permanecem separados; detalhes mostram somente atendimentos ligados ao cliente selecionado, incluindo finais; ausência de resultados é explícita; ID inexistente retorna 404.
- **Rastreabilidade:** T-003, T-010; S2, integrado em S3.

### RF-004 — Abrir atendimento

- **Descrição:** criar atendimento com cliente, tipo de serviço e solicitação obrigatórios; equipamento e valores opcionais.
- **Perfil autorizado:** administrador.
- **Fluxo:** selecionar cliente existente, preencher formulário e salvar; abrir detalhes com protocolo gerado.
- **Exceções:** cliente inexistente, tipo fora da lista, descrição vazia, valor negativo ou inválido e falha de banco.
- **Regras:** RN-004 a RN-007, RN-011 a RN-013.
- **Aceite:** criação válida gera protocolo único, imutável, abertura automática e estado Aberto; encerramento fica vazio; omitir opcionais é aceito; valor zero é aceito e negativo é recusado; campos automáticos enviados manualmente não são aplicados; falha de validação não cria registro parcial; não há campo para senhas.
- **Rastreabilidade:** T-004, T-005, T-010; S3.

### RF-005 — Corrigir informações do atendimento

- **Descrição:** atualizar informações descritivas e valores sem alterar estado implicitamente.
- **Perfil autorizado:** administrador.
- **Fluxo:** abrir edição, corrigir campos autorizados, validar e salvar; retornar aos detalhes.
- **Exceções:** dados inválidos e tentativa de sobrescrever protocolo, estado, datas automáticas ou histórico.
- **Regras:** RN-004, RN-005, RN-007, RN-010 a RN-013.
- **Aceite:** descrição e dados opcionais válidos são atualizados; valores continuam opcionais e não negativos; protocolo, abertura e encerramento não mudam; edição não cria transição; POST com estado forjado não o modifica. Correção descritiva em registro final preserva estado e encerramento, conforme PT-05.
- **Rastreabilidade:** T-005, T-010; S3.

### RF-006 — Pesquisar e filtrar atendimentos

- **Descrição:** listar atendimentos com busca por protocolo ou cliente e filtros por estado e data de abertura.
- **Perfil autorizado:** administrador.
- **Fluxo:** informar busca e filtros, aplicar em conjunto, consultar resultados e abrir detalhes; permitir limpar filtros.
- **Exceções:** nenhum resultado, estado inválido, datas inválidas ou intervalo invertido.
- **Regras:** RN-004, RN-006, RN-012 e RN-014; PT-06.
- **Aceite:** protocolo completo encontra seu atendimento; nome do cliente encontra todos os respectivos registros; estado selecionado restringe resultados; início e fim inclusivos consideram a data local de abertura; combinar filtros aplica interseção; intervalo invertido exibe erro; limpar filtros remove as restrições; nenhum resultado não é erro de servidor.
- **Rastreabilidade:** T-006, T-010; S3.

### RF-007 — Alterar estado e encerrar

- **Descrição:** executar ação explícita de transição, validar origem/destino e registrar auditoria obrigatória.
- **Perfil autorizado:** administrador.
- **Fluxo:** nos detalhes, escolher ação permitida, informar justificativa se cancelar e confirmar por POST; gravar transição e estado atomicamente.
- **Exceções:** transição não permitida, estado desatualizado, justificativa vazia, autor forjado e falha de persistência.
- **Regras:** RN-006, RN-008 a RN-010 e RN-014.
- **Aceite:** cada uma das sete transições permitidas da seção 6 funciona; todas as demais combinações são recusadas; ao concluir/cancelar, encerramento recebe a data/hora do servidor; cancelamento sem texto útil é recusado; usuário e horário são determinados pelo servidor; falha de histórico reverte toda a ação; reenvio da mesma transição não duplica evento; estados finais não permitem reabertura.
- **Rastreabilidade:** T-007, T-010; S4.

### RF-008 — Acrescentar observações

- **Descrição:** acrescentar texto ao histórico do atendimento com data e autor.
- **Perfil autorizado:** administrador.
- **Fluxo:** preencher nova observação nos detalhes e salvar; exibir junto às anteriores.
- **Exceções:** texto vazio, registro inexistente, tentativa de editar/excluir observação e conteúdo HTML malicioso.
- **Regras:** RN-010, RN-011, RN-013 e RN-014.
- **Aceite:** texto válido gera nova observação com autor autenticado e horário do servidor; texto em branco é recusado; não existem ações de edição/exclusão nem endpoints que as executem; correção gera outro registro; HTML é apresentado escapado e não executado; acrescentar observação não altera estado nem encerramento. PT-05 permite acréscimos em registros finais.
- **Rastreabilidade:** T-008, T-010; S4.

### RF-009 — Consultar detalhes e históricos

- **Descrição:** consultar todos os dados autorizados do atendimento, observações e mudanças de estado.
- **Perfil autorizado:** administrador.
- **Fluxo:** abrir detalhes pela lista, painel ou histórico do cliente; consultar eventos em ordem cronológica determinística.
- **Exceções:** nenhum evento, datas iguais e registro inexistente.
- **Regras:** RN-008 a RN-014.
- **Aceite:** cada mudança exibe origem, destino, data e responsável; observação exibe texto, data e autor; cancelado mostra justificativa; finais seguem consultáveis; empates de horário têm ordenação estável por ID; atendimento recém-criado mostra Aberto e abertura mesmo sem transições; cliente e atendimento podem ser navegados entre si.
- **Rastreabilidade:** T-008, T-009, T-010; S4.

### RF-010 — Consultar painel inicial

- **Descrição:** mostrar quantidades por estado e atendimentos abertos mais recentes.
- **Perfil autorizado:** administrador.
- **Fluxo:** após login ou pela navegação, acessar indicadores e links para detalhes.
- **Exceções:** nenhuma ocorrência em um estado ou nenhum atendimento aberto.
- **Regras:** RN-006 e RN-014; PT-07.
- **Aceite:** os cinco estados aparecem com contagens consistentes, inclusive zero; a lista contém apenas estados não finais, ordenados por abertura decrescente e ID como desempate; mudança de estado aparece após nova consulta; vazio tem mensagem clara; não há indicadores financeiros.
- **Rastreabilidade:** T-009, T-010; S5.

## 6. Regras de negócio

| ID | Regra confirmada |
| --- | --- |
| RN-001 | Um único perfil administrador; sem cadastro público, senha padrão ou aprovação de usuários. Conta criada pelo mecanismo nativo do Django. |
| RN-002 | Nome e telefone obrigatórios; e-mail e observações cadastrais opcionais. Nome e telefone não são únicos. Não coletar CPF, RG ou endereço. |
| RN-003 | Aceitar formatos brasileiros usuais de telefone e normalizar para pesquisa. |
| RN-004 | Atendimento tem cliente, tipo e descrição obrigatórios; protocolo único, automático e não editável. Equipamento, marca, modelo e número de série são opcionais. |
| RN-005 | Tipos fixos: Formatação de computador, Configuração de sistemas, Suporte técnico, Desenvolvimento de sites e Outros. Sem tela de gestão. |
| RN-006 | Todo atendimento começa Aberto; somente as transições abaixo são permitidas. Concluído e Cancelado são finais; retorno gera novo atendimento. |
| RN-007 | Valores estimado/final opcionais, decimais, zero permitido e negativos rejeitados; sem funções financeiras adicionais. |
| RN-008 | Toda mudança registra estado anterior, novo estado, data e usuário. Alteração é explícita e validada no servidor. |
| RN-009 | Abertura automática; encerramento automático ao concluir/cancelar. Cancelamento exige justificativa. |
| RN-010 | Sem exclusão de atendimentos pela interface. Observações registram texto/data/autor e não são editadas ou excluídas pela interface; correções são novos registros. |
| RN-011 | Não armazenar senhas de equipamentos, contas ou sistemas dos clientes; sem anexos. |
| RN-012 | Modelos de negócio com `created_at` e `updated_at`; datas com timezone consistente em America/Sao_Paulo. |
| RN-013 | Dados cadastrais e informações descritivas podem ser corrigidos. A política de retenção, anonimização e exclusão precisa ser definida antes de dados reais; ausência de exclusão na interface não significa retenção indefinida. |
| RN-014 | Todas as páginas e operações privadas exigem autorização no servidor. |

### 6.1 Máquina de estados

| Origem | Destinos permitidos | Encerramento |
| --- | --- | --- |
| Aberto (`open`) | Em andamento (`in_progress`), Cancelado (`cancelled`) | Somente ao cancelar |
| Em andamento (`in_progress`) | Aguardando cliente (`waiting_customer`), Concluído (`completed`), Cancelado (`cancelled`) | Ao concluir ou cancelar |
| Aguardando cliente (`waiting_customer`) | Em andamento (`in_progress`), Cancelado (`cancelled`) | Somente ao cancelar |
| Concluído (`completed`) | Nenhum | Preservado |
| Cancelado (`cancelled`) | Nenhum | Preservado |

São sete transições permitidas: duas de Aberto, três de Em andamento e duas de Aguardando cliente. Permanecer no mesmo estado não é transição. Não permitir Aberto → Concluído nem Aguardando cliente → Concluído.

### 6.2 Propostas para detalhamento das regras

- **PT-03 — Protocolo:** usar `UUIDField` com geração automática, `unique=True`, `editable=False`; apresentação integral para pesquisa e consulta, sem requisito de sequência numérica ou ausência de lacunas. Confirmar legibilidade em DP-03. Não gerar por contagem de registros.
- **PT-04 — Telefone:** remover espaços e pontuação de apresentação; aceitar DDD e número nacional com 10 ou 11 dígitos, com prefixo internacional `+55` opcional. Normalizar como `55` + número nacional; distinguir prefixo internacional explícito de DDD 55. Exemplo: `(11) 99999-9999` e `+55 11 99999-9999` equivalem. Não aceitar letras; não presumir DDD para número local. Prefixos de operadora, ramais e números internacionais não brasileiros dependem de DP-02. A validação sintática não comprova titularidade nem existência da linha.
- **PT-05 — Correções:** permitir editar tipo, descrição, dados de equipamento e valores, inclusive em estados finais; permitir novas observações nos finais. Não permitir editar protocolo, cliente vinculado, estado pelo formulário geral, abertura, encerramento, justificativa já registrada ou eventos. A troca de cliente vinculado não foi definida (DP-04). Correção de justificativa será nova observação. Não criar auditoria genérica de todas as edições, pois somente transições e observações foram exigidas.
- **PT-06 — Consultas:** nome com `icontains`; telefone sobre campo normalizado; protocolo por valor completo; busca de atendimento por nome do cliente. Filtro de abertura como intervalo de datas locais, inclusivo nas duas extremidades; construir limites conscientes de timezone. Paginação nativa sugerida de 25 itens, ordenação estável e filtros preservados entre páginas. Pesquisa sem acentos não é requisito confirmado.
- **PT-07 — Painel:** interpretar “abertos” como não finalizados: Aberto, Em andamento e Aguardando cliente. Mostrar os dez mais recentes, sem paginação no painel; lista completa na tela de atendimentos. Sem gráficos avançados. Interpretação registrada em DP-05.
- **PT-08 — Valores:** `DecimalField(max_digits=12, decimal_places=2)`, `null=True`, `blank=True`; moeda BRL proposta em DP-06. Entrada brasileira `0,00` aceita; mais de duas casas decimais recusadas sem arredondamento silencioso. `NULL` significa não informado e zero significa informado como zero. Não exigir valor final na conclusão, nem relação de ordem entre valores.

## 7. Requisitos não funcionais

| ID | Requisito e verificação planejada |
| --- | --- |
| RNF-001 | Interface em pt-BR, responsiva e consistente com referência verificada. Verificar telas a 360, 768 e 1280 px; nenhuma ação essencial cortada; tabela pode ter rolagem local identificada. |
| RNF-002 | Navegação completa por teclado, foco visível, rótulos associados, mensagens acessíveis e estados não dependentes só de cor. Meta proposta: contraste WCAG AA, 4,5:1 em texto comum, 3:1 em texto grande e componentes pertinentes. Verificação manual por tela. |
| RNF-003 | Validação e autorização no servidor, inclusive URLs diretas e parâmetros forjados; nenhuma gravação indevida nos testes de acesso e campos protegidos. |
| RNF-004 | CSRF em operações mutáveis, escape padrão, sessões nativas e métodos HTTP adequados. GET não altera negócio. Testar CSRF com verificações habilitadas. |
| RNF-005 | Produção com `DEBUG=False`, `ALLOWED_HOSTS` explícito, HTTPS, cookies seguros e configuração de proxy confiável. Executar `check --deploy` com configuração de produção e tratar achados antes de liberar. |
| RNF-006 | PostgreSQL em desenvolvimento, testes de integração e produção; migrations versionadas; FKs e constraints de integridade; transição e auditoria atômicas. Verificar concorrência no mesmo banco de referência. |
| RNF-007 | Segredos vindos do ambiente, sem credenciais padrão; `.env` ignorado, `.env.example` somente placeholders; imagem e logs sem segredos. Falhar de forma segura quando segredo obrigatório estiver ausente. |
| RNF-008 | Datas conscientes de timezone, `USE_TZ=True`, `TIME_ZONE='America/Sao_Paulo'`, `LANGUAGE_CODE='pt-br'`; persistência de instantes em UTC e apresentação/filtros locais. Testar limites de dia. |
| RNF-009 | Código conforme PEP 8 e convenções Django; apps por domínio; dependências justificadas; testes com runner nativo; README atualizado na implementação. |
| RNF-010 | Logs operacionais sem senhas, tokens, cookies, corpo de formulários ou dados pessoais desnecessários. Não registrar termos de pesquisa pessoais em logs de acesso; preferir caminho sem query string. Verificar amostras sanitizadas. |
| RNF-011 | Backup externo ao host e restauração validada antes de dados reais; retenção, frequência, RPO e RTO aprovados em DP-08. Não considerar volume Docker como backup. |
| RNF-012 | Política de dados pessoais aprovada antes de dados reais; incluir cópias, logs e restaurações no tratamento de retenção/exclusão, sem fixar prazo neste PRD. |
| RNF-013 | Consultas paginadas, índices de uso frequente e prevenção de N+1; metas de desempenho condicionadas a H-01 e DP-09. Sem SLA ou capacidade garantidos nesta etapa. |
| RNF-014 | Health checks, reinício controlado, logs com rotação e procedimento de atualização/rollback testado; indisponibilidade não gera confirmação falsa de gravação. |

## 8. Telas, navegação e design system

### 8.1 Mapa de navegação

`Login → Dashboard → Customers → CustomerDetail → TicketDetail`.
`Customers → CustomerCreate / CustomerUpdate`.
`Tickets → TicketCreate / TicketUpdate / TicketDetail`.
`TicketDetail → StatusChange / NoteCreate` como ações/formulários, sem exigir páginas separadas.

Os nomes acima são identificadores internos. Rótulos visíveis: Entrar, Painel, Clientes, Atendimentos e Sair. Cabeçalho/navegação compartilhados, título da página e indicação da seção atual. Links de retorno devem preservar contexto quando possível, sem aceitar destinos externos arbitrários.

| Tela | Conteúdo e ações |
| --- | --- |
| Login | Usuário, senha, erros genéricos e Entrar; sem cadastro ou recuperação por e-mail nesta versão |
| Painel | Cinco contagens, atendimentos não finais recentes e acesso aos detalhes |
| Clientes | Pesquisa nome/telefone, lista e novo cadastro |
| Cadastro/edição do cliente | Campos confirmados, obrigatoriedade, validação e salvar/voltar |
| Detalhes do cliente | Cadastro, editar, histórico relacionado e abrir atendimento |
| Atendimentos | Busca protocolo/cliente, filtros estado/abertura, lista paginada e abrir atendimento |
| Abertura/edição do atendimento | Dados permitidos; automáticos somente para leitura quando aplicável; estado não editável pelo formulário geral |
| Detalhes do atendimento | Protocolo, cliente, dados, valores opcionais, datas, estado, justificativa, observações, transições e ações permitidas |

### 8.2 Referências efetivamente verificadas

O arquivo `design_system/design-system.html` não existe na estrutura inspecionada. O endereço remoto `https://getdesign.md/design-md/wise/DESIGN.MD` não ficou acessível pela ferramenta de consulta. Não foi possível comparar essas duas referências.

O preview informado, `httos://getdesign.md/design.md/wise/priveview`, tem esquema e caminho aparentemente incorretos. Foram tentados explicitamente dois candidatos HTTPS: `https://getdesign.md/design.md/wise/preview` não ficou acessível; **`https://getdesign.md/design-md/wise/preview` foi acessado com sucesso**. Este último é referência provisória verificada; não é uma correção silenciosa do original.

No [preview consultado](https://getdesign.md/design-md/wise/preview), foram identificados verde primário `#9fe870`, fundo `#e8ebe6`, superfície branca `#ffffff`, texto `#0e0f0c` e corpo secundário `#454745`; Inter é a fonte de substituição indicada. Botões e cartões usam raio de 24 px; botões têm alvos de 48–56 px. O corpo de referência é 16/24 px, com pesos 400 e 600 para conteúdo comum. Esses elementos orientam a proposta visual do Pytende; não se pretende reproduzir seções de marketing ou componentes financeiros do preview.

### 8.3 Aplicação e acessibilidade propostas

**PT-09:** implementar CSS local com tokens em inglês, templates e formulários Django. Usar os elementos verificados como base de cartões, ações e tipografia; confirmar disponibilidade/licença da fonte antes de distribuí-la, com fallback `sans-serif`. Não adicionar framework de frontend.

Formulários terão rótulos persistentes, indicação textual de obrigatórios, erros por campo e resumo quando necessário. Usar botões reais, `fieldset`/`legend` quando pertinente e confirmação explícita para cancelamento com motivo. Estados terão rótulo textual. Foco não pode ficar oculto ou depender apenas de cor. Feedback de salvamento deve ser anunciável por tecnologia assistiva.

Não presumir que toda combinação do preview atende a contraste. Medir texto, badges, bordas, erros e foco no contexto final; ajustar tokens sem perder consistência e documentar ajustes. Navegação deve funcionar sem JavaScript; eventual JS será melhoria progressiva. Em celular, empilhar campos e filtros; manter rótulos e ações acessíveis. DP-01 controla validação final do design e eventuais divergências quando as referências faltantes estiverem disponíveis.

## 9. Modelo conceitual de dados

Os nomes e tipos a seguir são **propostas técnicas** para representar os requisitos confirmados. Limites de texto são salvaguardas propostas, sujeitos a DP-07; não regras já informadas pelo usuário.

### 9.1 Entidades

| Entidade | Campos propostos |
| --- | --- |
| `User` | Modelo nativo Django; identidade, credenciais com hash e flags administrativas; não criar campos de negócio desnecessários |
| `Customer` | `id`, `name` (200), `phone` (32, apresentação), `phone_normalized` (13, índice), `email` (opcional), `notes` (texto opcional), `created_at`, `updated_at` |
| `Ticket` | `id`, `protocol` (UUID único), `customer` (FK), `service_type` (choices), `request_description` (texto), `status` (choices), `equipment` (200), `brand` (100), `model` (100), `serial_number` (100), `estimated_amount`, `final_amount`, `opened_at`, `closed_at` (opcional), `cancellation_reason` (texto opcional condicional), `created_at`, `updated_at` |
| `TicketNote` | `id`, `ticket` (FK), `text`, `author` (FK para `settings.AUTH_USER_MODEL`), `created_at`, `updated_at`; imutável pelas interfaces da aplicação |
| `TicketStatusChange` | `id`, `ticket` (FK), `previous_status`, `new_status`, `changed_by` (FK), `created_at` (instante da mudança), `updated_at`; imutável pelas interfaces da aplicação |

`Customer` possui vários `Ticket`; cada `Ticket` possui várias `TicketNote` e `TicketStatusChange`; cada evento tem um autor. Equipamento é um conjunto de campos opcionais do atendimento, sem entidade de inventário. Tipo de serviço é `TextChoices`, sem tabela de gestão.

Valores internos sugeridos de `service_type`: `computer_formatting`, `system_configuration`, `technical_support`, `website_development`, `other`. Rótulos em português conforme RN-005.

### 9.2 Integridade e temporalidade

- **PT-10:** FKs com `PROTECT` para impedir remoção acidental de cliente com atendimentos e de autores/eventos vinculados; qualquer procedimento futuro de anonimização/exclusão deve avaliar essas restrições e cumprir DP-10, sem exclusão em cascata silenciosa.
- Índices propostos: `phone_normalized`, `opened_at`, composição `status`/`opened_at` e FKs; `protocol` com índice único. Pesquisa por trecho de nome não terá garantia de otimização por índice B-tree comum; avaliar somente se volume real exigir.
- Constraints: valores nulos ou maiores/iguais a zero; tipos/estados válidos; estado final exige `closed_at`, demais estados não têm encerramento; cancelado exige justificativa não vazia após normalização. `closed_at` não deve anteceder `opened_at`.
- `created_at` automático na criação; `updated_at` atualizado em alterações da própria entidade. Em eventos imutáveis, ambos permanecem iguais após criação. Acrescentar uma nota não precisa alterar `updated_at` do `Ticket`; sua data própria registra o acréscimo.
- `opened_at` representa abertura e não é editável. Criar em Aberto não exige uma transição fictícia; o histórico de mudanças começa na primeira alteração, e abertura permanece visível.
- Alteração de estado deve usar `transaction.atomic()` e `select_for_update()`, revalidar o estado atual e gravar evento/encerramento na mesma transação. Erro deve reverter todas as gravações.
- Não confiar somente em `ModelForm` ou `save()`: validações de domínio devem ser chamadas explicitamente nos caminhos de escrita, com constraints como defesa adicional. Escritas em massa que contornem regras não serão disponibilizadas.

## 10. Arquitetura e responsabilidades dos apps

### 10.1 Arquitetura proposta

Monólito Django com templates renderizados no servidor, formulários nativos, ORM e PostgreSQL. Views coordenam requisição, autorização e resposta; formulários validam entradas; modelos definem dados/invariantes; uma função de domínio em `tickets/services.py` concentra transições atômicas. Evitar camadas genéricas, repositórios abstratos e signals para regras centrais.

| Componente | Responsabilidade |
| --- | --- |
| `core` | Projeto principal existente; settings, URLs globais, WSGI/ASGI e infraestrutura compartilhada; sem ser depósito dos modelos de negócio |
| `customers` | Cadastro, correções, pesquisa e detalhes de clientes; apresenta atendimentos relacionados sem duplicar suas regras |
| `tickets` | Atendimento, tipos/estados fixos, transições, observações, histórico, pesquisa e painel por consultas agregadas |
| `django.contrib.auth` e `sessions` | Identidade, autenticação, sessões, validadores de senha e flags de autorização |
| `templates` e `static` | Layout e estilos compartilhados; organização por app para páginas específicas |

Não criar app por modelo: observações e transições pertencem a `tickets`. Não é necessário app `accounts` com usuário customizado, app financeiro ou app exclusivo para o painel simples.

### 10.2 Ambiente observado e versões propostas

Consulta local somente de leitura confirmou pasta `C:\Users\vinic\projects\pytende`, projeto `core`, `.venv`, `.env`, `.gitignore`, `manage.py` e `requirements.txt`. Não havia PRD, README, diretório de design ou arquivos de containers na raiz inspecionada. A listagem Git inicial não mostrou alterações.

`python --version` e `.venv/Scripts/python.exe --version` retornaram **3.13.17**. O Python global não tem Django; `.venv/Scripts/python.exe -m django --version` retornou **6.1.3**, igual ao pin de `requirements.txt`. O launcher `py` informou não encontrar instalações registradas, embora o executável direto funcione. Docker CLI retornou **29.9.2** e Compose **v5.6.1**, com aviso de acesso negado ao arquivo de configuração do Docker do usuário; não foi verificado daemon, execução de containers ou Swarm.

**PT-11:** manter Python **3.13**, no patch de segurança mais recente disponível ao implementar, e Django **6.1.3** como base já instalada, atualizando patches antes da liberação. A [matriz oficial Django](https://docs.djangoproject.com/en/6.1/faq/install/) confirma compatibilidade entre essas séries. A [política Python](https://devguide.python.org/versions/) indica suporte de segurança da série 3.13 até outubro de 2029; não foi validado que 3.13.17 é o patch mais recente.

Segundo a [página de versões Django consultada](https://www.djangoproject.com/download/), 6.1.3 é a versão oficial apresentada, com suporte estendido da série até dezembro de 2027. Para uma manutenção mais longa, 5.2 LTS é alternativa suportada até abril de 2028, mas não se propõe downgrade automático do projeto existente. Planejar atualização de série antes do fim de suporte. A página consultada é mutável e deverá ser conferida novamente na implementação.

**PT-12:** PostgreSQL **17**, com patch vigente e imagem fixada por digest na liberação, como base comum de desenvolvimento e produção. A [política oficial](https://www.postgresql.org/support/versioning/) apresenta suporte até novembro de 2029; a [documentação do backend Django](https://docs.djangoproject.com/en/6.1/ref/databases/#postgresql-notes) será referência para a seleção do driver. Não foi encontrado driver PostgreSQL no `requirements.txt` atual, nem verificado servidor PostgreSQL instalado.

### 10.3 Dependências e configuração propostas

| Dependência/recurso | Justificativa | Situação |
| --- | --- | --- |
| Django | Framework solicitado; ORM, formulários, auth, sessões, templates e testes nativos | Instalado na `.venv` e declarado |
| `psycopg` 3 | Driver necessário para PostgreSQL | Adição futura, versão exata e empacotamento a validar |
| Gunicorn | Servidor WSGI de produção em container Linux; `runserver` só em desenvolvimento | Adição futura; não usar executável Windows para produção |
| WhiteNoise | Proposta para servir estáticos coletados na mesma imagem, evitando serviço adicional | Adição futura, compatibilidade a validar |
| `asgiref`, `sqlparse`, `tzdata` | Dependências já declaradas; manter conforme resolução compatível; `tzdata` atende ambientes sem base de zonas do sistema | Pins existentes não foram auditados individualmente |
| `os.environ`, `decimal`, `uuid`, `zoneinfo` | Biblioteca padrão para configuração, precisão e identificadores/datas | Sem nova dependência |

Não acrescentar `django-environ`, biblioteca de telefone, framework CSS ou ferramenta de testes alternativa sem necessidade demonstrada. Compose pode injetar `.env` via `env_file`; execução fora de containers deverá receber variáveis do shell, sem presumir que Django lê `.env` automaticamente.

Prever `.venv` e `requirements.txt` na raiz, já existentes. Na implementação, fixar versões diretas e transitivas compatíveis no arquivo, validar instalação limpa e construção da imagem. `.env` e `.venv` já são ignorados pelo Git. `.env.example` será criado posteriormente com placeholders, sem credenciais reais. Nunca copiar `.env`, `.venv`, `.git` ou dados locais para a imagem.

### 10.4 Adequações ao código existente, somente planejadas

`core/settings.py` usa SQLite, `LANGUAGE_CODE='en-us'`, `TIME_ZONE='UTC'`, `DEBUG=True`, `ALLOWED_HOSTS=[]` e chave de desenvolvimento literal. Migrar para PostgreSQL, pt-BR, America/Sao_Paulo e segredos do ambiente, removendo chave literal e gerando outra para ambientes reais. Não foi lido o conteúdo de `.env` e não se afirma que contenha configuração válida. A URL atual contém somente Django Admin. Essas observações são diagnóstico, sem alteração de código nesta tarefa.

## 11. Segurança e controle de acesso

Usar `LoginView`, `LogoutView`, middleware de sessões, autenticação e CSRF nativos. Centralizar uma verificação reutilizável do administrador ativo e aplicá-la a todas as views privadas. Visitante em navegação privada vai ao login; usuário autenticado sem autorização recebe 403. Não usar ocultação de links como controle de acesso.

Formulários devem declarar campos explicitamente. Autor, protocolo, estado inicial, datas e vínculo de eventos são definidos no servidor. Transição é ação POST separada da edição comum. Cliente vinculado não é editável enquanto DP-04 não for resolvida. Aplicar a mesma regra ao Django Admin e a qualquer comando de negócio futuro.

Escape padrão dos templates; não aplicar `safe` em conteúdo de cliente. ORM parametrizado; sem SQL por concatenação. Formulários não terão campos de senha de cliente, e áreas de texto devem avisar para não registrar credenciais. Não prometer detecção automática de toda senha digitada em texto livre; conteúdo indevido identificado exigirá tratamento operacional conforme DP-10.

Produção: `DEBUG=False`, hosts explícitos, origens CSRF exatas quando necessárias, cookies de sessão e CSRF seguros e sessão com `HttpOnly`; HTTPS obrigatório. Configurar `SECURE_PROXY_SSL_HEADER` apenas se Traefik for a entrada confiável que sobrescreve o cabeçalho e o acesso direto ao app estiver bloqueado. Aplicar redirecionamento HTTPS e avaliar HSTS após validar TLS para evitar bloqueio operacional. Não expor banco nem painel de infraestrutura publicamente.

A [checklist oficial de deploy](https://docs.djangoproject.com/en/6.1/howto/deployment/checklist/) orienta a revisão de settings e o uso de `check --deploy`. **PT-13:** limitar tentativas de login no proxy, com parâmetros definidos na implantação; o backend nativo não fornece bloqueio automático por tentativas. Documentar recuperação operacional sem introduzir envio de e-mail. Provisionamento por `createsuperuser`; redefinição por `changepassword`; sem senha em scripts, argumentos registrados ou repositório.

Credenciais de banco da aplicação devem ter apenas privilégios necessários. Separar acesso de manutenção/migrations quando viável, proteger host, registry e backups. Não registrar dados pessoais completos em logs. Histórico é rastreabilidade de negócio, não mecanismo de imutabilidade contra alguém com acesso direto ao banco.

## 12. Estratégia de testes

Usar `python manage.py test`, `django.test.TestCase`, `TransactionTestCase` onde houver transações/concorrência e cliente HTTP nativo. A [documentação consultada](https://docs.djangoproject.com/en/6.1/topics/testing/overview/) fundamenta essa opção. Executar testes de banco em PostgreSQL isolado, nunca contra dados reais; credenciais de teste devem permitir criar/destruir somente banco de testes. Não acrescentar pytest por conveniência.

| ID | Cobertura planejada | Rastreabilidade | Sprint |
| --- | --- | --- | --- |
| T-001 | Login válido/inválido, inativo/não admin, saída POST, sessão invalidada e destino externo recusado | RF-001; RN-001/014; RNF-003/004 | S1 |
| T-002 | Campos de cliente, brancos, e-mail, duplicidades permitidas, formatos e normalização de telefone, timestamps | RF-002; RN-002/003/012 | S2 |
| T-003 | Pesquisa nome/telefone, duplicados, vazio, 404 e isolamento do histórico por cliente | RF-003; RN-002/003/014 | S2/S3 |
| T-004 | Abertura, protocolo único/imutável, cliente obrigatório, choices, estado/datas automáticos e opcionais | RF-004; RN-004/005/006/009 | S3 |
| T-005 | Decimal: vazio, zero, negativo, precisão, limites e entrada local; edição comum não altera campos protegidos, inclusive finais | RF-004/005; RN-007/010/012 | S3 |
| T-006 | Busca protocolo/cliente, combinação/limpeza de filtros, estado/data inválidos e limites de dia local | RF-006; RN-006/012; RNF-008/013 | S3 |
| T-007 | Matriz completa 5×5: sete transições válidas e demais recusadas; motivo vazio; finais; rollback por falha; duas conexões concorrentes; reenvio | RF-007; RN-006/008/009; RNF-006 | S4 |
| T-008 | Observação com autor/data, texto vazio, escape de HTML, ausência de editar/excluir, correção por acréscimo e consulta cronológica | RF-008/009; RN-010/011 | S4 |
| T-009 | Histórico origem/destino/autor/data, cancelamento, painel com cinco contagens, lista não final e estados vazios | RF-009/010; RN-006/008/009 | S4/S5 |
| T-010 | Inventário de URLs privadas: GET/POST sem login e com conta sem autorização; CSRF ativo; parâmetros forjados; nenhum bypass pelo admin | RF-001 a RF-010; RN-014; RNF-003/004/005 | S1–S5, incremental |
| T-011 | Revisão manual das telas em tamanhos definidos, teclado, foco, rótulos, erros, contraste e leitura de históricos | RF-001 a RF-010; RNF-001/002 | S2–S5 |
| T-012 | Build limpo, instalação fixa, settings de produção, estáticos, health checks, persistência, restauração isolada e ensaio de rollback | RNF-005/006/007/010/011/012/014 | S6 |

O teste de concorrência deve usar PostgreSQL real e conexões separadas; testes serializados não provam bloqueio. CSRF deve ser testado com `Client(enforce_csrf_checks=True)`. Verificar constraints também por tentativas de gravação fora de formulário onde aplicável.

Os critérios dos RF serão evidências de homologação. A jornada completa da seção 4 deve ser executada com dados sintéticos, incluindo espera, conclusão, cancelamento e novo atendimento para retorno. Não se fixa percentual arbitrário de cobertura; o mínimo é cobrir todas as regras, permissões, validações e fluxos críticos identificados. Nenhum teste da aplicação foi executado nesta etapa documental.

## 13. Deploy, operação, custos e backup

Todo o conteúdo desta seção é planejamento. Não foi provisionada infraestrutura, executado build ou realizado deploy.

### 13.1 Serviços e responsabilidades

| Serviço | Desenvolvimento: Docker Compose | Produção: Docker Swarm |
| --- | --- | --- |
| `app` | Django `runserver`, código local montado, ambiente de desenvolvimento | Imagem imutável, Gunicorn em `core.wsgi:application`, templates e estáticos coletados |
| `db` | PostgreSQL da mesma série de produção, volume nomeado e credenciais locais | Uma instância PostgreSQL com volume persistente e restrição ao nó de dados |
| `traefik` | Opcional; acesso local pode ocorrer diretamente | Entrada HTTP/HTTPS, roteamento por domínio e terminação TLS |
| Tarefa de manutenção | Comando pontual para migrations e testes | Execução única e supervisionada de migrations, sem rodar em todas as réplicas |
| Rotina de backup | Dados sintéticos quando necessário | Agendamento operacional no host ou infraestrutura para dump e cópia externa; não requer Celery |

WhiteNoise é a proposta de entrega de estáticos junto ao app. Não haverá serviço de mídia, filas, cache distribuído ou workers de negócio. Sessões ficam no banco nativo.

### 13.2 Build e publicação da imagem

Na implementação: criar Dockerfile com base Python compatível e fixada, instalar requirements, copiar somente arquivos necessários e executar como usuário sem privilégios. Configurar `.dockerignore` para excluir segredos, `.venv`, `.git`, backups e arquivos locais.

Após testes e revisão, gerar imagem identificada por versão/commit e digest, publicá-la em registry escolhido (DP-11) e registrar o digest aprovado. Os nós Swarm devem ter acesso autenticado se o registry for privado. Não usar apenas `latest` para implantação ou rollback. A publicação será uma atividade futura autorizada separadamente desta documentação.

Executar `collectstatic --noinput` na construção com configuração própria sem credenciais reais nem conexão obrigatória ao banco; não embutir chaves de produção em camadas de imagem. Validar que os caminhos coletados são servidos no container final. Se a implementação exigir coleta em tarefa de release, documentar o artefato resultante e sua versão, sem volume compartilhado mutável entre versões.

### 13.3 Diferenças entre Compose e Swarm

| Aspecto | Compose | Swarm |
| --- | --- | --- |
| Construção | `build` local disponível | Consumir imagem já publicada; `stack deploy` não faz build |
| Código | Bind mount para desenvolvimento | Código na imagem; não montar diretório local de desenvolvimento |
| Configuração | `.env` local e `env_file` conforme Compose | Injetar configuração explicitamente; não presumir auto carregamento/interpolação de `.env` igual ao Compose |
| Início | Pode usar dependência com saúde do banco | Não depender de `depends_on` para ordem/prontidão; app deve tolerar indisponibilidade temporária |
| Rede | Rede local e portas limitadas ao host de desenvolvimento | Overlay privada para banco; rede de borda para app/Traefik |
| Roteamento | Porta local do app | Labels em `deploy.labels` para provider Swarm e porta interna do serviço explícita |
| Operação | Reinício local | Réplicas, placement, limites, update e rollback no bloco `deploy` |

A [documentação Docker consultada](https://docs.docker.com/engine/swarm/stack-deploy/) diferencia o formato de stack do Compose atual e exige distribuição da imagem por registry. Validar o arquivo Swarm separadamente, sem assumir compatibilidade integral por compartilhar extensão YAML. Sem executar esses comandos nesta etapa.

### 13.4 Variáveis e gestão de segredos

Configurações propostas: `DJANGO_SECRET_KEY`, `DJANGO_DEBUG`, `DJANGO_ALLOWED_HOSTS`, `DJANGO_CSRF_TRUSTED_ORIGINS`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_HOST`, `POSTGRES_PORT`, `DOMAIN`, `ACME_EMAIL` e `LOG_LEVEL`. São nomes sugeridos para implementação, não variáveis já existentes verificadas. Idioma/timezone podem ficar fixos no código por serem requisitos.

Desenvolvimento: `.env` local ignorado, permissões locais adequadas e valores exclusivos para desenvolvimento. `.env.example` somente com placeholders como `<set-locally>`; sem senha funcional, token ou chave reutilizável.

Produção: gerenciar segredos fora do repositório e da stack em texto. **PT-14:** usar Docker secrets e adaptador mínimo de inicialização para disponibilizar os valores ao processo via ambiente, atendendo à configuração por variáveis; nunca imprimir valores. O sufixo `_FILE` não é automaticamente interpretado por Django: documentar a leitura explícita, eventual suporte da imagem PostgreSQL e as permissões de acesso. Não colocar segredos em labels ou argumentos de build. Rotação deve criar nova versão do secret, atualizar os serviços e retirar o anterior após verificar recuperação. Definir responsável e armazenamento seguro de credenciais de recuperação.

### 13.5 Domínio, DNS, HTTPS e roteamento

Domínio/subdomínio e provedor pendentes em DP-11; não presumir endereço da ViniTech. Apontar registros A e, se houver IPv6 operacional, AAAA para a entrada aprovada. Liberar 80/443 somente conforme estratégia TLS e SSH restrito para manutenção. Banco e Gunicorn não terão portas públicas.

Traefik deverá usar provider Swarm compatível com a versão escolhida, descoberta opt-in e regra exata de host, roteando para porta interna do app. Proposta simples: certificado ACME para um único host via HTTP-01 quando as condições de rede permitirem; não exigir wildcard, Cloudflare ou token DNS por copiar a referência SCSI. Persistir estado ACME com acesso restrito; proteger o acesso do proxy ao Docker API, preferindo acesso mínimo e documentando que socket Docker é sensível mesmo montado como somente leitura.

Validar emissão/renovação TLS, redirecionamento HTTP, `ALLOWED_HOSTS`, origens CSRF e cabeçalhos de proxy. Painel Traefik não é requisito e pode permanecer desabilitado. Definir domínio de health check sem ampliar hosts para curingas.

### 13.6 Banco, migrations e primeira liberação

Volume persistente dedicado ao PostgreSQL; uma réplica e placement no nó que possui o volume local. Swarm não transforma volume local em armazenamento replicado. Se o nó falhar, não permitir realocação sobre volume vazio aparentando banco novo. Definir disco, permissões e procedimento de recuperação. Desenvolvimento e produção usam bancos e credenciais distintos.

Sequência proposta: verificar backup recuperável → disponibilizar imagem/configurações → verificar banco pronto → suspender escritas se a migration exigir → executar `migrate` uma única vez → confirmar sucesso → atualizar app → validar estáticos, login, leitura e uma gravação sintética controlada → liberar uso. Primeira instalação inclui `createsuperuser` em sessão administrativa segura; nunca senha padrão na imagem. Manter um registro da versão do schema e imagem.

Não rodar migrations automaticamente em todas as réplicas de entrada. Mudança de versão major do PostgreSQL exige plano próprio, sem substituir a imagem e reutilizar cegamente o diretório de dados; a [política PostgreSQL](https://www.postgresql.org/support/versioning/) explica a necessidade de atualização de dados compatível.

### 13.7 Health checks e logs

Proposta: endpoint interno `/health/` com resposta mínima, sem dados ou versões, acessível apenas pela infraestrutura; consultar prontidão do banco e retornar erro quando indisponível. A exceção técnica ao login não libera páginas de negócio. Container check com `Host` válido, timeout e tolerância inicial; `pg_isready` no banco, complementado pela verificação de conexão real da aplicação. Testar comportamento quando banco fica indisponível para evitar ciclos de reinício sem diagnóstico.

Enviar logs de app/Gunicorn/Traefik para stdout/stderr com rotação e limite de armazenamento no coletor/host. Registrar nível, horário, erro e identificador técnico sem corpos de formulários, query strings pessoais, credenciais, cookies ou tokens. Monitorar disco, falhas de backup, indisponibilidade e validade TLS por verificação operacional; automações externas de monitoramento são opcionais, sem criar notificações de negócio nesta versão.

### 13.8 Backup, retenção, restauração e teste de recuperação

Backup obrigatório do PostgreSQL e das configurações necessárias à recuperação; segredos em cofre/armazenamento protegido separado. Não há anexos de clientes a copiar. Usar dump consistente, preferencialmente formato custom do `pg_dump`, com versão de ferramentas compatível; registrar imagem do app, schema e momento do backup. Copiar para armazenamento externo ao host, criptografado, com acesso restrito e verificação de integridade. Snapshot isolado no mesmo host não é recuperação suficiente.

**DP-08 e DP-10 devem ser resolvidas antes de dados reais:** periodicidade, janela de retenção, rotação, localização, RPO máximo de perda aceitável, RTO de recuperação e tratamento de dados excluídos em backups. Não há prazos de retenção aprovados neste documento. Enquanto essas decisões não existirem, o ambiente serve somente para homologação com dados sintéticos.

Procedimento de restauração planejado:

1. Selecionar cópia e versão compatíveis, verificar integridade e obter segredos de forma segura.
2. Restaurar em banco isolado vazio, impedindo acesso público e gravações no banco original.
3. Validar schema, contagens, relações cliente/atendimento, protocolos, eventos, valores e datas.
4. Executar testes de leitura e jornada controlada com aplicação compatível; verificar login sem divulgar credenciais.
5. Medir tempo de restauração e perda potencial, comparar com RPO/RTO aprovados e registrar resultado.
6. Definir corte de tráfego, eventuais escritas divergentes e autorização operacional antes de substituir produção.
7. Descartar ambiente de ensaio e cópias temporárias de acordo com a política aprovada.

Realizar ensaio antes da primeira liberação e definir recorrência em DP-08; repetir após mudanças relevantes de banco/backup. Manter procedimento de reaplicação de exclusões/anonimizações aprovadas após recuperação para evitar ressuscitar dados tratados. Backup só é considerado validado após restauração e conferência funcional, não apenas pela existência de um arquivo.

### 13.9 Atualização e rollback

Registrar imagem anterior, backup validado e compatibilidade do schema. Fazer atualização com paralelismo um e health check; escolher `start-first` somente se houver memória e compatibilidade entre versões. Para o cenário inicial de um nó/uma réplica, janela de manutenção curta pode ser mais simples; não prometer zero downtime.

Rollback Swarm reverte imagem/configuração, **não desfaz migrations nem recupera dados**. Planejar migrations compatíveis em etapas, adiando remoções de campos. Se schema não aceitar a imagem anterior, preferir correção adiante ou reversão de migration testada quando reversível. Restauração de backup é último recurso e pode perder escritas posteriores; documentar conciliação e aprovação do responsável. Testar o procedimento antes de declarar operação pronta.

### 13.10 Dimensionamento inicial e desempenho

**H-01:** um administrador, até três abas, até 10 mil clientes e 50 mil atendimentos, sem anexos; volumes são premissas para avaliação, não expectativa confirmada. **PT-15:** começar com um host Linux de 2 vCPU, 4 GiB RAM e 40 GiB SSD, uma réplica do app, dois workers Gunicorn, uma instância de banco e uma de Traefik. Um nó Swarm é compatível com a proposta, mas não fornece alta disponibilidade. Backup externo é indispensável; espaço deve ser revisto com banco, logs e imagens reais.

Orçamento inicial de limites para ensaio: app até 1 GiB, banco até 1,5 GiB e proxy até 256 MiB, preservando memória para SO e operações; medir antes de fixar. Atualizações podem exigir parada ou memória adicional. Não copiar as réplicas/workers da referência SCSI sem carga equivalente.

Meta proposta, pendente em DP-09: percentil 95 de até 2 s para listagem paginada e detalhes, medidos no servidor, sem latência de internet, com dados sintéticos do volume H-01, um usuário ativo, índices definidos e host PT-15. Registrar método, consultas e resultado; não é SLA nem desempenho comprovado. Crescer recursos ou revisar consultas somente com evidência.

### 13.11 Custos obrigatórios e opcionais

Não foram pesquisados preços comerciais. Os intervalos abaixo são **estimativas não verificadas em 07/11/2026**, em reais brasileiros, para um host pequeno, um domínio e baixo volume. Não são cotações; dependem de provedor, impostos, câmbio, região, tráfego e infraestrutura já disponível. Confirmar em DP-11 antes de contratar.

| Item | Natureza | Estimativa não verificada | Premissa |
| --- | --- | --- | --- |
| Servidor de produção | Obrigatório como recurso; pode ser existente | R$ 70–200/mês | Um host PT-15, operação própria, sem alta disponibilidade |
| Domínio | Obrigatório para o HTTPS público proposto; subdomínio pode aproveitar domínio existente | R$ 40–100/ano | Um domínio, renovação não cotada; custo incremental pode ser zero se existente |
| Backup externo | Obrigatório | R$ 10–50/mês | Baixo volume, retenção ainda a aprovar; armazenamento e recuperação variam |
| Registry de imagens | Obrigatório como capacidade | R$ 0–50/mês | Pode usar cota gratuita; privacidade e limites pendentes |
| Licenças da stack proposta | Sem contratação paga prevista | R$ 0/mês previsto | Validar termos das edições, imagens e uso do Docker no ambiente da organização |
| DNS/TLS automatizado | Obrigatório como capacidade | R$ 0/mês incremental previsto | Recursos incluídos/gratuitos a validar; não inclui mão de obra |
| Monitoramento externo pago | Opcional | R$ 0–100/mês | Plano gratuito ou pequeno serviço; sem integração de negócio |
| Banco gerenciado ou nós adicionais | Opcional, fora da topologia inicial | A cotar em BRL/mês | Somente se operação/capacidade justificar; sem faixa inventada |
| Implantação, manutenção e suporte | Esforço obrigatório; contratação opcional | A dimensionar em horas e, se contratado, BRL/hora | Disponibilidade e valor/hora desconhecidos |

Faixa de infraestrutura recorrente obrigatória estimada: **R$ 80–300/mês**, mais **R$ 40–100/ano** de domínio, somando host, backup e registry nas premissas acima. Não inclui mão de obra, excedentes, recuperação cobrada ou opcionais. Infraestrutura existente pode reduzir custo incremental, mas deve ter capacidade e responsabilidade definidas. Frequência/retenção maiores podem invalidar a faixa de backup.

## 14. Sprints pequenas

Sequência de entregas, sem prazo fechado. A duração depende da disponibilidade do desenvolvedor. Cada sprint inclui seus testes e atualização do README durante a implementação; nenhum README é criado nesta etapa. Sprints só concluem quando os critérios indicados forem demonstrados.

### S1 — Base, ambiente e autenticação

**Objetivo:** tornar executável a base segura para um administrador. **Dependências:** validação de versões e acesso ao ambiente de desenvolvimento.

- [ ] Configurar PostgreSQL via Compose e variáveis; manter `.venv`/requirements na raiz.
- [ ] Adequar `core`, idioma, timezone, segredos e configurações por ambiente; preparar `.env.example` com placeholders.
- [ ] Implementar login/logout e autorização central; provisionar conta com mecanismo nativo.
- [ ] Executar T-001 e início de T-010 com usuário sem autorização como fixture de teste, sem criar novo perfil de produto.
- [ ] Documentar configuração, criação administrativa, execução e testes no README.

**Conclusão:** RF-001 e RNF-003/004/007/008 demonstrados em ambiente PostgreSQL; nenhuma página privada disponível sem autorização.

### S2 — Clientes

**Objetivo:** cadastrar, corrigir e localizar clientes. **Dependências:** S1; PT-04 adotada ou DP-02 resolvida.

- [ ] Criar domínio `customers`, migrations e formulários confirmados.
- [ ] Implementar pesquisa, detalhes, duplicidades permitidas e normalização do telefone.
- [ ] Aplicar layout provisório verificado com rótulos, erros e teclado; registrar DP-01.
- [ ] Executar T-002, T-003 sem integração ainda indisponível, T-010 e T-011 correspondentes.
- [ ] Documentar campos, formatos aceitos e execução dos testes.

**Conclusão:** RF-002 e parte cadastral de RF-003 aceitos; histórico preparado para integração em S3, sem simular atendimentos inexistentes como funcionalidade entregue.

### S3 — Atendimento, valores e consultas

**Objetivo:** abrir e corrigir atendimentos e consultar listas/histórico do cliente. **Dependências:** S2; decisões de protocolo e valores resolvidas ou propostas explicitamente adotadas.

- [ ] Criar `tickets`, campos, choices, constraints, protocolo e migrations.
- [ ] Implementar abertura, edição com campos restritos, pesquisas e filtros.
- [ ] Integrar histórico de atendimentos no cliente e navegação entre detalhes.
- [ ] Executar T-003 completo, T-004, T-005, T-006, T-010 e T-011 pertinentes.
- [ ] Documentar tipos fixos, opcionais, limites e consultas no README.

**Conclusão:** RF-003 a RF-006 demonstrados; estado inicial e campos automáticos protegidos; sem transições por edição comum.

### S4 — Estados, observações e histórico

**Objetivo:** acompanhar e encerrar com integridade. **Dependências:** S3.

- [ ] Implementar serviço atômico de transição e controle de concorrência.
- [ ] Implementar cancelamento justificado e encerramento automático.
- [ ] Implementar observações somente por acréscimo e visualização de eventos.
- [ ] Garantir ausência de bypass de exclusão/edição pela interface e Django Admin.
- [ ] Executar T-007, T-008, parte histórica de T-009, T-010 e T-011 pertinentes.
- [ ] Documentar matriz de estados, correções e limitações de auditoria.

**Conclusão:** RF-007 a RF-009 aceitos, sete transições permitidas e demais recusadas; falha e concorrência não deixam estado/eventos inconsistentes.

### S5 — Painel e homologação funcional

**Objetivo:** consolidar jornada e interface. **Dependências:** S4; resolver/adotar interpretação de abertos e validar design em DP-01.

- [ ] Implementar contagens e lista recente sem indicadores financeiros.
- [ ] Revisar todas as telas, vazios, erros, navegação e responsividade.
- [ ] Executar T-009, T-010 completo, T-011 e jornada da seção 4 com dados sintéticos.
- [ ] Medir consultas no volume acordado se DP-09 tiver meta aprovada.
- [ ] Atualizar documentação de uso, regras e testes; conferir rastreabilidade RF/RN.

**Conclusão:** RF-001 a RF-010 homologados; referências visuais e ajustes acessíveis documentados; nenhuma funcionalidade fora do escopo introduzida.

### S6 — Preparação operacional e liberação futura

**Objetivo:** tornar a implantação revisável e recuperável. **Dependências:** S5; decisões de privacidade, backup, infraestrutura e credenciais concluídas antes de dados reais.

- [ ] Criar Dockerfile, Compose, stack Swarm e documentação de configuração/release, sem segredos nos artefatos.
- [ ] Validar versões de imagens e dependências, estáticos e settings de produção.
- [ ] Preparar build/registry, domínio/TLS, health checks, logs e runbook de migrations.
- [ ] Executar T-012 em homologação e ensaiar backup/restauração e rollback com dados sintéticos.
- [ ] Aprovar RPO/RTO, política de dados, retenção, custos e responsáveis.
- [ ] Atualizar README com configuração, execução, testes e procedimentos operacionais.

**Conclusão:** RNF e testes operacionais demonstrados, decisões bloqueadoras resolvidas e checklist de liberação preenchido. Deploy real depende de autorização futura e não é realizado pela elaboração deste PRD.

## 15. Riscos, hipóteses e decisões pendentes

### 15.1 Hipóteses

| ID | Hipótese | Validação |
| --- | --- | --- |
| H-01 | Carga pequena descrita na seção 13.10 | Confirmar volume, crescimento e uso real antes de dimensionar definitivamente |
| H-02 | Um único host e janelas de manutenção são aceitáveis | Responsável deve aceitar disponibilidade limitada antes de produção |
| H-03 | Containers Linux serão o alvo de produção, embora o workspace seja Windows | Validar host, operação do Docker e construção multiplataforma se aplicável |

### 15.2 Decisões pendentes

| ID | Decisão | Proposta ou limite provisório | Responsável/momento |
| --- | --- | --- | --- |
| DP-01 | Validar design system local/documento remoto e comparar com preview | Preview válido como base provisória; nenhuma divergência afirmada sem acesso; medir contraste e confirmar fonte | Responsável e implementação, até homologação visual S5 |
| DP-02 | Extensão da validação de telefone | PT-04 cobre DDD + 8/9 dígitos e `+55`; operadora, ramais e outros formatos não presumidos | Responsável, S2 |
| DP-03 | Formato visível do protocolo | UUID integral em PT-03; confirmar conveniência operacional antes de persistir esquema definitivo | Responsável, S3 |
| DP-04 | Troca de cliente e limites de correção em finais | PT-05 bloqueia troca de vínculo; permite correções descritivas/valores e acréscimos em finais sem reabrir | Responsável, S3/S4 |
| DP-05 | Significado de abertos no painel e quantidade recente | PT-07: três estados não finais, dez registros | Responsável, S5 |
| DP-06 | Moeda, precisão e teto de valores | PT-08: BRL, duas casas e 12 dígitos totais; sem regra financeiro-contábil | Responsável, S3 |
| DP-07 | Limites de campos textuais | Tamanhos propostos para campos curtos; definir teto seguro para textos longos sem truncamento silencioso | Implementação e responsável, S2–S4 |
| DP-08 | Frequência/retenção de backup, RPO/RTO e recorrência do ensaio | Sem prazo aprovado; definir armazenamento externo, recuperação e responsável | Responsável operacional, bloqueia dados reais |
| DP-09 | Volume real e metas de desempenho/disponibilidade | H-01/PT-15 e meta proposta não comprovados | Responsável, até S6 |
| DP-10 | Política de retenção, anonimização e exclusão de dados pessoais | Definir tratamento de banco, textos livres, logs, backups e restauração; procedimento restrito fora da interface, sem módulo novo presumido | Responsável pela ViniTech, bloqueia dados reais |
| DP-11 | Host, domínio, registry, TLS, versões de infraestrutura e custos reais | Stack mínima; estimativas sem cotação não autorizam contratação | Responsável operacional, antes da implantação |
| DP-12 | Atualização de patches e dependências de produção | Confirmar patch Python mais recente, driver/Gunicorn/WhiteNoise compatíveis, imagens e agenda de upgrade Django antes de dezembro de 2027 | Implementação/manutenção, S1 e novamente S6 |

As lacunas acima não impedem a elaboração do PRD. Propostas são valores de trabalho para planejamento, não aprovação silenciosa de políticas ou autorização para uso de dados reais. Não foram feitas perguntas adicionais porque não existe contradição que impeça documentar a versão.

### 15.3 Riscos e mitigação

- **Dados sensíveis em texto livre:** orientar a não incluir senhas, limitar coleta e definir tratamento de registros indevidos; não prometer reconhecimento automático de segredos.
- **Perda de integridade por caminhos alternativos:** centralizar transição, usar transação/constraints e testar acesso direto e admin.
- **Concorrência entre abas:** bloquear linha, revalidar origem e recusar ação desatualizada; não é necessária colaboração multiusuário.
- **Host único e volume local:** indisponibilidade/perda do host exigem restauração; backup externo e ensaio são critérios de liberação.
- **Rollback com schema incompatível:** migrations em etapas, backup e teste; rollback de container não reverte dados.
- **Referências incompletas ou mutáveis:** registrar links, arquivos consultados e limitações; validar novamente antes de implementação.
- **Suporte de versões:** aplicar patches e planejar evolução antes do fim de suporte, sem instalar ou alterar dependências nesta entrega.
- **Retenção indefinida por omissão:** bloquear dados reais até política aprovada, abrangendo backups e textos livres.

## 16. Referências consultadas e limitações

Todas as consultas abaixo ocorreram em 07/11/2026. Conteúdo externo foi tratado como referência, sem substituir o escopo definido pelo responsável. Respostas web podem vir de cache; links para `main` não representam snapshot imutável verificado.

### 16.1 Workspace

Pasta confirmada por `Get-Location` e raiz por `git rev-parse --show-toplevel`: `C:\Users\vinic\projects\pytende`. Inspeção somente de leitura com listagem da raiz, `rg --files`, `git ls-files`, status e leitura dos arquivos relevantes.

Arquivos efetivamente lidos: `requirements.txt`, `.gitignore`, `manage.py`, `core/settings.py`, `core/urls.py`, `core/asgi.py`, `core/wsgi.py` e `.venv/pyvenv.cfg`. `.env` teve apenas existência observada, sem leitura do conteúdo. Não foi encontrado `AGENTS.md` na enumeração do projeto nem nos caminhos ancestrais consultados. Não foi consultado conteúdo de arquivos inexistentes.

Comandos de versão foram descritos na seção 10. Não foram executados `pip install`, testes da aplicação, migrations, comandos de provisionamento, build, publicação ou deploy. Docker CLI disponível não comprova Docker daemon operacional; houve aviso de acesso negado ao arquivo de configuração do usuário.

### 16.2 Repositório SCSI, branch `main`

A [árvore de arquivos](https://github.com/pycodebr/scsi/tree/main) foi consultada. Foram lidos os seguintes arquivos:

| Arquivo consultado | Observação útil para o Pytende |
| --- | --- |
| [Dockerfile](https://raw.githubusercontent.com/pycodebr/scsi/main/Dockerfile) | Estrutura de imagem Python e instalação de requirements como referência; não copiar instalações/entrypoints sem necessidade |
| [docker-compose.yml](https://raw.githubusercontent.com/pycodebr/scsi/main/docker-compose.yml) | Separação app/banco, volume e health check como referência; filas e workers excluídos |
| [docker-stack.yml](https://raw.githubusercontent.com/pycodebr/scsi/main/docker-stack.yml) | Exemplo de Traefik, labels de serviço, saúde e atualização; dimensionamento e estratégia DNS próprios do SCSI não adotados |
| [core/settings.py](https://raw.githubusercontent.com/pycodebr/scsi/main/core/settings.py) | Organização de settings e configuração por ambiente; não adotar tenant, backend de usuário customizado, fallback SQLite ou segredo padrão |
| [mkdocs.yml](https://raw.githubusercontent.com/pycodebr/scsi/main/mkdocs.yml) | Referência de organização da documentação operacional; README basta para a primeira versão, sem dependência MkDocs |

Síntese arquitetural: aproveitar separação por domínios e documentação operacional; reduzir a stack ao necessário. Não copiar entidades, regras de seguros, integrações ou funcionalidades do SCSI.

Tentativas de leitura de `docs/architecture.md` e `docs/deploy.md` falharam; seus conteúdos não foram analisados. A API de commit e a página de histórico de `main` também não ficaram acessíveis. **Commit analisado: não determinado**; não afirmar que os arquivos acessados em endpoints mutáveis correspondem ao mesmo commit. Registrar SHA e revisar os arquivos nesse snapshot quando a referência for reutilizada na implementação.

### 16.3 Design

- Local `design_system/design-system.html`: ausente na inspeção.
- [DESIGN.MD informado](https://getdesign.md/design-md/wise/DESIGN.MD): tentativa de acesso sem conteúdo disponível.
- Preview original com `httos` e `priveview`: não usado como fonte válida.
- Candidato `https://getdesign.md/design.md/wise/preview`: tentativa sem conteúdo disponível.
- [Preview verificado](https://getdesign.md/design-md/wise/preview): conteúdo textual da página consultado; fundamenta somente os elementos registrados na seção 8. Não houve inspeção visual por screenshot ou teste de contraste do site.

### 16.4 Documentação técnica oficial

- [Django: downloads e suporte](https://www.djangoproject.com/download/): versões e suporte da série.
- [Django 6.1: compatibilidade com Python](https://docs.djangoproject.com/en/6.1/faq/install/): compatibilidade das séries propostas.
- [Python: estado de suporte](https://devguide.python.org/versions/): ciclo de segurança da série 3.13; patch mais recente não confirmado.
- [Django: bancos de dados](https://docs.djangoproject.com/en/6.1/ref/databases/#postgresql-notes): referência do backend PostgreSQL; driver exato a validar na implementação.
- [PostgreSQL: política de versões](https://www.postgresql.org/support/versioning/): suporte e cuidados de atualização de série major.
- [Django: checklist de deploy](https://docs.djangoproject.com/en/6.1/howto/deployment/checklist/): configurações de segurança e verificação de produção.
- [Django: testes](https://docs.djangoproject.com/en/6.1/topics/testing/overview/): runner e recursos nativos.
- [Docker: implantação de stack Swarm](https://docs.docker.com/engine/swarm/stack-deploy/): formato de stack, imagem publicada e registry.

Não foram pesquisados preços, provedores comerciais ou políticas jurídicas de retenção. Não foi verificada vulnerabilidade individual de todas as dependências. Recomendações de operação detalhadas são propostas de arquitetura, não afirmações de homologação das ferramentas e versões ainda não escolhidas.

### 16.5 Fechamento e alcance das evidências

O fechamento deste documento utiliza exclusivamente os requisitos recebidos e as informações já coletadas; não houve repetição das consultas externas. Versões locais citadas representam as saídas dos comandos executados na inspeção, e não uma certificação de origem, integridade ou atualização dos pacotes. As informações de compatibilidade e suporte representam o conteúdo retornado pelas páginas oficiais consultadas naquela ocasião, sujeito a cache e alteração posterior. Não se afirma que o patch Python instalado seja o mais recente, que as dependências futuras estejam homologadas ou que a infraestrutura esteja operacional. Essas verificações permanecem em DP-12 e nos critérios futuros de liberação.

Referências inacessíveis, ausência do design system local, commit SCSI não determinado, daemon Docker não verificado e preços não pesquisados permanecem limitações explícitas. Não impedem a conclusão documental; a validação visual, operacional e de versões deverá ocorrer nas sprints correspondentes. A política de dados pessoais e o plano aprovado de recuperação continuam obrigatórios antes do uso com dados reais.

## 17. Checklist de conclusão

### 17.1 Entrega deste PRD

- [x] Pasta de trabalho confirmada antes da escrita.
- [x] Estrutura e arquivos existentes inspecionados em modo somente leitura.
- [x] Primeira versão limitada às definições funcionais recebidas.
- [x] Requisitos confirmados separados de propostas, hipóteses e decisões pendentes.
- [x] Perfil único e matriz de permissões definidos, com autorização no servidor.
- [x] RF com descrição, perfil, fluxo, exceções, regras e critérios verificáveis.
- [x] Estados finais, sete transições, cancelamento e históricos consistentes.
- [x] Rastreabilidade entre RF, regras, testes planejados e sprints.
- [x] Modelo conceitual, apps coesos e dependências justificadas.
- [x] Ambiente consultado e compatibilidade de séries confrontada com fontes oficiais.
- [x] Referências efetivamente acessadas e limitações de design/commit registradas.
- [x] Plano Compose/Swarm, segredos, backup, recuperação, custos e rollback incluídos.
- [x] Sprints pequenas com testes, documentação e critérios de conclusão, sem prazos inventados.
- [x] Política de dados e retenção tratada como pendência antes de dados reais.
- [x] Nenhuma implementação de aplicação, alteração de README, migration, instalação, commit, push ou deploy realizada por esta entrega.

### 17.2 Critérios futuros de liberação da primeira versão

- [ ] RF-001 a RF-010 e RN/RNF homologados com evidências.
- [ ] T-001 a T-012 concluídos, com falhas relevantes corrigidas.
- [ ] Design e acessibilidade validados em todas as telas.
- [ ] Dependências/patches e configurações de produção revisados.
- [ ] Política de retenção, anonimização e exclusão aprovada antes de dados reais.
- [ ] RPO/RTO, backup externo e retenção aprovados; restauração testada e registrada.
- [ ] Imagem, schema, migrations e rollback ensaiados em homologação.
- [ ] Domínio, TLS, custos e responsabilidades operacionais definidos.
- [ ] README atualizado durante a implementação, com configuração, execução e testes.
- [ ] Autorização de implantação obtida em etapa futura.
