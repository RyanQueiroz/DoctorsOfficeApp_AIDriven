# Relatório de Modernização: DoctorsOfficeApp (Consultório, POC)

Branch `refactoring`, outubro de 2026. Backend ASP.NET Core 8 + EF Core 8 + MongoDB; frontend Angular 19.

Este relatório consolida a auditoria técnica e a refatoração estruturada do sistema implementado nas aulas anteriores. O objetivo não foi adicionar funcionalidade nem corrigir defeitos, e sim deixar o código sustentável: menos repetição, responsabilidades no lugar certo, sintaxe atual da plataforma e uma rede de testes que prova que o comportamento observável não mudou.

A estrutura segue as três fases pedidas: blindagem (testes de caracterização e mapeamento da dívida), execução (transformações assistidas, uma por commit) e validação (paridade funcional, auditoria de segurança, revisão independente e uma rodada de correções a partir dela).

## Sumário executivo

| Medida | Antes (`main`, 8ff4b1d) | Depois (`refactoring`) | Variação |
| --- | ---: | ---: | ---: |
| Linhas de código do backend (sem Migrations) | 4 267 | 3 405 | −20 % |
| Controllers (linhas) | 719 | 459 | −36 % |
| Services (linhas) | 2 069 | 1 249 | −40 % |
| Data: contexto, seed e consultas (linhas) | 594 | 480 | −19 % |
| DTOs (linhas; agora incluem o mapeador `Mapeamentos.cs`) | 274 | 425 | +55 % |
| `Dominio/` + `Infraestrutura/` (novos: exceções, `UsuarioAtual`, papéis, opções, filtro, armazenamento) | 0 | 163 | novo |
| Maior arquivo C# | 596 (`DomainControllers.cs`) | 344 (`ProntuarioService.cs`) | −42 % |
| Maior método | 421 (`SeedAsync`) | 249 (`SeedAsync`, só dados) | −41 % |
| Métodos com mais de 60 linhas | 7 | 2 | −71 % |
| `TriagemChatService` (+ `RoteiroTriagem`) | 493 | 291 | −41 % |
| `DbInitializer.cs` | 434 | 265 | −39 % |
| `catch` em controllers (os 2 restantes são as sondas do health check) | 29 | 2 | −93 % |
| DTOs montados à mão fora do mapeador | 33 | 3 | −91 % |
| `.Include(` em serviços | 58 | 27 | −53 % |
| Buscas no banco só para descobrir quem é o solicitante | 17 | 0 | −100 % |
| Literais `new MensagemMongo { ... }` | 14 | 0 | −100 % |
| Exceções da BCL usadas como sinal de regra de negócio | 69 | 0 | −100 % |
| Leituras de `IConfiguration[...]` e `GetConnectionString` | 16 | 3 | −81 % |
| `[Authorize(Roles = "...")]` com string solta | 18 | 0 | −100 % |
| DTOs de entrada mutáveis (`class` com `set`) | 12 | 0 | −100 % |
| Pacotes NuGet | 9 | 7 | −22 % |
| Artefatos de build versionados | 141 (37 MB) | 0 | −100 % |
| Frontend TypeScript (`src/app`) | 2 961 | 2 978 | +1 % |
| Template e CSS mortos do `AppComponent` | 336 | 0 | −100 % |
| Usos de `any` no frontend | 11 | 0 | −100 % |
| Testes automatizados | 0 | 59 (45 HTTP + 14 unitários) | novo |
| Linhas de teste | 0 | 1 616 | novo |
| Documentação (`.ai`, README, `docs/refactoring`) | 183 | 2 562 | +1 300 % |

Como ler: o código de produção do backend encolheu 20 % no total, com os serviços (−40 %) e os controllers (−36 %) concentrando o ganho. `DTOs/` cresceu porque passou a hospedar o mapeador que antes estava copiado dentro dos serviços, e `Dominio/` e `Infraestrutura/` são 163 linhas novas que substituem repetição espalhada. O que cresceu de verdade foi o que não existia: testes e documentação.

Diff da branch no backend (sem Migrations): 185 arquivos, 1 753 inserções, 8 027 remoções (as remoções incluem os 141 artefatos de build). Todos os 19 commits passam na suíte de caracterização que foi gravada contra o código original antes de qualquer mudança estrutural. As mudanças deliberadas de contrato estão na seção 5.

