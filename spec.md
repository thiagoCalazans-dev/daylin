# Daylin — especificação do MVP

**Estado:** especificação de planejamento; nenhuma feature foi implementada. Este documento reúne as regras confirmadas e os defaults técnicos propostos para orientar a implementação. A visão visual está em [design.md](design.md), e rotas/pastas em [architecture.md](architecture.md).

## Convenções e decisões confirmadas

- Código, nomes de funções, classes, arquivos de código, pastas técnicas, tabelas, colunas, JSON, endpoints e URLs **sempre em inglês**. Documentação e textos da interface podem estar em português.
- Um projeto Next.js com App Router e um deploy na Vercel; Supabase para Auth/Postgres/Storage privado; Vercel Workflow para processamento; OpenAI API com chave do projeto no servidor.
- Cada plano usa um único PDF com texto selecionável (até 100 páginas) **ou** texto colado. Sem OCR no MVP.
- O banco inicial de perguntas é criado quando o material é processado, com cobertura do conteúdo. As sessões selecionam perguntas desse banco; não há geração diária automática.
- Perguntas de múltipla escolha e resposta curta; as respostas curtas são avaliadas com IA a partir da fonte e de critérios de correção.
- O usuário define dias de estudo, minutos por dia e idioma das perguntas (português ou idioma original do material).
- **Uma nova sessão por plano por dia.** Uma sessão incompleta pode ser retomada depois, inclusive em outro dia. Não abrir uma segunda sessão no dia em que uma sessão do mesmo plano for concluída.
- Cada sessão concluída avança uma casa. Erros não removem progresso. Um dia sem estudar não cria casa e desloca a previsão de conclusão.
- Revisão curta ao fim de cada região. Nomes exibidos em português: Clareira, Trilha, Serra, Caverna, Cume ao pôr do sol. Identificadores ingleses propostos para o código: `clearing`, `trail`, `ridge`, `cave`, `sunsetSummit`.
- O mascote é um pangolim. As imagens de origem em `design/mascot/` mantêm seus nomes históricos; cópias usadas em `public/mascot/` terão nomes ingleses, por exemplo `pangolin-neutral.png`.

## Estados e conceitos

- `StudyPlan`: configurações do estudo e estado `processing | ready | failed | completed`. Um usuário pode possuir vários planos, cada um com um material.
- `Material`: origem `pdf | pastedText`, caminho privado ou texto, estado de extração e metadados. Texto extraído é dividido em `SourceSection` com referência a página ou posição no texto.
- `Question`: tipo `multipleChoice | shortAnswer`, enunciado, resposta esperada, explicação, critérios de correção, idioma e referência verificável à `SourceSection`.
- `StudySession`: sequência dentro do plano e estado `inProgress | completed`. Contém `SessionQuestion` com ordem estável. Uma sessão iniciada permanece retomável até concluir.
- `AnswerAttempt`: resposta, avaliação, feedback e momento do envio. A primeira tentativa válida de cada card conta para a sessão; novas chamadas com a mesma chave de idempotência devolvem o mesmo resultado.
- `ReviewState`: próximo momento de revisão e histórico resumido por pergunta e plano.
- `UsageEvent`: operação, modelo, tokens de entrada/saída, custo estimado, plano e chamada relacionada. O custo é estimativa, não valor de faturamento.

## Regras das funcionalidades

### F01 — Conta e acesso ao piloto

1. O cadastro recebe apenas as vagas disponíveis no piloto; o limite inicial é configurável, com alvo de aproximadamente 50 pessoas.
2. A reserva de vaga deve ser atômica. Falha de criação da conta não pode consumir uma vaga permanentemente.
3. O usuário autenticado só acessa seus planos, materiais, sessões, perguntas e uso. Toda rota privada valida identidade e propriedade no servidor. Tabelas e bucket privado aplicam políticas de acesso coerentes.
4. A escolha entre login por senha e link de acesso ainda será fechada; a estrutura deve usar Supabase Auth sem acoplar regras de estudo ao método de login.

**Aceite:** um usuário não lê nem altera dados de outro; o cadastro bloqueia novas entradas ao atingir o limite sem criar vagas fantasmas.

### F02 — Material e criação do plano

1. O formulário aceita PDF com texto selecionável e até 100 páginas, ou texto colado. Rejeita documento vazio, tipo incompatível e PDF sem texto útil, explicando o motivo.
2. O PDF vai do navegador ao bucket privado por autorização temporária emitida pelo servidor. A criação do plano confirma que o objeto pertence ao usuário; o cliente não fornece um caminho arbitrário.
3. O usuário informa `durationDays`, `minutesPerDay` e `questionLanguage`. `durationDays` é a meta de dias com sessão concluída, não uma data fixa. A estimativa deve indicar se a duração parece insuficiente e sugerir mais dias antes da confirmação.
4. Salvar o fuso horário IANA do plano para determinar o dia local de suas sessões. O fuso pode começar com o valor informado pelo navegador; alteração posterior exige regra explícita, para não liberar sessões extras.
5. O plano passa a `processing` e mostra estado, erro e opção segura de tentar de novo.

