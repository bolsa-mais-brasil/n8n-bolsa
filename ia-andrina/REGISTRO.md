# IA SDR / Closer das caixas Andrina (Profissionaliza Mais Brasil)

Registro do que foi pedido, decidido, configurado e testado (28/09 a 30/09/2026). A partir de 30/09/2026 o agente sai dos Flows do Chatwoot e passa a rodar no **n8n**.

- Chatwoot: https://chat.bmbr.com.br (conta 1, "Bolsa Mais Brasil")
- Caixas atendidas: **Andrina Bmb (217)** e **ANDRINA BMB3 (216)**
- n8n: https://n8n-bolsa.bmbr.com.br (webhooks em https://webhookbolsa.bmbr.com.br)
- IA: Claude (API Anthropic), modelo `claude-sonnet-5-5`. Resumo e leitura de imagem com `claude-haiku-4-5`.
- Prompt: `ia-andrina/prompt.md` neste repositório. É a fonte oficial: o n8n lê este arquivo a cada atendimento.

---

## 1. Como funciona agora (n8n)

- A IA atende os leads das caixas Andrina pelo WhatsApp como SDR/closer.
  - Tenta fechar na conversa, mandando o formulário de cadastro.
  - Se não fechar, marca a reunião no Meet.
- Ela só responde conversas que têm a **etiqueta da IA**. No modo teste é `ia-n8n`; depois do corte volta a ser `ia-liberada`.
  - Lead novo que chega sem atendente ganha a etiqueta sozinho (automação 24 do Chatwoot).
  - Conversa que fica sem atendente ganha a etiqueta sozinha (automação 23).
  - Conversa atribuída a alguém só é atendida pela IA se uma pessoa colocar a etiqueta na mão.
  - Quando a etiqueta entra, a IA lê o histórico e responde a mensagem que ficou pendente do lead.
  - Para desligar a IA numa conversa, é só tirar a etiqueta.
- **Áudio:** agora é transcrito (OpenAI, gpt-4o-mini-transcribe) e a IA responde normalmente. Antes ela parava e marcava @Sistema.
- **Imagem:** é descrita pelo Claude e entra na conversa como texto.
- **Espera antes de responder:** buffer de 20 a 40 segundos. Se o lead mandar várias mensagens seguidas, a IA responde uma vez só, lendo todas.
- **Memória:** a IA lê as últimas 15 mensagens direto do Chatwoot, inclusive as que alguém da equipe escreveu. O que é mais antigo vira um resumo curto guardado no Redis.
- **Horários de reunião:** vêm direto do Google Calendar da conta contato@bolsamaisbrasil.com.
  - Só entram os eventos com o título exato **"Reunião Profissionaliza"** e link do Meet, dos próximos 7 dias e que começam daqui a mais de 30 minutos.
  - Ninguém precisa mais atualizar os horários na mão toda semana.
- **Respostas:** mensagens curtas, no máximo 2 balões, sem travessão, no jeito da equipe.
- **Lista de cursos:** a IA manda os 2 PDFs (sem logo e com logo).
- **Matriz curricular:** a IA diz que consegue e já manda, e deixa nota marcando @Sistema para enviar.
- **Passagem para humano:** a IA passa para a Juliana e tira a etiqueta da IA sozinha.

---

## 2. Os 3 workflows no n8n

### 2.1 IA Andrina | Atendimento (Claude)

Caminho de uma mensagem:

1. **Webhook do Chatwoot** recebe `message_created` e `conversation_updated`.
2. **Filtrar evento** deixa passar três casos:
   - mensagem do lead nas caixas 216/217, em conversa com a etiqueta da IA (fora grupos e status);
   - etiqueta da IA recém-colocada numa conversa (ativação);
   - chamada interna de follow-up.
3. **Mídia:** áudio vai para a transcrição e imagem para a descrição. O texto gerado fica guardado no Redis por 30 dias.
4. **Buffer:** marca esta mensagem como a última da conversa, espera 20 a 40 s e só segue se nenhuma outra chegou nesse tempo.
5. **Contexto:**
   - conversa e etiquetas atuais;
   - até 40 mensagens do Chatwoot;
   - mídias transcritas;
   - resumo;
   - prompt do GitHub;
   - agenda da semana.
6. **Monta contexto:**
   - usa as 15 últimas mensagens como diálogo;
   - junta os dados da conversa (nome, data e hora, resumo, horários de reunião, modo follow-up);
   - não responde se a última mensagem já é nossa, porque alguém da equipe já respondeu.
7. **Claude** responde em JSON com a resposta e as ações. O prompt fica em cache para sair mais barato.
8. **Interpreta resposta:**
   - divide em até 2 balões (`---`) e tira travessão;
   - não repete a última mensagem enviada;
   - PULAR significa não mandar nada.
9. **Confere de novo** se não chegou mensagem nova enquanto a IA pensava. Se chegou, desiste, porque a execução nova responde tudo.
10. **Saídas**, nesta ordem:
    - mensagens com 3,5 s entre os balões;
    - os PDFs da lista;
    - as ações;
    - o Kanban;
    - a atualização do resumo;
    - o contador de follow-up.

