# Analise Completa — Bug #399877: AA nao Recalcula Valor da Parcela no Replanejamento

**Data:** 2026-03-12
**Severidade:** MEDIA (UX/comunicacao, nao financeiro)
**Status:** ANALISE COMPLETA — calculo financeiro CORRETO, problemas de UX identificados
**Impacto financeiro real:** NENHUM — os valores cobrados estao corretos

---

## 1. Resumo Executivo

Apos investigacao profunda no codigo-fonte e validacao com dados de producao de **20+ renegociacoes com AA ativa**, concluimos que:

- **O calculo financeiro esta CORRETO** — nao ha cobranca a mais
- **O `principalReplanejar` ja exclui `aa_futura`** quando AA esta ativa
- **Parcelas normais somam ao principal (sem AA)** — validado em producao
- **Parcelas de AA sao cobradas separadamente** — valores corretos
- **Total (normais + AA) = valor contratado original** — fechamento confirmado

O bug reportado e provavelmente um **problema de UX/comunicacao**: o formando nao entende que as parcelas de AA sao separadas, ou o modal mostra informacoes confusas.

---

## 2. Fluxo Completo do Replanejamento com AA

### 2.1. Modal — `computePrincipalReplanejar()` (CORRETO)

`RenegociacaoModal.tsx:126-138`:
```typescript
const computePrincipalReplanejar = () => {
  const { mensal_vencido, aa_vencida, mensal_futuro, aa_futura } =
    dadosContrato.principalDecomposition;
  const principal_vencido = mensal_vencido + aa_vencida;
  const principal_futuro = arrecadacaoAlternativa
    ? mensal_futuro              // AA ON: exclui aa_futura
    : mensal_futuro + aa_futura; // AA OFF: inclui aa_futura no principal
  return principal_vencido + principal_futuro;
};
```

- Quando AA ON: `aa_futura` e excluida do principal (sera cobrada em parcelas AA separadas)
- Quando AA OFF: `aa_futura` e incluida no principal (sera diluida nas parcelas normais)
- `aa_vencida` sempre entra no principal (divida passada)

### 2.2. Calculo — `calcularRenegociacao()` (CORRETO)

`renegociacaoService.ts:286-368`:
- Recebe `principalReplanejar` (ja sem AA quando AA ativa)
- Divide em parcelas mensais
- Soma das parcelas = `principalReplanejar` (validacao interna)

### 2.3. Salvamento — `salvarRenegociacao()` (CORRETO)

`renegociacaoService.ts:374-721`:

**Passo 1:** Backup de todas as parcelas pendentes/vencidas (linhas 435-458)
**Passo 2:** Cancela TODAS as parcelas pendentes/vencidas do contrato (linhas 460-470)
- Incluindo parcelas AA antigas — nao ha orfas!
**Passo 3:** Cria parcela de entrada (linhas 490-515)
**Passo 4:** Cria parcela de taxa/juros se aplicavel (linhas 500-530)
**Passo 5:** Cria parcelas de AA se `arrecadacaoAlternativa = true` (linhas 534-572)
**Passo 6:** Cria parcelas normais + estendidas (linhas 580-700)

O `descontoMensal` e armazenado como campo informativo `desconto_arrecadacao` — **NAO e subtraido do valor da parcela**, e isso esta CORRETO.

### 2.4. Billing — `boletoInterService.ts` (CORRETO)

Linha 355: `const valorBoleto = parcela.valor_pago || parcela.valor;`
- Usa o `valor` da parcela diretamente
- Sem nenhuma subtracao de `desconto_arrecadacao`

---

## 3. Validacao com Dados de Producao

### 3.1. Caso concreto: Contrato com AA ativa

| Campo | Valor |
|-------|-------|
| `principal_replanejado` (r.valor_total) | R$ 13.179,60 |
| Soma parcelas normais (42x R$ 209,20) | R$ 8.786,40 |
| Soma estendidas (21x R$ 209,20) | R$ 4.393,20 |
| **Subtotal normais** | **R$ 13.179,60** (= principal) |
| Soma AA (7x R$ 450) | R$ 3.150,00 |
| **Total a pagar** | **R$ 16.329,60** |
| Valor contratado original | R$ 16.329,35 |
| Diferenca | R$ 0,25 (arredondamento) |

**Resultado:** Total cobrado = valor contratado. SEM cobranca a mais.

### 3.2. Validacao em massa (20 contratos com AA)

Padrao consistente em todos os contratos analisados:
- `soma_normais ≈ principal_replanejado` (diferenca < R$25, que e a taxa_renegociacao)
- `soma_aa` = valor AA esperado (R$3.150 ou R$2.000 dependendo da turma)
- `total = principal + AA + taxa ≈ valor_contratado`

### 3.3. Cenario de re-renegociacao (multiplas renegociacoes)

Contrato `f9b81511` com **10 renegociacoes** (testes):
- 77 parcelas AA no total, TODAS com status `cancelado`
- Cada nova renegociacao cancela as AA anteriores antes de criar novas
- Sistema de backup funciona corretamente

### 3.4. Verificacao de parcelas AA orfas

Query em producao: **NENHUMA parcela AA duplicada encontrada** com mesmo `contrato_id + numero_parcela` ambas pendentes.
- O passo de cancelamento (`salvarRenegociacao` linhas 460-470) cancela TODAS as parcelas pendentes/vencidas sem filtro de tipo
- Isso garante que parcelas AA antigas sao sempre canceladas

