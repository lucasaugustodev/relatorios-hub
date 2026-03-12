# Analise — Bug #399995: Upgrade de Plano com Sobrecobranca

**Data:** 2026-03-12
**Bug:** #399995
**Status:** BUG CONFIRMADO — CRITICO
**Severidade:** FINANCEIRA ALTA — R$ 297.144,91 em sobrecobranca

---

## 1. Resumo Executivo

O sistema de upgrade de plano esta **cobrando valores massivamente acima do correto** de todos os formandos que fizeram upgrade. A causa raiz e que `calcularAlteracao()` inclui `ACORDOS_NAO_QUITADOS` no calculo — parcelas de renegociacao que representam o **principal reestruturado completo** do plano antigo. Quando o formando faz upgrade, essas parcelas deveriam ser canceladas junto com o contrato antigo, mas em vez disso sao somadas como divida adicional ao plano novo.

### Impacto em producao

| Metrica | Valor |
|---------|-------|
| Upgrades ativos afetados | **24** |
| Sobrecobranca total | **R$ 297.144,91** |
| Media por formando | **R$ 12.381,04** |
| Maior sobrecobranca individual | **R$ 46.423,41** |
| Valor correto total dos planos | R$ 371.767,23 |
| % sobrecobranca vs valor real | **79,9%** |

---

## 2. Causa Raiz

### 2.1. ACORDOS_NAO_QUITADOS inclui principal reestruturado da renegociacao

O arquivo `src/services/mudancaPlanoService.ts` usa a funcao `calcularAlteracao()` (linha 295) que aplica a formula:

```
SALDO_BASE = MULTA - PRINCIPAL_QUITADO
PENDENCIAS = JUROS_MULTAS_PENDENTES + ACORDOS_NAO_QUITADOS
valorFinal = SALDO_BASE + PENDENCIAS  → vira DEBITO
```

**O problema esta em `ACORDOS_NAO_QUITADOS`** (linhas 368-378):

```typescript
// Busca parcelas de renegociacao NAO pagas do contrato antigo
const ACORDOS_NAO_QUITADOS = (parcelas || [])
  .filter(p => {
    const tipo = p.tipo || 'normal';
    return ['renegociacao', 'entrada_renegociacao'].includes(tipo) &&
      ['pendente', 'vencido'].includes(p.status);
  })
  .reduce((sum, p) => {
    const valorTotal = parseFloat(p.valor || '0');
    const valorPago = parseFloat(p.valor_pago || '0');
    return sum + Math.max(0, valorTotal - valorPago);
  }, 0);
```

**O filtro pega os tipos errados.** A renegociacao (`renegociacaoService.ts`) cria 3 tipos de parcela:

| Tipo | Linha | O que e |
|------|-------|---------|
| `taxa_renegociacao` | L535 | **Juros e multas negociados** (o "acordo" real) |
| `entrada_renegociacao` | L701 | Primeira parcela do **principal reestruturado** |
| `renegociacao` | L701 | Demais parcelas do **principal reestruturado** |

O filtro captura `renegociacao` + `entrada_renegociacao` = **principal reestruturado** (o mesmo dinheiro que ja esta em `A_valorPlano`). NAO captura `taxa_renegociacao` (os juros/multas reais).

**Resultado: captura exatamente o contrario do que deveria.**

#### Prova via backup (Aninha Trindade)

O backup das parcelas canceladas pelo upgrade mostra os tipos ORIGINAIS:

| Tipo Original | Qtd | Soma | O que e |
|---|---|---|---|
| `entrada_renegociacao` | 1 | R$ 596,00 | Principal reestruturado (CAPTURADO) |
| `renegociacao` | 42 | R$ 25.032,00 | Principal reestruturado (CAPTURADO) |
| `taxa_renegociacao` | 1 | R$ 954,56 | Juros/multas reais (NAO CAPTURADO) |

O filtro captura: R$ 596 + R$ 25.032 = **R$ 25.628** ← exatamente o valor no JSON!
O acordo real (juros/multas) de R$ 954,56 **nao e capturado**.

O formando paga o principal do plano antigo DUAS VEZES: uma vez como `ACORDOS_NAO_QUITADOS` no debito, e outra vez embutido no preco integral do plano novo.

