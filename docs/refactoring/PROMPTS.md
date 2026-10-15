# Prompts utilizados

Registro integral dos prompts desta atividade, na ordem em que foram usados. O relatório (`RELATORIO_MODERNIZACAO.md`) traz versões resumidas de cada um ao lado da intervenção correspondente; aqui está o texto completo.

## 1. Prompt do aluno que orientou toda a sessão

Enviado ao agente (Claude Code, modelo Claude Fable 5.1, skills `pstack:poteto-mode` e `pstack:thermo-nuclear-code-quality-review`) com o repositório recém-clonado:

> preciso utilizar essas técnicas de refactor:
>
> Técnicas de Refactoring com IA. A IA atua como um par sênior que identifica oportunidades de melhoria que passam despercebidas no dia a dia.
> - DRY (Don't Repeat Yourself): Detecção de padrões duplicados em arquivos distintos. Agentes de IA cruzam informações de todo o repositório para sugerir abstrações e reduzir a redundância.
> - KISS (Keep It Simple, Stupid): Simplificação de lógicas densas. A IA ajuda a quebrar algoritmos complexos em passos menores e mais intuitivos, priorizando a clareza sobre a sofisticação desnecessária.
> - YAGNI (You Ain't Gonna Need It): Eliminação de funcionalidades "para o futuro". A IA identifica trechos de código que não estão sendo chamados ou que resolvem problemas que ainda não existem, focando na entrega de valor imediata.
> - Identificação de Código Morto: Agentes conseguem mapear caminhos de execução e sugerir a remoção com segurança de classes e métodos nunca utilizados.
> - Abstração de Lógica (Decoupling): Identificar quando uma classe acumulou responsabilidades demais e propor a separação em componentes menores (SOLID).
> - Modernização Sintática: Substituição de construções verbosas e antigas por novas facilidades da linguagem (independente da stack).
>
> utilizar essa formas de validação:
>
> Segurança e Validação de Refactoring. Refatorar sem testes é apenas "estragar código de forma organizada".
> - Test-Snapshotting: Antes de iniciar a refatoração, use a IA para gerar testes que capturem o comportamento atual do sistema (mesmo que o comportamento esteja errado, ele é o seu baseline).
> - Paridade Funcional: Após a transformação, rode a suíte de testes. Se a IA sugere mudar uma lógica, ela deve provar que o resultado final para o usuário permanece o mesmo.
>
> precisamos fazer essas 3 etapas antes de iniciar a refatoração lembrando refatoração não é para arrumar nenhum bug e nem arrumar nada apenas refatorar e melhorar o código:
>
> 1. Fase de Pré-Refatoração (Blindagem): Baseline de Testes de Caracterização; Mapeamento de Acoplamento e Dívida Técnica.
> 2. Fase de Execução (Transformação Assistida): Migração de Versão e Otimização de Records; Refatoração de Domínio (Desacoplamento); Modernização de Coleções e Streams.
> 3. Fase de Pós-Refatoração (Validação e Evolução): Implementação de Virtual Threads; Audit de Segurança e Tratamento de Exceções; Relatório de Modernização (Dívida Paga).
>
> preciso entregar essa atividade então precisa estar na branch `refactoring` e ser enviada ao GIT. Entregáveis: 1. Código Refatorado (branch `refactoring`). 2. Relatório de Modernização (PDF ou Markdown) com: Diagnóstico de Dívida Técnica (Code Smells, Violação de Padrões, Obsolescência); Plano de Refatoração e Execução (para cada problema: Problema, Técnica Aplicada, Prompt de Refatoração).
>
> utilizar /pstack:poteto-mode /pstack:thermo-nuclear-code-quality-review

## 2. Rede de segurança (testes de caracterização)

Instrução formulada pelo agente para si mesmo antes de qualquer mudança estrutural:

> Antes de qualquer refatoração, escreva testes de integração HTTP que congelem o comportamento atual de todos os endpoints, usando WebApplicationFactory e Testcontainers com PostgreSQL e MongoDB reais, seedados pelo DbInitializer. Capture status, corpo `{ mensagem }`, RBAC por perfil em cada recurso, o roteiro determinístico do chat etapa por etapa, os dois casos de 409 do agendamento, as transições de status, evolução e conclusão, notificações, pesquisa e dashboard por instituição e global. Se o comportamento atual parecer errado (por exemplo, 500 em vez de 403, ou um campo sempre false), congele-o mesmo assim e marque com comentário; não corrija nada. Depois prove que o pin morde com uma mutação deliberada no código e confirme que a suíte fica vermelha.

## 3. Diagnóstico de dívida técnica (agente de auditoria)

Brief enviado a um subagente independente (modelo Claude Opus, esforço de raciocínio alto), com o repositório ainda no estado original:

> You are a software architect performing a technical-debt audit (read-only review) of a small academic project. Do NOT modify any source file. Your only output is a set of Markdown report files.
>
> Read first, in full: the `thermo-nuclear-code-quality-review` skill. Apply its standards, review questions, "code judo" ambition, output expectations, and the layered report format (one summary + one detail file per subsystem). This audit is of the WHOLE current codebase (there is no diff yet), so treat "the current branch's changes" as "the entire project as it stands".
>
> Project: backend ASP.NET Core 8 Web API at backend/Consultorio.Api (C#, EF Core + Npgsql, MongoDB.Driver, ASP.NET Identity, JWT). Ignore Migrations/, bin/, obj/. Read everything else. Frontend: Angular standalone SPA at frontend/src/app. Docs: .ai/*.md and README.md describe intended architecture/standards (note where the code diverges). Repo hygiene: check whether build artifacts are tracked and whether a backend .gitignore exists.
>
> The assignment asks an AI agent to scan the project and list, with evidence (file:line, counts, sizes):
> 1. Code Smells: long methods, classes with multiple responsibilities, duplicated blocks (count the copies), N+1 query loops, magic strings, string-content-based control flow, dead code, config duplication.
> 2. Violação de Padrões (SOLID, layering, coupling): controllers holding exception-to-HTTP mapping policy 25+ times instead of one handler; a service doing authorization + persistence + presentation mapping; IConfiguration read deep inside services; base controller that is itself routable; business rules living in a switch on an int step with inline bot copy; filesystem I/O inside a domain service. Be concrete and rank by maintainability impact.
> 3. Obsolescência / modernização: manual foreach + List.Add where a LINQ projection fits; mutable DTO classes where record fits; IConfiguration with ?? fallbacks where IOptions binding fits; pre-.NET-8 exception handling; frontend repeated patterns, any usage, hard-coded base URL.
>
> Also include the ambitious "code judo" proposals: which restructurings would delete whole categories of complexity while preserving behavior (this is a refactoring, NOT a bug-fixing pass; note real bugs you spot in a separate short "Bugs observados (não corrigir nesta refatoração)" section).
>
> Measure: wc -l per source file, count duplicated blocks with grep (show the commands), list files over ~400 lines and methods over ~60 lines.
>
> Output: docs/refactoring/diagnostico/00_resumo.md (verdict, findings as prose with file:line, remediation sequence) plus detail files 01 to 05. Write all prose in Portuguese (pt-BR). Every claim carries a file:line or a command output you actually ran. Do not invent evidence.

## 4. Prompts por intervenção

Cada um dirigiu um commit e foi executado com o pin verde antes e depois.

**2.1 Higiene do repositório** (415c865, d1ad4ba)
> Liste o que está versionado e não deveria estar, e as dependências sem nenhuma referência no código. Remova do índice e adicione um .gitignore. Ao final, descreva o problema resolvido e a técnica aplicada.

**2.2 Código morto e base controller** (304637e)
> Para cada campo, parâmetro e dependência injetada, conte os usos no repositório. Remova o que tem zero leituras. Torne a classe base dos controllers abstrata e sem rota. Mantenha o pin verde.

**2.3 Tratador global de exceções** (03f54ed, revisto em e8ceee8)
> Centralize o mapeamento de exceções para HTTP num único ponto que preserve o corpo `{ mensagem }` lido pelo frontend. Crie tipos de exceção de domínio para 400, 403 e 409 e troque os lançamentos nos serviços por substituição mecânica; exceções da BCL não podem mais virar 4xx. Remova todos os try/catch dos controllers. As rotas que hoje vazam 500 vão passar a responder 403/400: atualize os testes que congelaram o 500 e registre a mudança no commit. Adicione um teste que prova que o cabeçalho CORS continua presente em resposta de erro.

**2.4 Um ponto de montagem por DTO** (a8476ae, revisto em 3aa322c)
> Cruze os serviços e encontre cada DTO montado à mão mais de uma vez. Defina uma única projeção por DTO e migre todos os chamadores no mesmo passo, sem manter a versão antiga; as listas devem projetar no SQL. Substitua o N+1 da lista de usuários por uma projeção que resolve PacienteId e MedicoId no próprio SQL. A regra de consentimento entre instituições deve existir uma vez, no domínio. Reescreva os dois foreach do histórico como um predicado por perfil, preservando o comportamento atual do Enfermeiro (lista vazia), que já está congelado em teste.

**2.5 Roteiro da triagem como função pura** (8d44753)
> Antes de mexer, complete o pin com os ramos do roteiro que ainda não estão cobertos (alergia "não sei", "sim, tomo medicamentos", os dois "não" diretos e resposta vazia). Depois extraia o switch para uma classe pura em que a pergunta é indexada pela etapa de destino, de modo que texto duplicado seja impossível. Crie fábricas para as mensagens do bot e do paciente. Escreva testes unitários da classe pura que rodem sem containers.

**2.6 Opções tipadas e armazenamento fora do domínio** (34a4fc2)
> Faça a validação e a assinatura do JWT lerem a mesma classe de opções, validada na subida (chave de pelo menos 32 bytes). Remova os segredos padrão do código e as chaves de ambiente alternativas que só um dos lados lia. Tire a leitura de configuração e o acesso a disco de ProntuarioService para um singleton de infraestrutura.

**2.7 UsuarioAtual a partir dos claims** (75f3a92, ce4fea8)
> Os serviços perguntam ao banco quem é o usuário embora o token já diga. Crie um record montado uma vez a partir dos claims, passe-o aos serviços no lugar dos pares (id, perfil) e troque cada busca por UsuarioId que só servia para obter um id pelo claim correspondente. O mesmo tipo deve emitir e ler os claims. Preserve todas as mensagens de erro e a matriz de RBAC congelada.

**2.8 Seed por helper e produto cartesiano** (6aee21c)
> Reescreva o seed usando o helper de criação de usuário já existente e uma única expressão declarativa para os horários. Os ids, nomes, datas e quantidades têm de ser idênticos: o pin verifica contagem de slots, nomes e datas do seed.

**2.9 DTOs de entrada como records** (d483f38)
> Converta os DTOs de entrada para records imutáveis sem alterar a desserialização: mantenha os valores padrão de cada propriedade.

**2.10 Frontend mínimo** (8b9c114)
> No frontend, só o que o compilador consegue provar: elimine os any usando os tipos que já existem, centralize a URL da API e troque a concatenação de query string por HttpParams. Remova arquivos que nenhum componente referencia. Pare onde seria preciso um teste de interface.

**2.11 Divisão física** (e50e14e)
> Divida DomainControllers.cs e AuxiliaryServices.cs em um arquivo por tipo com um script, sem alterar nenhuma linha de código. Descarte os rótulos de seção.

## 5. Revisão independente pós-refatoração (agente revisor)

Brief enviado a um segundo subagente (modelo Claude Opus, esforço alto), sem acesso ao raciocínio do primeiro:

> You are an independent, demanding code-quality reviewer. Do NOT modify any source file, do NOT commit. Your only output is a set of Markdown report files plus a short reply.
>
> Read first, in full: the `thermo-nuclear-code-quality-review` skill. Apply its standards (code-judo ambition, review questions, what to flag, remedies, tone, layered report format, approval bar).
>
> Review the backend changes of the branch `refactoring` versus `main` (git diff main..HEAD -- backend/Consultorio.Api, ignoring Migrations/). Read the resulting files in full. Also read the test project (characterization suite via WebApplicationFactory + Testcontainers, and unit tests) because it is the behavior pin the refactor claims to hold. A pre-refactor audit of the ORIGINAL code lives in docs/refactoring/diagnostico/. The refactor deliberately made a few behavior changes, documented in commit messages; judge those on their merits but do not report them as "undocumented".
>
> You may build and run the tests (Docker is running). Do not run the app or npm. Do not edit files under backend/ or frontend/.
>
> Review questions to answer with evidence (file:line of the NEW code):
> 1. Structural regressions: anything that got more coupled, more indirect, or harder to scan than before? Are the new extension-method homes the right layer? Is UsuarioAtual well placed? Is Paciente.TemAutorizacaoAtiva a good domain method or a smell?
> 2. Missed code-judo: where could a further restructuring delete whole categories of complexity while keeping the HTTP contract pinned by the tests? Be concrete with before/after sketches.
> 3. Behavior-preservation risks the pin might not catch: EF Core navigation fixup, claim values instead of DB lookups, Add vs AddAsync, dropped Includes, FindAsync with nullable key, null-forgiving assignments, ToDictionary keyed by Mongo id, GroupBy translation. For each, say whether it is safe, and if not, what minimal test would expose it. If you can prove a risk by running the suite or writing a throwaway check, do so and report the result.
> 4. Spaghetti / boundary / type-contract: leftover string statuses, role strings, IConfiguration still read, tuple returns, bool-returning service methods mapped in controllers, single-implementation interfaces.
> 5. Test quality: is the suite asserting behavior (not implementation)? Brittle or tautological asserts? Gaps in HTTP contract coverage? Is one long scenario test acceptable?
> 6. File sizes before/after, remaining duplicated blocks, anything that crossed a sensible size threshold.
>
> Output: docs/refactoring/revisao/00_resumo.md (verdict against the skill's approval bar, findings ranked, remediation sequence) plus 01_estrutura_e_code_judo.md, 02_riscos_de_comportamento.md, 03_testes.md with measurements, commands you ran and their output, and worked before/after sketches. Portuguese (pt-BR). Every claim carries file:line or a command output you actually ran.

## 6. Rodada de correções a partir da revisão

Instruções formuladas pelo agente após triar os achados (commits e1f1e05, e8ceee8, 3aa322c, ce4fea8, e50e14e):

> Primeiro feche o pin: transforme as sondas da revisão em testes (consentimento ativo e revogado no histórico e no download, as cinco rotas que saíram de 500 sem teste, usuários com perfil sem registro, validação reprovada, upload acima de 5 MB, documento Mongo no formato antigo) e rode os dois mutantes do consentimento para confirmar que agora derrubam a suíte. Depois: tipos de exceção de domínio e um IExceptionFilter do MVC no lugar do IExceptionHandler, para que falhas de infraestrutura voltem a ser 500 e 4xx esperados não gerem log de erro; projeções Expression nas listas com recarga pelo id após cada criação, apagando as atribuições de navegação com `!`; um só tipo que emite e lê os claims e constantes para os papéis; um arquivo por tipo. Um commit por item, pin verde em cada um.
