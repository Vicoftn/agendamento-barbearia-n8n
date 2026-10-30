# Conversas de teste

Roteiros para validar o fluxo e gravar o vídeo. Os testes 1 a 3 cobrem o essencial; o 4 e o 5 cobrem remarcação e cancelamento; o 6 cobre um erro que apareceu em testes reais. As respostas do agente abaixo são exemplos do comportamento esperado; o texto exato muda a cada execução, mas o conteúdo deve ser equivalente.

**Antes de cada teste**, clique em **New chat** (ou recarregue o chat) para começar uma sessão limpa. Senão a memória mistura as conversas.

As datas dependem do dia em que você testar. Use sempre um dia da semana que ainda esteja por vir.

---

## Teste 1 · Agendamento normal

**Preparação:** nenhuma. Confira só que a quarta-feira às 15h está livre na agenda.

| Quem | Mensagem |
|---|---|
| Cliente | Oi, queria marcar um corte |
| Agente | Oi! Claro. Qual é o seu nome e para qual dia e horário você gostaria? |
| Cliente | Sou o Lucas, quarta às 15h |
| Agente | Perfeito, Lucas. Ficaria assim: corte, quarta-feira, 21/10, às 15h, R$ 40. Posso confirmar? |
| Cliente | Pode marcar |
| Agente | Pronto, Lucas! Seu corte está agendado para quarta-feira, 21/10, às 15h. Até lá! |

**O que conferir:**
- [ ] O agente pediu só nome, dia e horário (o serviço já tinha sido dito).
- [ ] Na aba *Executions*, a mensagem "quarta às 15h" mostra *Verificar disponibilidade* sendo usada.
- [ ] Na mensagem "Pode marcar", aparecem *Verificar disponibilidade* e depois *Criar agendamento*.
- [ ] No Google Calendar existe o evento **"Corte - Lucas"** na quarta, das 15h00 às 15h30.

---

## Teste 2 · Horário ocupado

**Preparação:** no Google Calendar, na mesma agenda que o workflow usa, crie um evento qualquer ("Ocupado") na **quinta-feira das 10h00 às 11h00**.

| Quem | Mensagem |
|---|---|
| Cliente | Boa tarde! Sou o Rafael, quero corte e barba na quinta às 10h |
| Agente | Rafael, quinta às 10h já está ocupado. Tenho livres às 11h e às 11h30. Algum desses funciona? |
| Cliente | 11h então |
| Agente | Fechado: corte + barba, quinta-feira, 22/10, às 11h, R$ 60. Confirma? |
| Cliente | Sim |
| Agente | Agendado, Rafael! Te esperamos quinta-feira, 22/10, às 11h. |

**O que conferir:**
- [ ] O agente não ofereceu 10h nem 10h30 (corte + barba às 10h30 terminaria às 11h30 e bateria com o evento até 11h).
- [ ] Os horários sugeridos foram verificados com a ferramenta antes de serem oferecidos (veja nas *Executions* várias chamadas de *Verificar disponibilidade*).
- [ ] O evento criado é **"Corte + barba - Rafael"**, das 11h00 às 12h00 (duração de 60 minutos).

---

## Teste 3 · Fora do horário de funcionamento

**Preparação:** nenhuma.

| Quem | Mensagem |
|---|---|
| Cliente | Oi, sou o Pedro. Dá pra fazer a barba domingo às 10h? |
| Agente | Oi, Pedro! Aos domingos e segundas a barbearia fica fechada. Posso marcar na terça-feira, 27/10, às 10h? |
| Cliente | E sábado às 18h45, corte e barba? |
| Agente | Às 18h45 não dá: corte + barba leva 1 hora e fechamos às 19h, então o último horário é 18h. Quer sábado, 24/10, às 18h? |
| Cliente | Pode ser às 18h |
| Agente | Então fica: corte + barba, sábado, 24/10, às 18h, R$ 60. Confirma? |
| Cliente | Confirmo |
| Agente | Tudo certo, Pedro! Agendado para sábado, 24/10, às 18h. |

