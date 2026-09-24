# Daylin — proposta de funcionalidades e estrutura técnica

**Status: proposta técnica para discussão.** As decisões de produto estão em [design.md](design.md), e as regras detalhadas para implementação em [spec.md](spec.md). Nenhuma pasta de código ou rota foi implementada.

## Princípio de organização

Um **monólito modular** em Next.js: interface, rotas HTTP e fluxo assíncrono no mesmo projeto/deploy da Vercel. As áreas de negócio abaixo são módulos de código, não serviços ou deploys independentes. Supabase e OpenAI são serviços externos.

No App Router, `page.tsx` define uma tela, `route.ts` define um endpoint e diretórios entre parênteses organizam telas sem entrar na URL. A lógica do produto deve ficar fora das páginas e dos endpoints, em `src/features/`.

## Funcionalidades do primeiro piloto

| Ordem | Funcionalidade | Resultado para o usuário |
| --- | --- | --- |
| 1 | Cadastro, login e limite de vagas do piloto | Entra em uma conta própria e vê apenas seus estudos. |
| 2 | Criar plano com texto colado ou PDF privado | Envia um material, escolhe idioma, duração e minutos por dia. |
| 3 | Processamento assíncrono e banco inicial de perguntas | Acompanha o estado do material até o plano ficar pronto. |
| 4 | Sessão diária com cards | Responde múltipla escolha e resposta curta, recebe feedback e pode retomar sessão interrompida. |
| 5 | Revisões e tabuleiro | Revê perguntas no momento adequado, conclui a sessão e sobe uma casa do vulcão. |
| 6 | Medição de uso da IA | O projeto registra tokens e custo estimado para analisar o piloto. |

Uma primeira entrega interna pode usar **texto colado** para validar geração, sessão e correção antes de adicionar o upload de PDF; isso não muda o escopo final do piloto.

## Domínios de negócio

| Módulo | Responsabilidade | Dados principais |
| --- | --- | --- |
| `auth` | Cadastro, login, identidade, fuso horário e limite de vagas. | Perfil, vaga do piloto. |
| `materials` | Receber PDF/texto, validar, guardar privado e extrair trechos com referência à fonte. | Material, trecho de origem, estado do processamento. |
| `plans` | Preferências de estudo, estimativa de carga, prazo e estado do plano. | Plano de estudo. |
| `questions` | Gerar e validar banco inicial; guardar alternativas, resposta, critério de correção e origem. | Pergunta, referência ao trecho. |
| `sessions` | Montar sessão com perguntas novas e revisões, registrar respostas e concluir uma casa. | Sessão, itens da sessão, tentativas. |
| `progress` | Agendar revisões, contar perguntas inéditas, calcular região, sequência e conclusão prevista. | Estado de revisão e projeção do progresso. |
| `usage` | Medir chamadas à IA, tokens, modelo e custo estimado. | Evento de uso. |

Esses módulos não precisam de pacotes, serviços HTTP internos ou abstrações genéricas. Cada um pode começar com poucos arquivos: `schema.ts`, `service.ts`, `repository.ts` e componentes próprios quando houver interface.

## Fluxos e estados

### Criar um plano

1. Após autenticação, o usuário informa material e preferências do plano.
2. Para PDF, o servidor autoriza um caminho específico em um bucket privado; o navegador envia o arquivo diretamente ao Supabase Storage. O PDF não atravessa o corpo de uma função Next.js. Para texto colado, o servidor recebe e valida o tamanho.
3. O servidor grava material e plano como `processing` e inicia um Workflow com seus IDs, nunca com o documento inteiro no argumento.
4. O Workflow baixa/lê o material privado, extrai e divide o texto em trechos, estima a carga, cria o banco inicial com cobertura ao longo do documento e grava as referências de origem.
5. O plano passa a `ready` ou `failed`. A tela de preparação consulta o estado para mostrar progresso ou erro recuperável.

O banco inicial deve conter perguntas **suficientes para o plano e as revisões previstas**, sem tentar esgotar todas as perguntas possíveis do documento. O número exato será calibrado no piloto. Evitar nova geração diária por padrão.

### Estudar

1. Ao abrir o plano pronto, o servidor retoma a sessão inacabada, se existir. Pode abrir **no máximo uma sessão nova por plano por dia local**, e não abre outra no mesmo dia após uma conclusão.
2. A seleção mistura perguntas inéditas e revisões vencidas; cada resposta é persistida imediatamente.
3. Múltipla escolha é corrigida de forma determinística. Resposta curta é avaliada pela IA com a pergunta, resposta esperada, critério e trecho de origem. O resultado inclui feedback compreensível.
4. Ao concluir, o sistema atualiza revisão, perguntas inéditas, casa do tabuleiro, região e previsão de término em uma operação consistente.
5. A revisão curta de fim de região entra na própria sessão de fronteira, mantendo a regra de **uma casa por sessão concluída**.

O progresso do mapa depende de sessões concluídas. A passagem de região depende das perguntas inéditas estudadas. Erros não retiram casas ou progresso; influenciam a revisão futura. Sessões incompletas podem ser retomadas; sessões concluídas podem ser revisitadas apenas para leitura.

## Rotas de interface propostas

| URL | Tela |
| --- | --- |
| `/` | Apresentação breve do produto e chamada para começar. |
| `/login` e `/signup` | Autenticação e entrada no piloto. |
| `/app` | Planos do usuário e acesso à sessão atual. |
| `/app/plans/new` | Envio do material e configuração do plano. |
| `/app/plans/[planId]` | Tabuleiro, estado do processamento e progresso do plano. |
| `/app/plans/[planId]/sessions/[sessionId]` | Cards da sessão, feedback, conclusão e consulta posterior. |

