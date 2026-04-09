# Validacao Consolidada — Bug #399885: Lote Agendado Nao Ativou

**Data da correcao:** 2026-04-09
**Arquivo principal:** `supabase/functions/manage-lotes-schedule/index.ts`
**Banco de dados:** Supabase (amylxrskjhqwrlarqfcz)

---

## Resumo dos 4 bugs corrigidos

### Bug 1 (ALTA) — Lote pausado bloqueava ativacao agendada

**Problema:** O `return null` no check de `status_venda === "pausado"` (L129-134) acontecia ANTES da verificacao de horario agendado, impedindo que lotes pausados fossem ativados no horario programado.

**Correcao:** Movida a verificacao de ativacao por horario para ANTES do check de pausado. Agora a condicao de ativacao aceita tanto `inativo` quanto `pausado`:

```typescript
if (
  (lote.status_venda === "inativo" || lote.status_venda === "pausado") &&
  dataHoraInicio &&
  nowBrasilia >= dataHoraInicio &&
  (!dataHoraFim || nowBrasilia < dataHoraFim)
)
```

O check de `pausado` foi movido para DEPOIS, bloqueando apenas as demais operacoes (pausa por fim de horario, etc).

**Mesmo fix aplicado em `ativarProximoLote()`:** Removido o bloqueio que impedia ativacao do proximo lote na cadeia quando este estava pausado (L425-429). Agora lotes pausados podem ser ativados como proximo lote.

---

### Bug 2 (MEDIA) — Ativacoes duplicadas (race condition)

**Problema:** Duas execucoes consecutivas do CRON (~60s) podiam ativar o mesmo lote porque a primeira ativacao nao tinha feito commit antes da segunda execucao ler o status.

**Evidencia:** Logs mostravam lotes sendo ativados 2x em sequencia (ex: PRE LOTE ativado 13:30:01 e novamente 13:31:02).

**Correcao:** Adicionada guarda otimista na funcao `ativarLote()`:

```typescript
const { data: updated, error: updateError } = await supabase
  .from("lotes")
  .update({ status_venda: "ativo", ... })
  .eq("id", lote.id)
  .in("status_venda", ["inativo", "pausado"])  // so atualiza se ainda nao ativo
  .select();

if (!updated || updated.length === 0) {
  console.log(`[CRON] Lote ja ativado por outra execucao. Ignorando duplicata.`);
  return;
}
```

---

### Bug 3 (BAIXA) — Log registrava status errado na pausa

**Problema:** `pausarLote()` definia `status_venda: "pausado"` no update, mas o log em `lotes_logs` registrava `status_novo: "inativo"`.

**Correcao:** Alterado de `status_novo: "inativo"` para `status_novo: "pausado"` na insercao do log (L357).

---

### Bug 4 (MEDIA) — Campo `disponivel` inconsistente com `status_venda`

**Problema:** 62 lotes no banco tinham `disponivel = true` com `status_venda` em "pausado" ou "inativo", podendo causar vendas em lotes fechados.

**Correcao em 2 partes:**

1. **Dados existentes** — Executado SQL para corrigir todos os 62 registros:
   ```sql
   UPDATE lotes SET disponivel = false
   WHERE status_venda IN ('pausado', 'inativo') AND disponivel = true;
   ```

2. **Trigger preventivo** — Criada funcao + trigger `trigger_sync_lote_disponivel` que sincroniza automaticamente `disponivel` sempre que `status_venda` mudar:
   - `pausado` / `inativo` → `disponivel = false`
   - `ativo` → `disponivel = true`

---

## Verificacao pos-correcao

| Item | Status |
|------|--------|
| Lotes inconsistentes no banco | **0** (62 corrigidos) |
| Trigger `sync_lote_disponivel` ativo | **Sim** |
| Edge function atualizada | **Sim** (4 alteracoes) |
| Race condition protegida | **Sim** (guarda otimista) |
| Log de pausa corrigido | **Sim** (`pausado` em vez de `inativo`) |

---

## Arquivos alterados

| Arquivo | Alteracao |
|---------|-----------|
| `supabase/functions/manage-lotes-schedule/index.ts` | Bugs 1, 1b, 2, 3 |
| Banco de dados (SQL direto) | Bug 4 — data fix + trigger |

## Proximos passos

- **Deploy da edge function** `manage-lotes-schedule` para producao
- Monitorar logs do CRON nas proximas 24h para confirmar ausencia de duplicatas
- Verificar se lotes agendados com status pausado ativam corretamente no horario
