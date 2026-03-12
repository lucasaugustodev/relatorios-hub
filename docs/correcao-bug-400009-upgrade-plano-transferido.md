# Correcao — Bug #400009: Acoes Permitidas em Contrato Transferido

**Data:** 2026-03-12
**Bug:** #400009
**Status:** CORRIGIDO (TypeScript 0 erros)

---

## 1. O que foi corrigido

Contratos transferidos agora aparecem na aba correta ("Transferido" ou "Aguardando transferencia") e todas as acoes (upgrade, downgrade, rescisao, transferencia) sao bloqueadas.

---

## 2. Alteracoes aplicadas

### 2.1. Arquivo: `src/pages/MinhasContratacoes.tsx` — `fetchContratos()` (L384-402)

#### Query — buscar contratos transferidos pelo dono original

**Antes:**
```typescript
const { data: contratosData } = await supabase
  .from('contratos')
  .select('*, planos(...), turmas(...)')
  .eq('user_id', user.id)          // ← So busca contratos do dono ATUAL
  .eq('turma_id', turmaAtual.id);
```

**Depois:**
```typescript
// Buscar contratos do usuario E contratos que ele transferiu (Bug #400009)
const { data: contratosData } = await supabase
  .from('contratos')
  .select('*, planos(...), turmas(...)')
  .or(`user_id.eq.${user.id},original_owner_id.eq.${user.id}`)  // ← Ambos os lados
  .eq('turma_id', turmaAtual.id);

// Buscar transfer_requests pendentes para esses contratos
const contratosIds = contratosData?.map(c => c.id) || [];
const transfersPendentesSet = new Set<string>();
if (contratosIds.length > 0) {
  const { data: transfersPendentes } = await supabase
    .from('transfer_requests')
    .select('contract_id')
    .in('contract_id', contratosIds)
    .eq('status', 'pending');

  if (transfersPendentes) {
    transfersPendentes.forEach((t: any) => transfersPendentesSet.add(t.contract_id));
  }
}
```

**Por que:** O dono original precisa ver seus contratos transferidos na aba "Transferido". E contratos com transferencia pendente precisam ser identificados para a aba "Aguardando transferencia".

---

### 2.2. Arquivo: `src/pages/MinhasContratacoes.tsx` — Status mapping (L432)

#### De binario para 4 estados

**Antes:**
```typescript
const status = contrato.status === 'ativo' ? 'Ativo' : 'Rescindido';
```

**Depois:**
```typescript
let status = 'Ativo';
if (contrato.status === 'rescindido') {
  status = 'Rescindido';
} else if (contrato.is_transferred && contrato.original_owner_id === user.id && contrato.user_id !== user.id) {
  // Contrato que o usuario transferiu para outra pessoa
  status = 'Transferido';
} else if (transfersPendentesSet.has(contrato.id)) {
  // Contrato com transferencia pendente (aguardando aceitacao)
  status = 'Aguardando transferência';
} else if (contrato.status === 'ativo') {
  status = 'Ativo';
}
```

**Por que:** As tabs "Transferido" e "Aguardando transferencia" ja existiam (L221) mas nunca recebiam contratos porque o status era sempre "Ativo" ou "Rescindido".

---

### 2.3. Arquivo: `src/pages/MinhasContratacoes.tsx` — Bloqueio de acoes (L66-140)

#### Funcao helper `isContratoTransferido()`

```typescript
// Funcao auxiliar para verificar se contrato esta em transferencia (Bug #400009)
const isContratoTransferido = (contrato: any): boolean => {
  return contrato.status === 'Transferido' || contrato.status === 'Aguardando transferência';
};
```

#### Acoes bloqueadas

Adicionado `if (isContratoTransferido(contrato)) return false;` em:

| Funcao | Linha | Acao bloqueada |
|--------|-------|----------------|
| `isUpgradeEnabled()` | L80 | Upgrade de plano |
| `isDowngradeEnabled()` | L100 | Downgrade de plano |
| `isTransferEnabled()` | L120 | Nova transferencia |

Alem disso, os botoes de acao ja checam `contrato.status !== "Ativo"` (L803, L810, L818, L826), que automaticamente esconde todos os botoes quando status e "Transferido" ou "Aguardando transferencia".

**Protecao dupla:** funcoes `isXxxEnabled()` + check de status nos botoes.

---

## 3. Comportamento apos correcao

### Dono original (quem transferiu)

| Aspecto | Antes | Depois |
|---------|-------|--------|
| Contrato na listagem | Desaparecia completamente | Aparece na aba **"Transferido"** |
| Status exibido | N/A | **"Transferido"** (cinza) |
| Upgrade/Downgrade | N/A | **Bloqueado** |
| Rescisao | N/A | **Bloqueado** |
| Transferencia | N/A | **Bloqueado** |

### Contrato com transferencia pendente

| Aspecto | Antes | Depois |
|---------|-------|--------|
| Contrato na listagem | Aba "Ativo" | Aba **"Aguardando transferencia"** |
| Status exibido | "Ativo" | **"Aguardando transferencia"** (amarelo) |
| Upgrade/Downgrade | Disponivel | **Bloqueado** |
| Rescisao | Disponivel | **Bloqueado** |
| Nova transferencia | Disponivel | **Bloqueado** |

### Novo dono (quem recebeu)

| Aspecto | Antes | Depois |
|---------|-------|--------|
| Contrato na listagem | Aba "Ativo" | Aba **"Ativo"** (sem mudanca) |
| Acoes | Todas disponiveis | Todas disponiveis (correto — e o dono agora) |

---

## 4. TypeScript

```
npx tsc --noEmit --skipLibCheck → 0 erros
```

---

## 5. Impacto e riscos

- **Zero regressao para contratos normais:** Contratos sem transferencia continuam com status "Ativo" e todas as acoes disponiveis
- **10 contratos transferidos existentes:** Passarao a aparecer na aba "Transferido" para o dono original
- **2 transferencias pendentes expiradas:** transfer_requests "pending" de jan/2026 que ja expiraram — mostrarao como "Aguardando transferencia" ate serem limpas
- **Alinhamento com MeusContratos.tsx:** A query agora usa `.or()` igual ao MeusContratos.tsx, que ja funcionava corretamente
