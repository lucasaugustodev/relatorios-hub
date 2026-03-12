# Classificacao de Tarefas — 11/03/2026

---

## BUGS (17 tarefas)

---

### 1. [#391363](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/391363/) — IMPORTANTE valores nao batem (contratado, a pagar)

**Severidade:** Alta | **Etapa:** Reteste | **Prazo:** 11/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Valor no replanejamento nao bate com o valor devido. Plano de R$ 17.169,85 gerou apenas R$ 14.492,76 para pagar. Faltou cobrar R$ 2.677,09.

**Detalhes numericos:**
- Valor do plano: R$ 17.169,85
- Valor gerado para pagamento: R$ 14.492,76 (faltando R$ 2.677,09)
- Parcelas escolhidas: 34 | Parcelas geradas: 33
  - 5 parcelas de AA: R$ 400,00 cada = R$ 2.000,00
  - 28 parcelas normais: R$ 446,17 cada = R$ 12.492,76
  - Total gerado: R$ 14.492,76

**Comentarios relevantes (05/03):**
- Caroline descobriu que o sistema usava a data da 1a parcela para TODAS as outras (ignorando o dia original do contrato = dia 30)
- Conta de teste: `meusuario+teste1102@somosahub.com.br`
- BOT postou correcao tecnica: toggle "Alterar data de vencimento" — quando DESLIGADO, so a 1a parcela muda; quando LIGADO, parcelas 2+ usam o novo dia
- Caroline reportou problema residual: meses com 30/31 dias gerando datas incorretas

**Correcoes ja aplicadas:** Commits `43cf466`, `62b53f2`, `164af32`, `a92434e` — corrigindo overflow de data, ajustarParaDiaUtil, e toggle de vencimento

---

### 2. [#399891](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399891/) — Nao apareceu no acesso hub a solicitacao de rescisao

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Apos fazer uma solicitacao de rescisao, ela nao apareceu na visao do hub/administrador.

**Reproduzir:** Fazer solicitacao de rescisao e verificar se aparece no painel do admin.

**Arquivos:** 2 screenshots anexados

---

### 3. [#399885](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399885/) — Programei um lote as 11h e ele nao ativou

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Usuario programou um lote de ativacoes para as 11h e o sistema nao executou o agendamento no horario programado. O lote simplesmente nao ativou.

**Reproduzir:** Agendar um lote com horario futuro e verificar se ele e executado automaticamente.

**Arquivos:** 2 screenshots (agendamento e lote nao ativado)

---

### 4. [#399883](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399883/) — Replanejamento gerando parcela fora do periodo de pgto da turma

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
A turma permite inicio do pagamento em 01/04/2026, mas ao fazer o replanejamento financeiro, o sistema gerou parcela para marco (calculou "hoje + 5 dias" = 16/03 em vez de respeitar o calendario da turma). Alem disso, permitiu selecionar 40 parcelas quando o limite correto da turma e 39.

**Dois problemas:**
1. **Data da 1a parcela:** sistema ignora o inicio do periodo financeiro da turma
2. **Quantidade de parcelas:** permite 40 quando o maximo e 39

---

### 5. [#399901](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399901/) — INCONSISTENCIA CONV. EXTRA

**Severidade:** Alta | **Etapa:** Backlog | **Sem prazo**
**Proprietaria:** Izadora Carvalho | **Responsavel:** Lucas Augusto

**Descricao completa:**
Dois problemas no convite extra:

1. **Valor divergente:** O valor do 2o lote e R$ 1.200,00 mas na hora do pagamento aparece R$ 1.000,00. Diferenca de R$ 200,00 sem justificativa.
2. **Arrecadacao alternativa indevida:** A opcao de arrecadacao alternativa esta habilitada para convites extras, mas este servico nao deveria ter essa opcao.

**Arquivos:** 2 screenshots

---

### 6. [#399903](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399903/) — Estou em dia e ele fala que to inadimplente

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Sistema marca o aluno como inadimplente mesmo estando em dia. Caroline levanta a hipotese: "deve ser pq a parcela vence hj" — ou seja, no dia do vencimento (11/03), o sistema ja considera inadimplente antes que o pagamento seja processado.

**Problema provavel:** Comparacao de datas usa `<` ao inves de `<=`, ou verifica inadimplencia antes do processamento de pagamento do dia.

**Arquivos:** 3 screenshots

---

### 7. [#399905](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399905/) — Nao permitiu liberar manual a aptidao

