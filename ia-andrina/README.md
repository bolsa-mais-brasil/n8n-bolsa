# IA Andrina (Profissionaliza Mais Brasil) no n8n

Agente SDR/closer que atende pelo WhatsApp as caixas **Andrina Bmb (217)** e **ANDRINA BMB3 (216)** do Chatwoot (conta 1).

## Arquivos desta pasta

- `REGISTRO.md`: histórico completo do projeto (pedidos, decisões, estrutura no n8n, corte e pendências).
- `prompt.md`: prompt do agente. O n8n lê este arquivo a cada atendimento (GET no GitHub), então mudar aqui muda a IA na hora.
  - Trechos entre `<!-- TRAVA INÍCIO -->` e `<!-- TRAVA FIM -->` são fixos: preços, regras do produto, links, lista de cursos, formulário, regra de ouro, reuniões e formato de resposta. A melhoria automática diária não pode mexer neles.
  - Placeholders preenchidos pelo n8n: `{{contato}}`, `{{agora}}`, `{{resumo}}`, `{{reunioes_semana}}`, `{{isFollow}}`, `{{tentativas}}`.
- `relatorios/AAAA-MM-DD.md`: relatório diário do que a melhoria automática mudou no prompt e por quê. Os relatórios não trazem telefone nem nome completo de lead.

## Workflows no n8n (n8n-bolsa.bmbr.com.br)

1. **IA Andrina | Atendimento (Claude)**
   - Webhook do Chatwoot (message_created e conversation_updated).
   - Filtro: caixas 216/217 com a etiqueta da IA (`ia-liberada`; `ia-n8n` no modo teste).
   - Áudio é transcrito na OpenAI (gpt-4o-mini-transcribe) e imagem é descrita pelo Claude.
   - Buffer de 20 a 40 s no Redis: só a última mensagem de uma rajada gera resposta.
   - Contexto: últimas 15 mensagens do Chatwoot mais um resumo do que veio antes (Redis `andrina:resumo:<conversa>`).
   - Horários de reunião: eventos da semana no Google Calendar com o nome fixo e link do Meet.
   - O Claude responde em JSON (resposta e ações). A resposta sai em até 2 balões.
   - Ações: etiquetas, nota @Sistema, reunião (prioridade alta, lembrete 10 min antes, atribui Sistema), passar para a Juliana, mover no Kanban (funil 13, etapa 55), enviar os 2 PDFs da lista.
2. **IA Andrina | Follow-up**: de hora em hora, em horário comercial, retoma quem parou de responder: até 3 tentativas, com 24 h entre elas.
3. **IA Andrina | Melhoria diária do prompt**: todo dia de madrugada lê as conversas das últimas 24 h e propõe ajustes só nas partes livres do prompt. Aplica sozinho com trava e sobe o relatório em `relatorios/`.

## Configuração

Tudo fica no topo do nó **Filtrar evento** (objeto `CFG`): etiqueta, caixas, modo teste, modelo, janela, buffer, ids (Sistema 1, Juliana 9), etapa do Kanban, nome do evento da agenda e PDFs.

## Corte (sair do Chatwoot Flows)

1. Em `CFG`: `etiqueta_ia: 'ia-liberada'` e `modo_teste: false`.
2. Desativar no Chatwoot os flows 6 ("SDR Closer IA | Andrina") e 11 ("IA liga pela etiqueta ia-liberada").
3. As automações 23 e 24 do Chatwoot continuam colocando `ia-liberada` nas conversas sem atendente.
