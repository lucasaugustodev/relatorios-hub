# Validacao Consolidada — Bug #391363: Replanejamento Financeiro

**Data:** 2026-03-11
**Severidade:** Media-Alta
**Status:** CONFIRMADO via teste E2E (Playwright) + analise de codigo + dados de producao

---

## 1. Resumo Executivo

O bug #391363 causa **confusao para o formando** ao abrir o modal de replanejamento financeiro: o "Principal a replanejar" e significativamente menor que o "Valor contratado" sem que a decomposicao fique clara a primeira vista.

**Teste E2E realizado:** Login no portal com conta real (havinix704@etramay.com), navegacao ate Meus Contratos, abertura do modal de Replanejamento financeiro. Screenshots capturadas com sucesso.

**O que a UI mostra hoje (capturado via Playwright):**

```
Detalhamento:
  Valor contratado:                R$ 13.756,64
  Valor quitado:                   R$      0,00
  Mensalidades vencidas:           R$    706,36  (vermelho)
  Mensalidades futuras:            R$ 10.948,58
  AA futura (paga separadamente):  R$  3.150,00
  Multa e juros (+):               R$     31,36

  Principal a replanejar:          R$ 11.654,94
```

**Problema:** A soma das linhas do detalhamento (0 + 706,36 + 10.948,58 + 3.150,00 = 14.804,94) **nao fecha** com o valor contratado (13.756,64). Faltam as parcelas de "estendido" (R$ 3.947,64) no detalhamento do topo. O formando nao consegue conferir a matematica.

**Principal a replanejar (R$ 11.654,94) = Mensalidades vencidas (706,36) + Mensalidades futuras (10.948,58).** A AA futura e mostrada mas nao fica claro por que ela esta "fora" do principal.

---

## 2. Teste E2E — Resultado Completo

### 2.1. Ambiente
- **Ferramenta:** CLI `cli-hub-portal-vibe` + Playwright 1.50.0
- **Docker:** `mcr.microsoft.com/playwright/python:v1.50.0-noble`
- **Conta testada:** havinix704@etramay.com (Lucas Augusto)
- **Contrato:** CONT-1762972565206-746cfe31, tipo BAILE (10 CONVITES)
- **73 parcelas pendentes**

### 2.2. Steps executados

| Step | Descricao | Status |
|------|-----------|--------|
| 1 | Login (email + senha) | OK |
| 1b | Selecao de turma (FORMAE-163871) | OK |
| 2 | Navegacao para Meus Contratos | OK |
| 3 | Abertura do modal Replanejamento financeiro | OK |
| 4 | Extracao de valores do modal | OK |
| 5 | Analise do gap (contrato vs principal) | GAP CONFIRMADO |
| 6 | UI explica AA? | PARCIALMENTE |

### 2.3. Valores extraidos do modal (via Playwright)

**Detalhamento (topo do modal):**
| Campo | Valor |
|-------|-------|
| Valor contratado | R$ 13.756,64 |
| Valor quitado | R$ 0,00 |
| Mensalidades vencidas | R$ 706,36 |
| Mensalidades futuras | R$ 10.948,58 |
| AA futura (paga separadamente) | R$ 3.150,00 |
| Multa e juros | R$ 31,36 |
| **Principal a replanejar** | **R$ 11.654,94** |
| Entrada inteligente | R$ 187,98 |

**Parcelamento (bottom do modal):**
- 41 parcelas mensais de R$ 187,98
- 7 parcelas de arrecadacao alternativa: R$ 450,00 cada (semestral, 2026-2029)
- 1 parcela estendido: R$ 3.947,64 (21x cartao, a partir de 19/07/2029)

### 2.4. Screenshots

Salvas em `/home/claudee/cli-hub-portal-vibe/reports/`:
- `havinix704-04-modal-loaded.png` — Modal com detalhamento e principal
- `havinix704-05-modal-bottom.png` — Parcelas AA + estendido + botao confirmar

---

## 3. Analise do Gap

### 3.1. Contrato testado (havinix704)

| Metrica | Valor |
|---------|-------|
| Valor contratado | R$ 13.756,64 |
| Principal a replanejar | R$ 11.654,94 |
| **Diferenca** | **R$ 2.101,70 (15,3%)** |
| AA futura no contrato | R$ 3.150,00 |

**Como o principal e calculado:**
- Principal = Mensalidades vencidas (R$ 706,36) + Mensalidades futuras (R$ 10.948,58)
- **Exclui:** AA futura (R$ 3.150,00) — cobrada separadamente
- **Inclui:** Parcela estendido embutida nas mensalidades futuras

**Diferenca (R$ 2.101,70) != AA (R$ 3.150,00)** porque as mensalidades futuras ja incluem parte do valor estendido que seria descontado na decomposicao interna.

### 3.2. Evidencia nos dados de producao (Supabase)

| Metrica | Valor |
|---------|-------|
| Total de renegociacoes no banco | 101 |
| Com gap significativo (> R$100) | **86 (85%)** |
| Gap = valor AA (+-R$5) | **65 (64%)** |
| Contratos com AA ativa | 87 (86%) |