## 1. Fase de pré-refatoração (blindagem)

### 1.1 Baseline de testes de caracterização (Test-Snapshotting)

O projeto não tinha nenhum teste. Antes de mover qualquer estrutura, foi criado `backend/Consultorio.Api.Tests`, que congela o comportamento observável pela borda HTTP:

- `WebApplicationFactory<Program>` sobe a API real, com as migrations e o seed do `DbInitializer`.
- Testcontainers sobe PostgreSQL 16 e MongoDB 7 reais (as mesmas imagens do `docker-compose.yml`), então `FromSqlRaw ... FOR UPDATE`, índices parciais e a coleção do Mongo são exercitados de verdade.
- O baseline inicial tinha 34 testes cobrindo os 36 endpoints: login e `me`, catálogo público, pacientes, histórico e documentos, autorizações de compartilhamento, o roteiro completo do chat de triagem, fila e validação do enfermeiro, agendamento com os dois casos de 409, transições de status, evolução e conclusão, notificações, pesquisa de satisfação e dashboard por instituição e global. A matriz de RBAC é verificada por perfil em cada recurso.
- Comportamentos peculiares foram congelados de propósito, com comentário no teste: histórico devolve `temPesquisaRespondida=false` mesmo com pesquisa respondida (falta um `Include`), a fila do Admin sem instituição vem vazia, o Enfermeiro recebe histórico sem consultas nem documentos e baixa qualquer documento, e nove rotas respondiam 500 para erros de regra.

A suíte foi validada por mutação antes de ser usada: trocar o texto que decide o 409 no controller derrubou o cenário ponta a ponta com `Expected: Conflict, Actual: BadRequest`. Durante a execução ela cresceu para 45 testes de integração e 14 unitários do roteiro puro (9 ms, sem containers). A revisão independente (seção 3.4) rodou a suíte final contra o código original: 33 de 37 passaram, e as 4 falhas são exatamente as mudanças de contrato declaradas, o que prova que os testes acrescentados descrevem o comportamento antigo e não foram moldados ao código novo.

Prompt usado para gerar a rede de segurança (resumido): *"Antes de qualquer refatoração, escreva testes de integração HTTP que congelem o comportamento atual de todos os endpoints, usando WebApplicationFactory e Testcontainers com PostgreSQL e MongoDB reais. Capture status, corpo `{ mensagem }`, RBAC por perfil e o roteiro determinístico do chat. Se o comportamento atual parecer errado, congele-o mesmo assim e marque com comentário; não corrija nada. Depois prove que o pin morde com uma mutação deliberada."*

### 1.2 Mapeamento de acoplamento e dívida técnica

Um agente de IA fez a auditoria do código original no padrão "thermo-nuclear code quality review" (leitura integral, medição por comandos reproduzíveis, foco em reestruturações que apagam categorias de complexidade). O resultado completo está em `docs/refactoring/diagnostico/` (`00_resumo.md` e cinco arquivos de detalhe com os comandos e as evidências `arquivo:linha`).

Prompt usado (resumido): *"Aja como um Arquiteto de Software. Analise o projeto inteiro (backend ASP.NET Core e frontend Angular) e gere um relatório de dívida técnica focado em violações de SOLID e acoplamento excessivo. Liste code smells, violações de padrão e obsolescência com evidência arquivo:linha e contagens obtidas por grep e wc. Proponha as reestruturações que eliminam categorias inteiras de complexidade sem mudar comportamento. Separe, em seção própria, os bugs observados que não devem ser corrigidos nesta refatoração."*

Achados principais, nas três categorias pedidas:

**Code smells**

