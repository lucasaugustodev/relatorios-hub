# Correcao — Bug #399907: Desconto Valor Fixo nao Multiplica pela Quantidade

**Data:** 2026-03-12
**Bug:** #399907
**Commit:** `2bfa5d1`
**Status:** CORRIGIDO E TESTADO (29/29 testes passando)

---

## 1. O que foi corrigido

A funcao `calcularDescontoTipoUsuario` em `src/hooks/useCalculoParcelamento.ts` aplicava o desconto `valor_fixo` uma unica vez, independente da quantidade de convites comprados.

### Antes (bugado):
```typescript
case 'comissao':
  if (descontos.comissao) {
    desconto = descontos.comissao.tipo === 'valor_fixo'
      ? descontos.comissao.valor || 0          // R$ 200 fixo, sempre
      : valorComCorrecao * ((descontos.comissao.valor || 0) / 100);
  }
  break;
```

### Depois (corrigido):
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
        ? (descontos.comissao.valor || 0) * quantidade   // ← multiplica por quantidade
        : valorComCorrecao * ((descontos.comissao.valor || 0) / 100);
    }
    break;
```

### Chamada atualizada (linha 86-91):
```typescript
const descontoTipoUsuario = calcularDescontoTipoUsuario(
  plano,
  valorComCorrecao,
  tipoUsuario,
  quantidade        // ← agora passa quantidade
);
```

---

## 2. Arquivos alterados

| Arquivo | Alteracao |
|---------|----------|
| `src/hooks/useCalculoParcelamento.ts` | Adicionado parametro `quantidade` na funcao `calcularDescontoTipoUsuario`, multiplicacao por quantidade no caso `valor_fixo` para todos os tipos (comissao, aderente, nao_aderente, plano_social). Chamada atualizada para passar `quantidade`. |
| `src/hooks/__tests__/useCalculoParcelamento.test.ts` | 29 testes unitarios cobrindo todos os cenarios (Vitest) |
| `test-bug-399907.js` | Script de teste standalone (Node.js) para validacao rapida |

---

## 3. Resultado dos testes (29/29 passaram)

### Caso original do bug
| Teste | Resultado |
|-------|-----------|
| 5 convites R$1.200, desc comissao R$200/un → desconto R$1.000 | PASS |
| Valor final = R$5.000 (nao R$5.800 do bug) | PASS |

### valor_fixo com multiplas quantidades
| Teste | Resultado |
|-------|-----------|
| comissao 5x R$200 = R$1.000 | PASS |
| aderente 3x R$50 = R$150 | PASS |
| plano_social 4x R$100 = R$400 | PASS |
| nao_aderente 10x R$0 = R$0 | PASS |

### Convite Extra (Lote 2, R$800)
| Teste | Resultado |
|-------|-----------|
| comissao 3x R$200 = R$600 | PASS |
| aderente 5x R$100 = R$500 | PASS |
| plano_social 2x R$150 = R$300 | PASS |

### Regressao: quantidade = 1
| Teste | Resultado |
|-------|-----------|
| comissao 1x = R$200 | PASS |
| aderente 1x = R$50 | PASS |
| default (sem parametro) = R$200 | PASS |

### Porcentagem (nao afetado)
| Teste | Resultado |
|-------|-----------|
| comissao 10% de R$5.000 = R$500 | PASS |
| aderente 5% de R$3.000 = R$150 | PASS |
| plano_social 15% de R$2.000 = R$300 | PASS |

### Edge cases
| Teste | Resultado |
|-------|-----------|
| descontos undefined = R$0 | PASS |
| tipo desconhecido = R$0 | PASS |
| quantidade 0 = R$0 | PASS |

### Fluxo completo (9 cenarios)
| Cenario | Desconto | Final | Resultado |
|---------|----------|-------|-----------|
| comissao 1x R$1.200 | R$200 | R$1.000 | PASS |
| comissao 2x R$1.200 | R$400 | R$2.000 | PASS |
| comissao 3x R$1.200 | R$600 | R$3.000 | PASS |
| comissao 5x R$1.200 | R$1.000 | R$5.000 | PASS |
| comissao 10x R$1.200 | R$2.000 | R$10.000 | PASS |
| aderente 1x R$1.200 | R$50 | R$1.150 | PASS |
| aderente 5x R$1.200 | R$250 | R$5.750 | PASS |
| plano_social 3x R$1.200 | R$300 | R$3.300 | PASS |
| nao_aderente 5x R$1.200 | R$0 | R$6.000 | PASS |

---

## 4. TypeScript

```
npx tsc --noEmit --skipLibCheck → 0 erros
```

---

## 5. Impacto e riscos

- **Zero regressao:** Quantidade default = 1, comportamento identico para contratos existentes
- **Zero impacto em producao atual:** Todos os 475 contratos tem quantidade = 1
- **Correcao preventiva:** Quando lojinha abrir convites extras em quantidade, o calculo sera correto
- **Desconto porcentagem:** Nao afetado (ja aplicava sobre o total que inclui quantidade)
- **Adesao:** Nao afetada (quantidade sempre = 1, descontos dos lotes = R$0)
