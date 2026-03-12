# Correcao — Bug #399995: Upgrade de Plano com Sobrecobranca

**Data:** 2026-03-12
**Bug:** #399995
**Status:** CORRIGIDO E TESTADO (35/35 testes passando)

---

## 1. O que foi corrigido

O filtro de `ACORDOS_NAO_QUITADOS` em `calcularAlteracao()` capturava parcelas tipo `renegociacao` e `entrada_renegociacao` (principal reestruturado do plano antigo), quando deveria capturar apenas `taxa_renegociacao` (juros e multas negociados).

### Impacto do bug

| Metrica | Valor |
|---------|-------|
| Upgrades ativos afetados | **24** |
| Sobrecobranca total | **R$ 297.144,91** |
| Media por formando | **R$ 12.381,04** |
| Maior sobrecobranca individual | **R$ 46.423,41** |

### Causa raiz

A renegociacao (`renegociacaoService.ts`) cria 3 tipos de parcela:

| Tipo | Linha | O que representa |
|------|-------|-----------------|
| `taxa_renegociacao` | L535 | Juros e multas negociados (o "acordo" real) |
| `entrada_renegociacao` | L701 | 1a parcela do principal reestruturado |
| `renegociacao` | L701 | Demais parcelas do principal reestruturado |

O filtro capturava `renegociacao` + `entrada_renegociacao` = **principal inteiro do plano antigo** (mesmo dinheiro que `A_valorPlano`). Resultado: formando pagava o plano antigo + plano novo + multa.

---

## 2. Alteracao aplicada

### Arquivo: `src/services/mudancaPlanoService.ts` — `calcularAlteracao()` (L367-378)

**Antes:**
```typescript
// ACORDOS_NAO_QUITADOS = parcelas de renegociação não pagas
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

**Depois:**
```typescript
// ACORDOS_NAO_QUITADOS = apenas taxas de renegociação (juros/multas negociados) não pagas
// IMPORTANTE: NÃO incluir parcelas tipo 'renegociacao' ou 'entrada_renegociacao' —
// essas representam o principal reestruturado (mesmo dinheiro que A_valorPlano),
// não juros/multas. Incluí-las causa dupla cobrança no upgrade (Bug #399995).
const ACORDOS_NAO_QUITADOS = (parcelas || [])
  .filter(p => {
    const tipo = p.tipo || 'normal';
    return tipo === 'taxa_renegociacao' &&
      ['pendente', 'vencido'].includes(p.status);
  })
  .reduce((sum, p) => {
    const valorTotal = parseFloat(p.valor || '0');
    const valorPago = parseFloat(p.valor_pago || '0');
    return sum + Math.max(0, valorTotal - valorPago);
  }, 0);
```

---

## 3. Resultado dos testes (35/35 passaram)

### Filtro corrigido vs bugado
| Teste | Resultado |
|-------|-----------|
| BUGADO captura principal (R$ 25.628) | PASS |
| CORRIGIDO captura apenas taxa (R$ 954,56) | PASS |

### Calculo completo (caso Aninha Trindade)
| Teste | Resultado |
|-------|-----------|
| MULTA = 15% x 25.628,01 = R$ 3.844,20 | PASS |
| ACORDOS = R$ 954,56 (apenas taxa) | PASS |
| SALDO_BASE = R$ 3.844,20 | PASS |
| PENDENCIAS = R$ 954,56 | PASS |
| DEBITO = R$ 4.798,76 (era R$ 29.472,20) | PASS |
| Tipo = FORMANDO_PAGA | PASS |

### Regressao — sem renegociacao
| Teste | Resultado |
|-------|-----------|
| Contrato limpo: ACORDOS = 0 | PASS |
| DEBITO = apenas MULTA | PASS |
| Comportamento identico ao anterior | PASS |

### Taxa ja paga / parcialmente paga
| Teste | Resultado |
|-------|-----------|
| Taxa paga: ACORDOS = 0 | PASS |
| Taxa parcial (R$ 800, R$ 300 pago): ACORDOS = R$ 500 | PASS |

### Cenarios complexos
| Teste | Resultado |
|-------|-----------|
| Formando com 3 parcelas pagas + taxa pendente | PASS |
| Downgrade sem multa: credito correto | PASS |
| 2 renegociacoes: so taxa pendente entra | PASS |
| Parcelas renegociacao vencidas: NAO entram | PASS |
| Taxa vencida: ENTRA corretamente | PASS |

### Caso real Ingrid (taxa 40%)
| Teste | Resultado |
|-------|-----------|
| MULTA = 40% x 14.778,10 = R$ 5.911,24 | PASS |
| ACORDOS = R$ 1.200 (taxa) vs bugado R$ 13.757 | PASS |
| DEBITO = R$ 7.111,24 vs bugado R$ 19.668 | PASS |

---

## 4. TypeScript

```
npx tsc --noEmit --skipLibCheck → 0 erros
```

---

## 5. Impacto e riscos

- **Zero regressao para contratos SEM renegociacao:** Quando nao ha parcelas tipo `taxa_renegociacao`, `ACORDOS_NAO_QUITADOS` = 0 — identico ao comportamento anterior
- **Afeta upgrade E rescisao:** A funcao `calcularAlteracao()` e usada para ambos. Na rescisao, o comportamento anterior (capturar principal) tambem era incorreto — as parcelas de renegociacao representam o mesmo principal, nao divida adicional
- **Parcelas existentes nao sao corrigidas:** Os 24 contratos com sobrecobranca precisam de script de correcao manual

### Limitacao conhecida
- Os **24 contratos existentes** com sobrecobranca NAO sao corrigidos automaticamente. Requer script de ajuste ou decisao da equipe financeira para recalcular parcelas.