- Política de erro HTTP copiada 27 vezes nos controllers (16 cópias literais de `StatusCode(403, new { mensagem = ex.Message })`), com o 409 decidido por `ex.Message.Contains("já foi reservado")`.
- Nove DTOs montados à mão em 24 pontos; só `ConsultaDto` tinha quatro cópias do mesmo construtor posicional, uma delas com fallbacks como `?? "Médico"` que nunca disparavam.
- `ProcessarMensagemAsync` com 221 linhas, 167 delas num `switch` com 14 literais `new MensagemMongo { Id = Guid.NewGuid().ToString(), Remetente = "Bot", ... }` e duas perguntas duplicadas palavra por palavra.
- `DbInitializer.SeedAsync` com 422 linhas num único método, nove usuários criados pelo mesmo bloco de três passos com o `IdentityResult` descartado, e os horários gerados por quatro literais quase iguais.
- Código morto: `AppDbContext` injetado e não usado em `ColetasController`, `EnviarMensagemChatDto.EtapaEsperada`, `ConversaMongo.VersaoRoteiro` e `ConversaMongo.Status` (gravados, nunca lidos), parâmetro `medicoId` aceito e ignorado em `ListarConsultasAsync`, `.http` de template apontando para `/weatherforecast`, dois pacotes NuGet sem nenhuma referência, `app.component.html` de 336 linhas gerado pelo CLI e nunca usado.
- N+1: uma consulta ao Mongo por coleta na fila do enfermeiro, uma consulta por usuário na lista do admin, quatro `COUNT` iguais mudando só o status no dashboard.

**Violação de padrões (SOLID, camadas, acoplamento)**

- Controllers decidindo política de erro (responsabilidade da infraestrutura) e serviços redescobrindo no banco quem é o solicitante (17 consultas `FirstOrDefaultAsync(x => x.UsuarioId == ...)`), embora o JWT já carregasse `pacienteId`, `medicoId` e `instituicaoId`. A classe base assumia `Paciente` quando o claim de perfil faltava.
- `ProntuarioService` lendo `IConfiguration` e criando diretório em disco no construtor (serviço `Scoped`, logo a cada request), além de ler e gravar arquivos: infraestrutura dentro do serviço de domínio.
- A regra "médico pode ver dado desta instituição de origem" escrita três vezes (dois `foreach` espelhados no histórico e uma consulta ao banco no download).
- Chave JWT lida de fontes diferentes para assinar (`JWT_SECRET_KEY` ou `Jwt:SecretKey`) e para validar (só `Jwt:SecretKey`), com textos padrão diferentes embutidos no código; `Jwt:ExpirationHours` nunca lida.
- Classe base de controllers não abstrata e com `[ApiController][Route("api")]`, registrada como controller sem ações.

**Obsolescência**

