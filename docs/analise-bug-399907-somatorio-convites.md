# Analise — Bug #399907: Erro na Somatoria dos Valores (Convite Extra)

**Data:** 2026-03-12
**Severidade:** Alta (afeta dinheiro)
**Status:** CONFIRMADO no codigo
**Impacto em producao:** Nenhum ate agora (todos os 475 contratos tem quantidade = 1)

---

## 1. Resumo

O desconto por tipo de usuario (`valor_fixo`) e aplicado **uma unica vez sobre o total**, em vez de ser **multiplicado pela quantidade**. Quando um usuario compra multiplos convites extras na lojinha, o desconto fixo nao escala com a quantidade.

**Exemplo concreto (Lote "2 LOTE" — CONVITE BAILE):**
- Valor unitario: R$ 1.200,00
- Desconto comissao: R$ 200,00 (valor_fixo)
- Quantidade: 5 convites

| | Esperado | Gerado |
|--|----------|--------|
| Valor base | 5 x R$ 1.200 = R$ 6.000 | 5 x R$ 1.200 = R$ 6.000 |
| Desconto | 5 x R$ 200 = **R$ 1.000** | 1 x R$ 200 = **R$ 200** |
| Valor final | **R$ 5.000** | **R$ 5.800** |
| Diferenca | | **+R$ 800 a mais pro formando** |

---

## 2. Codigo com Problema

### 2.1. `src/hooks/useCalculoParcelamento.ts` — funcao `calcularDescontoTipoUsuario` (linhas 333-375)

```typescript
function calcularDescontoTipoUsuario(
  plano: PlanoComLote,
  valorComCorrecao: number,  // ja multiplicado por quantidade (linha 79)
  tipoUsuario: string
): number {
  if (!plano.lote.descontos) return 0;
  const descontos = plano.lote.descontos;
  let desconto = 0;

  switch (tipoUsuario) {
    case 'comissao':
      if (descontos.comissao) {
        desconto = descontos.comissao.tipo === 'valor_fixo'
          ? descontos.comissao.valor || 0          // ← BUG: R$ 200 fixo, independente da quantidade
          : valorComCorrecao * ((descontos.comissao.valor || 0) / 100);  // % esta OK (aplica sobre total)
      }
      break;
    // ... mesmo padrao para aderente, nao_aderente, plano_social
  }
  return desconto;
}
```

**O problema:** No caso `valor_fixo`, retorna o valor bruto do desconto (ex: R$ 200) sem multiplicar pela quantidade. No caso `porcentagem`, funciona corretamente porque aplica sobre `valorComCorrecao` que ja inclui a multiplicacao por quantidade (linha 79: `valorBase = plano.lote.valor * quantidade`).

### 2.2. Contexto de chamada (linhas 78-90)

```typescript
// Linha 79 — valor base JA multiplicado por quantidade
const valorBase = plano.lote.valor * quantidade;

// Linha 82-83 — correcao sobre o total (OK)
const valorCorrecao = calcularCorrecao(plano, valorBase, dataSimulacao);
const valorComCorrecao = valorBase + valorCorrecao;

// Linha 86-90 — desconto tipo usuario: NAO recebe quantidade!
const descontoTipoUsuario = calcularDescontoTipoUsuario(
  plano,
  valorComCorrecao,
  tipoUsuario
);
```

A funcao `calcularDescontoTipoUsuario` nao recebe `quantidade` como parametro, entao nao tem como multiplicar o desconto fixo.

### 2.3. Mesmo problema no desconto de adesao (linhas 377-408)

```typescript
function calcularDescontoAdesao(
  plano: PlanoComLote,
  valorComDescontoTipoUsuario: number,
  dataSimulacao?: string
): number {
  // ...
  } else if (tipoDesconto === 'VALOR' && financeiroPlano.valorDesconto) {
    return parseFloat(financeiroPlano.valorDesconto...) || 0;  // ← tambem nao multiplica por quantidade
  }
}
```

O desconto de adesao com tipo `VALOR` (fixo) tambem nao multiplica pela quantidade. Porem, o desconto de adesao e configurado no **plano** (nao no lote), e planos de convite extra provavelmente nao tem desconto de adesao. Ainda assim, a logica esta incorreta.

---

## 3. Dados de Producao

### 3.1. Lotes com desconto fixo > 0 (convites extras)