Ações que a IA pode pedir (campos do JSON):

| Campo | O que o n8n faz |
|---|---|
| `etiquetas` | Coloca `ia-reuniao`, `ia-fechamento` ou `ia-sem-interesse` |
| `reuniao` | Nota @Sistema com dia, hora e link, atribui ao Sistema, prioridade alta, lembrete 10 min antes (sem duplicar, grava `ia_lembrete`) e etiqueta `ia-reuniao` |
| `mover_kanban` | Move ou cria o card no funil 13 "Andrina", etapa 55 Negociação |
| `nota_sistema` | Nota privada marcando @Sistema (dúvida, matriz, horário pedido, cadastro recebido) |
| `passar_humano` | Atribui à Juliana (id 9), tira a etiqueta da IA e deixa nota |
| `enviar_lista` | Envia "Lista de cursos Profissionaliza - sem logo.pdf" e "... com logo.pdf" |

A configuração fica toda no topo do nó **Filtrar evento** (objeto `CFG`):
- etiqueta, caixas, modo teste e telefones de teste;
- modelos, janela e buffer;
- ids (Sistema 1, Juliana 9) e etapa do Kanban;
- agenda e nome do evento;
- PDFs.

### 2.2 IA Andrina | Follow-up

- Roda de hora em hora, de segunda a sábado, das 9h às 19h.
- Pega as conversas com a etiqueta da IA em que a última mensagem é nossa há mais de 24 h.
- Pula as que têm `ia-sem-interesse` ou `ia-fechamento`.
- Chama o atendimento em modo follow-up. A IA manda uma retomada curta, sem repetir a última frase, ou responde PULAR.
- No máximo 3 tentativas por lead (contador no Redis, zera quando o lead responde) e 10 retomadas por hora.

### 2.3 IA Andrina | Melhoria diária do prompt

- Todo dia às 3h07 lê as conversas das caixas Andrina com etiqueta de IA das últimas 24 h.
- Tira telefone, e-mail, CPF e nome antes de mandar para análise.
- O Claude propõe ajustes pequenos no prompt, no máximo 5.
- O n8n aplica sozinho, com trava:
  - só mexe em texto fora dos blocos `<!-- TRAVA ... -->`;
  - não aceita número, link, valor ou placeholder novo;
  - não aceita travessão;
  - se a conferência final falhar, descarta tudo.
- Sobe o prompt novo e o relatório do dia em `ia-andrina/relatorios/AAAA-MM-DD.md`.
- Seções travadas:
  - o que a gente oferece;
  - como funciona na prática;
  - cursos;
  - valor da mensalidade;
  - reuniões;
  - formulário;
  - regra de ouro;
  - formato de resposta.

---

## 3. O que mudou no prompt na migração

- As **ferramentas** do Chatwoot (etiqueta, nota, atribuir, Kanban) viraram os campos do JSON da seção **COMO RESPONDER (FORMATO OBRIGATÓRIO)**.
- Os códigos `[[LISTA]]` e `[[REUNIAO|...]]` saíram. Agora são os campos `enviar_lista` e `reuniao`.
- **Áudio e imagem:** a IA recebe o texto transcrito ou a descrição e responde normalmente, sem comentar que foi transcrito. Se não der para abrir, pede para o lead escrever.
- **Dados da conversa** entram no fim, num bloco separado, para o prompt ficar em cache:
  - nome, data e hora, resumo, horários de reunião e modo follow-up;
  - aviso de que mensagens nossas podem ter sido escritas por alguém da equipe.
- **Ativação pela etiqueta:** a IA continua de onde parou ou responde PULAR se não houver nada pendente.
- Placeholders: `{{contato}}`, `{{agora}}`, `{{resumo}}`, `{{reunioes_semana}}`, `{{isFollow}}`, `{{tentativas}}`.
- Marcadores de TRAVA nas seções que a melhoria automática não pode mexer.
- Todo o conteúdo comercial continua igual:
  - preços, regras, lista de 224 cursos, objeções, formulário e tom da equipe;
  - cumprimento com o nome e preço só quando perguntarem;
  - certificado, cancelamento, CNPJ e presencial.

---

## 4. Linha do tempo dos pedidos