**Severidade:** Media | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
O sistema nao permitiu realizar a liberacao manual da aptidao financeira de um aluno. O botao ou acao de "liberar aptidao" que deveria estar disponivel para operadores nao esta funcionando.

**Arquivos:** 1 screenshot (`image (973) (6).png`)

---

### 8. [#399907](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399907/) — ERRO NA SOMATORIA DOS VALORES

**Severidade:** Alta | **Etapa:** Backlog | **Sem prazo**
**Proprietaria:** Izadora Carvalho | **Responsavel:** Lucas Augusto

**Descricao completa:**
Ao comprar convite extra, o valor unitario e R$ 1.200,00. Para pagar 1 convite, o valor sai R$ 1.000,00 (com desconto). Porem, ao comprar 5 convites, o total sai R$ 5.800,00 — o desconto nao esta proporcional a quantidade.

**Calculo esperado:** 5 x R$ 1.000 = R$ 5.000 (ou 5 x R$ 1.200 = R$ 6.000 sem desconto)
**Calculo gerado:** R$ 5.800 (valor inconsistente — nem preco cheio nem com desconto)

**Problema:** O calculo de desconto na somatoria esta errado — nao aplica o desconto proporcionalmente.

**Arquivos:** 1 screenshot (`image (532) (2).png`)

---

### 9. [#399911](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399911/) — LIMITE DE COMPRA NAO HABILITADO

**Severidade:** Alta | **Etapa:** Backlog | **Sem prazo**
**Proprietaria:** Izadora Carvalho | **Responsavel:** Lucas Augusto

**Descricao completa:**
O limite de 5 convites extras por aderente nao esta sendo respeitado. O usuario conseguiu comprar mais do que 5 fazendo compras separadas (ex: compra 3 + compra 3 = 6 convites, ultrapassando o limite de 5).

**Problema:** O limite nao e aplicado cumulativamente entre transacoes distintas. Ex: 1a compra = 3 convites (OK), 2a compra = 3 convites (OK individualmente, mas total = 6, deveria bloquear).

**Arquivos:** 1 screenshot (`image (532) (3).png`)

---

### 10. [#399915](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399915/) — Acesso hub, nao atualizou os valores

**Severidade:** Media | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
O portal Hub nao atualizou a quantidade de parcelas e os valores quitados. Apos pagamento ou alteracao de plano, a tela de acesso/dashboard continua exibindo dados desatualizados.

**Arquivos:** 1 screenshot (`image (973) (7).png`)

---

### 11. [#399923](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399923/) — Liberou desconto pra quem nao e comissao

**Severidade:** Alta | **Etapa:** Backlog | **Sem prazo**
**Proprietaria:** Izadora Carvalho | **Responsavel:** Lucas Augusto

**Descricao completa:**
Usuario "dodo" (que NAO e comissao) recebeu desconto de comissao nos convites extras. Pagou R$ 1.000,00 ao inves de R$ 1.200,00.

**Problema:** O preco diferenciado por tipo de participante nao esta sendo filtrado corretamente. O desconto de comissao esta sendo aplicado para usuarios que nao sao comissao. Para "dodo", o convite do 2o lote apareceu como R$ 1.000,00 quando deveria ser R$ 1.200,00.

**Arquivos:** 1 screenshot (`image (973) (8).png`)

---

### 12. [#399995](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399995/) — ERRO NO UPGRADE DE PLANO

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Multiplos problemas no fluxo de upgrade de plano:

1. **Valor de debito inexplicado:** "Esta aparecendo um valor de debito a pagar, somando o valor do plano, sem explicacao" — o sistema exibe debito adicional sem detalhar a origem
2. **Valores pagos nao abatidos:** "Nao somou os valores que ja havia pago" — o calculo do upgrade nao desconta o que ja foi pago no plano atual
3. **Parcelamento estendido indevido:** "Nao permanece as mesmas condicoes de pagamento do plano atual. Coloca um parcelamento estendido nao solicitado." O campo inicial deveria puxar a forma atual de pagamento como padrao, dando opcao de trocar depois

**Comportamento esperado:** A tela deveria (a) mostrar credito do plano atual ja pago, (b) manter o mesmo parcelamento como padrao, (c) exibir origem de qualquer debito adicional.

**Arquivos:** 1 screenshot (`image (532) (5).png`)

---

### 13. [#400001](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/400001/) — Termo de transferencia nao apareceu