| Lote | Plano | Valor Unit. | Comissao | Aderente | Plano Social |
|------|-------|------------|----------|----------|-------------|
| PRE LOTE | CONVITE BAILE | R$ 800 | R$ 200 | R$ 50 | R$ 100 |
| LOTE 1 | CONVITE BAILE | R$ 1.000 | R$ 200 | R$ 50 | R$ 100 |
| 2 LOTE | CONVITE BAILE | R$ 1.200 | R$ 200 | R$ 50 | R$ 100 |
| LOTE 1 | CONVITE EXTRA | R$ 700 | R$ 200 | R$ 100 | R$ 150 |
| LOTE 2 | CONVITE EXTRA | R$ 800 | R$ 200 | R$ 100 | R$ 150 |

Todos usam `tipo: 'valor_fixo'` — portanto todos sao afetados pelo bug.

### 3.2. Impacto real ate agora

- **475 contratos** no banco, todos com `quantidade = 1`
- Nenhum usuario comprou convite extra com quantidade > 1 ate o momento
- **Impacto financeiro real: R$ 0,00** (ninguem foi afetado ainda)
- Porem, quando a lojinha abrir para compra de convites extras em quantidade, o bug vai gerar cobranca a mais

### 3.3. Planos principais — nao afetados

Todos os lotes de plano principal (PLANO PRINCIPAL) tem descontos com `valor: 0`, entao o bug nao os afeta. O bug e exclusivo de **convites extras na lojinha**.

---

## 4. Cenarios de Impacto

### Cenario 1: Comissao compra 3 convites (LOTE 1 — R$ 1.000)
| | Correto | Bugado |
|--|---------|--------|
| Base | 3 x R$ 1.000 = R$ 3.000 | R$ 3.000 |
| Desconto | 3 x R$ 200 = R$ 600 | R$ 200 |
| Final | **R$ 2.400** | **R$ 2.800** |
| Prejuizo formando | | **+R$ 400** |

### Cenario 2: Aderente compra 5 convites (LOTE 2 — R$ 800)
| | Correto | Bugado |
|--|---------|--------|
| Base | 5 x R$ 800 = R$ 4.000 | R$ 4.000 |
| Desconto | 5 x R$ 100 = R$ 500 | R$ 100 |
| Final | **R$ 3.500** | **R$ 3.900** |
| Prejuizo formando | | **+R$ 400** |

### Cenario 3: Porcentagem (hipotetico) — funciona OK
Se o desconto fosse 10% (porcentagem), o calculo ja funciona:
- Base: 5 x R$ 1.000 = R$ 5.000
- Desconto: R$ 5.000 x 10% = R$ 500 (correto, aplica sobre total)

---

## 5. Escopo do Bug

| Componente | Afetado? | Por que |
|------------|----------|---------|
| Lojinha (convites extras) | SIM | Quantidade pode ser > 1, desconto fixo nao multiplica |
| Adesao (plano principal) | NAO | Quantidade sempre = 1, descontos dos lotes = R$ 0 |
| Replanejamento | NAO | Nao usa lotes/descontos |
| Upgrade de plano | NAO | Nao usa lotes/descontos |

---

## 6. Correcao Proposta

### 6.1. `calcularDescontoTipoUsuario` — adicionar parametro `quantidade`

```typescript
function calcularDescontoTipoUsuario(
  plano: PlanoComLote,
  valorComCorrecao: number,
  tipoUsuario: string,
  quantidade: number = 1       // ← NOVO parametro
): number {
  // ...
  case 'comissao':
    if (descontos.comissao) {
      desconto = descontos.comissao.tipo === 'valor_fixo'
        ? (descontos.comissao.valor || 0) * quantidade   // ← multiplicar por quantidade
        : valorComCorrecao * ((descontos.comissao.valor || 0) / 100);
    }
    break;
  // ... aplicar para todos os tipos
}
```

### 6.2. Chamada na `useCalculoParcelamento` (linha 86)

```typescript
const descontoTipoUsuario = calcularDescontoTipoUsuario(
  plano,
  valorComCorrecao,
  tipoUsuario,
  quantidade        // ← passar quantidade
);
```

### 6.3. `calcularDescontoAdesao` — mesmo ajuste (se aplicavel)

O desconto de adesao com tipo `VALOR` tambem deveria multiplicar por quantidade, caso algum plano de convite extra configure desconto de adesao fixo. Porem, na pratica atual, nenhum plano de convite extra tem desconto de adesao, entao e uma correcao preventiva.

---

## 7. Complexidade

- **Muito baixa** — alteracao em 1 arquivo, 1 funcao, adicionar 1 parametro
- Sem mudanca de schema, sem migration
- Nao afeta nenhum contrato existente (todos tem quantidade = 1)
- Risco zero de regressao nos fluxos existentes
