# Correcao — Bug #399883 + #400011: Parcelas Fora do Periodo Financeiro

**Data:** 2026-03-12
**Bugs:** #399883 (replanejamento) + #400011 (upgrade)
**Status:** CORRIGIDO E TESTADO (27/27 testes passando)

---

## 1. O que foi corrigido

O sistema gerava parcelas com datas de vencimento que **ultrapassavam o `dataUltimaParcela`** configurado na turma. O problema afetava:
- **51 contratos** em producao
- **906 parcelas** fora do periodo
- **R$ 446.403,98** em valor total fora do periodo

### Arquivos alterados

| Arquivo | Alteracao |
|---------|----------|
| `src/services/renegociacaoService.ts` | `calcularRenegociacao()`: novo param `dataUltimaParcela`, validacao no loop, ajuste da ultima parcela ao cortar. `salvarRenegociacao()`: busca `dados_pagamento` da turma, validacao defense-in-depth antes do insert. |
| `src/components/RenegociacaoModal.tsx` | Passa `dadosContrato.dadosPagamento.dataUltimaParcela` para `calcularRenegociacao()`. |
| `src/services/mudancaPlanoService.ts` | `executarMudancaPlano()`: busca `dados_pagamento` da turma, respeita `dataPrimeiraParcela` minima, valida datas no loop de parcelas normais e estendidas. |
| `test-bug-399883-400011.js` | 27 testes standalone (Node.js) |

---

## 2. Detalhes das alteracoes

### 2.1. `renegociacaoService.ts` — `calcularRenegociacao()`

**Antes:**
```typescript
export const calcularRenegociacao = (
  principalReplanejar: number,
  entradaPercentual: number,
  numeroParcelas: number,
  dataPrimeiraParcela?: string,
  parcelamentoEstendido: boolean = false
): ResumoRenegociacao => {
  // ...
  for (let i = 0; i < numeroParcelas; i++) {
    const dataVencimento = adicionarMesesComDiaFixo(dataBase, i, diaVencimento);
    // ← NENHUMA validacao de data
    parcelas.push({ ... });
  }
  return { numeroParcelas, parcelas };
};
```

**Depois:**
```typescript
export const calcularRenegociacao = (
  principalReplanejar: number,
  entradaPercentual: number,
  numeroParcelas: number,
  dataPrimeiraParcela?: string,
  parcelamentoEstendido: boolean = false,
  dataUltimaParcela?: string                // ← NOVO
): ResumoRenegociacao => {
  // ...
  const dataLimite = dataUltimaParcela
    ? new Date(dataUltimaParcela + 'T23:59:59') : null;

  for (let i = 0; i < numeroParcelas; i++) {
    const dataVencimento = adicionarMesesComDiaFixo(dataBase, i, diaVencimento);

    // ← NOVO: Validar contra periodo financeiro
    if (dataLimite && dataVencimento > dataLimite) {
      break;
    }
    parcelas.push({ ... });
  }

  // ← NOVO: Ajustar ultima parcela se houve corte
  if (parcelas.length > 0 && parcelas.length < numeroParcelas) {
    const faltante = principalReplanejar - somaAtual;
    parcelas[parcelas.length - 1].valor += faltante;
  }

  return { numeroParcelas: parcelas.length, parcelas };
};
```

### 2.2. `renegociacaoService.ts` — `salvarRenegociacao()`

**Antes:**
```typescript
const { data: contratoOriginal } = await supabase
  .from('contratos')
  .select('dia_vencimento')
  .eq('id', contratoId)
  .single();
```

**Depois:**
```typescript
const { data: contratoOriginal } = await supabase
  .from('contratos')
  .select('dia_vencimento, turma_id, turmas(dados_pagamento)')  // ← JOIN com turma
  .eq('id', contratoId)
  .single();

const dataUltimaParcela = contratoOriginal?.turmas?.dados_pagamento?.dataUltimaParcela;

// ... antes do insert ...
// ← NOVO: Defense-in-depth
if (dataUltimaParcela) {
  const limiteDate = new Date(dataUltimaParcela + 'T23:59:59');
  const parcelasForaPeriodo = novasParcelas.filter(p =>
    new Date(p.data_vencimento) > limiteDate
  );
  if (parcelasForaPeriodo.length > 0) {
    throw new Error(`${parcelasForaPeriodo.length} parcela(s) com vencimento apos periodo financeiro`);
  }
}
```

### 2.3. `RenegociacaoModal.tsx`

**Antes:**
```typescript
const resumo = calcularRenegociacao(
  principalReplanejar, entradaInteligente, totalParcelas,
  dataPrimeiraParcela, isParcelamentoEstendidoAtivo
);
```

