# Analise e Plano de Correcao — Bug #399923 + #399901 + #399907

## Desconto de comissao aplicado para TODOS os usuarios

**Data:** 2026-03-12
**Severidade:** CRITICA (perda financeira direta)
**Status:** Confirmado no codigo + dados de producao

---

## 1. Resumo do Problema

O tipo de usuario esta **hardcoded como `'comissao'`** em dois componentes criticos:
- `SelecaoParcelamentoLojinha.tsx` (lojinha/convites extras)
- `SelecaoParcelamento.tsx` (adesao/contratacao principal)

Isso faz com que **todos os usuarios** — aderentes, nao-aderentes e plano social — recebam o desconto de comissao, que e o **maior desconto disponivel**.

### Impacto financeiro real (dados de producao)

Exemplo: Lote "2 LOTE" do plano CONVITE BAILE (R$ 1.200/convite):

| Tipo Usuario | Desconto Configurado | Preco Correto | Preco Cobrado (bug) | Perda/convite |
|-------------|---------------------|---------------|---------------------|---------------|
| Comissao | R$ 200 (fixo) | R$ 1.000 | R$ 1.000 | R$ 0 |
| **Aderente** | R$ 50 (fixo) | **R$ 1.150** | R$ 1.000 | **-R$ 150** |
| **Nao aderente** | R$ 0 | **R$ 1.200** | R$ 1.000 | **-R$ 200** |
| Plano social | R$ 100 (fixo) | R$ 1.100 | R$ 1.000 | -R$ 100 |

**Se um nao-aderente compra 5 convites: perde R$ 1.000 (5 x R$ 200).**

Outro exemplo: Lote "LOTE 2" do plano CONVITE EXTRA (R$ 800):
- Comissao: -R$ 200 → R$ 600
- Aderente: -R$ 100 → R$ 700 (mas paga R$ 600, perda R$ 100/convite)
- Nao aderente: R$ 0 → R$ 800 (mas paga R$ 600, perda R$ 200/convite)

---

## 2. Causa Raiz no Codigo

### 2.1. Default hardcoded (BUG PRINCIPAL)

**Arquivo: `src/components/SelecaoParcelamentoLojinha.tsx` — linha 52:**
```typescript
const [tipoUsuario, setTipoUsuario] = useState<
  'comissao' | 'aderente' | 'nao_aderente' | 'plano_social'
>('comissao');  // ← DEFAULT ERRADO: todo mundo começa como 'comissao'
```

**Arquivo: `src/pages/SelecaoParcelamento.tsx` — linha 44:**
```typescript
const [tipoUsuario, setTipoUsuario] = useState<string>("comissao");
// ← MESMO BUG no fluxo de adesao
```

### 2.2. Seletor de debug em producao

**`SelecaoParcelamentoLojinha.tsx` — linhas 537-551:**
```typescript
{/* Tipo de Usuário - Debug */}
<div className="mb-6 p-4 bg-green-50 border border-green-200 rounded-lg">
  <label className="block text-sm font-medium text-green-800 mb-2">
    👤 Tipo de Usuário (Debug - Provisório)
  </label>
  <select value={tipoUsuario} onChange={(e) => setTipoUsuario(e.target.value as any)}>
    <option value="comissao">Comissão</option>
    <option value="aderente">Aderente</option>
    <option value="nao_aderente">Não Aderente</option>
    <option value="plano_social">Plano Social</option>
  </select>
```

**`SelecaoParcelamento.tsx` — linhas 2060-2068:** seletor identico.

Esses seletores estao **visiveis em producao** — qualquer usuario pode trocar seu tipo manualmente e pagar menos.

### 2.3. Tipo real nunca e buscado do banco

Em nenhum momento o codigo busca o `tipo_participante` do usuario na tabela `turma_participantes`. Os hooks `usePlanosLojinha.ts` e `useTurmaPlanos.ts` buscam planos, lotes, aptidao, mas **nunca buscam o tipo do usuario**.

### 2.4. Como o desconto e aplicado

