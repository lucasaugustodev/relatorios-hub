# Analise — Bug #399885: Lote Agendado Nao Ativou no Horario Programado

**Data:** 2026-03-12
**Bug:** #399885
**Status:** ANALISADO — 4 problemas encontrados
**Severidade:** Alta
**Arquivo principal:** `supabase/functions/manage-lotes-schedule/index.ts`

---

## 1. Contexto

O sistema de lotes usa um modelo hibrido:
- **Database Triggers** — desativacao por quantidade (imediata, sem race conditions)
- **Edge Function CRON** (`manage-lotes-schedule`) — ativacao/desativacao por data/hora (executa a cada ~60s)

3 edge functions rodam a cada minuto:
| Funcao | Intervalo | Status |
|--------|-----------|--------|
| `manage-lotes-schedule` | ~60s | 200 OK |
| `verificar-aptidao-financeira` | ~60s | 200 OK (~10s exec) |
| `verificar-status-boleto-inter` | ~60s | 200 OK |

O CRON esta saudavel — o problema nao e de execucao, mas de **logica**.

---

## 2. Bug 1 (PRINCIPAL): Lote pausado bloqueia ativacao agendada

**Arquivo:** `supabase/functions/manage-lotes-schedule/index.ts` linhas 129-134

### Problema

```typescript
// Linha 129 — retorna ANTES de verificar horario agendado
if (lote.status_venda === "pausado") {
  console.log(`[CRON] Lote "${lote.nome_lote}" está pausado. Ignorando ativação automática.`);
  return null;  // ← horario agendado NUNCA e verificado
}
```

Quando um lote e pausado manualmente e depois tem um horario de ativacao programado, o CRON ignora completamente porque o `return null` acontece antes da verificacao de horario.

### Mesmo bug em `ativarProximoLote()` (L425-429)

```typescript
if (proximoLote.status_venda === "pausado") {
  console.log(`[CRON] Próximo lote "${proximoLote.nome_lote}" está pausado. Não será ativado automaticamente.`);
  return false;  // ← proximo lote da cadeia nao ativa se estiver pausado
}
```

### Correcao proposta

Mover a verificacao de `pausado` para DEPOIS da logica de ativacao por horario:

```typescript
// Verificar se ha ativacao programada PRIMEIRO
if (dataHoraInicio && nowBrasilia >= dataHoraInicio && (!dataHoraFim || nowBrasilia < dataHoraFim)) {
  // Verificar limite de quantidade
  if (lote.regra_virada === "quantidade_e_data_hora" &&
      lote.quantidade_limite > 0 &&
      lote.quantidade_vendida >= lote.quantidade_limite) {
    return null;
  }

  // Ativar se estiver inativo OU pausado (horario agendado tem prioridade)
  if (lote.status_venda === 'inativo' || lote.status_venda === 'pausado') {
    await ativarLote(supabase, lote, "Data/hora início atingida", nowBrasilia);
    return { lote_id: lote.id, nome: lote.nome_lote, acao: "ativado", motivo: "Data/hora início atingida" };
  }
}

// SO ENTAO verificar se esta pausado para outras operacoes
if (lote.status_venda === "pausado") {
  return null;
}
```

---

## 3. Bug 2: Ativacoes duplicadas (race condition)

### Evidencia nos logs (2026-03-11)

| Timestamp (UTC) | Lote | Acao |
|-----------------|------|------|
| 13:30:01 | PRE LOTE | ativado (inativo → ativo) |
| 13:31:02 | PRE LOTE | ativado **NOVAMENTE** (inativo → ativo) |

A segunda execucao do CRON ainda ve `status_venda = "inativo"` — a ativacao anterior nao fez commit a tempo ou ha race condition entre invocacoes.

### Mesmo padrao com "LOTE COM DESCONTO"

| Timestamp | Lote ID | Acao |
|-----------|---------|------|
| 13:19:01 | da82b882 | ativado |
| 13:20:01 | 862c1d05 | ativado |
| 13:21:01 | 862c1d05 | ativado **NOVAMENTE** |
| 13:22:01 | 90ef842c | ativado |