**Aceite:** material privado, parâmetros persistidos, mensagens claras para PDFs inadequados e nenhum plano duplicado por repetição do pedido de criação.

### F03 — Preparar o banco de perguntas

1. Um Workflow recebe IDs de material/plano, lê o documento privado, extrai texto e registra seções com referência à origem. Não passa o documento inteiro como argumento do Workflow.
2. Distribui a geração por seções para cobrir o documento, evitando concentrar perguntas nas primeiras páginas. Gera quantidade compatível com sessões previstas e revisões, sujeita a limite de custo.
3. A IA devolve dados estruturados para `Question`; o servidor valida esquema e conteúdo. Cada pergunta precisa de resposta e referência à seção que a sustenta. Questões duplicadas, sem resposta sustentada ou com alternativa correta ambígua são descartadas ou refeitas dentro do orçamento.
4. O processamento é idempotente: repetição ou retomada do Workflow não duplica perguntas. Estado final é `ready` se houver banco utilizável, ou `failed` com motivo recuperável.
5. Não regenerar perguntas em cada visita ou a cada dia. Regeneração manual/expansão do banco fica para uma decisão futura.

**Aceite:** um texto de teste cobrindo início, meio e fim produz perguntas ligadas a suas seções; um processamento repetido não duplica registros.

### F04 — Montar e retomar a sessão diária

1. A regra é **por plano e por data local do plano**, não por conta inteira. Planos diferentes podem oferecer uma sessão cada no mesmo dia.
2. Se houver sessão `inProgress`, abrir essa sessão, mesmo que tenha começado em outro dia. Não criar outra enquanto ela existir.
3. Se uma sessão foi concluída na data local atual, mostrar o resultado e a próxima disponibilidade; não criar outra nessa data.
4. Na ausência das condições acima, criar no máximo uma sessão nova na data local. Requisições simultâneas ou repetidas recebem a mesma sessão.
5. Selecionar revisões vencidas e perguntas inéditas sem repetir a mesma pergunta duas vezes na sessão. A ordem dos cards e o estado da sessão são persistidos; reabrir a tela não remonta ou reinicia a sessão.
6. **Proposta de UX:** sessões concluídas podem ser revisitadas em modo leitura; revisitar não altera respostas, sequência, revisão ou tabuleiro.

**Aceite:** iniciar duas vezes no mesmo dia retorna a mesma sessão; fechar e reabrir retoma do card seguinte; terminar hoje uma sessão antiga impede abrir outra hoje.

### F05 — Responder e receber feedback

1. Um card por vez, com envio explícito. Persistir a resposta antes de avançar para o próximo card. O navegador mantém a resposta digitada até receber confirmação do servidor, permitindo reenviar após falha de rede.
2. Múltipla escolha é corrigida comparando o identificador da alternativa; nenhuma chamada à IA é necessária.
3. Resposta curta envia à IA apenas o contexto necessário: pergunta, resposta do usuário, resposta esperada, critério e trecho de origem. A avaliação retorna `correct | partial | incorrect`, explicação breve e sugestão de revisão. A resposta do usuário não altera a questão armazenada.
4. Em falha ou avaliação inconclusiva da IA, conservar a resposta e permitir nova tentativa de correção. Se já existir resultado persistido para a chave de idempotência, devolvê-lo sem nova chamada à IA. Se a chamada anterior tiver resultado incerto, registrar qualquer nova chamada efetivamente feita; não prometer custo zero em uma repetição externa. Não mostrar acerto inventado.
5. Feedback explica o raciocínio e aponta para trecho/página de origem quando disponível. O estado visual do pangolim acompanha acerto, erro ou alerta sem humilhar o usuário.

**Aceite:** questões objetivas são determinísticas; respostas curtas usam critério/fonte; falha de rede não perde uma resposta enviada; repetir uma chamada não duplica tentativas.

### F06 — Conclusão, revisão e tabuleiro

1. A sessão só pode concluir quando seus cards têm resultados persistidos. Concluir é idempotente e avança **exatamente uma casa**.
2. A agenda de revisão é atualizada conforme o resultado: erros e respostas parciais voltam antes; acertos espaçam a próxima revisão. Os intervalos exatos ficam em configuração e precisam de calibração no piloto.
3. A região visual depende do número de perguntas inéditas estudadas. Revisões repetidas não aceleram a mudança de região. Erros não causam retrocesso.
4. A sessão que encerra uma região inclui revisão curta desse conteúdo. Ao faltar um dia, a quantidade de sessões restantes não cai; a data prevista de conclusão é recalculada.
5. A sequência de estudo conta dias locais com sessão concluída. Retomar uma sessão antiga conta no dia em que ela for concluída.