**Depois:**
```typescript
const resumo = calcularRenegociacao(
  principalReplanejar, entradaInteligente, totalParcelas,
  dataPrimeiraParcela, isParcelamentoEstendidoAtivo,
  dadosContrato?.dadosPagamento?.dataUltimaParcela  // ← NOVO
);
```

### 2.4. `mudancaPlanoService.ts` — `executarMudancaPlano()`

**Antes:**
```typescript
const dataPrimeiraParcela = new Date();
dataPrimeiraParcela.setDate(mudanca.dia_vencimento_novo || 10);
// ← Sem buscar turma, sem validar periodo

for (let i = 0; i < qtdParcelasNormais; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(...);
  // ← Sem validacao
  parcelas.push({ ... });
}
```

**Depois:**
```typescript
// ← NOVO: Buscar periodo financeiro da turma
const { data: turmaParaPeriodo } = await supabase
  .from('turmas')
  .select('dados_pagamento')
  .eq('id', mudanca.turma_id)
  .single();

const dataLimiteTurma = turmaParaPeriodo?.dados_pagamento?.dataUltimaParcela
  ? new Date(dataUltimaParcela + 'T23:59:59') : null;
const dataInicioTurma = turmaParaPeriodo?.dados_pagamento?.dataPrimeiraParcela
  ? new Date(dataPrimeiraParcela) : null;

// ← NOVO: Respeitar data minima da turma
if (dataInicioTurma && dataPrimeiraParcela < dataInicioTurma) { ... }

for (let i = 0; i < qtdParcelasNormais; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(...);

  // ← NOVO: Validar contra limite
  if (dataLimiteTurma && dataVencimento > dataLimiteTurma) {
    break;
  }
  parcelas.push({ ... });
}

// ← NOVO: Parcelas estendidas tambem validam o limite
if (dataLimiteTurma && dataUltimaNormal > dataLimiteTurma) {
  // Nao cria estendidas fora do periodo
}
```

---

## 3. Resultado dos testes (27/27 passaram)

### Comportamento original preservado
| Teste | Resultado |
|-------|-----------|
| Sem dataUltimaParcela: 63 parcelas | PASS |
| Soma = principal sem limite | PASS |

### Com limite — todas cabem
| Teste | Resultado |
|-------|-----------|
| Limite em 2031-12-31: 63 parcelas ok | PASS |
| Ultima parcela dentro do limite | PASS |

### Com limite — parcelas cortadas (cenario do bug)
| Teste | Resultado |
|-------|-----------|
| Turma BIOMEDICINA (limite 2029-07-18): 40 parcelas (era 63) | PASS |
| Ultima <= 2029-07-18 | PASS |
| Soma = R$ 13.179,60 (ajuste na ultima) | PASS |

### Limite curto
| Teste | Resultado |
|-------|-----------|
| Limite 2026-12-31: 9 parcelas (era 24) | PASS |
| Soma = principal | PASS |

### Edge cases
| Teste | Resultado |
|-------|-----------|
| dataPrimeiraParcela APOS limite: 0 parcelas | PASS |
| Limite exato no ultimo mes: 6 parcelas ok | PASS |
| Sem dataPrimeiraParcela, com limite futuro | PASS |

### Principal grande com corte
| Teste | Resultado |
|-------|-----------|
| R$ 50.000, 60→25 parcelas pelo limite | PASS |
| Soma = R$ 50.000,00 (exato) | PASS |
| Ultima parcela <= 2028-04-10 | PASS |

### Casos reais de producao
| Teste | Resultado |
|-------|-----------|
| BIOMEDICINA PUC: era 62 parcelas ate 2031, agora 44 ate 2029-07 | PASS |
| Soma = R$ 13.395,00 | PASS |
| FORMAE Teste1.2: era 62 ate 2031, agora 42 ate 2029-04 | PASS |
| Soma = R$ 9.378,00 | PASS |

---

## 4. TypeScript

```
npx tsc --noEmit --skipLibCheck → 0 erros
```

---

## 5. Impacto e riscos

- **Zero regressao:** Sem `dataUltimaParcela` (param default undefined), comportamento identico ao anterior
- **Parcelas existentes nao sao afetadas:** A correcao atua apenas na GERACAO de novas parcelas
- **Defense-in-depth:** Mesmo que o calculo falhe, o `salvarRenegociacao()` valida antes do insert
- **Upgrade coberto:** `mudancaPlanoService` agora busca e respeita periodo da turma
- **Soma sempre fecha:** Quando parcelas sao cortadas, a ultima absorve o valor restante

### Limitacao conhecida
- As **906 parcelas existentes** fora do periodo NAO sao corrigidas automaticamente. Requer script de correcao ou decisao manual da equipe financeira.
