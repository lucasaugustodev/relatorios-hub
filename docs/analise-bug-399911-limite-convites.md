# Analise — Bug #399911: Limite de Compra Nao Cumulativo Entre Transacoes

**Data:** 2026-03-12
**Bug:** #399911
**Status:** CONFIRMADO — dados reais mostram limite ultrapassado

---

## 1. Descricao do problema

O limite de convites extras por usuario (`limite_por_usuario`) configurado no lote nao e verificado cumulativamente. O usuario consegue comprar mais do que o limite fazendo compras separadas.

**Exemplo:** Limite = 5 convites. Usuario compra 3 (OK) + compra 3 (OK individualmente) = 6 total, ultrapassando o limite.

---

## 2. Evidencia no banco de dados

### Usuarios que ultrapassaram o limite

| user_id | plano_id | total_comprado | limite | excedeu |
|---------|----------|----------------|--------|---------|
| fd2cec94-0b2b-4459-a300-6b4d90b25c76 | a249e778 | **8** | 5 | **+3** |
| dc6a6ae5-1a74-413d-bdaf-47e35221775c | a249e778 | **8** | 5 | **+3** |

Ambos compraram 8 unidades no mesmo lote ("LOTE COM DESCONTO DE ADESAO") com `limite_por_usuario = 5`.

### Query de verificacao

```sql
SELECT c.user_id, l.plano_id,
       COUNT(*) as total_contratos,
       SUM(c.quantidade) as total_quantidade,
       l.limite_por_usuario
FROM contratos c
JOIN lotes l ON l.id = c.lote_id
WHERE c.status IN ('ativo', 'pendente')
GROUP BY c.user_id, l.plano_id, l.limite_por_usuario
HAVING SUM(c.quantidade) > COALESCE(l.limite_por_usuario, 999);
```

---

## 3. Causa raiz

### 3.1. Frontend — `SelecaoQuantidade.tsx` (L17-25)

```typescript
// Linha 17 — usa limite bruto sem descontar compras anteriores
const limiteUsuario = plano.lote.limite_por_usuario || 999;

// Linha 25 — maxQuantidade nao considera historico de compras
const maxQuantidade = Math.min(limiteUsuario, estoqueDisponivel);
```

O componente permite selecionar ate `limiteUsuario` unidades em CADA transacao, sem verificar quantas o usuario ja comprou no total.

### 3.2. Backend — `Contratacao.tsx` (L900-920) e `ConfirmacaoContratacaoLojinha.tsx` (L120-139)

Ambos os pontos de criacao de contrato validam disponibilidade de estoque (`validar_disponibilidade_turma`) mas **nao validam o limite cumulativo por usuario**.

### 3.3. Hook de dados — `usePlanosLojinha.ts` (L310-328)

O hook monta o objeto `PlanoComLote` com os dados do lote, mas nao busca a quantidade ja comprada pelo usuario. O `LoteAtivo` nao tinha campo para essa informacao.

---

## 4. Fluxo de compra (onde falta validacao)

```
Lojinha (lista planos)
  → SelecaoQuantidade (❌ permite ate limiteUsuario sem descontar)
  → SelecaoParcelamentoLojinha (configura parcelas)
  → Contratacao.tsx (❌ cria contrato sem validar limite cumulativo)
     OU
  → ConfirmacaoContratacaoLojinha.tsx (❌ cria contrato sem validar limite cumulativo)
```

### Dois pontos de criacao de contrato

| Arquivo | Linha | Uso |
|---------|-------|-----|
| `src/pages/Contratacao.tsx` | L922-956 | Fluxo principal via lojinha |
| `src/components/ConfirmacaoContratacaoLojinha.tsx` | L141-146 | Fluxo alternativo de confirmacao |

Ambos precisam de validacao.

---

## 5. Escopo do limite: por PLANO (nao por lote)

O `limite_por_usuario` esta na tabela `lotes`, mas o limite deve ser cumulativo **por plano**, nao por lote individual. Motivo: lotes sao janelas temporais do mesmo produto. Quando um lote expira e o proximo ativa, o usuario ja comprou unidades no lote anterior.

Dados confirmam: usuarios compram no mesmo plano_id atraves de diferentes lotes ao longo do tempo.

---

## 6. Correcao necessaria (3 pontos)

### 6.1. `usePlanosLojinha.ts` — buscar quantidade ja comprada

Adicionar ao `LoteAtivo`:
- Campo `quantidade_ja_comprada: number`
- Query: `contratos WHERE user_id + plano_id + status IN (ativo, pendente)` → `SUM(quantidade)`

### 6.2. `SelecaoQuantidade.tsx` — usar limite restante

```typescript
const limiteRestante = Math.max(0, limiteUsuario - quantidadeJaComprada);
const maxQuantidade = Math.min(limiteRestante, estoqueDisponivel);
```

- Se `limiteRestante = 0`: alerta vermelho + botao desabilitado
- Se parcialmente usado: mostrar "voce ja comprou X, restam Y"

### 6.3. Backend — validar antes do INSERT

Em ambos os pontos de criacao (`Contratacao.tsx` e `ConfirmacaoContratacaoLojinha.tsx`):

```typescript
// Antes do INSERT
const { data: comprasExistentes } = await supabase
  .from('contratos')
  .select('quantidade')
  .eq('user_id', user.id)
  .eq('plano_id', planoId)
  .in('status', ['ativo', 'pendente']);

const totalJaComprado = comprasExistentes.reduce(
  (sum, c) => sum + (c.quantidade || 1), 0
);

if (totalJaComprado + novaQuantidade > limite_por_usuario) {
  throw new Error('Limite por usuario excedido');
}
```

---

## 7. Riscos e consideracoes

- **Zero regressao para planos SEM limite:** Quando `limite_por_usuario` e null, o limite padrao e 999 (sem restricao)
- **Protecao dupla:** Frontend bloqueia + backend valida = impossivel burlar
- **2 usuarios existentes** com 8 compras (limite 5) — nao sao corrigidos automaticamente, precisam de decisao manual
- **Limite por plano vs por lote:** A correcao agrupa por plano_id, nao por lote_id, para cobrir cenarios de rotacao de lotes