**Aceite:** completar duas vezes não avança duas casas; revisões não inflam a contagem inédita; faltar um dia muda apenas a previsão, não o progresso já obtido.

### F07 — Uso da IA e controle de custo

1. A chave da OpenAI fica apenas em variável de ambiente do servidor, nunca no navegador, banco ou repositório.
2. Registrar por chamada o modelo, operação (`questionGeneration | shortAnswerGrading`), tokens reportados, custo estimado e estado. Não registrar o texto integral dos materiais ou respostas em logs de aplicação.
3. Limitar tamanho e quantidade de chamadas por material e por usuário. Atingir um limite gera estado/feedback visível, sem consumo ilimitado.
4. Modelo e tabela de preços usados na estimativa ficam configuráveis; não fixar preço de API no código de negócio.

**Aceite:** cada geração/correção bem-sucedida tem evento de uso; o total pode ser agregado por plano; uma falha não é apresentada como custo exato de faturamento.

## Interfaces e rotas

As URLs e a árvore de pastas estão em [architecture.md](architecture.md). Contratos mínimos de endpoints:

| Endpoint | Entrada principal | Saída principal |
| --- | --- | --- |
| `POST /api/auth/signup` | Dados de cadastro | Conta admitida ou limite atingido. |
| `POST /api/materials/upload-intent` | Nome, tipo e tamanho do PDF | Caminho permitido e token temporário. |
| `POST /api/plans` | Material, `durationDays`, `minutesPerDay`, `questionLanguage`, `timeZone` | `planId`, estado `processing`. |
| `GET /api/plans/[planId]/status` | `planId` | Estado de processamento e erro tratável. |
| `POST /api/plans/[planId]/sessions` | `planId` | Sessão nova, retomada ou próxima disponibilidade. |
| `POST /api/sessions/[sessionId]/answers` | Card, resposta e chave de idempotência | Resultado e feedback. |
| `POST /api/sessions/[sessionId]/complete` | `sessionId` e chave de idempotência | Casa alcançada e próximo dia disponível. |

Rotas de interface: `/`, `/login`, `/signup`, `/app`, `/app/plans/new`, `/app/plans/[planId]` e `/app/plans/[planId]/sessions/[sessionId]`. Não expor IDs de outros usuários mesmo quando conhecidos.

## Ordem sugerida de implementação

1. **Base:** projeto Next.js, Supabase Auth, políticas de acesso, migrações, estados e estrutura de módulos.
2. **Primeiro fluxo completo com texto:** cadastro → plano com texto colado → Workflow → banco de perguntas → sessão → respostas → avanço; medição de uso desde a primeira chamada à IA.
3. **PDF privado:** upload direto ao Supabase, extração, referências de página e erros de arquivo.
4. **Jogo e retenção:** tabuleiro do vulcão, pangolim, revisões curtas, cálculo da região, sequência e previsão de conclusão.
5. **Validação do piloto:** acessibilidade e celular, limites de custo, recuperação de falhas e testes dos invariantes principais.

Verificação mínima: testes das regras de dia local/retomada, idempotência de conclusão, seleção de revisão e propriedade dos dados; teste de fluxo real com texto e PDF. Não criar testes que só reproduzam a implementação.

## Parâmetros ainda abertos

- Método de login, mecanismo exato da admissão e tamanho máximo em MB para PDF/texto colado.
- Fórmula de estimativa de carga e número de perguntas por sessão, por dia e por região.
- Proporção de inéditas/revisões e intervalos da revisão espaçada.
- Tolerância para respostas `partial`, limites de custo e política de reprocessamento.
- Experiência final para plano concluído e eventual revisão depois do cume.
- Como representar as cinco regiões em planos com menos de cinco sessões; definir duração mínima ou adaptar o mapa.

Esses parâmetros devem ser configuráveis ou resolvidos antes do trecho da implementação que depende deles; não devem ser escondidos em números espalhados pelo código.

## Fontes técnicas

- [OpenAI Docs: Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) para respostas estruturadas e tratamento de saídas incompletas/recusas; [contagem de tokens](https://developers.openai.com/api/docs/guides/token-counting) para medição de uso.
- [Next.js: Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers) e [route groups](https://nextjs.org/docs/app/api-reference/file-conventions/route-groups).
- [Supabase: upload com autorização temporária](https://supabase.com/docs/reference/javascript/file-buckets-createsigneduploadurl) e [armazenamento privado](https://supabase.com/docs/guides/storage/serving/downloads).
- [Vercel: Workflow com Next.js](https://vercel.com/academy/workflow-foundations/set-up-the-pizza-tracker).