O formando acaba pagando:
1. O valor INTEGRAL do plano novo (entrada + parcelas)
2. MAIS a MULTA de upgrade (% do plano antigo)
3. MAIS o saldo da renegociacao (= basicamente o plano antigo inteiro de novo)

### 2.2. DEBITO adicionado POR CIMA do preco do plano novo

Em `calcularMudancaPlano()` (linha 571):

```typescript
} else if (tipoResultado === 'DEBITO') {
  debitoParcelaAjuste = valorDebito; // ← valor INTEGRAL do debito
}
```

E depois (linha 575-576):

```typescript
const valorBaseParcelamento = valorNovoPlano - valorEntradaOriginal;
const valorParcelaComDesconto = (valorBaseParcelamento - creditoAplicadoParcelas) / numeroParcelas;
```

O `valorNovoPlano` NAO e reduzido pelo debito — o debito e simplesmente adicionado como parcela extra. O formando paga o plano novo INTEIRO + o debito.

### 2.3. Parcelamento estendido inflando o total

17 dos 24 upgrades ativos tem `tem_parcelamento_estendido = true` com 21 parcelas extras, multiplicando a sobrecobranca.

---

## 3. Prova com dados de producao (rescisao_calculation JSON)

### Caso 1: Aninha Trindade (PIOR CASO)

| Campo | Valor |
|-------|-------|
| Plano antigo | R$ 25.628,01 |
| Plano novo | R$ 34.318,71 |
| MULTA (15%) | R$ 3.844,20 |
| PRINCIPAL_QUITADO | R$ 0,00 |
| **ACORDOS_NAO_QUITADOS** | **R$ 25.628,00** |
| SALDO_BASE | R$ 3.844,20 |
| PENDENCIAS | R$ 25.628,00 |
| **DEBITO (ajuste)** | **R$ 29.472,20** |
| Entrada novo | R$ 6.863,74 |
| Parcelas (45x) | R$ 73.878,38 |
| **TOTAL A PAGAR** | **R$ 80.742,12** |
| **SOBRECOBRANCA** | **R$ 46.423,41 (135%)** |

**O que deveria pagar:** R$ 34.318,71 (valor do plano novo)
**O que vai pagar:** R$ 80.742,12

### Caso 2: Ingrid Gomes Correa

| Campo | Valor |
|-------|-------|
| Plano antigo | R$ 14.778,10 |
| Plano novo | R$ 17.169,85 |
| MULTA (40%) | R$ 5.911,24 |
| ACORDOS_NAO_QUITADOS | R$ 13.357,30 |
| **DEBITO** | **R$ 19.268,54** |
| **TOTAL A PAGAR** | **R$ 44.769,58** |
| **SOBRECOBRANCA** | **R$ 27.599,73 (161%)** |

### Caso 3: Maria Joaquina Gomes

| Campo | Valor |
|-------|-------|
| Plano antigo | R$ 13.688,20 |
| Plano novo | R$ 16.329,35 |
| MULTA (20%) | R$ 2.737,64 |
| ACORDOS_NAO_QUITADOS | R$ 13.239,60 |
| **DEBITO** | **R$ 15.977,24** |
| **TOTAL A PAGAR** | **R$ 41.988,07** |
| **SOBRECOBRANCA** | **R$ 25.658,72 (157%)** |

### Caso 4: Criselda Isabelle

| Campo | Valor |
|-------|-------|
| Plano antigo | R$ 13.688,20 |
| Plano novo | R$ 16.329,35 |
| MULTA (20%) | R$ 2.737,64 |
| ACORDOS_NAO_QUITADOS | R$ 13.688,36 |
| **DEBITO** | **R$ 16.426,00** |
| **TOTAL A PAGAR** | **R$ 39.286,94** |
| **SOBRECOBRANCA** | **R$ 22.957,59 (141%)** |

---

## 4. Todos os upgrades com sobrecobranca (producao)