**`src/hooks/useCalculoParcelamento.ts` — funcao `calcularDescontoTipoUsuario` (linhas 333-375):**
```typescript
function calcularDescontoTipoUsuario(plano, valorComCorrecao, tipoUsuario) {
  const descontos = plano.lote.descontos;
  switch (tipoUsuario) {
    case 'comissao':
      desconto = descontos.comissao.tipo === 'valor_fixo'
        ? descontos.comissao.valor
        : valorComCorrecao * (descontos.comissao.valor / 100);
      break;
    case 'aderente':
      // ...mesmo padrao
    case 'nao_aderente':
      // ...mesmo padrao
  }
  return desconto;
}
```

O `tipoUsuario` vem sempre como `'comissao'` → aplica desconto de comissao para todos.

---

## 3. Estrutura dos Descontos no Lote

Os descontos sao configurados no campo JSON `descontos` da tabela `lotes`:

```json
{
  "comissao":     { "tipo": "valor_fixo", "valor": 200 },
  "aderente":     { "tipo": "valor_fixo", "valor": 50 },
  "nao_aderente": { "tipo": "valor_fixo", "valor": 0 },
  "plano_social": { "tipo": "valor_fixo", "valor": 100, "ativo": false }
}
```

Cada tipo pode ter:
- `tipo`: `"valor_fixo"` (em reais) ou `"porcentagem"`
- `valor`: o valor do desconto
- `ativo`: flag opcional (usado em plano_social)

### Lotes com descontos configurados em producao:

| Lote | Plano | Valor Base | Comissao | Aderente | Nao Ader. |
|------|-------|-----------|----------|----------|-----------|
| PRE LOTE | CONVITE BAILE (Psico Faminas) | R$ 800 | -R$ 200 | -R$ 50 | R$ 0 |
| LOTE 1 | CONVITE BAILE (Psico Faminas) | R$ 1.000 | -R$ 200 | -R$ 50 | R$ 0 |
| 2 LOTE | CONVITE BAILE (Psico Faminas) | R$ 1.200 | -R$ 200 | -R$ 50 | R$ 0 |
| LOTE 2 | CONVITE EXTRA (Eng Comp) | R$ 800 | -R$ 200 | -R$ 100 | R$ 0 |

---

## 4. Mapeamento tipo_participante (banco) → tipoUsuario (codigo)

A tabela `turma_participantes` tem o campo `tipo_participante` com valores:
- `'Comissão'` → mapeia para `'comissao'`
- `'Presidente'` → mapeia para `'comissao'` (mesmo beneficio)
- `'Padrão'` → precisa checar se e aderente ou nao

Para determinar se um usuario `'Padrão'` e aderente, existe a funcao RPC `check_user_is_aderente(user_id, turma_id)` que verifica se o usuario tem contrato ativo com categoria principal na turma.

**Logica de mapeamento (Lojinha):**
```
campo plano_social = true                                     → 'plano_social'
tipo_participante = 'Comissão' ou 'Presidente'                → 'comissao'
tipo_participante = 'Padrão' + check_user_is_aderente = true  → 'aderente'
tipo_participante = 'Padrão' + check_user_is_aderente = false → 'nao_aderente'
```

**Logica de mapeamento (Adesao — sem check de aderencia):**
```
tipo_participante = 'Comissão' ou 'Presidente' → 'comissao'
qualquer outro caso                            → 'nao_aderente'
```

**Nota:** `check_user_is_aderente(user_id, turma_id)` verifica se o usuario tem contrato ativo com categoria `is_principal = true`. Categorias sao definidas no backoffice (Gestao > Categorias). Atualmente todas as categorias principais se chamam "PLANO PRINCIPAL".

---

## 5. Plano de Correcao

### 5.1. Escopo da correcao: Lojinha vs Adesao

**Aderencia so se aplica na Lojinha** (convites extras, produtos adicionais), nao na adesao. Na adesao o usuario ainda nao tem contrato, entao nao pode ser aderente.

- **Lojinha (`SelecaoParcelamentoLojinha.tsx`):** Precisa checar aderencia via `check_user_is_aderente()` + tipo_participante
- **Adesao (`SelecaoParcelamento.tsx`):** So precisa checar tipo_participante (comissao/presidente vs padrao). Default seguro: `'nao_aderente'`