### Correcao proposta

Adicionar guarda contra duplicatas com verificacao otimista:

```typescript
// Usar match condicional para evitar ativacao duplicada
const { data: updated, error: updateError } = await supabase
  .from("lotes")
  .update({
    status_venda: "ativo",
    disponivel: true,
    updated_at: timestamp.toISOString(),
    historico: [...(lote.historico || []), historicoItem],
  })
  .eq("id", lote.id)
  .in("status_venda", ["inativo", "pausado"])  // ← so atualiza se ainda nao estiver ativo
  .select();

if (!updated || updated.length === 0) {
  console.log(`[CRON] Lote "${lote.nome_lote}" ja foi ativado por outra execucao. Ignorando.`);
  return null;
}
```

---

## 4. Bug 3: Log registra status errado na pausa

**Arquivo:** `supabase/functions/manage-lotes-schedule/index.ts` linhas 353-362

### Problema

`pausarLote()` define `status_venda: "pausado"`, mas o log registra `status_novo: "inativo"`:

```typescript
// Linha 356 — status_novo inconsistente
const { error: logError } = await supabase.from("lotes_logs").insert({
  lote_id: lote.id,
  acao: "desativado_automatico_data",
  status_anterior: lote.status_venda,
  status_novo: "inativo",           // ← ERRADO: deveria ser "pausado"
  motivo: `${motivo} - Lote pausado`,
  ...
});
```

### Correcao

```typescript
status_novo: "pausado",  // ← consistente com o update real
```

---

## 5. Bug 4: `disponivel` inconsistente com `status_venda`

### Dados atuais no banco

| Lote | status_venda | disponivel | Problema |
|------|-------------|------------|----------|
| LOTE 1 (02561ba2) | pausado | **true** | Deveria ser false |
| LOTE UNICO (eca22a4c) | inativo | **true** | Deveria ser false |

Lotes pausados/inativos com `disponivel = true` podem causar vendas em lotes que deveriam estar fechados.

### Causa provavel

Quando o lote e criado ou editado manualmente no admin, o campo `disponivel` nao e sincronizado com `status_venda`. A edge function faz isso corretamente na ativacao/pausa automatica, mas a UI pode nao estar fazendo.

### Correcao proposta

1. Script de correcao para dados existentes:
```sql
UPDATE lotes SET disponivel = false WHERE status_venda IN ('pausado', 'inativo') AND disponivel = true;
```

2. Adicionar trigger no banco para manter consistencia:
```sql
CREATE OR REPLACE FUNCTION sync_lote_disponivel()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.status_venda IN ('pausado', 'inativo') THEN
    NEW.disponivel := false;
  ELSIF NEW.status_venda = 'ativo' THEN
    NEW.disponivel := true;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_sync_lote_disponivel
BEFORE UPDATE ON lotes
FOR EACH ROW
WHEN (OLD.status_venda IS DISTINCT FROM NEW.status_venda)
EXECUTE FUNCTION sync_lote_disponivel();
```

---

## 6. Resumo das correcoes

| # | Bug | Severidade | Arquivo | Linhas |
|---|-----|-----------|---------|--------|
| 1 | Lote pausado bloqueia ativacao | **Alta** | manage-lotes-schedule/index.ts | 129-134, 425-429 |
| 2 | Ativacoes duplicadas | **Media** | manage-lotes-schedule/index.ts | ativarLote() |
| 3 | Log status errado | **Baixa** | manage-lotes-schedule/index.ts | 356 |
| 4 | disponivel inconsistente | **Media** | DB + admin UI | - |

### Ordem de implementacao sugerida

1. Bug 1 — Corrigir logica de prioridade (agendamento > pausa manual)
2. Bug 2 — Adicionar guarda contra duplicatas
3. Bug 4 — Script SQL + trigger para consistencia
4. Bug 3 — Corrigir status no log
