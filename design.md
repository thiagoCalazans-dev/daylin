# Daylin — decisões de produto e design

Estado: **planejamento do MVP**. Este documento registra o que foi decidido até agora e as perguntas que ainda precisam de resposta. Não há aplicativo implementado.

## Visão

Daylin será um jogo de microlearning para qualquer tema. A pessoa envia algo que precisa estudar, a IA prepara perguntas baseadas nesse conteúdo e o aplicativo organiza sessões curtas ao longo do período escolhido. A mensagem central é: **o importante é estudar com constância, não com velocidade**.

## Escopo do MVP

| Área | Decisão atual |
| --- | --- |
| Plataforma | Aplicativo web responsivo. |
| Piloto | Contas individuais, cadastro aberto inicialmente para um grupo pequeno, por volta de 50 usuários. |
| Material | Um material por plano de estudo: PDF com texto selecionável de até 100 páginas ou texto colado. Sem OCR no MVP. |
| Planejamento | O usuário informa o período em dias e os minutos disponíveis por dia. O sistema sugere aumentar o período se o tempo não comportar o material. |
| Idioma | O usuário escolhe perguntas em português ou no idioma original do material. |
| Perguntas | Múltipla escolha e resposta curta, com feedback e explicação baseados na fonte. Respostas curtas são avaliadas por IA com critérios de correção. |
| Entrega | Perguntas dentro do aplicativo, sem envio por e-mail ou WhatsApp no MVP. |
| Progresso | Uma nova sessão por plano por dia, com retomada de sessão incompleta. Avanço no tabuleiro e sequência de estudo; dias perdidos deslocam a conclusão prevista. |

## Fluxo de estudo

1. O usuário cria a conta, envia um PDF ou cola o texto e escolhe idioma, dias e minutos por dia.
2. O aplicativo extrai o texto e processa o material de forma assíncrona.
3. A IA prepara **um banco inicial de perguntas no processamento do envio**. A proposta não é criar perguntas novas todos os dias. Cada pergunta deve estar ligada ao trecho do material usado para formulá-la e ter resposta/critério de correção.
4. A sessão do dia seleciona perguntas ainda não vistas e revisões do banco, conforme o progresso. O usuário responde um card por vez e recebe feedback imediato.
5. Uma sessão incompleta pode ser retomada depois. Ao concluí-la, o usuário avança uma casa no tabuleiro e não abre outra sessão do mesmo plano nesse dia. Se não estudar em um dia, não surge uma casa vazia: a conclusão prevista apenas muda.

A revisão espaçada faz parte da proposta do MVP; o intervalo exato e a proporção entre perguntas novas e revisões ainda precisam de definição.

## Tabuleiro e narrativa visual

O tabuleiro é uma **subida lúdica de um vulcão**. Cada casa representa uma sessão de estudo, e o caminho concluído mostra visualmente o avanço. A paisagem muda de tempos em tempos conforme a quantidade de **perguntas inéditas estudadas**, sem depender de acertos ou erros. O número exato de perguntas por região ainda está em aberto. Erros geram feedback e revisão, mas nunca fazem o personagem voltar casas.

| Etapa | Clima e significado |
| --- | --- |
| **Clareira** | Base verde e ensolarada: convite para começar. |
| **Trilha** | Caminho de terra em direção à montanha: primeiros passos da subida. |
| **Serra** | Pedras, vento e nuvens: a jornada exige mais esforço. |
| **Caverna** | Interior do vulcão com cristais e brilho de lava: ponto de maior intensidade, ainda lúdico. |
| **Cume ao pôr do sol** | Céu rosa e dourado, vista ampla: sensação de conquista e chegada. |

Ao fim de cada região, haverá uma **revisão curta** do conteúdo estudado ali. Um desafio final na saída da caverna foi sugerido, mas seu formato ainda não foi definido.

Dentro de cada casa/sessão, as perguntas aparecem como **cards de estudo em pixel art**. O cenário acompanha a região atual. A interface deve ser colorida, acolhedora e legível no celular, com espírito de aventura inspirado de modo amplo por jogos como Zelda e Celeste, mas com personagens e arte originais.

## Mascote

O mascote escolhido é um **pangolim**: ele avança sem pressa, e suas escamas podem reforçar visualmente a ideia de conhecimento acumulado. Seu desenho atual tem rosto bege-mel, escamas verde-oliva e esmeralda, focinho alongado, cauda escamada, óculos quadrados pretos, lenço laranja e tênis azul-marinho. A expressão deve ser curiosa e encorajadora. Um erro é uma oportunidade de aprender, sem punição visual.

Sprites existentes, em PNG transparente:

- `design/mascot/pangolim-neutro.png`
- `design/mascot/pangolim-acerto.png`
- `design/mascot/pangolim-erro.png`
- `design/mascot/pangolim-alerta.png`
- `design/mascot/pangolim-combo.png`
- `design/mascot/pangolim-ofensiva.png` — pose de desafio de aprendizado com lápis luminoso, sem violência.

Os prompts usados estão em `design/mascot/pangolim-prompts.md`. Os arquivos `tartaruga-*.png` são estudos anteriores, substituídos pelo pangolim.

## Arquitetura pretendida para o piloto

A proposta detalhada de funcionalidades, domínios, rotas e pastas está em [architecture.md](architecture.md). As regras das features e a ordem de implementação estão em [spec.md](spec.md). Nomes técnicos e rotas serão em inglês; os textos da interface podem estar em português.

- **Next.js na Vercel:** interface, rotas de servidor e um único projeto/deploy da aplicação.
- **Vercel Workflow:** processamento assíncrono do documento e criação do banco de perguntas, sem aplicação worker em deploy separado.
- **Supabase:** autenticação, banco de dados e armazenamento privado dos materiais.
- **OpenAI API:** criação de perguntas e avaliação de respostas curtas. A chave do projeto ficará somente em variáveis de ambiente do servidor.
- **Medição de uso:** registrar tokens e custo estimado por processamento de material e por correção para orientar a futura precificação.

“Um único deploy” refere-se à aplicação Next.js; Supabase e OpenAI continuam serviços externos. A intenção é validar o MVP com custo baixo, verificando limites dos planos gratuitos e o consumo real da API durante o piloto.

## Fora do MVP inicial

- OCR para PDFs escaneados.
- Múltiplos materiais no mesmo plano.
- Notificações por e-mail, WhatsApp ou push.
- Chave OpenAI fornecida por cada usuário ou IA hospedada pelo próprio projeto.
- Sistema extenso de XP, níveis e recompensas.
- Geração diária de um banco inteiramente novo de perguntas.

## Questões em aberto

- Público principal do piloto: estudantes, profissionais ou ambos.
- Quantidade de perguntas por sessão, por região e por revisão; regra precisa da revisão espaçada.
- Como estimar a carga do material para sugerir dias e minutos de estudo.
- Limites de tamanho para texto colado e para armazenamento por usuário.
- Critérios de qualidade das perguntas e como mostrar o trecho de origem no feedback.
- Nome do pangolim e detalhes finais do tabuleiro, cards e estados visuais.
- Política de retenção/remoção dos materiais e limites de custo da API para o piloto.
