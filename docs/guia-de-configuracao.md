# Guia de configuração e explicação dos nós

## Parte 1 · Importar o workflow

1. No n8n Cloud, crie um workflow novo (botão **Create workflow**).
2. No menu de três pontos (**⋯**) no canto superior direito, escolha **Import from File...** e selecione `workflow/barbearia-agendamento.json`.
3. O canvas vai mostrar 10 nós e 2 notas. Os nós do Gemini e do Google Calendar vão aparecer com aviso de credencial faltando; isso é esperado.
4. Salve (**Ctrl+S** ou **Cmd+S**).

Atalho: você também pode abrir o JSON num editor de texto, copiar tudo e colar direto no canvas com **Ctrl+V**.

## Parte 2 · Credenciais

### 2.1 Gemini (modelo de IA)

1. Acesse [Google AI Studio](https://aistudio.google.com/) com sua conta Google.
2. Clique em **Get API key** e depois em **Create API key**. Copie a chave.
3. No n8n, abra o nó **Google Gemini Chat Model**.
4. Em **Credential to connect with**, escolha **Create new credential**. O tipo é *Google Gemini(PaLM) Api*.
5. Deixe o **Host** como está (`https://generativelanguage.googleapis.com`), cole a chave em **API Key** e salve. O n8n testa a conexão na hora.
6. Ainda no nó, confira o campo **Model**. O workflow vem com `gemini-3.5-flash`. Se ele não aparecer na lista, escolha outro modelo **Flash** (evite as versões *Lite*: elas falham com mais frequência ao chamar ferramentas).
7. Abra o nó **Gemini reserva** e selecione a **mesma credencial**. Ele vem com `gemini-3-flash-preview`.

**Sobre a cota gratuita:** sem faturamento ativo, a API do Gemini permite poucas chamadas por dia para cada modelo (nos meus testes, 20). Cada mensagem do cliente gasta de 2 a 4 chamadas. Para testar à vontade ou gravar demonstrações, ative o faturamento no AI Studio; o custo dos modelos Flash fica em centavos.

Guarde a chave fora do repositório. Ela fica salva só dentro do n8n, e o JSON exportado não leva credenciais.

### 2.2 Google Calendar

**Recomendado antes de começar:** no [Google Calendar](https://calendar.google.com/), crie uma agenda separada chamada "Doctor Barber" (Outras agendas → **+** → Criar nova agenda). Assim seus compromissos pessoais não bloqueiam horários da barbearia, e os testes não bagunçam sua agenda principal.

1. No n8n, abra o nó **Verificar disponibilidade**.
2. Em **Credential to connect with**, escolha **Create new credential** (tipo *Google Calendar OAuth2 API*).
3. No n8n Cloud aparece o botão **Sign in with Google**. Clique, escolha sua conta e autorize o acesso à agenda.
   - Se em vez do botão o n8n pedir *Client ID* e *Client Secret*, siga a [documentação de OAuth do Google no n8n](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/).
4. No campo **Calendar**, selecione a agenda "Doctor Barber" (ou a principal, se preferir).
5. Faça o mesmo nos nós **Criar agendamento**, **Buscar agendamentos do cliente** e **Cancelar agendamento**: **mesma credencial** (ela já aparece na lista) e **mesma agenda**.

As quatro ferramentas precisam apontar para a mesma agenda. Se uma consultar uma agenda e outra criar em outra, o agente não enxerga os próprios agendamentos.

### 2.3 Testar

1. Salve o workflow.
2. Clique em **Open chat** na parte de baixo do canvas e mande "Oi".
3. Enquanto o agente responde, os nós acendem no canvas. Quando ele usa uma ferramenta, o nó da ferramenta fica verde.
4. Para ver o que aconteceu em cada mensagem, abra a aba **Executions**.

Para testar no editor não é preciso publicar o workflow. Publicar só é necessário se você quiser um link público do chat (opção *Make Chat Publicly Available* no nó de gatilho).

## Parte 3 · O que cada nó faz

O workflow tem um nó principal (o agente), sete nós "plugados" nele e um nó final que garante a resposta. Os plugados não rodam em sequência: o agente chama cada um quando precisa.

```
Mensagem do cliente ──► Recepcionista (AI Agent) ──► Garantir resposta ──► chat
                           ▲   ▲   ▲      ▲   ▲   ▲   ▲
          Gemini ──────────┘   │   │      │   │   │   └── Cancelar agendamento
          Gemini reserva ──────┘   │      │   │   └────── Buscar agendamentos do cliente
          Memória ─────────────────┘      │   └────────── Criar agendamento
                                          └────────────── Verificar disponibilidade
```

### Mensagem do cliente (Chat Trigger)

- **O que faz:** é a porta de entrada. Cada mensagem digitada no chat dispara uma execução do workflow e entrega dois dados principais: `chatInput` (o texto) e `sessionId` (um identificador da conversa).
- **Por que está aqui:** sem gatilho o workflow não roda. Ele também fornece o chat pronto do n8n, então não precisei construir interface.
- **Configuração:** modo de resposta *When Last Node Finishes*, ou seja, o chat mostra o que o último nó (*Garantir resposta*) devolver.
- **Para alterar:** marcando *Make Chat Publicly Available*, o chat ganha um link público ou pode ser embutido num site.

### Recepcionista (AI Agent)

- **O que faz:** é o cérebro do fluxo. Recebe a mensagem, junta com o prompt de sistema e o histórico, manda tudo para o modelo e decide, a partir da resposta do modelo, se precisa chamar alguma ferramenta antes de responder.
- **Por que está aqui:** o pedido do cliente vem em linguagem livre ("quero cortar o cabelo quarta de tarde"). Um fluxo fixo de perguntas não lida bem com isso; o agente entende, pergunta só o que falta e usa as ferramentas na ordem certa.
- **Configuração:**
  - *System Message*: o prompt com os dados da barbearia e as regras (veja `prompt-do-agente.md`).
  - *Max Iterations* = 10: limite de idas e vindas entre modelo e ferramentas numa mesma mensagem, para evitar loop infinito.
  - *Enable Streaming* desligado, combinando com o modo de resposta do gatilho.
  - *Enable Fallback Model* ligado: se o modelo principal der erro, o agente usa o *Gemini reserva*.
  - Em *Settings*, *Retry On Fail* ligado (2 tentativas, 2 s de intervalo), para erros temporários da API.
  - Em *Settings*, *On Error* = *Continue*: se o modelo falhar de vez, o fluxo segue para *Garantir resposta*, e o cliente recebe uma mensagem amigável em vez de "Error in workflow".
- **Para alterar:** quase tudo de comportamento se muda no prompt, sem mexer em nó nenhum.

### Google Gemini Chat Model

- **O que faz:** é o modelo de linguagem que lê a conversa e gera as respostas e os pedidos de uso de ferramenta.
- **Por que está aqui:** o nó AI Agent não tem modelo próprio; você pluga o que quiser. Escolhi Gemini porque a API tem camada gratuita.
- **Configuração:** temperatura 0.3. Quanto mais baixa, mais previsível; para agendamento isso é melhor do que criatividade.
- **Para alterar:** dá para trocar por OpenAI, Anthropic, Groq etc. apagando este nó e plugando outro no mesmo encaixe *Chat Model*. O resto do workflow não muda.

### Gemini reserva

- **O que faz:** é um segundo modelo (`gemini-3-flash-preview`) que o agente usa quando o principal falha.
- **Por que está aqui:** nos testes, o modelo principal devolveu erro 503 ("alta demanda") em horário de pico e erro 429 (cota diária esgotada). Cada modelo tem cota e fila próprias, então a reserva mantém o atendimento funcionando.

### Memória da conversa (Simple Memory)

- **O que faz:** guarda as últimas 20 trocas de mensagem de cada conversa, separadas pelo `sessionId` que vem do gatilho.
- **Por que está aqui:** cada mensagem é uma execução nova do workflow. Sem memória, o agente esqueceria o nome do cliente assim que ele mandasse o horário.
- **Para alterar:** *Context Window Length* define quantas trocas ficam. Para produção, trocaria por uma memória em banco (nó *Postgres Chat Memory* ou *Redis Chat Memory*), porque a Simple Memory vive na memória do servidor do n8n e pode se perder num reinício.

### Verificar disponibilidade (Google Calendar Tool)

- **O que faz:** pergunta ao Google Calendar se existe algum evento entre um horário de início e um de fim. Devolve `available: true` ou `available: false`.
- **Por que está aqui:** é o que impede marcar dois clientes no mesmo horário. O agente usa antes de oferecer um horário e de novo logo antes de criar o evento.
- **Como os campos são preenchidos:** `inicio` e `fim` usam `$fromAI(...)`. Isso diz ao n8n: "este valor quem decide é o modelo". O texto dentro do `$fromAI` explica ao modelo o formato esperado (ISO 8601 com -03:00).
- **Para alterar:** a *Description* da ferramenta é o que o modelo lê para decidir quando usá-la. Se o agente estiver usando a ferramenta na hora errada, ajuste esse texto.

### Criar agendamento (Google Calendar Tool)

- **O que faz:** cria o evento na agenda com início, fim, título e descrição.
- **Por que está aqui:** é a ação final do atendimento.
- **Como os campos são preenchidos:** `inicio`, `fim`, `titulo` ("Corte + barba - João Silva") e `descricao` vêm do modelo via `$fromAI`. No fim da descrição, o n8n acrescenta sozinho a linha `Código da conversa: <sessionId>`, que o modelo não controla.
- **Proteção:** a descrição da ferramenta e o prompt mandam usá-la só depois de confirmação explícita do cliente.

### Buscar agendamentos do cliente (Google Calendar Tool)

- **O que faz:** lista os eventos futuros da agenda que têm o código desta conversa na descrição. Devolve id, título, início e fim de cada um.
- **Por que está aqui:** para cancelar, remarcar ou conferir, o agente precisa achar o evento certo.
- **Proteção:** o filtro de busca (`query`) é o `sessionId` vindo do gatilho, fixado no nó, e não um campo preenchido pelo modelo. Assim o agente só enxerga os agendamentos feitos nesta conversa e não consegue mexer no horário de outro cliente, mesmo que alguém digite o nome dessa pessoa.
- **Detalhe técnico:** o Gemini exige pelo menos um campo preenchido pelo modelo em cada ferramenta. Por isso o fim do período de busca (`ate`) usa `$fromAI`.

### Cancelar agendamento (Google Calendar Tool)

- **O que faz:** apaga um evento da agenda pelo ID.
- **Por que está aqui:** completa o atendimento (cancelar) e é a segunda metade de remarcar, que é "criar o novo e depois apagar o antigo".
- **Proteção:** o prompt só deixa usar IDs que vieram de *Buscar agendamentos do cliente*, e só depois de o cliente confirmar.

### Garantir resposta (Edit Fields)

- **O que faz:** copia a resposta do agente para o chat. Se a resposta vier vazia, troca por uma mensagem padrão pedindo para o cliente mandar "conferir meu agendamento".
- **Por que está aqui:** às vezes o Gemini falha ao montar uma chamada de ferramenta e devolve texto vazio. Sem este nó, o cliente via `{ "output": "" }` no chat.
- **Como funciona:** a expressão `($json.output || '').trim() || 'mensagem padrão'` usa a resposta do agente se ela tiver texto e a mensagem padrão se não tiver.

### As duas notas no canvas

São *Sticky Notes*, só comentários no canvas. Não rodam nada; servem para quem abrir o workflow entender o que fazer primeiro.

## Parte 4 · O caminho de uma mensagem, passo a passo

Exemplo: o cliente já disse que se chama Lucas e quer corte. Agora manda "quarta às 15h".

1. **Gatilho** recebe `chatInput = "quarta às 15h"` e o `sessionId` da conversa.
2. **Agente** busca na **Memória** as trocas anteriores dessa sessão (onde estão "Lucas" e "corte").
3. **Agente** monta o pedido ao **Gemini**: prompt de sistema (já com a data de hoje calculada) + histórico + mensagem nova + descrição das quatro ferramentas.
4. **Gemini** responde: "quero usar *Verificar disponibilidade* com `inicio = 2026-10-21T15:00:00-03:00` e `fim = 2026-10-21T15:30:00-03:00`".
5. **n8n** executa o nó da ferramenta com esses valores. O Google Calendar responde `available: true`.
6. **Agente** devolve esse resultado ao **Gemini**, que agora escreve o texto final: o resumo e o pedido de confirmação.
7. A resposta aparece no chat e a **Memória** salva a pergunta e a resposta.

Na mensagem seguinte ("pode marcar"), o mesmo ciclo roda: verifica de novo e chama *Criar agendamento*.

**Remarcar** segue a mesma lógica, com mais passos: *Buscar agendamentos do cliente* → *Verificar disponibilidade* do novo horário → resumo e confirmação → *Criar agendamento* do novo → *Cancelar agendamento* do antigo. A ordem é proposital: se a criação falhar, o cliente não fica sem horário.

## Parte 5 · Para a entrevista

**Resumo em 30 segundos:** "É um agente de IA no n8n que atende pelo chat como recepcionista de barbearia. Ele coleta nome, serviço, dia e horário em linguagem natural, valida as regras de funcionamento, consulta o Google Calendar para não marcar em cima de outro cliente, pede confirmação e cria o evento. Também cancela e remarca, mas só agendamentos feitos na mesma conversa. Usei Gemini como modelo, com um modelo reserva, memória por sessão e quatro ferramentas do Google Calendar que o próprio agente decide quando chamar."

**Decisões que vale saber justificar:**

- **Verificar de novo antes de criar:** entre a sugestão do horário e o "sim" do cliente, alguém pode ter ocupado aquele horário.
- **Lista dos próximos 14 dias no prompt:** modelos erram dia da semana com frequência. Calcular no n8n e entregar pronto é mais confiável.
- **Temperatura baixa:** a tarefa pede consistência, não criatividade.
- **Agenda separada:** evita que compromissos pessoais bloqueiem horários e isola os dados de teste.
- **Sem credenciais no JSON:** o arquivo pode ir para o GitHub sem expor chaves.
- **Busca filtrada pelo código da conversa:** a segurança do cancelamento não depende do modelo obedecer ao prompt; o filtro está fixo no nó.
- **Remarcar = criar o novo, depois cancelar o antigo:** se algo falhar no meio, o pior caso é um horário duplicado (fácil de corrigir), nunca o cliente perder o horário.

**Limitações conhecidas (e como melhoraria):**

- As regras de funcionamento estão só no prompt; o modelo pode errar. Melhoria: uma ferramenta de código que valida dia e horário de forma determinística antes de consultar a agenda.
- Dois clientes confirmando o mesmo horário no mesmo segundo poderiam gerar conflito. Melhoria: checar e criar numa única etapa com trava, ou revalidar depois de criar.
- Só cancela ou remarca agendamentos da mesma conversa; se o cliente voltar outro dia, num chat novo, não consegue. Melhoria: identificar o cliente por telefone (por exemplo, com WhatsApp como canal) e usar o número no lugar do código da conversa.
- Simple Memory não é persistente. Melhoria: Postgres ou Redis.
- O chat é o do n8n. Melhoria: trocar o gatilho por WhatsApp ou Telegram mantendo o mesmo agente.

**Um problema real que resolvi (bom para contar):** nos testes, o cliente confirmava e recebia uma resposta vazia, e o evento não era criado. Abrindo a execução no n8n, vi que o modelo *Flash-Lite* devolvia `MALFORMED_FUNCTION_CALL` ao tentar chamar *Criar agendamento*, ferramenta com quatro campos preenchidos pelo modelo. Troquei para um modelo Flash completo, adicionei um modelo reserva para erros da API e o nó *Garantir resposta* para o cliente nunca receber resposta vazia. Depois, nos testes de gravação, apareceram erros 429 (cota gratuita diária esgotada) e 503 (modelo sobrecarregado). Resolvi com modelos de fila menos disputada, menos tentativas automáticas e o agente configurado para seguir em caso de erro, mostrando uma mensagem amigável.
