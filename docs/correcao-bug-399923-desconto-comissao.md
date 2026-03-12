# Correcao — Bug #399923: Desconto de Comissao Aplicado a Todos os Usuarios

**Data:** 2026-03-12
**Bug:** #399923
**Documento de analise:** `analise-bug-399923-desconto-comissao.md`
**Status:** Aguardando aprovacao

---

## 1. Resumo da Correcao

O `tipoUsuario` esta hardcoded como `'comissao'` em dois componentes, fazendo com que **todos os usuarios recebam desconto de comissao** na lojinha e na adesao. A correcao envolve:

1. **Lojinha:** Detectar automaticamente o tipo do usuario (comissao, aderente, nao_aderente) via `turma_participantes` + RPC `check_user_is_aderente`
2. **Adesao:** Detectar via `turma_participantes.tipo_participante` (sem checar aderencia — nao se aplica no fluxo de adesao)
3. **Remover seletores debug** visiveis em producao em ambos os componentes

---

## 2. Arquivos a Alterar

| # | Arquivo | O que fazer |
|---|---------|-------------|
| 1 | `src/hooks/usePlanosLojinha.ts` | Adicionar fetch do `tipoUsuario` (turma_participantes + check_user_is_aderente) e exportar |
| 2 | `src/pages/Lojinha.tsx` | Passar `tipoUsuario` como prop para `SelecaoParcelamentoLojinha` |
| 3 | `src/components/SelecaoParcelamentoLojinha.tsx` | Receber `tipoUsuario` como prop, remover useState, remover debug selector |
| 4 | `src/pages/SelecaoParcelamento.tsx` | Buscar `tipo_participante` do usuario logado, remover default `'comissao'`, remover debug selector |

---

## 3. Alteracoes Detalhadas

### 3.1. `src/hooks/usePlanosLojinha.ts`

**Objetivo:** Buscar o tipo do usuario e checar aderencia. Exportar `tipoUsuario` no retorno do hook.

**Apos a linha 194** (onde ja temos `user` autenticado), adicionar:

```typescript
// 5.0 Determinar tipo do usuario para descontos
let tipoUsuarioDetectado: 'comissao' | 'aderente' | 'nao_aderente' | 'plano_social' = 'nao_aderente';

// Buscar tipo_participante na turma
const { data: participante } = await supabase
  .from('turma_participantes')
  .select('tipo_participante')
  .eq('user_id', user.id)
  .eq('turma_id', turmaAtual.id)
  .maybeSingle();

if (participante?.tipo_participante) {
  const tipo = participante.tipo_participante;
  // Mapear valores do banco para tipos de desconto
  if (tipo === 'Comissão' || tipo === 'Presidente') {
    tipoUsuarioDetectado = 'comissao';
  } else {
    // Usuario padrao — checar se e aderente (tem contrato ativo com categoria principal)
    const { data: isAderente } = await supabase.rpc('check_user_is_aderente', {
      p_user_id: user.id,
      p_turma_id: turmaAtual.id,
    });
    tipoUsuarioDetectado = isAderente ? 'aderente' : 'nao_aderente';
  }
}
```

**No retorno do hook** (aprox. linha 340), adicionar `tipoUsuario: tipoUsuarioDetectado`:

```typescript
// Antes:
return { planos, categorias, dadosPagamento, loading, error };

// Depois:
return { planos, categorias, dadosPagamento, tipoUsuario: tipoUsuarioDetectado, loading, error };
```

**Adicionar state** para tipoUsuario no hook (junto com os outros states):

```typescript
const [tipoUsuario, setTipoUsuario] = useState<'comissao' | 'aderente' | 'nao_aderente' | 'plano_social'>('nao_aderente');
```

E no fetchPlanos, setar: `setTipoUsuario(tipoUsuarioDetectado);`

---

### 3.2. `src/pages/Lojinha.tsx`

**Objetivo:** Passar `tipoUsuario` do hook para o componente de parcelamento.

**Linha 28** — desestruturar `tipoUsuario`:

```typescript
// Antes:
const { planos, categorias, dadosPagamento, loading, error } = usePlanosLojinha();

// Depois:
const { planos, categorias, dadosPagamento, tipoUsuario, loading, error } = usePlanosLojinha();
```

**Linha 220** — passar prop:

```tsx
// Antes:
<SelecaoParcelamentoLojinha
  plano={planoSelecionado}
  quantidade={quantidadeSelecionada}
  ...
/>

// Depois:
<SelecaoParcelamentoLojinha
  plano={planoSelecionado}
  quantidade={quantidadeSelecionada}
  tipoUsuario={tipoUsuario}
  ...
/>
```

---

### 3.3. `src/components/SelecaoParcelamentoLojinha.tsx`

**Objetivo:** Receber `tipoUsuario` como prop em vez de usar useState hardcoded. Remover debug selector.

**Linha 26-40** — adicionar `tipoUsuario` na interface:

```typescript
interface SelecaoParcelamentoLojinhaProps {
  plano: PlanoComLote;
  quantidade: number;
  tipoUsuario: 'comissao' | 'aderente' | 'nao_aderente' | 'plano_social'; // NOVO
  dadosPagamento?: any;
  onConfirmar: (...) => void;
  onVoltar: () => void;
}
```

**Linha 42-48** — desestruturar `tipoUsuario` dos props:

```typescript
export function SelecaoParcelamentoLojinha({
  plano,
  quantidade,
  tipoUsuario,  // NOVO — vem do hook usePlanosLojinha
  dadosPagamento: dadosPagamentoProp,
  onConfirmar,
  onVoltar
}: SelecaoParcelamentoLojinhaProps) {
```

**Linha 52** — REMOVER o useState:

```typescript
// REMOVER esta linha:
const [tipoUsuario, setTipoUsuario] = useState<'comissao' | 'aderente' | 'nao_aderente' | 'plano_social'>('comissao');
```

**Linhas 537-551** — REMOVER INTEIRO o bloco debug selector:

```tsx
// REMOVER TUDO de "Tipo de Usuário - Debug" ate o </select> + info de desconto
// Aproximadamente linhas 537-600 (todo o <div className="mb-6 p-4 bg-green-50...">)
```

---

### 3.4. `src/pages/SelecaoParcelamento.tsx` (Adesao)

**Objetivo:** Buscar `tipo_participante` do usuario logado. Sem checar aderencia (nao se aplica em adesao — o usuario esta contratando o primeiro plano).

**Linha 44** — mudar default de `'comissao'` para `'nao_aderente'`:

```typescript
// Antes:
const [tipoUsuario, setTipoUsuario] = useState<string>("comissao");

// Depois:
const [tipoUsuario, setTipoUsuario] = useState<string>("nao_aderente");
```

**Adicionar useEffect** (apos os outros useEffects, aprox. linha 100) para buscar o tipo real:

```typescript
// Buscar tipo_participante do usuario logado
useEffect(() => {
  const fetchTipoUsuario = async () => {
    const { data: { user } } = await supabase.auth.getUser();
    const turmaId = localStorage.getItem('turmaId');
    if (!user || !turmaId) return;

    const { data: participante } = await supabase
      .from('turma_participantes')
      .select('tipo_participante')
      .eq('user_id', user.id)
      .eq('turma_id', turmaId)
      .maybeSingle();

    if (participante?.tipo_participante) {
      const tipo = participante.tipo_participante;
      if (tipo === 'Comissão' || tipo === 'Presidente') {
        setTipoUsuario('comissao');
      } else {
        setTipoUsuario('nao_aderente');
        // Na adesao, nao checamos aderencia — o usuario esta contratando agora
      }
    }
  };
  fetchTipoUsuario();
}, []);
```

**Linhas 2056-2070** — REMOVER INTEIRO o bloco debug selector:

```tsx
// REMOVER todo o <div className="mb-6 p-4 bg-green-50..."> com o select de tipo usuario
```

---

## 4. Logica de Determinacao do Tipo

### 4.1. Lojinha (com aderencia)