- `try/catch` por ação onde o .NET 8 oferece um filtro de exceção global.
- `IConfiguration["..."] ?? "default"` onde cabe `IOptions<T>` com validação na subida.
- `foreach` + `List.Add` onde cabe projeção LINQ; quatro `COUNT` onde cabe um `GroupBy`; `Include` em massa onde cabe `Select` com projeção.
- Classes mutáveis com `{ get; set; } = string.Empty` para DTOs de entrada onde cabem `record`s imutáveis.
- Construtores com campos `private readonly` repetidos onde cabem construtores primários (C# 12).
- `JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear()`, sem efeito no .NET 8 (o JwtBearer valida com `JsonWebTokenHandler`).
- Frontend: query string concatenada à mão onde cabe `HttpParams`; `any` onde o tipo já existia; URL da API fixa em dois serviços.

## 2. Fase de execução (transformação assistida)

Cada intervenção é um commit que mantém a suíte verde. A tabela registra o problema, a técnica e o prompt que dirigiu a mudança.

### 2.1 Higiene do repositório (415c865, d1ad4ba)

- **Problema**: 141 artefatos de build (`bin/`, `obj/`, 37 MB, com caminhos `D:\...` gravados) versionados; `.http` apontando para uma rota inexistente; `BCrypt.Net-Next` e `Microsoft.AspNetCore.OpenApi` referenciados sem nenhum uso.
- **Técnica**: remoção de código morto (YAGNI); `.gitignore` no backend.
- **Prompt**: *"Liste o que está versionado e não deveria estar, e as dependências sem nenhuma referência no código. Remova do índice e adicione um .gitignore. Ao final, descreva o problema resolvido e a técnica aplicada."*

### 2.2 Código morto e base controller (304637e)

- **Problema**: injeção sem uso, campos nunca lidos, classe base roteável, `AuthController` reimplementando a leitura de claim que a base já fazia.
- **Técnica**: identificação de código morto por grep de usos; `abstract` na base.
- **Prompt**: *"Para cada campo, parâmetro e dependência injetada, conte os usos no repositório. Remova o que tem zero leituras. Torne a classe base dos controllers abstrata e sem rota. Mantenha o pin verde."*

### 2.3 Tratador global de exceções (03f54ed, revisto em e8ceee8)

- **Problema**: 27 `catch` repetindo a mesma tradução de exceção para HTTP; 409 decidido por texto; nove ações deixando a exceção vazar como 500.
- **Técnica**: primeiro um `IExceptionHandler` (.NET 8) mapeando `InvalidOperationException` e `UnauthorizedAccessException`; a revisão independente mostrou que isso aplicava a política à API inteira e transformava falhas de infraestrutura em 4xx (seção 3.4). A versão final usa tipos de domínio (`RegraDeNegocioException` 400, `ConflitoException` 409, `AcessoNegadoException` 403) e um `IExceptionFilter` do MVC, com o mesmo alcance dos `try/catch` originais. Controllers viram expressões de uma linha com construtores primários.
- **Prompt**: *"Centralize o mapeamento de exceções para HTTP num único ponto que preserve o corpo `{ mensagem }` lido pelo frontend. Crie tipos de exceção de domínio para 400, 403 e 409 e troque os lançamentos nos serviços por substituição mecânica; exceções da BCL não podem mais virar 4xx. Remova todos os try/catch dos controllers. As rotas que hoje vazam 500 vão passar a responder 403/400: atualize os testes que congelaram o 500 e registre a mudança no commit. Adicione um teste que prova que o cabeçalho CORS continua presente em resposta de erro."*
- **Resultado**: `DomainControllers.cs` de 596 para 303 linhas (depois dividido em 14 arquivos); 0 linhas de log `fail` por 4xx esperado (eram 45 com o `IExceptionHandler`).

### 2.4 Um ponto de montagem por DTO e projeção única de usuário (a8476ae, revisto em 3aa322c)

- **Problema**: 24 construções manuais de DTO; cadeia de `Include` repetida 3 vezes; bloco `pacienteId/medicoId` repetido 4 vezes com N+1 no admin; criação de usuário Identity duplicada; regra de consentimento escrita 3 vezes; `ObterHistoricoPacienteAsync` com 147 linhas e dois `foreach` espelhados.
- **Técnica**: DRY. Na primeira versão, métodos de extensão `ToDto()` sobre entidades carregadas por `Include`. A revisão apontou que isso só mudava a complexidade de lugar (navegações atribuídas à mão, `!` mentirosos). A versão final define uma `Expression<Func<Entidade, Dto>>` por DTO em `DTOs/Mapeamentos.cs`, usada com `Select` nas listas (o EF Core traduz para SQL, inclusive a disponibilidade do slot como `NOT EXISTS`) e recarregada pelo id logo após cada criação; `ToDto()` é a mesma expressão compilada, usada só em agregados já carregados. Também: `Usuarios.CriarUsuarioAsync()`, método de domínio `Paciente.InstituicaoPodeVer()` (política clínica única para histórico e download), predicado `podeVerOrigem` por perfil no lugar dos loops, `GroupBy` no dashboard.
- **Prompt**: *"Cruze os serviços e encontre cada DTO montado à mão mais de uma vez. Defina uma única projeção por DTO e migre todos os chamadores no mesmo passo, sem manter a versão antiga; as listas devem projetar no SQL. Substitua o N+1 da lista de usuários por uma projeção que resolve PacienteId e MedicoId no próprio SQL. A regra de consentimento entre instituições deve existir uma vez, no domínio. Reescreva os dois foreach do histórico como um predicado por perfil, preservando o comportamento atual do Enfermeiro (lista vazia), que já está congelado em teste."*
- **Resultado**: `.Include(` de 58 para 27; atribuições de navegação com `!` de 3 para 0.

### 2.5 Roteiro da triagem como função pura (8d44753)

- **Problema**: máquina de estados com transição, preenchimento do resumo e texto das perguntas misturados num `switch` de 167 linhas; perguntas duplicadas; cinco buscas iguais da conversa no Mongo; três `ReplaceOne` iguais; N consultas ao Mongo na fila.
- **Técnica**: extração para `RoteiroTriagem` (sem I/O), com `Avancar()` e `Pergunta()` indexada pela etapa de chegada (cada texto existe uma vez por construção); fábricas `MensagemMongo.Bot()` e `DoPaciente()`; helpers `ColetaDoPacienteAsync`, `ConversaAsync`, `SalvarConversaAsync`; `$in` na fila.
- **Prompt**: *"Antes de mexer, complete o pin com os ramos do roteiro que ainda não estão cobertos (alergia 'não sei', 'sim, tomo medicamentos', os dois 'não' diretos e resposta vazia). Depois extraia o switch para uma classe pura em que a pergunta é indexada pela etapa de destino, de modo que texto duplicado seja impossível. Crie fábricas para as mensagens do bot e do paciente. Escreva testes unitários da classe pura que rodem sem containers."*
- **Resultado**: `ProcessarMensagemAsync` de 220 para 35 linhas; 14 testes unitários em 9 ms.

### 2.6 Opções tipadas e armazenamento fora do domínio (34a4fc2)

- **Problema**: JWT lido de fontes divergentes para assinar e validar; `Jwt:ExpirationHours` nunca lida; segredos padrão no código; `ProntuarioService` lendo configuração, criando diretório a cada request e fazendo I/O de disco.
- **Técnica**: `JwtOptions` e `StorageOptions` com `ValidateDataAnnotations().ValidateOnStart()`; `JwtBearerOptions` configurado via `IOptions<JwtOptions>`; singleton `ArmazenamentoArquivos`; `.env.example` na convenção `Secao__Chave`.
- **Prompt**: *"Faça a validação e a assinatura do JWT lerem a mesma classe de opções, validada na subida (chave de pelo menos 32 bytes). Remova os segredos padrão do código e as chaves de ambiente alternativas que só um dos lados lia. Tire a leitura de configuração e o acesso a disco de ProntuarioService para um singleton de infraestrutura."*
- **Auditoria de segurança**: a API passa a recusar subir sem chave JWT configurada; a chave de assinatura e a de validação não podem mais divergir; `Program.cs` deixa de ter uma senha de banco e uma chave JWT embutidas como fallback.

### 2.7 `UsuarioAtual` a partir dos claims (75f3a92, ce4fea8)

- **Problema**: 11 métodos recebendo `(Guid solicitanteUsuarioId, PerfilUsuario perfil)` e 17 consultas ao banco só para descobrir o Paciente, o Médico ou a instituição de quem chama; três helpers na classe base com fallbacks mortos e default silencioso para `Paciente`; nomes de claim duplicados entre quem emite e quem lê; 18 papéis como strings soltas.
- **Técnica**: desacoplamento pela fronteira (boundary discipline): record `UsuarioAtual(Id, Perfil, InstituicaoId, PacienteId, MedicoId)`, que é o único tipo que conhece os nomes dos claims (`ParaClaims()` emite no login, `De()` lê a cada requisição); 22 métodos de serviço passam a recebê-lo; sobram só as buscas que precisam da entidade (instituição do médico no consentimento), agora por chave primária. `Dominio/Papeis` deriva os nomes de papel de `nameof(PerfilUsuario.X)`.
- **Prompt**: *"Os serviços perguntam ao banco quem é o usuário embora o token já diga. Crie um record montado uma vez a partir dos claims, passe-o aos serviços no lugar dos pares (id, perfil) e troque cada busca por UsuarioId que só servia para obter um id pelo claim correspondente. O mesmo tipo deve emitir e ler os claims. Preserve todas as mensagens de erro e a matriz de RBAC congelada."*

### 2.8 Seed por helper e produto cartesiano (6aee21c)

- **Problema**: 422 linhas num método; nove blocos idênticos de criação de usuário com resultado ignorado; quatro literais de geração de horário.
- **Técnica**: reuso de `CriarUsuarioAsync`; `AddRange` único; query expression `dias úteis × médicos × horas × meia-hora` no lugar dos loops.
- **Prompt**: *"Reescreva o seed usando o helper de criação de usuário já existente e uma única expressão declarativa para os horários. Os ids, nomes, datas e quantidades têm de ser idênticos: o pin verifica contagem de slots, nomes e datas do seed."*

### 2.9 DTOs de entrada como records (d483f38)

- **Problema**: 12 DTOs de entrada mutáveis, inconsistentes com os 16 de saída que já eram records.
- **Técnica**: modernização sintática: `record` com `init` e os mesmos valores padrão (para que campo ausente continue chegando como `""`, não `null`).
- **Prompt**: *"Converta os DTOs de entrada para records imutáveis sem alterar a desserialização: mantenha os valores padrão de cada propriedade."*

### 2.10 Frontend mínimo (8b9c114)

- **Problema**: URL da API fixa em dois serviços; 11 `any`; query string concatenada à mão em três métodos; template do CLI de 336 linhas sem uso.
- **Técnica**: constante única `API_URL`; interfaces `UsuarioInfo`, `HealthStatus`, `Mensagem`, `MensagemChat`; `HttpParams` via helper; remoção de código morto. Validado por `ng build` (não há testes de frontend; por isso o escopo ficou em mudanças verificáveis pelo compilador).
- **Prompt**: *"No frontend, só o que o compilador consegue provar: elimine os any usando os tipos que já existem, centralize a URL da API e troque a concatenação de query string por HttpParams. Remova arquivos que nenhum componente referencia. Pare onde seria preciso um teste de interface."*

### 2.11 Divisão física (e50e14e)

`DomainControllers.cs` (11 controllers e a base) e `AuxiliaryServices.cs` (5 serviços sem relação) foram divididos em um arquivo por tipo com um script `awk` que separa os tipos de topo e descarta os rótulos de seção, depois que o conteúdo já tinha encolhido. Nenhuma linha de código mudou.

### 2.12 Documentação (be318ad)

`.ai/*.md` e `README.md` descreviam .NET 6-7, Angular 16, Angular Material, AutoMapper, uma rota `/api/enfermeiro/validar` e um campo `Disponivel` inexistentes. Como esses arquivos são o contexto que um agente de IA lê antes de escrever código, a divergência gera código para outro sistema. Foram alinhados ao sistema real e ganharam os ADRs 005 (política única de erro) e 006 (testes de caracterização).

## 3. Fase de pós-refatoração (validação e evolução)

### 3.1 Paridade funcional

A suíte de caracterização foi executada após cada commit e está verde em `HEAD`: 59 testes, 45 de integração contra PostgreSQL e MongoDB reais e 14 unitários. Como o pin foi gravado antes das mudanças, validado por mutação e rodado retroativamente contra o código original pela revisão independente, o resultado é a prova de que o contrato HTTP (status, corpo, RBAC, roteiro do chat, consentimento, dados do seed) permaneceu o mesmo, exceto pelas mudanças deliberadas da seção 5.

Comando: `cd backend && dotnet test Consultorio.Api.Tests` (requer Docker; cerca de 15 s numa máquina sem carga, dos quais a maior parte é subir os containers).

### 3.2 Processamento assíncrono (equivalente a Virtual Threads)

O item "Virtual Threads" do roteiro é específico de Java. O equivalente em .NET já estava em uso: todo I/O (EF Core, MongoDB, disco) é `async/await` sobre o thread pool, sem threads bloqueadas. O ganho de desempenho desta refatoração veio de reduzir idas ao banco e dados trafegados, não de trocar o modelo de threads: a fila do enfermeiro deixou de fazer uma consulta ao Mongo por coleta (`$in` único), a lista de usuários do admin deixou de fazer uma consulta por usuário (subconsultas na projeção), o dashboard trocou quatro `COUNT` por um `GROUP BY`, as listas passaram a projetar só as colunas do DTO no SQL (a disponibilidade de cada slot virou `NOT EXISTS` em vez de carregar todas as consultas de todos os slots), e 17 consultas de identidade do solicitante deixaram de existir.

### 3.3 Auditoria de segurança e tratamento de exceções

- Um único `TratadorGlobalDeExcecoes` (`IExceptionFilter`) responde 400/403/409 com `{ mensagem }` para as três exceções de domínio. Exceções da BCL (configuração, disco, banco) nunca são mapeadas: continuam 500 e registradas como erro, como no original. Nenhum 4xx esperado gera log de erro.
- Nenhum segredo padrão no código; a chave JWT é obrigatória e validada na subida; assinatura e validação leem a mesma fonte.
- Um token sem claim de perfil ou de identificador é recusado em vez de ser tratado como `Paciente`.
- O `BsonIgnoreExtraElements` em `ConversaMongo` garante que documentos gravados pela versão antiga (com `Status` e `VersaoRoteiro`) continuem legíveis, e há um teste que insere um documento no formato antigo e o lê pela API.

### 3.4 Revisão independente pós-refatoração

Um segundo agente, sem acesso ao raciocínio do primeiro, revisou o diff `main..refactoring` no mesmo padrão thermo-nuclear, com permissão para rodar a suíte, aplicar mutantes e escrever sondas HTTP descartáveis comparando o código original e o novo. O relatório está em `docs/refactoring/revisao/` (`00_resumo.md` e três detalhes).

Veredito da revisão: "pedir mudanças", reconhecendo a melhoria líquida (4 267 para 3 289 linhas naquele momento, 30 `catch` para 3, N+1 removidos, roteiro puro) mas com dois itens bloqueantes. O que foi feito com cada achado:

1. **Tratador global mapeando exceções da BCL em toda a API (bloqueante).** Confirmado por sonda: com `MongoDatabaseName` ausente, `/api/health` respondia `400 {"mensagem": ...}`; um diretório sem permissão virava 403 com o caminho do servidor na mensagem; cada 4xx esperado gerava um log `fail` com stack trace (45 por execução). Corrigido em e8ceee8: tipos de domínio e `IExceptionFilter`. Resultado: 0 logs `fail`, infraestrutura volta a ser 500.
2. **Pin sem a regra de consentimento (bloqueante).** Dois mutantes (`TemAutorizacaoAtiva => false` e médico vendo qualquer origem) passavam com 51/51, e a revisão contou 9 rotas que saíram de 500 (o commit falava em 4). Corrigido em e1f1e05: 8 testes novos (consentimento ativo e revogado no histórico e no download, as 5 rotas sem pin, usuários com perfil sem registro, validação reprovada, upload acima de 5 MB, documento Mongo antigo). Os dois mutantes agora derrubam o cenário.
3. **Code-judo perdido no mapeamento.** Aceito e feito em 3aa322c (projeções `Expression`, seção 2.4).
4. **`UsuarioAtual` como saco de parâmetros; regra de dono copiada 14 vezes.** Parcialmente aceito: a política clínica agora existe uma vez (`Paciente.InstituicaoPodeVer`). As 14 verificações de dono continuam inline porque cada uma tem uma mensagem de erro própria, congelada pelo pin; um helper com a mensagem como parâmetro economizaria uma linha por uso sem remover a decisão. Fica como próximo passo.
5. **Contratos como strings.** Aceito para claims (um só tipo emite e lê) e papéis (`Dominio/Papeis`), em ce4fea8. Não feito: enums para `"Bot"`/`"Opcoes"` e para as etapas 1 a 9 (formato persistido no Mongo, ganho pequeno) e `MongoOptions` (duas linhas de leitura com `?? throw`).
6. **Arquivos-gaveta.** Aceito em e50e14e. A divisão de `ProntuarioService` em atendimento, histórico, documentos e autorizações não foi feita: os quatro assuntos compartilham a política clínica, e separá-los exige mais interfaces e registros de DI do que remove.
7. **Interfaces de implementação única e ciclo entre pastas.** O ciclo entre `Infraestrutura` e `Services` foi quebrado movendo `UsuarioAtual` e as exceções para `Dominio/`. As 10 interfaces foram mantidas (ver seção 4).
8. **Lógica residual nos controllers** (`Guid.Empty` para o Admin na fila, `?? dto.InstituicaoId` no slot, 401 ao iniciar coleta sem registro de Paciente) e **higiene do pin** (dashboard por diferença, cenário único de 400 linhas): registrados como próximos passos. O filtro de slots por data passou a usar o próximo dia útil com `Assert.NotEmpty`, como sugerido.

## 4. O que ficou de fora e por quê

- **Divisão de `ProntuarioService`** em quatro serviços: compartilham a política clínica e a verificação de vínculo médico-paciente; separar hoje acrescenta interfaces e registros sem apagar lógica.
- **Helper para a regra de dono**: 14 verificações com mensagens distintas congeladas pelo pin; o helper só esconderia a mesma decisão.
- **Interfaces de implementação única** (`IAgendaService` etc.): mantidas; removê-las é mudança de muitos arquivos com ganho pequeno, e o projeto pode querer dublês no futuro.
- **Enums para strings do Mongo e para as etapas do roteiro**, `MongoOptions`: formato persistido ou duas linhas de código; ganho pequeno frente ao risco.
- **Frontend além do que o compilador prova**: tratamento uniforme de erro (`finalize` + mensagem), guardas de rota por perfil, controle de fluxo nativo (`@if`/`@for`) e selos tipados estão detalhados em `diagnostico/04_frontend.md`, mas sem testes de interface não há como provar paridade.
- **Health checks nativos** (`AddHealthChecks`): mudariam a forma da resposta de `/api/health`, que o frontend lê.
- **Higiene do pin**: dashboard medido por diferença e cenário dividido em fases com um paciente por teste.

## 5. Mudanças conscientes de contrato

Registradas nos commits e nos testes, para não se confundirem com a refatoração pura:

1. Nove rotas que deixavam a exceção vazar como 500 passam a responder 403/400 com `{ mensagem }`, como todas as outras: `GET`/`PATCH /api/pacientes/{id}` para outro paciente, `POST /api/coletas` com instituição inexistente, `POST /api/pacientes` com e-mail repetido, histórico, documentos e autorizações de paciente inexistente, notificações e pesquisa para usuário Paciente sem registro. `AdminController` deixa de transformar qualquer `Exception` em 400. Todas as nove estão pinadas.
2. Variáveis de ambiente `JWT_*` e `MONGO_*` deixam de ser lidas (só um dos lados do JWT as lia); vale a convenção `Jwt__SecretKey`, documentada em `.env.example`. Sem chave JWT configurada a API não sobe.
3. Token sem claim de perfil ou de identificador é recusado (403) em vez de assumido como `Paciente`. O claim `perfil`, sem nenhum leitor, deixa de ser emitido.
4. `ConversaMongo.Status`, `ConversaMongo.VersaoRoteiro` e `EnviarMensagemChatDto.EtapaEsperada` deixam de ser gravados/aceitos. Nenhum leitor existia; documentos antigos continuam legíveis.

## 6. Bugs observados e não corrigidos

Encontrados durante a leitura e deixados fora da refatoração de propósito (corrigi-los muda comportamento). Lista completa com `arquivo:linha` em `diagnostico/00_resumo.md`, seção "Bugs observados". Os mais relevantes:

- Senha do PostgreSQL em `appsettings.json` difere da do `docker-compose.yml`; o erro do seed é engolido e a API sobe sem banco.
- O histórico do paciente devolve `temPesquisaRespondida=false` sempre, por falta de `Include(PesquisaSatisfacao)` (congelado em `ProntuarioTests`).
- Transições de consulta não são validadas: é possível concluir consulta cancelada e iniciar atendimento de consulta concluída; atendente ou médico de qualquer instituição pode cancelar.
- `ObterConsultaPorIdAsync` e `GET /api/coletas/{id}` liberam leitura a qualquer médico, atendente ou enfermeiro, de qualquer instituição (congelado em `TriagemAgendamentoTests`).
- O Enfermeiro baixa qualquer documento clínico: o download não tem ramo para esse perfil (congelado em `ProntuarioTests`).
- `FinalizarEnvioEnfermeiroAsync` não verifica a etapa: a coleta pode ser enviada na primeira pergunta.
- O enfermeiro valida coleta de outra instituição; a fila do Admin sem instituição vem vazia.
- `CriarUsuarioAsync` (admin) cria o usuário antes de validar que médico precisa de instituição, deixando um usuário órfão; um usuário Paciente criado pelo admin não ganha registro de `Paciente`.
- Na triagem, `Contains("Sim")` também aceita "assim"; o frontend gera um `clienteMessageId` novo a cada envio, anulando a idempotência.

## 7. Como reproduzir

```bash
git checkout refactoring
docker info                                   # Docker precisa estar rodando
cd backend && dotnet test Consultorio.Api.Tests
cd ../frontend && npm ci && npx ng build
```

Arquivos de apoio:

- `docs/refactoring/PROMPTS.md`: texto integral de todos os prompts (do aluno, dos dois agentes e de cada intervenção).
- `docs/refactoring/diagnostico/`: auditoria do código original (resumo + 5 detalhes).
- `docs/refactoring/revisao/`: revisão independente do diff (resumo + 3 detalhes), feita antes da rodada de correções e1f1e05 a e50e14e.
- `backend/Consultorio.Api.Tests/`: rede de segurança (caracterização + unitários).
- `git log --reverse main..refactoring`: um commit por intervenção, cada um com o resultado do pin na mensagem.
