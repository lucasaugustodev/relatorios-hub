# Analise — Bug #399883 + #400011: Parcelas com Datas Fora do Periodo Financeiro

**Data:** 2026-03-12
**Severidade:** ALTA — 51 contratos afetados em producao, R$ 446.403,98 em parcelas fora do periodo
**Status:** CONFIRMADO com dados de producao
**Bugs:** #399883 (replanejamento) + #400011 (upgrade)

---

## 1. Resumo

O sistema gera parcelas com datas de vencimento que **ultrapassam o `dataUltimaParcela`** configurado na turma. O problema ocorre em dois fluxos:

1. **Replanejamento** (`renegociacaoService.ts`) — quando `dataPrimeiraParcela` nao e fornecida, usa `new Date()` sem validar contra o periodo da turma
2. **Upgrade de plano** (`mudancaPlanoService.ts`) — **nunca busca** `dados_pagamento` da turma, calcula datas livremente

Ambos os fluxos **nao validam** se as datas geradas ficam dentro de `[dataPrimeiraParcela, dataUltimaParcela]` da turma.

---

## 2. Impacto em Producao

| Metrica | Valor |
|---------|-------|
| Contratos afetados | **51** |
| Parcelas fora do periodo | **906** |
| Valor total fora do periodo | **R$ 446.403,98** |
| Parcela mais antiga fora | 2025-10-31 |
| Parcela mais distante fora | **2031-02-10** |

### Breakdown por tipo:

| Tipo | Tipo Parcela | Contratos | Parcelas | Valor |
|------|-------------|-----------|----------|-------|
| normal | normal | 48 | 565 | R$ 193.099,23 |
| normal | estendido | 34 | 300 | R$ 235.750,59 |
| renegociacao | normal | 2 | 19 | R$ 4.166,06 |
| rescindidas | estendido | 1 | 21 | R$ 12.779,55 |

### Casos concretos:

- **BIOMEDICINA PUC** (turma limite: 2029-07-18) — parcelas ate **2031-02-10** (19 meses alem!)
- **FORMAE Teste1.2** (turma limite: 2029-04-19) — parcelas ate **2031-01-14** (21 meses alem!)
- **MEDICINA CIENCIAS MEDICAS 84** (turma limite: 2030-12-06) — 23+ parcelas estendidas de R$12 no mesmo dia 2030-12-12

---

## 3. Causa Raiz

### 3.1. Replanejamento — `renegociacaoService.ts`

**`calcularRenegociacao()` (linhas 286-369):**
```typescript
export const calcularRenegociacao = (
  principalReplanejar: number,
  entradaPercentual: number,
  numeroParcelas: number,
  dataPrimeiraParcela?: string,     // Unico parametro de data
  parcelamentoEstendido: boolean = false
): ResumoRenegociacao => {
```

**Problema 1:** Nao recebe `dataUltimaParcela` como parametro. Impossivel validar.

**Problema 2 (linhas 313-319):** Fallback para `new Date()`:
```typescript
let dataBase: Date;
if (dataPrimeiraParcela) {
  const [ano, mes, dia] = dataPrimeiraParcela.split('-').map(Number);
  dataBase = new Date(ano, mes - 1, dia);
} else {
  dataBase = new Date();  // ← USA HOJE, ignora periodo da turma
}
```

**Problema 3 (linhas 326-351):** Loop gera parcelas sem checar limite:
```typescript
for (let i = 0; i < numeroParcelas; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(dataBase, i, diaVencimento);
  // ← NUNCA checa se dataVencimento > dataUltimaParcela
  parcelas.push({ ... });
}
```

**Nota:** O modal (`RenegociacaoModal.tsx`) tem `calcularMaxParcelasDisponiveis()` que limita o numero de parcelas pelo periodo. Porem:
- A validacao e apenas no NUMERO de parcelas, nao nas DATAS efetivas
- Se `dataPrimeiraParcela` nao e fornecida, o calculo usa hoje como base
- O servico nao faz nenhuma validacao propria — confia no modal

### 3.2. Upgrade — `mudancaPlanoService.ts`

**`executarMudancaPlano()` (linhas 921-926):**
```typescript
const dataPrimeiraParcela = new Date();                       // ← USA HOJE
dataPrimeiraParcela.setDate(mudanca.dia_vencimento_novo || 10);
if (dataPrimeiraParcela <= new Date()) {
  dataPrimeiraParcela.setMonth(dataPrimeiraParcela.getMonth() + 1);
}
// ← NUNCA busca dados_pagamento da turma
// ← NUNCA valida contra dataUltimaParcela
```