O resultado da sessão pode aparecer na própria tela de estudo; não precisa de rota própria no MVP. As páginas privadas validam sessão e propriedade do plano no servidor.

## Endpoints HTTP propostos

| Método e URL | Uso |
| --- | --- |
| `POST /api/auth/signup` | Reservar vaga de piloto e cadastrar conta sem ultrapassar o limite. |
| `POST /api/materials/upload-intent` | Validar PDF e devolver autorização temporária para upload privado direto ao Supabase. |
| `POST /api/plans` | Criar plano a partir de texto ou PDF já enviado e disparar processamento. |
| `GET /api/plans/[planId]/status` | Consultar estado `processing`, `ready` ou `failed`. |
| `POST /api/plans/[planId]/sessions` | Criar ou retomar a sessão disponível. |
| `POST /api/sessions/[sessionId]/answers` | Registrar uma tentativa e devolver feedback. |
| `POST /api/sessions/[sessionId]/complete` | Finalizar a sessão e atualizar o avanço. |

Leituras para páginas privadas podem ir diretamente do componente de servidor ao Supabase, sem criar um endpoint GET duplicado para cada tela. Todos os endpoints verificam a identidade e a propriedade do recurso; o cliente não escolhe `userId` ou caminho de armazenamento arbitrário. A correção de resposta e a conclusão precisam ser idempotentes para não duplicar tentativas, casas ou custos ao repetir uma requisição.

## Estrutura de pastas proposta

```text
/
├── AGENTS.md
├── design.md
├── architecture.md
├── spec.md
├── design/mascot/                 # source artwork and prompts; not public
├── public/mascot/                 # future runtime copies with English names
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                # /
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── signup/page.tsx
│   │   ├── (private)/
│   │   │   └── app/
│   │   │       ├── layout.tsx      # requires authentication
│   │   │       ├── page.tsx        # /app
│   │   │       └── plans/
│   │   │           ├── new/page.tsx
│   │   │           └── [planId]/
│   │   │               ├── page.tsx
│   │   │               └── sessions/[sessionId]/page.tsx
│   │   └── api/
│   │       ├── auth/signup/route.ts
│   │       ├── materials/upload-intent/route.ts
│   │       ├── plans/
│   │       │   ├── route.ts
│   │       │   └── [planId]/
│   │       │       ├── status/route.ts
│   │       │       └── sessions/route.ts
│   │       └── sessions/[sessionId]/
│   │           ├── answers/route.ts
│   │           └── complete/route.ts
│   ├── features/
│   │   ├── auth/
│   │   ├── materials/
│   │   ├── plans/
│   │   ├── questions/
│   │   ├── sessions/
│   │   ├── progress/
│   │   └── usage/
│   ├── workflows/
│   │   └── process-material.ts
│   ├── lib/
│   │   ├── supabase/              # browser, server, and internal clients
│   │   └── openai/                # API client and configuration
│   ├── components/ui/            # shared UI without business rules
│   └── proxy.ts                   # Supabase session refresh
└── supabase/migrations/          # schema and access policies
```

`src/app/` descreve URLs e compõe telas; `src/features/` contém regras de negócio; `src/lib/` concentra integrações externas; `src/workflows/` coordena o processamento. O Workflow chama serviços dos módulos, sem duplicar regras. Os sprites só iriam a `public/mascot/` quando a interface fosse implementada, com nomes como `pangolin-neutral.png`. Todo código, nome de arquivo de código, identificador e URL deve estar em inglês; os textos mostrados ao usuário podem estar em português.

## Modelo de dados inicial, ainda sujeito a ajuste

`profiles`, `materials`, `source_sections`, `study_plans`, `questions`, `study_sessions`, `session_questions`, `attempts`, `review_states` e `usage_events`. Uma tabela ou função de admissão controlaria o limite de vagas de forma atômica. Todas as tabelas com dados do usuário terão regras de acesso por proprietário; arquivos ficam em bucket privado. O processo interno usa credenciais de servidor apenas onde necessário.

## Pontos para fechar antes da implementação

1. **Fuso horário:** como definir e alterar o dia local de cada plano sem permitir liberar sessões extras acidentalmente?
2. **Admissão:** cadastro aberto até o limite ou convites para os primeiros participantes? O documento de design registra cadastro aberto como intenção atual.
3. **Carga de perguntas:** quantas por minuto de estudo, qual proporção de revisão e quantas perguntas inéditas acionam cada região?
4. **Correção:** qual nível de tolerância para respostas curtas e como tratar avaliação inconclusiva da IA?
5. **Limites e falhas:** tamanho máximo de arquivo/texto, orçamento de processamento por usuário e experiência de reprocessamento quando o material falhar.

## Referências técnicas consultadas

- [Next.js App Router: páginas e layouts](https://nextjs.org/docs/app/getting-started/layouts-and-pages), [grupos de rotas](https://nextjs.org/docs/app/api-reference/file-conventions/route-groups) e [Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers).
- [Supabase Auth para SSR](https://supabase.com/docs/guides/auth/server-side/creating-a-client?framework=nextjs&package-manager=npm&queryGroups=framework&queryGroups=package-manager), [upload com URL assinada](https://supabase.com/docs/reference/javascript/file-buckets-createsigneduploadurl) e [arquivos privados](https://supabase.com/docs/guides/storage/serving/downloads).
- [Vercel Workflows](https://vercel.com/workflows) e [integração com Next.js](https://vercel.com/academy/workflow-foundations/set-up-the-pizza-tracker).
