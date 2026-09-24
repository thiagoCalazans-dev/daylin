# Orientações para agentes — Daylin

Este repositório está na fase de definição do produto. Leia [design.md](design.md), [architecture.md](architecture.md) e [spec.md](spec.md) antes de propor mudanças. **Não iniciar a implementação sem pedido explícito do usuário.** Se ele pedir implementação em uma conversa futura, a instrução mais recente prevalece.

## Objetivo

Criar um jogo de microlearning para estudar qualquer tema a partir de material enviado pelo usuário. A experiência deve valorizar a constância: sessões curtas fazem o personagem avançar por um tabuleiro que sobe um vulcão lúdico.

## Decisões que devem orientar propostas futuras

- MVP web responsivo, com contas individuais e piloto pequeno (aproximadamente 50 inscrições no início).
- Um material por plano: PDF com texto selecionável, até 100 páginas, ou texto colado. OCR fica fora do MVP.
- Gerar um banco de perguntas durante o processamento do material enviado; selecionar perguntas desse banco nas sessões seguintes, com revisão espaçada. Não gerar um lote novo diariamente por padrão.
- Misturar múltipla escolha e resposta curta. Avaliar respostas curtas com IA, usando o material e critérios de correção.
- Usuário escolhe dias de estudo, minutos por dia e idioma das perguntas (português ou idioma original do material). Sugerir mais dias quando o tempo informado não couber no conteúdo. Dias perdidos adiam a previsão de conclusão.
- Perguntas e progresso ficam no aplicativo; notificações por e-mail ou WhatsApp não fazem parte do MVP.
- Arquitetura pretendida para o piloto: um projeto Next.js com um único deploy na Vercel, Vercel Workflow para processamento assíncrono, Supabase para autenticação/banco/armazenamento privado e OpenAI API para geração e correção. Não há worker Node em deploy separado.
- Uma chave de API da OpenAI do projeto, guardada como segredo no servidor, será usada no piloto. Registrar uso de tokens e custo estimado por material e por correção. Não pedir nem registrar a chave em arquivos, prompts ou mensagens.
- O mascote atual é um **pangolim** em pixel art. Seus seis sprites estão em `design/mascot/pangolim-*.png`; os prompts estão em `design/mascot/pangolim-prompts.md`. Os sprites `tartaruga-*.png` são versões anteriores e não devem orientar novas propostas de mascote.
- O tabuleiro sobe, em ordem: **Clareira → Trilha → Serra → Caverna → Cume ao pôr do sol**. Cada casa representa uma sessão de estudo concluída. Erros não fazem o usuário retroceder.
- Existe no máximo uma nova sessão por plano por dia local. Uma sessão incompleta pode ser retomada em outro dia; concluí-la impede iniciar outra no mesmo dia.
- Todo código, identificador, nome de arquivo de código, pasta técnica, tabela, campo JSON, rota e endpoint deve estar em inglês. A documentação e a interface podem estar em português. Imagens históricas em `design/mascot/` mantêm seus nomes; cópias para uso no app devem receber nomes ingleses.

## Como manter a documentação

- Distinguir decisões confirmadas de ideias ainda abertas. Não transformar uma sugestão em requisito sem confirmação do usuário.
- Manter `design.md`, `architecture.md` e `spec.md` coerentes quando uma decisão mudar, distinguindo regra confirmada de proposta técnica.
- Preservar os arquivos de arte já criados. Ao explorar novas versões, salvar com novos nomes, sem sobrescrever as imagens existentes sem pedido explícito.
- Manter o conteúdo em português claro e separar visão do produto, experiência, arquitetura e pendências.