**Loop de parcelas (linhas 1014-1031):**
```typescript
for (let i = 0; i < qtdParcelasNormais; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(dataPrimeira, i, diaVencimentoConfig);
  // ← NUNCA checa se dataVencimento > turma.dataUltimaParcela
  parcelas.push({ data_vencimento: dataVencimento.toISOString().split('T')[0] });
}
```

**Loop estendidas (linhas 1034-1051):** Mesmo problema — nenhuma validacao.

---

## 4. Fluxo Correto de Referencia — `SelecaoParcelamento.tsx`

O fluxo de contratacao original **ja implementa corretamente** a validacao:

```typescript
// SelecaoParcelamento.tsx:117-154
const calcularMaxParcelasDisponiveis = () => {
  if (!dadosPagamento?.dataUltimaParcela) {
    return dadosPagamento?.numeroMaximoParcelas || 60;
  }
  const dataUltima = new Date(dadosPagamento.dataUltimaParcela);
  const diffAnos = dataUltima.getFullYear() - primeiraParcela.getFullYear();
  const diffMeses = dataUltima.getMonth() - primeiraParcela.getMonth();
  const mesesDisponiveis = (diffAnos * 12) + diffMeses + 1;
  const maxConfigurado = dadosPagamento?.numeroMaximoParcelas || 60;
  return Math.max(1, Math.min(mesesDisponiveis, maxConfigurado));
};
```

---

## 5. Plano de Correcao

### 5.1. `renegociacaoService.ts` — Adicionar validacao de datas

**Arquivo:** `src/services/renegociacaoService.ts`

**Mudanca 1:** Adicionar parametro `dataUltimaParcela` em `calcularRenegociacao()`:
```typescript
export const calcularRenegociacao = (
  principalReplanejar: number,
  entradaPercentual: number,
  numeroParcelas: number,
  dataPrimeiraParcela?: string,
  parcelamentoEstendido: boolean = false,
  dataUltimaParcela?: string           // ← NOVO
): ResumoRenegociacao => {
```

**Mudanca 2:** Validar fallback de `dataBase` contra periodo da turma:
```typescript
let dataBase: Date;
if (dataPrimeiraParcela) {
  const [ano, mes, dia] = dataPrimeiraParcela.split('-').map(Number);
  dataBase = new Date(ano, mes - 1, dia);
} else {
  dataBase = new Date();
}
dataBase.setHours(0, 0, 0, 0);

// ← NOVO: Validar contra periodo financeiro
if (dataUltimaParcela) {
  const dataLimite = new Date(dataUltimaParcela);
  const ultimaParcelaGerada = adicionarMesesComDiaFixo(dataBase, numeroParcelas - 1, dataBase.getDate());
  if (ultimaParcelaGerada > dataLimite) {
    // Calcular max parcelas que cabem no periodo
    const mesesDisponiveis = (dataLimite.getFullYear() - dataBase.getFullYear()) * 12
      + dataLimite.getMonth() - dataBase.getMonth() + 1;
    if (mesesDisponiveis < numeroParcelas) {
      console.warn(`⚠️ [RENEGOCIACAO] ${numeroParcelas} parcelas ultrapassam periodo. Max: ${mesesDisponiveis}`);
      // Ajustar ou lancar erro conforme regra de negocio
    }
  }
}
```

**Mudanca 3:** No loop, validar cada data:
```typescript
for (let i = 0; i < numeroParcelas; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(dataBase, i, diaVencimento);

  // ← NOVO: Impedir parcela alem do limite
  if (dataUltimaParcela) {
    const dataLimite = new Date(dataUltimaParcela);
    if (dataVencimento > dataLimite) {
      console.warn(`⚠️ Parcela ${i+1} (${dataVencimento.toISOString()}) ultrapassa limite (${dataUltimaParcela})`);
      // Opcao A: Ajustar para dataLimite
      // Opcao B: Parar de gerar parcelas
      // Opcao C: Lancar erro
    }
  }
  // ... resto do loop
}
```

### 5.2. `RenegociacaoModal.tsx` — Passar `dataUltimaParcela` para o servico

**Arquivo:** `src/components/RenegociacaoModal.tsx`

Na chamada a `calcularRenegociacao()`:
```typescript
const resumo = calcularRenegociacao(
  principalReplanejar,
  entradaInteligente,
  totalParcelas,
  dataPrimeiraParcela,
  isParcelamentoEstendidoAtivo,
  dadosContrato?.dadosPagamento?.dataUltimaParcela  // ← NOVO
);
```

### 5.3. `mudancaPlanoService.ts` — Buscar e respeitar periodo da turma