```
turma_participantes.tipo_participante == 'Comissão' ou 'Presidente'
  → tipoUsuario = 'comissao'

turma_participantes.tipo_participante == 'Padrão'
  → check_user_is_aderente(user_id, turma_id)
    → true  → tipoUsuario = 'aderente'
    → false → tipoUsuario = 'nao_aderente'
```

### 4.2. Adesao (sem aderencia)

```
turma_participantes.tipo_participante == 'Comissão' ou 'Presidente'
  → tipoUsuario = 'comissao'

turma_participantes.tipo_participante == 'Padrão' (ou qualquer outro)
  → tipoUsuario = 'nao_aderente'
  (nao checa aderencia — o usuario esta contratando agora)
```

### 4.3. Plano Social

O `plano_social` e um tipo especial que nao depende de `tipo_participante`. A logica de determinacao de plano social sera tratada em uma melhoria futura (provavelmente via flag no lote ou na categoria). Por enquanto, o desconto de plano social so sera aplicavel manualmente pelo backoffice.

---

## 5. Mapeamento de Valores

| DB (`turma_participantes.tipo_participante`) | Codigo (`tipoUsuario`) | Desconto aplicado |
|----------------------------------------------|------------------------|-------------------|
| `Comissão` | `comissao` | `lote.descontos.comissao` |
| `Presidente` | `comissao` | `lote.descontos.comissao` |
| `Padrão` + aderente | `aderente` | `lote.descontos.aderente` |
| `Padrão` + nao aderente | `nao_aderente` | `lote.descontos.nao_aderente` |

---

## 6. RPC `check_user_is_aderente`

Ja existe (migration `20251128000000`). Assinatura:

```sql
check_user_is_aderente(p_user_id UUID, p_turma_id UUID) → BOOLEAN
```

Retorna `true` se o usuario tem contrato ativo com categoria onde `is_principal = true` na turma.

Ja e chamada em:
- `OnboardingGuard.tsx:146`
- `Portal.tsx:206`
- `backoffice/Usuarios.tsx:258`

**Nao e chamada** (e deveria ser) em:
- `usePlanosLojinha.ts` — sera adicionada nesta correcao

---

## 7. O que sera REMOVIDO

### 7.1. Debug selectors em producao

Dois seletores identicos estao visiveis para todos os usuarios em producao:

1. **`SelecaoParcelamentoLojinha.tsx:537-551`** — dropdown verde "Tipo de Usuario (Debug - Provisorio)"
2. **`SelecaoParcelamento.tsx:2056-2070`** — mesmo dropdown

Estes permitem que qualquer usuario mude seu tipo e receba descontos indevidos. Serao removidos completamente.

### 7.2. Estado local hardcoded

1. **`SelecaoParcelamentoLojinha.tsx:52`** — `useState('comissao')` → substituido por prop
2. **`SelecaoParcelamento.tsx:44`** — `useState("comissao")` → default muda para `'nao_aderente'` + fetch automatico

---

## 8. Impacto e Riscos

### Impacto positivo
- Usuarios que nao sao comissao param de receber desconto indevido
- Aderentes passam a receber desconto correto automaticamente
- Debug selectors removidos eliminam vulnerabilidade de manipulacao de preco

### Riscos
- **Baixo:** Se `turma_participantes` nao tiver registro para o usuario, default e `nao_aderente` (seguro)
- **Baixo:** Se `check_user_is_aderente` falhar, default e `nao_aderente` (seguro)
- **Medio:** Usuarios de comissao que estao acostumados com o desconto continuarao recebendo normalmente — sem mudanca para eles

### Testes recomendados
1. Login como usuario Comissao → verificar desconto de comissao aplicado
2. Login como usuario Padrao sem contrato principal → verificar sem desconto (nao_aderente)
3. Login como usuario Padrao com contrato principal ativo → verificar desconto aderente na lojinha
4. Verificar que debug selector nao aparece mais em producao

---

## 9. Estimativa de Complexidade

- **Baixa complexidade** — 4 arquivos, sem mudanca de schema, sem migration
- A RPC ja existe, a tabela `turma_participantes` ja tem os dados
- Maior parte da mudanca e remocao de codigo (debug selectors) e adicao de 1 query + 1 RPC call