**Severidade:** Media | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
No fluxo de transferencia de plano, o "termo de transferencia" nao aparece para o usuario. Nao existe campo para configurar isso na turma — Caroline acredita que seria uma configuracao interna do sistema.

**Arquivos:** 1 screenshot

---

### 14. [#399997](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399997/) — Erro no link que explica valores da transferencia

**Severidade:** Baixa | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
No modal "Passo 2 de 4 — Resumo da Transferencia", ha um link que deveria explicar os valores. Ao clicar, da erro.

**Valores vistos no screenshot:**
- Valor contratado: R$ 10.412,40
- Valor pago do contrato (Encargos por atraso): R$ 46,71
- Multa e juros de parcelas vencidas: R$ 0,00
- Valor a pagar: R$ 9.952,80

**Historico:** Caroline editou a descricao e trocou screenshots (15:10)

**Arquivos:** 2 screenshots

---

### 15. [#400007](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/400007/) — Transferi um plano mas nao atualizou

**Severidade:** Media | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Apos a transferencia de plano ser aceita pela outra parte ("dodo"), o status ficou inconsistente:
- Na aba **"Todos"**: aparece como **ativo** (INCORRETO)
- Na aba **"Transferencias"**: aparece como **transferido** (correto)

**Problema:** O status do contrato nao e atualizado na listagem geral apos a transferencia ser aceita.

**Arquivos:** 2 screenshots

---

### 16. [#400009](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/400009/) — Alteracao de plano que foi transferido

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Dois problemas:

1. **Logica incorreta:** O sistema permite fazer upgrade de um plano que ja foi transferido para outra pessoa ("dodo"). Deveria bloquear essa acao.
2. **Mensagem confusa:** Apos a acao, aparece mensagem misturando tres contextos: rescisao + ativacao de plano + transferencia.

**Comentarios de Lucas (18:59):**
- "Na verdade e feita a rescisao de um plano e a adesao em outro"
- "A gente precisa ajustar esse texto mesmo porque ta ruim"
- "Relativo a conseguir fazer upgrade pos transferencia, ai e ajuste mesmo"

**Arquivos:** 1 screenshot

---

### 17. [#400011](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/400011/) — Ao fazer upgrade gerou parcela fora do periodo financeiro

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
A turma encerra o periodo financeiro em 29/06, mas ao fazer upgrade, o sistema gerou parcela para julho — ultrapassando o limite.

**Importante:** Na adesao inicial as parcelas foram geradas corretamente (dentro do periodo). O bug ocorre **somente no fluxo de upgrade**, nao na adesao.

**Arquivos:** 2 screenshots

---

## BUGS — Replanejamento (2 tarefas)

---

### 18. [#399877](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399877/) — ERRO NO REPLANEJAMENTO FINANCEIRO

**Severidade:** Alta | **Etapa:** Backlog | **Prazo:** 18/03/2026
**Proprietaria:** Lucas Augusto | **Responsavel:** Lucas Augusto

**Descricao completa:**
No fluxo de replanejamento financeiro, ao selecionar uma "arrecadacao alternativa", o valor da parcela deveria ser recalculado mas permanece inalterado. O campo de valor nao reage a mudanca.

**Historico:** Tarefa criada por Izadora Carvalho, proprietario alterado para Lucas Augusto.

**Arquivos:** 2 screenshots

---

### 19. [#397137](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/397137/) — REPLANEJAMENTO — boleto pago antes do replanejamento

**Severidade:** Media | **Etapa:** Reteste | **Prazo:** 18/03/2026
**Proprietaria:** Isabelle Matos | **Responsavel:** Isabelle Matos | **Marcadores:** melhoria

**Descricao completa:**
Carol pagou um boleto de R$ 15,00 + R$ 3,00 de taxa antes de dar baixa no sistema. Em seguida fez um replanejamento. O valor pago deveria ter entrado como HubCash para abatimento da nova parcela.

**Sugestao:** Eduardo sugeriu adicionar aviso no replanejamento: "Caso tenha pagamento em processamento, esperar para replanejar."

**Comentarios:**
- **BOT (06/03 18:24):** Implementacao concluida. Edge function `verificar-status-boleto-inter` (v6): quando parcela esta cancelada (replanejamento), credita valor como HubCash na wallet do aluno. Commit `4b97a84`.
- **Isabelle (06/03 16:46):** "arrasou amigo / vou testar de novo semana que vem"

**Status:** Correcao implementada, aguardando validacao de Isabelle.