**Arquivo:** `src/services/mudancaPlanoService.ts`

**Mudanca 1:** Buscar `dados_pagamento` da turma (antes da linha 921):
```typescript
// ← NOVO: Buscar periodo financeiro da turma
const { data: turmaData } = await supabase
  .from('turmas')
  .select('dados_pagamento')
  .eq('id', mudanca.turma_id)
  .single();

const dadosPagamento = turmaData?.dados_pagamento;
const dataLimiteTurma = dadosPagamento?.dataUltimaParcela
  ? new Date(dadosPagamento.dataUltimaParcela)
  : null;
const dataInicioTurma = dadosPagamento?.dataPrimeiraParcela
  ? new Date(dadosPagamento.dataPrimeiraParcela)
  : null;
```

**Mudanca 2:** Validar `dataPrimeiraParcela` contra periodo (linhas 921-926):
```typescript
const dataPrimeiraParcela = new Date();
dataPrimeiraParcela.setDate(mudanca.dia_vencimento_novo || 10);
if (dataPrimeiraParcela <= new Date()) {
  dataPrimeiraParcela.setMonth(dataPrimeiraParcela.getMonth() + 1);
}

// ← NOVO: Respeitar data minima da turma
if (dataInicioTurma && dataPrimeiraParcela < dataInicioTurma) {
  dataPrimeiraParcela.setTime(dataInicioTurma.getTime());
  dataPrimeiraParcela.setDate(mudanca.dia_vencimento_novo || 10);
}
```

**Mudanca 3:** Validar no loop de parcelas (linhas 1014-1031):
```typescript
for (let i = 0; i < qtdParcelasNormais; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(dataPrimeira, i, diaVencimentoConfig);

  // ← NOVO: Validar contra limite da turma
  if (dataLimiteTurma && dataVencimento > dataLimiteTurma) {
    console.warn(`⚠️ [UPGRADE] Parcela ${i+2} ultrapassa periodo financeiro. Parando.`);
    break;
  }

  parcelas.push({ ... });
}
```

**Mudanca 4:** Mesmo para parcelas estendidas (linhas 1034-1051).

### 5.4. `salvarRenegociacao()` — Validacao de seguranca (defense in depth)

**Arquivo:** `src/services/renegociacaoService.ts`

Antes de inserir parcelas no banco (linhas 700-720):
```typescript
// ← NOVO: Validacao final antes do insert
if (dadosPagamento?.dataUltimaParcela) {
  const dataLimite = new Date(dadosPagamento.dataUltimaParcela);
  const parcelasForaPeriodo = novasParcelas.filter(p =>
    new Date(p.data_vencimento) > dataLimite
  );
  if (parcelasForaPeriodo.length > 0) {
    throw new Error(
      `${parcelasForaPeriodo.length} parcelas excedem o periodo financeiro da turma (${dadosPagamento.dataUltimaParcela})`
    );
  }
}
```

---

## 6. Arquivos a Alterar

| Arquivo | Alteracao | Risco |
|---------|----------|-------|
| `src/services/renegociacaoService.ts` | Adicionar `dataUltimaParcela` param + validacao no loop | BAIXO — param opcional com default |
| `src/components/RenegociacaoModal.tsx` | Passar `dataUltimaParcela` na chamada | BAIXO — campo ja disponivel |
| `src/services/mudancaPlanoService.ts` | Buscar `dados_pagamento` + validar datas | MEDIO — novo fetch + logica |
| `src/services/renegociacaoService.ts` | Defense-in-depth no `salvarRenegociacao()` | BAIXO — validacao extra |

---

## 7. Correcao de Dados em Producao

Apos o fix, sera necessario corrigir as **906 parcelas** que ja estao fora do periodo:

**Opcao A (conservadora):** Relatorio para equipe financeira decidir caso a caso
**Opcao B (automatica):** Script que:
1. Identifica parcelas fora do periodo
2. Recalcula datas dentro do periodo (mantendo mesmo numero de parcelas se possivel)
3. Ou cancela parcelas excedentes e redistribui valor nas parcelas dentro do periodo

Recomendo **Opcao A** por seguranca — a equipe financeira deve validar antes de alterar parcelas ativas.

---

## 8. Prioridade de Implementacao

1. **PRIMEIRO:** Fix no `mudancaPlanoService.ts` (mais critico — nao tem NENHUMA validacao)
2. **SEGUNDO:** Fix no `renegociacaoService.ts` (menos critico — modal ja limita parcelas)
3. **TERCEIRO:** Defense-in-depth no `salvarRenegociacao()`
4. **QUARTO:** Correcao de dados existentes