**O que conferir:**
- [ ] Para "domingo às 10h" e "sábado às 18h45", o agente recusou **sem** chamar *Verificar disponibilidade* (a regra é checada antes da agenda).
- [ ] A sugestão alternativa respeitou o funcionamento e a duração do serviço.
- [ ] O evento final termina às 19h00, nunca depois.

---

## Teste 4 · Trocar para o combo (remarcar)

**Preparação:** nenhuma. Use a mesma sessão logo depois de concluir o Teste 1 (corte na quarta às 15h).

| Quem | Mensagem |
|---|---|
| Cliente | Quanto fica para fazer a barba também? |
| Agente | A barba custa R$ 30. Posso marcar logo depois do seu corte, às 15h30, ou trocar seu horário para corte + barba, que sai por R$ 60 no total. O que prefere? |
| Cliente | Troca para o combo |
| Agente | Fica assim: corte + barba, quarta-feira, 21/10, das 15h às 16h, R$ 60, no lugar do corte das 15h. Confirma? |
| Cliente | Confirmo |
| Agente | Pronto, Lucas! Seu horário agora é corte + barba na quarta-feira, 21/10, às 15h. |

**O que conferir:**
- [ ] O agente **não** disse que as 15h estão ocupadas por outra pessoa (o horário é do próprio cliente).
- [ ] Nas *Executions*, a ordem foi: *Buscar agendamentos do cliente* → *Verificar disponibilidade* (15h30 às 16h) → *Criar agendamento* → *Cancelar agendamento*.
- [ ] No Google Calendar sobrou **um** evento só: **"Corte + barba - Lucas"**, das 15h00 às 16h00.

---

## Teste 5 · Cancelar

**Preparação:** nenhuma. Use a mesma sessão de um teste em que o agendamento foi criado.

| Quem | Mensagem |
|---|---|
| Cliente | Preciso cancelar meu horário |
| Agente | Encontrei seu corte + barba na quarta-feira, 21/10, às 15h. Confirma o cancelamento? |
| Cliente | Sim |
| Agente | Pronto, seu horário foi cancelado. Se quiser marcar outro dia, é só me dizer. |

**O que conferir:**
- [ ] O agente pediu confirmação antes de cancelar.
- [ ] O evento sumiu do Google Calendar.
- [ ] Num **New chat**, pedir "quero cancelar meu agendamento" → o agente responde que não encontrou agendamento feito por este chat (ele não enxerga horários de outras conversas).

---

## Teste 6 · Nome inválido

| Quem | Mensagem |
|---|---|
| Cliente | Quero um corte sábado às 10h |
| Agente | Claro! Qual é o seu nome para o agendamento? |
| Cliente | xxx 123 |
| Agente | Pode me passar seu nome para eu registrar o horário? O primeiro nome já basta. |

**O que conferir:**
- [ ] O agente não repetiu o texto inválido e não seguiu para o resumo sem um nome de verdade.

---

## Testes extras (opcionais)

- **Informação aos poucos:** "quero marcar" → o agente pergunta tudo que falta sem repetir.
- **Mudar de ideia:** depois do resumo, responder "na verdade quero só barba" → o agente refaz o resumo com o novo serviço e preço antes de criar.
- **Horário quebrado:** "terça às 14h10" → o agente explica a grade de meia em meia hora e sugere 14h ou 14h30.

## Dica para o vídeo

Grave a tela dividida: o chat do n8n de um lado e o Google Calendar do outro. Rode o Teste 2 (horário ocupado) e emende o Teste 4 (troca para o combo) na mesma conversa, atualizando o Calendar depois de cada confirmação. No n8n, deixe o canvas visível enquanto o agente responde, para os nós das ferramentas acenderem.