| Formando | Plano Novo | Total a Pagar | Sobrecobranca |
|----------|-----------|---------------|---------------|
| Aninha Trindade | R$ 34.318,71 | R$ 80.742,12 | **R$ 46.423,41** |
| Ingrid Gomes Correa | R$ 17.169,85 | R$ 44.769,58 | **R$ 27.599,73** |
| Maria Joaquina Gomes | R$ 16.329,35 | R$ 41.988,07 | **R$ 25.658,72** |
| Aninha | R$ 16.329,35 | R$ 39.286,98 | **R$ 22.957,63** |
| Criselda Isabelle | R$ 16.329,35 | R$ 39.286,94 | **R$ 22.957,59** |
| Joao Henrique (21k) | R$ 21.282,66 | R$ 36.301,90 | **R$ 15.019,24** |
| Isabelle Matos | R$ 21.282,66 | R$ 36.010,02 | **R$ 14.727,36** |
| Teste CT-UPG-01 | R$ 16.329,35 | R$ 26.434,70 | **R$ 10.105,35** |
| Teste CT-UPG-06 | R$ 16.329,35 | R$ 26.434,70 | **R$ 10.105,35** |
| Teste CT-UPG-05 | R$ 16.329,35 | R$ 26.434,70 | **R$ 10.105,35** |
| Teste CT-UPG-02 | R$ 16.329,35 | R$ 26.434,70 | **R$ 10.105,35** |
| Teste CT-UPG-03 | R$ 16.329,35 | R$ 26.434,70 | **R$ 10.105,35** |
| lucas augusto | R$ 16.329,35 | R$ 25.634,46 | **R$ 9.305,11** |
| Priscilla Aguiar | R$ 10.000,00 | R$ 19.224,73 | **R$ 9.224,73** |
| Eduardo Reuter | R$ 11.047,05 | R$ 19.796,88 | **R$ 8.749,83** |
| Izadora Carvalho | R$ 12.290,65 | R$ 20.123,64 | **R$ 7.832,99** |
| Lucas Augusto | R$ 13.688,20 | R$ 20.540,59 | **R$ 6.852,39** |
| lucas (16k) | R$ 16.329,35 | R$ 22.489,70 | **R$ 6.160,35** |
| lucas augusto (13k) | R$ 13.688,20 | R$ 19.369,82 | **R$ 5.681,62** |
| Criselda Isabelle (13k) | R$ 13.688,20 | R$ 19.369,68 | **R$ 5.681,48** |
| Ana Trindade | R$ 13.688,20 | R$ 18.661,86 | **R$ 4.973,66** |
| lucas augusto goncalo | R$ 16.329,35 | R$ 19.718,86 | **R$ 3.389,51** |
| Priscilla Aguiar (5k) | R$ 5.000,00 | R$ 6.809,38 | **R$ 1.809,38** |
| Priscilla Aguiar (5k-2) | R$ 5.000,00 | R$ 6.613,43 | **R$ 1.613,43** |

---

## 5. Bug secundario: valor_pago_plano_antigo nao creditado

O campo `valor_pago_plano_antigo` esta como `0.00` mesmo quando o formando JA havia pago parcelas:

| Formando | Registrado | Real Pago | Nao Creditado |
|----------|-----------|-----------|---------------|
| Isabelle Matos | R$ 0,00 | R$ 537,28 | **R$ 537,28** |
| Maria Joaquina Gomes | R$ 0,00 | R$ 490,06 | **R$ 490,06** |
| Izadora Carvalho | R$ 0,00 | R$ 482,27 | **R$ 482,27** |

A funcao `calcularAlteracao()` busca parcelas com `status === 'pago'` (linha 337). Porem o valor registrado em `mudancas_plano.valor_pago_plano_antigo` nao reflete o calculo real, sugerindo que as parcelas foram pagas DEPOIS do upgrade ou ha divergencia no status armazenado.

---

## 6. Fluxo do bug (passo a passo)

