# Analise — Bug #400009: Acoes Permitidas em Contrato Transferido

**Data:** 2026-03-12
**Bug:** #400009
**Status:** CONFIRMADO — 3 problemas encontrados

---

## 1. Descricao do problema

Quando um contrato e transferido, ele nao desaparece da listagem ativa do dono original. O sistema continua permitindo acoes (upgrade, downgrade, rescisao, nova transferencia) em contratos que ja foram transferidos ou que estao aguardando transferencia.

O contrato deveria ir para uma aba de "Contratos Transferidos" e nao permitir mais nenhuma acao.

---

## 2. Evidencia no banco de dados

### 2.1. Contratos transferidos com status "ativo"

Todos os contratos transferidos mantêm `status = 'ativo'` apos a transferencia:

| contrato_id | status | is_transferred | original_owner_id | user_id (novo dono) |
|-------------|--------|----------------|--------------------|--------------------|
| 36a4fdd5 | **ativo** | true | 1df1604a | e9fc2e67 |
| 4733438a | **ativo** | true | 901f8a97 | 0d4e5a99 |
| 59c178a0 | **ativo** | true | cfeab6e7 | cb166faf |
| 3c1a48b0 | **ativo** | true | ed6846b9 | c4ab304b |
| d7284dd2 | **ativo** | true | 3bdf7ea6 | d326e1d6 |

O campo `status` nunca muda para "transferido" — apenas `is_transferred` vira `true` e `user_id` muda para o novo dono.

### 2.2. Transferencias pendentes (aguardando aceitacao)

| transfer_id | contrato | from | to | status | expires_at |
|-------------|----------|------|----|--------|------------|
| cf00356c | 4223296b | Maria Joaquina | teste carol | **pending** | 2026-01-24 |
| 086a4447 | ca4009b6 | Priscilla Aguiar | Priscilla Aguiar | **pending** | 2026-01-24 |

Durante o periodo de espera (10 dias), o contrato continua com `user_id` do dono original, status "ativo", e todas as acoes disponiveis.

---

## 3. Causa raiz — 3 problemas

### 3.1. Problema 1: `fetchContratos` nao busca contratos transferidos pelo dono original

**Arquivo:** `src/pages/MinhasContratacoes.tsx` linhas 384-392

```typescript
const { data: contratosData } = await supabase
  .from('contratos')
  .select(`*, planos(...), turmas(...)`)
  .eq('user_id', user.id)       // ← So busca contratos do dono ATUAL
  .eq('turma_id', turmaAtual.id);
```

Consequencia: O dono original que transferiu o contrato simplesmente nao ve mais o contrato — nao aparece em nenhuma aba, nem em "Transferido". As tabs "Transferido" e "Aguardando transferencia" existem (L221) mas estao **sempre vazias**.

Para comparacao, `MeusContratos.tsx` faz a query correta:
```typescript
.or(`user_id.eq.${user.id},original_owner_id.eq.${user.id}`)
```

### 3.2. Problema 2: Status mapeado como binario (Ativo/Rescindido)

**Arquivo:** `src/pages/MinhasContratacoes.tsx` linha 432

```typescript
// Status baseado no campo do contrato
const status = contrato.status === 'ativo' ? 'Ativo' : 'Rescindido';
```

Apenas 2 status possiveis. Nunca produz "Transferido" ou "Aguardando transferencia", mesmo quando `is_transferred = true` ou existe `transfer_request` pendente.

### 3.3. Problema 3: Acoes nao verificam transferencia

**Arquivo:** `src/pages/MinhasContratacoes.tsx` linhas 77-112

```typescript
// isUpgradeEnabled — NAO verifica transferencia
const isUpgradeEnabled = (contrato: any): boolean => {
  if (!contrato?.planos) return false;
  if (!contrato.planos.alteracao) return false;
  if (contrato.planos.alteracao.upgrade !== true) return false;
  // ❌ Nao verifica is_transferred
  // ❌ Nao verifica transfer_request pendente
  // ...
};

// isDowngradeEnabled — MESMO problema (L96-112)
// isTransferEnabled — MESMO problema (L115-140)
// isRescissionEnabled — MESMO problema (L65-73)
```

Botoes de acao (L768-793):
```typescript
// Todos checam apenas: contrato.status !== "Ativo"
hidden: contrato.status !== "Ativo" || ...
```

Como `status` e sempre "Ativo" (Problema 2), os botoes nunca sao escondidos para contratos transferidos.

---

## 4. Fluxo atual vs esperado

### Cenario: Dono original apos transferencia aceita

| Aspecto | Atual (bug) | Esperado |
|---------|-------------|----------|
| Contrato na listagem | **Desaparece** (nao busca por original_owner_id) | Aparece na aba "Transferido" |
| Status exibido | N/A | "Transferido" |
| Acoes disponiveis | N/A | Nenhuma |

### Cenario: Contrato com transferencia pendente (10 dias)

| Aspecto | Atual (bug) | Esperado |
|---------|-------------|----------|
| Contrato na listagem | Aba **"Ativo"** | Aba **"Aguardando transferencia"** |
| Status exibido | **"Ativo"** | "Aguardando transferencia" |
| Upgrade/Downgrade | **Disponivel** | Bloqueado |
| Rescisao | **Disponivel** | Bloqueado |
| Nova transferencia | **Disponivel** | Bloqueado |

---

## 5. Correcao necessaria

### 5.1. `fetchContratos` — buscar contratos transferidos + transfer_requests

```typescript
// Buscar contratos do usuario E contratos que ele transferiu
const { data: contratosData } = await supabase
  .from('contratos')
  .select('*, planos(...), turmas(...)')
  .or(`user_id.eq.${user.id},original_owner_id.eq.${user.id}`)
  .eq('turma_id', turmaAtual.id);

// Buscar transfer_requests pendentes para esses contratos
const { data: transfersPendentes } = await supabase
  .from('transfer_requests')
  .select('contract_id, to_user_name, status')
  .in('contract_id', contratosIds)
  .eq('status', 'pending');
```

### 5.2. Status — verificar is_transferred e transfer_requests

```typescript
// Determinar status correto
let status = 'Ativo';
if (contrato.status === 'rescindido') {
  status = 'Rescindido';
} else if (contrato.is_transferred && contrato.original_owner_id === user.id && contrato.user_id !== user.id) {
  status = 'Transferido';
} else if (transfersPendentes.has(contrato.id)) {
  status = 'Aguardando transferência';
}
```

### 5.3. Acoes — bloquear em contratos transferidos/pendentes

```typescript
const isUpgradeEnabled = (contrato: any): boolean => {
  // Bloquear se transferido ou aguardando transferencia
  if (contrato.status === 'Transferido' || contrato.status === 'Aguardando transferência') return false;
  // ... resto da logica
};
```

---

## 6. Riscos e consideracoes

- **Zero regressao para contratos normais:** Contratos sem transferencia continuam com status "Ativo" e todas as acoes disponiveis
- **MeusContratos.tsx ja funciona corretamente** — usa `.or()` para buscar ambos os lados da transferencia. MinhasContratacoes.tsx precisa alinhar
- **Impacto nos 10 contratos transferidos existentes:** Passarao a aparecer na aba "Transferido" para o dono original
- **2 transferencias pendentes expiradas:** contract_id `4223296b` e `ca4009b6` tem transfer_requests "pending" mas ja expiraram (jan/2026). Pode precisar de cleanup