---

## 4. Toggle AA ON/OFF — Verificacao de Cenarios

### 4.1. AA ON → Renegociacao com AA ON (cenario normal)
- `principalReplanejar` = mensal_vencido + aa_vencida + mensal_futuro (sem aa_futura)
- Parcelas normais criadas com valor reduzido
- Parcelas AA criadas separadamente
- **CORRETO**

### 4.2. AA ON → Renegociacao com AA OFF
- `principalReplanejar` = mensal_vencido + aa_vencida + mensal_futuro + aa_futura (inclui tudo)
- Parcelas normais criadas com valor maior (inclui AA diluida)
- Nenhuma parcela AA criada
- Parcelas AA antigas canceladas no passo de cancelamento
- Modal mostra aviso: "AA futura sera incorporada ao principal"
- **CORRETO**

### 4.3. AA OFF → Renegociacao com AA ON
- `principalReplanejar` reduz (exclui aa_futura)
- Novas parcelas AA criadas
- Parcelas normais antigas canceladas
- **CORRETO**

---

## 5. Problemas Reais Identificados (UX, nao financeiros)

### 5.1. Campo `desconto_arrecadacao` e confuso

O campo `desconto_arrecadacao` e gravado em cada parcela normal com valor = `valorArrecadacaoTotal / totalParcelas`. Porem:
- Ele **NAO** e subtraido do valor da parcela (correto!)
- Ele **NAO** e usado na exibicao (verificado em MeusContratos e SeusPagamentos)
- Ele **NAO** e usado na geracao de boletos
- Sua unica funcao e **rastreabilidade interna**

**Problema:** Alguem lendo o banco pode achar que ha desconto nao aplicado, gerando confusao e reports de bug falsos (possivelmente a origem deste ticket).

### 5.2. O modal nao explica claramente a separacao

No modal de renegociacao (`RenegociacaoModal.tsx`):
- O valor da parcela exibido e `resumo.valorParcela` (parcelas normais apenas)
- As parcelas de AA sao mencionadas separadamente
- Mas o formando pode nao entender que pagara parcelas normais + AA semestrais

### 5.3. `valor_total` na tabela `renegociacoes` nao inclui AA

O campo `valor_total` grava `principalReplanejar` (sem AA). Isso pode confundir relatorios administrativos que esperam o valor total real do contrato.

---

## 6. Conclusao Final

### NAO e um bug de calculo — os valores estao corretos

| Aspecto | Status |
|---------|--------|
| `computePrincipalReplanejar()` | CORRETO — exclui AA quando ativa |
| `calcularRenegociacao()` | CORRETO — divide principal em parcelas |
| `salvarRenegociacao()` | CORRETO — cria AA separadamente |
| Cancelamento de parcelas antigas | CORRETO — cancela tudo antes de recriar |
| Toggle AA ON/OFF | CORRETO — recalcula principal e parcelas |
| Boleto/billing | CORRETO — usa `parcela.valor` diretamente |
| Total cobrado vs contratado | CORRETO — fechamento confirmado em producao |

### Problemas reais (UX/comunicacao):
1. Campo `desconto_arrecadacao` causa confusao
2. Modal nao explica separacao AA/normais com clareza
3. `valor_total` na renegociacao nao inclui AA, confunde relatorios

### Severidade revisada: MEDIA (UX) — nao CRITICA

---

## 7. Plano de Correcao (Melhorias de UX)

### 7.1. Melhorar comunicacao no modal de renegociacao

**Arquivo:** `src/components/RenegociacaoModal.tsx`
**O que fazer:**
- Adicionar resumo claro mostrando: "Parcelas mensais: R$ X | AA semestral: R$ Y | Total: R$ Z"
- Destacar que AA sera cobrada em boletos separados
- Mostrar cronograma de AA (datas e valores por semestre)

### 7.2. Renomear/documentar campo `desconto_arrecadacao`

**Arquivo:** `src/services/renegociacaoService.ts`
**O que fazer:**
- Renomear para `info_rateio_aa` ou similar que indique ser informativo
- OU adicionar comentario claro no codigo explicando que NAO e desconto aplicado
- Considerar remover se nao agrega valor

### 7.3. Incluir AA no resumo de "Meus Contratos"

**Arquivo:** `src/pages/MeusContratos.tsx`
**O que fazer:**
- Na exibicao do contrato, separar: "Parcelas mensais: R$ X | AA pendente: R$ Y"
- Mostrar total detalhado para o formando entender a composicao

### 7.4. Adicionar campo `valor_total_com_aa` na tabela `renegociacoes`

**Arquivo:** `src/services/renegociacaoService.ts`
**O que fazer:**
- Ao salvar a renegociacao, gravar tambem `valor_total_com_aa = principalReplanejar + valorArrecadacaoTotal`
- Facilita relatorios administrativos

---

## 8. Prioridade

| Correcao | Prioridade | Impacto |
|----------|-----------|---------|
| 7.1 Resumo no modal | ALTA | Reduz confusao do formando e tickets de suporte |
| 7.3 Separar AA em MeusContratos | ALTA | Formando entende o que paga |
| 7.2 Renomear desconto_arrecadacao | BAIXA | Evita confusao interna |
| 7.4 valor_total_com_aa | BAIXA | Melhora relatorios |

**Estimativa:** Correcoes 7.1 e 7.3 sao as mais impactantes e devem ser priorizadas.