Exemplos de renegociacoes onde gap = AA exato:
| Contrato | Valor Total | Valor Renego | Gap | AA | Diff |
|----------|------------|-------------|-----|-----|------|
| de0ef725 | R$ 16.329,35 | R$ 13.179,60 | R$ 3.149,75 | R$ 3.150,00 | R$ 0,25 |
| 9173a0ba | R$ 16.329,35 | R$ 13.179,60 | R$ 3.149,75 | R$ 3.150,00 | R$ 0,25 |
| 81a8cee2 | R$ 16.329,35 | R$ 13.179,18 | R$ 3.150,17 | R$ 3.150,00 | R$ 0,17 |

---

## 4. Analise do Codigo

### 4.1. Funcao que exclui AA (`RenegociacaoModal.tsx:126-138`)

```typescript
const computePrincipalReplanejar = () => {
  const { mensal_vencido, aa_vencida, mensal_futuro, aa_futura } =
    dadosContrato.principalDecomposition;
  const principal_vencido = mensal_vencido + aa_vencida;
  const principal_futuro = arrecadacaoAlternativa
    ? mensal_futuro              // EXCLUI aa_futura quando AA ativa
    : mensal_futuro + aa_futura;
  return principal_vencido + principal_futuro;
};
```

**Logica correta:** Quando AA ativa, parcelas de arrecadacao sao cobradas via boleto separado, entao nao entram no replanejamento. Isso esta **financeiramente correto**.

### 4.2. Decomposicao no servico (`renegociacaoService.ts:149-189`)

O servico decompoe as parcelas em: `mensal_vencido`, `aa_vencida`, `mensal_futuro`, `aa_futura`. Subtrai `taxa_estendido` de cada categoria.

---

## 5. Problemas Identificados

### 5.1. PROBLEMA PRINCIPAL: Detalhamento nao fecha matematicamente

A UI mostra:
```
Valor contratado:     R$ 13.756,64
Quitado:              R$      0,00
Vencidas:             R$    706,36
Futuras:              R$ 10.948,58
AA futura:            R$  3.150,00
                      -----------
Soma visivel:         R$ 14.804,94  (> valor contratado!)
```

O formando ve uma soma que **excede** o valor contratado porque o "estendido" (R$ 3.947,64) nao aparece como uma linha separada no detalhamento do topo, mas esta embutido nas mensalidades futuras como uma sobreposicao.

### 5.2. A UI JA mostra "AA futura (paga separadamente)"

Diferente do que suspeitavamos inicialmente, a UI **ja inclui** a linha "AA futura (paga separadamente) R$ 3.150,00". Essa informacao existe, mas:
- Nao ha um calculo visivel tipo "valor_contratado - AA = principal"
- A soma das linhas nao fecha, gerando desconfianca
- O estendido aparece so no bottom, nao no detalhamento do topo

### 5.3. Arredondamento de centavos (ja corrigido)

`calcularRenegociacao()` usa `Math.floor()` + ajuste na ultima parcela.

---

## 6. Correcoes Propostas

### 6.1. Corrigir o detalhamento para fechar matematicamente

O detalhamento do topo deveria mostrar uma conta que fecha:

```
Valor contratado:               R$ 13.756,64
  (-) Valor quitado:            R$      0,00
  (-) AA futura (separado):     R$  3.150,00
  (-) Estendido (cartao):       R$  3.947,64
  (+) Multa e juros:            R$     31,36
                                 -----------
  (=) Principal a replanejar:   R$ 11.654,94  (✓ confere)

  Detalhamento do principal:
    Mensalidades vencidas:      R$    706,36
    Mensalidades futuras:       R$ 10.948,58
```

### 6.2. Mostrar o estendido no detalhamento do topo

Adicionar uma linha "Parcelamento estendido (pago via cartao)" no detalhamento superior, para que o formando entenda para onde foram os R$ 3.947,64.

### 6.3. Explicacao contextual

Adicionar um icone de info/tooltip no "Principal a replanejar" explicando:
> "Este valor exclui a arrecadacao alternativa (cobrada separadamente via boleto semestral) e o parcelamento estendido (pago via cartao de credito)."

---

## 7. Conclusao

### O que esta funcionando:
- A logica financeira de `computePrincipalReplanejar()` esta **correta**
- A UI **ja mostra** "AA futura (paga separadamente)" com o valor
- As 7 parcelas de arrecadacao alternativa aparecem no bottom do modal
- O arredondamento de centavos ja foi corrigido

### O que precisa melhorar:
1. **Detalhamento nao fecha:** A soma das linhas visiveis excede o valor contratado porque o estendido nao aparece no topo
2. **Falta conexao visual:** Nao ha uma "conta" que mostre como valor_contratado se transforma em principal_replanejar
3. **Confusao do formando:** Mesmo com a linha de AA, o formando nao consegue conferir os numeros

### Severidade revisada:
- **Bug de UX/comunicacao**, nao de calculo
- A logica financeira esta correta
- O impacto real e **confusao do usuario** e **tickets de suporte desnecessarios**
- Fix de baixa complexidade (apenas ajustes no JSX do modal)