**Aderente = usuario que ja tem contrato ativo com plano de categoria `is_principal = true` na turma.** Isso e definido no backoffice em Gestao > Categorias.

A RPC `check_user_is_aderente(p_user_id, p_turma_id)` ja existe e ja e usada em 3 lugares:
- `OnboardingGuard.tsx:146`
- `Portal.tsx:206`
- `backoffice/Usuarios.tsx:258`

Mas **nunca e chamada na Lojinha ou SelecaoParcelamentoLojinha**.

### 5.2. Buscar tipo real do usuario na Lojinha

Adicionar fetch no hook `usePlanosLojinha.ts`:

```typescript
// Em usePlanosLojinha.ts, apos buscar o user (linha ~189)
const fetchTipoUsuario = async (userId: string, turmaId: string) => {
  // 1. Buscar tipo_participante da turma
  const { data: participante } = await supabase
    .from('turma_participantes')
    .select('tipo_participante, plano_social')
    .eq('user_id', userId)
    .eq('turma_id', turmaId)
    .eq('status', 'Ativo')
    .single();

  if (!participante) return 'nao_aderente'; // default seguro

  // 2. Plano social tem prioridade
  if (participante.plano_social) return 'plano_social';

  // 3. Comissao/Presidente → desconto de comissao
  if (['Comissão', 'Presidente'].includes(participante.tipo_participante)) {
    return 'comissao';
  }

  // 4. Para tipo 'Padrão', verificar aderencia
  //    Aderente = tem contrato ativo com categoria is_principal
  //    Usa a RPC que ja existe no banco
  const { data: isAderente } = await supabase
    .rpc('check_user_is_aderente', {
      p_user_id: userId,
      p_turma_id: turmaId
    });

  return isAderente ? 'aderente' : 'nao_aderente';
};
```

**Na adesao (`SelecaoParcelamento.tsx`):** Nao precisa checar aderencia, so tipo_participante:

```typescript
const fetchTipoUsuario = async (userId: string, turmaId: string) => {
  const { data: participante } = await supabase
    .from('turma_participantes')
    .select('tipo_participante')
    .eq('user_id', userId)
    .eq('turma_id', turmaId)
    .eq('status', 'Ativo')
    .single();

  if (!participante) return 'nao_aderente';
  if (['Comissão', 'Presidente'].includes(participante.tipo_participante)) {
    return 'comissao';
  }
  return 'nao_aderente'; // na adesao ninguem e aderente ainda
};
```

### 5.3. Passar tipo como prop, remover useState hardcoded

**SelecaoParcelamentoLojinha.tsx:**
```typescript
// ANTES (bug):
const [tipoUsuario, setTipoUsuario] = useState<...>('comissao');

// DEPOIS (fix):
// Receber tipoUsuario como prop do componente pai (Lojinha.tsx)
interface SelecaoParcelamentoLojinhaProps {
  // ...props existentes
  tipoUsuario: 'comissao' | 'aderente' | 'nao_aderente' | 'plano_social';
}
```

**SelecaoParcelamento.tsx:**
```typescript
// ANTES (bug):
const [tipoUsuario, setTipoUsuario] = useState<string>("comissao");

// DEPOIS (fix):
// Buscar tipo_participante internamente (sem check de aderencia)
```

### 5.3. Remover seletor de debug

Remover completamente os seletores de debug (linhas 537-551 de SelecaoParcelamentoLojinha e 2060-2068 de SelecaoParcelamento) ou ocultar atras de flag admin:

```typescript
// Opcao 1: Remover completamente
// (deletar o bloco <div className="mb-6 p-4 bg-green-50...">)

// Opcao 2: Ocultar para usuarios normais
{isAdmin && (
  <div className="mb-6 p-4 bg-green-50 border border-green-200 rounded-lg">
    ...
  </div>
)}
```

### 5.4. Default seguro

Se por qualquer motivo o fetch falhar, o default deve ser `'nao_aderente'` (o tipo que paga MAIS), nao `'comissao'` (que paga MENOS):

```typescript
const [tipoUsuario, setTipoUsuario] = useState<...>('nao_aderente');
```