---

## MELHORIAS / FEATURES (5 tarefas)

---

### 20. [#399895](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399895/) — Habilitar aba historico

**Tipo:** Feature | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Liberar a aba de "historico" no portal para visualizacao. Caroline considera essencial para o lancamento do sistema: "pra liberar o sistema, acho mt importante termos acesso a essas infos."

**Arquivos:** 1 screenshot

---

### 21. [#399899](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399899/) — Visualizacao de pacotes disponiveis

**Tipo:** Melhoria UX | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
Mesmo que o aderente nao esteja apto para comprar um pacote/lote, ele deveria conseguir visualizar o lote disponivel e ver o motivo do bloqueio (ex: inadimplencia, restricao). Objetivo: motivar o usuario a resolver suas pendencias para poder comprar.

**Arquivos:** 1 screenshot

---

### 22. [#399977](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399977/) — MELHORIA DIDATICA

**Tipo:** Melhoria UX | **Etapa:** Backlog | **Sem prazo**
**Proprietaria:** Izadora Carvalho | **Responsavel:** Lucas Augusto

**Descricao completa:**
Ajustes visuais na tela "Meus Contratos":

1. **Codigo do contrato:** Reduzir destaque — nao precisa estar tao evidente na primeira coluna
2. **Titulo da contratacao:** Mostrar apenas o tipo de servico (ex: "Convite Extra"), sem o nome da turma
3. **Coluna de quantidade:** Adicionar coluna mostrando a quantidade adquirida (ex: "2 convites extras")

**Arquivos:** 1 screenshot

---

### 23. [#399991](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/399991/) — Parcela estimada

**Tipo:** Melhoria | **Etapa:** Backlog | **Prazo:** 16/03/2026
**Proprietaria:** Caroline Botelho Murad | **Responsavel:** Lucas Augusto

**Descricao completa:**
A "parcela estimada" no portal esta mostrando o **menor valor possivel** (sem diluicao AA, sem estendido), quando deveria refletir as configuracoes atuais do plano do aderente:

- Se dilui a AA → considerar valor COM diluicao
- Se nao dilui → considerar valor SEM diluicao
- Se tem estendido → considerar COM estendido
- Se nao tem estendido → considerar SEM estendido

**Arquivos:** 1 screenshot

---

### 24. [#384795](https://hub.bitrix24.com.br/workgroups/group/217/tasks/task/view/384795/) — Conferir restricoes por aderente de acordo com quitacao e inadimplencia

**Tipo:** Feature/Teste | **Etapa:** Backlog | **Prioridade:** Alta | **Prazo:** 17/03/2026
**Proprietario:** Eduardo Reuter | **Responsavel:** Caroline Botelho Murad | **Participante:** Lucas Augusto

**Descricao completa:**
Validar as restricoes de compra/acesso por aderente baseado em dois criterios: quitacao e inadimplencia. Objetivos:
- Testar no ciclo semanal
- Criar cenarios de teste para o time

**Historico:** Tarefa criada em 23/01/2026, prazo adiado multiplas vezes desde fevereiro. Marcada como alta prioridade por Caroline em 10/03.

---

## Resumo

| Categoria | Quantidade |
|---|---|
| Bugs | 19 |
| Melhorias / Features | 5 |
| **Total** | **24** |

### Bugs por area

| Area | Qtd | IDs | Status |
|---|---|---|---|
| Convite Extra (valores/limites/desconto) | 4 | #399901, #399907, #399911, #399923 | Backlog |
| Transferencia | 3 | #400001, #399997, #400007 | Backlog |
| Upgrade/Mudanca de plano | 3 | #399995, #400009, #400011 | Backlog |
| Replanejamento | 3 | #391363, #399883, #399877 | Reteste/Backlog |
| Aptidao financeira | 2 | #399903, #399905 | Backlog |
| Rescisao | 1 | #399891 | Backlog |
| Lote/Ativacao | 1 | #399885 | Backlog |
| Dashboard/Valores | 1 | #399915 | Backlog |
| Replanejamento + HubCash | 1 | #397137 | Reteste (correcao aplicada) |

### Correcoes ja aplicadas

| ID | Tarefa | Commits | Status |
|---|---|---|---|
| #391363 | Valores nao batem (replanejamento) | `43cf466`, `62b53f2`, `164af32`, `a92434e` | Reteste |
| #397137 | Boleto pago antes do replanejamento | `4b97a84` | Reteste |