```
1. Formando tem contrato antigo (plano R$ 25.628,01)
   └─ Contrato foi RENEGOCIADO → 42 parcelas tipo 'renegociacao' (R$ 25.032)
      + 1 entrada_renegociacao (R$ 596) = R$ 25.628 de principal reestruturado

2. Formando solicita UPGRADE para plano R$ 34.318,71

3. calcularAlteracao() executa:
   ├─ MULTA = 15% × R$ 25.628,01 = R$ 3.844,20
   ├─ PRINCIPAL_QUITADO = R$ 0,00 (nada pago)
   ├─ ACORDOS_NAO_QUITADOS = R$ 25.628,00 ← parcelas renegociacao pendentes
   │   (sao o principal reestruturado, NAO juros/multas!)
   ├─ SALDO_BASE = R$ 3.844,20 - R$ 0 = R$ 3.844,20
   ├─ PENDENCIAS = R$ 0 + R$ 25.628,00 = R$ 25.628,00
   └─ DEBITO = R$ 3.844,20 + R$ 25.628,00 = R$ 29.472,20

4. calcularMudancaPlano() aplica:
   ├─ Entrada = R$ 6.863,74 (20% do plano novo)
   ├─ 45 parcelas = R$ 73.878,38 (plano novo integral)
   ├─ Parcela ajuste (DEBITO) = R$ 29.472,20 (multa + plano antigo)
   └─ TOTAL = R$ 6.863,74 + R$ 73.878,38 = R$ 80.742,12

5. executarMudancaPlano() depois:
   └─ Cancela as parcelas de renegociacao → tipo 'cancelada_mudanca_plano'
      (mas o DEBITO ja foi calculado com elas como divida!)

6. Resultado: formando paga R$ 80.742 por um plano de R$ 34.318 (235%!)
   = Plano novo inteiro + plano antigo inteiro + multa
```

---

## 7. Arquivos afetados

| Arquivo | Problema |
|---------|----------|
| `src/services/mudancaPlanoService.ts` — `calcularAlteracao()` (L295-461) | Formula de rescisao soma ACORDOS_NAO_QUITADOS como divida no upgrade |
| `src/services/mudancaPlanoService.ts` — `calcularMudancaPlano()` (L466-652) | DEBITO adicionado POR CIMA do preco integral do plano novo |
| `src/services/mudancaPlanoService.ts` — `executarMudancaPlano()` (L831+) | Cria parcela de ajuste + parcelas normais + estendidas sem validacao |
| `src/components/MudancaPlanoModal.tsx` | UI nao mostra breakdown detalhado da composicao do debito |

---

## 8. Correcao necessaria

### O que o upgrade FAZ HOJE (errado):

```
1. Cobra o plano novo INTEIRO (entrada + parcelas)
2. MAIS a MULTA (% do plano antigo)
3. MAIS o principal reestruturado da renegociacao (ACORDOS_NAO_QUITADOS)
   → parcelas tipo 'renegociacao' que representam o PAGAMENTO do plano antigo
   → essas parcelas serao CANCELADAS pelo upgrade, mas ja foram contadas como divida
4. = Formando paga plano novo + plano antigo + multa
```

### O que DEVERIA fazer:

O `ACORDOS_NAO_QUITADOS` deveria capturar **apenas juros e multas negociados** (`taxa_renegociacao`), nao o principal reestruturado (`renegociacao` / `entrada_renegociacao`):

```typescript
// CORRECAO: Filtrar apenas taxa_renegociacao (juros/multas reais)
const ACORDOS_NAO_QUITADOS = (parcelas || [])
  .filter(p => {
    const tipo = p.tipo || 'normal';
    return tipo === 'taxa_renegociacao' &&  // ← ANTES: ['renegociacao', 'entrada_renegociacao']
      ['pendente', 'vencido'].includes(p.status);
  })
  .reduce((sum, p) => {
    const valorTotal = parseFloat(p.valor || '0');
    const valorPago = parseFloat(p.valor_pago || '0');
    return sum + Math.max(0, valorTotal - valorPago);
  }, 0);
```

Isso faria o calculo correto para Aninha Trindade:
- ACORDOS_NAO_QUITADOS = R$ 954,56 (taxa real) em vez de R$ 25.628 (principal)
- DEBITO = R$ 3.844,20 + R$ 954,56 = R$ 4.798,76 em vez de R$ 29.472,20

**Nota:** A mudanca no filtro afeta TANTO upgrade quanto rescisao. Verificar se a rescisao tambem deveria usar `taxa_renegociacao` ou se para rescisao o comportamento atual e correto.

---

## 9. Recomendacoes

1. **URGENTE**: Suspender upgrades ate correcao ser aplicada
2. **Corrigir `calcularAlteracao()`**: Nao incluir `ACORDOS_NAO_QUITADOS` para tipo `upgrade` — parcelas do contrato antigo serao canceladas
3. **Corrigir logica de debito**: O debito (se houver) deve ser a DIFERENCA entre planos, nao um valor adicional
4. **Script de correcao**: Recalcular e corrigir os 24 contratos ativos com sobrecobranca
5. **Validacao**: Adicionar check que impeca total_a_pagar > valor_plano_novo × 1.5 (safety limit)