---

## 6. Bug Relacionado #399907 — Desconto nao multiplica por quantidade

Na funcao `calcularDescontoTipoUsuario`, quando o tipo e `valor_fixo`, o desconto e aplicado uma vez sobre o total, nao multiplicado pela quantidade:

```typescript
// ATUAL (possivelmente errado para valor_fixo):
desconto = descontos.comissao.valor || 0;  // Ex: R$ 200 fixo, independente da quantidade

// O CORRETO depende da regra de negocio:
// Se R$ 200 e "por unidade": desconto = 200 * quantidade
// Se R$ 200 e "sobre o total": desconto = 200 (atual)
```

**Verificar com a equipe:** O desconto fixo de R$ 200 e por unidade ou sobre o total da compra?

Se for por unidade (cenario mais provavel para convites):
```typescript
case 'comissao':
  if (descontos.comissao) {
    desconto = descontos.comissao.tipo === 'valor_fixo'
      ? (descontos.comissao.valor || 0) * quantidade  // ← multiplicar
      : valorComCorrecao * ((descontos.comissao.valor || 0) / 100);
  }
```

---

## 7. Arquivos a Alterar

| Arquivo | Alteracao |
|---------|-----------|
| `src/hooks/usePlanosLojinha.ts` | Adicionar fetch de tipo_participante + check_user_is_aderente (aderencia so na lojinha) |
| `src/components/SelecaoParcelamentoLojinha.tsx` | Receber tipoUsuario como prop, remover useState, remover debug selector |
| `src/pages/Lojinha.tsx` | Passar tipoUsuario para SelecaoParcelamentoLojinha |
| `src/pages/SelecaoParcelamento.tsx` | Buscar tipo_participante (sem check aderencia), remover debug selector |
| `src/hooks/useCalculoParcelamento.ts` | (Bug #399907) Multiplicar desconto fixo por quantidade se regra for "por unidade" |
| `src/components/SelecaoQuantidade.tsx` | (Melhoria) Mostrar preco com desconto aplicado |

---

## 8. Prioridade de Implementacao

1. **URGENTE:** Trocar default de `'comissao'` para `'nao_aderente'` (1 minuto, para parar a hemorragia)
2. **ALTO:** Implementar fetch real do tipo_participante nos hooks
3. **ALTO:** Remover seletores de debug de producao
4. **MEDIO:** Corrigir multiplicacao de desconto por quantidade (#399907)
5. **BAIXO:** Mostrar preco com desconto na SelecaoQuantidade

---

## 9. Queries SQL para Auditoria

### Ver todos os lotes com descontos diferentes entre tipos:
```sql
SELECT l.nome_lote, l.valor,
  (l.descontos->'comissao'->>'valor')::numeric as desc_comissao,
  (l.descontos->'aderente'->>'valor')::numeric as desc_aderente,
  (l.descontos->'nao_aderente'->>'valor')::numeric as desc_nao_ader,
  p.nome_plano, t.nome as turma
FROM lotes l
JOIN planos p ON p.id = l.plano_id
JOIN turmas t ON t.id = p.turma_id
WHERE l.descontos IS NOT NULL
AND (l.descontos->'comissao'->>'valor')::numeric != (l.descontos->'aderente'->>'valor')::numeric
ORDER BY l.created_at DESC;
```

### Ver contratos que podem ter sido afetados (compras com desconto indevido):
```sql
SELECT c.id, c.valor_total, c.user_id, c.turma_id,
  tp.tipo_participante,
  l.descontos,
  CASE
    WHEN tp.tipo_participante IN ('Comissão','Presidente') THEN 'comissao'
    ELSE 'nao_comissao'
  END as tipo_real
FROM contratos c
JOIN lotes l ON l.id = c.lote_id
JOIN turma_participantes tp ON tp.user_id = c.user_id AND tp.turma_id = c.turma_id
WHERE tp.tipo_participante NOT IN ('Comissão', 'Presidente')
AND l.descontos IS NOT NULL
AND (l.descontos->'comissao'->>'valor')::numeric > 0
AND c.status = 'ativo'
ORDER BY c.created_at DESC;
```