1. Criar o agente SDR/closer com base no histórico das caixas Andrina.
2. "Muito rápida e com cara de IA": prompt no tom da equipe, frases curtas, 2 balões, lista de frases proibidas e espera antes de responder.
3. Quebrar objeções sem prometer nada (QUEBRA DE OBJEÇÕES e REGRA DE OURO).
4. Evitar resposta duplicada (hoje: buffer e conferência no Redis).
5. Quando não souber, marcar @Sistema (não passar para a Juliana).
6. Cadastro recebido: marcar @Sistema para gerar o link de pagamento.
7. Responder se tem ou não o curso pelo catálogo.
8. Reunião com link do Meet por horário, @Sistema, prioridade alta e lembrete 10 min antes.
9. Remarketing: até 3 tentativas, pulando quem não tem interesse.
10. O parceiro só não altera matriz curricular e carga horária.
11. 22 cenários de objeção testados.
12. CNPJ, cancelamento sem multa (aviso de 10 dias) e presencial.
13. Preço só quando o lead perguntar.
14. Primeiro cumprimentar com o nome.
15. Ignorar mensagens de Status do WhatsApp.
16. Pelo menos 1 curso novo por dia; pagamento direto na conta do parceiro (Asaas ou Mercado Pago).
17. Enriquecimento com as conversas da equipe de parceiros (conta 53, caixa 221): seção COMO FUNCIONA NA PRÁTICA.
18. Script comercial: seção COMO A EQUIPE VENDE.
19. "Escola de cursos profissionalizantes", nunca "escola de cursos online".
20. Certificado com a logo do parceiro, logo do Grupo Bolsa Mais Brasil no rodapé e QR Code no cantinho.
21. Remarketing em lotes (56 leads em 4 lotes de 14), sem disparar tudo de uma vez.
22. Leads do remarketing atribuídos ao Sistema.
23. Etiqueta como liga e desliga da IA.
24. Conversas já atribuídas só com etiqueta manual.
25. Áudio: primeiro a IA parava e marcava @Sistema; no n8n passou a transcrever.
26. Lista oficial de cursos em PDF (224 cursos, atualizada toda sexta) no prompt.
27. Lista de cursos: envia os 2 PDFs; matriz: marca @Sistema.
28. Matriz: nunca dizer "vou pedir", dizer que consegue e já manda.
29. **Migração para o n8n (30/09/2026):**
    - Chatwoot entra e sai; Claude via API Anthropic;
    - buffer de 20 a 40 s; últimas 15 mensagens mais resumo;
    - áudio transcrito e imagem lida;
    - prompt lido do GitHub; horários da agenda do Google;
    - follow-up por cron;
    - melhoria diária do prompt com trava e relatório no Git;
    - começa só pelas caixas Andrina, com teste antes do corte.

---

## 5. Etiquetas, pessoas e ids

| Etiqueta | Uso |
|---|---|
| ia-liberada | Liga a IA na conversa (depois do corte) |
| ia-n8n | Liga a IA do n8n no modo teste |
| ia-reuniao | IA marcou reunião |
| ia-fechamento | Lead mandou cadastro ou quer pagar |
| ia-sem-interesse | Lead recusou |
| rmk-sem-resposta, rmk-reuniao, rmk-lote-1 a 4 | Remarketing |

- Sistema: id 1 (recebe as notas e as reuniões)
- Juliana Guerra: id 9 (recebe o lead quando passa para humano)
- Funil 13 "Andrina": etapas 53 Novo Lead, 54 Fila, 55 Negociação, 56 Fechado

Chaves no Redis (credencial "Buffer"):

| Chave | O que guarda |
|---|---|
| `andrina:ult:<conversa>` | Última mensagem da conversa (buffer) |
| `andrina:midia:<conversa>:<mensagem>` | Texto do áudio ou da imagem |
| `andrina:resumo:<conversa>` | Resumo da conversa |
| `andrina:fu:<conversa>` | Tentativas de follow-up |

---

## 6. Corte (sair dos Flows do Chatwoot)

1. Testar com o número de teste, com a etiqueta `ia-n8n`.
2. No `CFG` do atendimento, trocar `etiqueta_ia` para `ia-liberada` e `modo_teste` para `false`.
3. Na Config do follow-up, trocar a etiqueta para `ia-liberada`.
4. Desativar no Chatwoot os flows 6 ("SDR Closer IA | Andrina") e 11 ("IA liga pela etiqueta ia-liberada").
5. Ativar os 3 workflows no n8n.
6. As automações 23 e 24 do Chatwoot continuam colocando `ia-liberada`.

---

## 7. Pendências e cuidados

1. **Repositório público:** este repositório está público e tem backups de workflow com token escrito nos nós. Deixar privado e trocar os tokens expostos.
2. **Webhook do Chatwoot** para `https://webhookbolsa.bmbr.com.br/webhook/ia-andrina` (eventos message_created e conversation_updated): falta criar.
3. **Lista de cursos:** atualizar toda sexta a lista do prompt e os 2 PDFs.
4. **PDF da lista** tem duplicados para conferir: "Manicure e Pedicure" (20h e 385h) e "Power BI" (30h e 60h).
5. **Segurança:** o acesso de administrador da escola demo circula para leads no WhatsApp. Vale criar um acesso só de visualização.
6. **Conversa com a etiqueta da IA e alguém da equipe falando junto:** a IA também responde. Tire a etiqueta quando quiser atender sozinho.

O que não foi colocado no prompt de propósito:
- "valor promocional / reajuste semana que vem" e "valor já com desconto";
- reembolso de 7 e 60 dias (confuso, vai para @Sistema);
- aviso de cancelamento de 15 dias (mantidos os 10 dias);
- senhas e dados pessoais de parceiros.
