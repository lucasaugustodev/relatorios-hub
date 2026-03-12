# Plano Detalhado de Correção — 11/03/2026

Documento gerado a partir da análise do `classificacao-tarefas-11-03.md`, confrontado com o código-fonte do projeto e o monitor em produção.

---

## Correções já aplicadas

### ✅ BUG #391363 — IMPORTANTE: valores não batem (contratado, a pagar)

**Status:** CORRIGIDO E TESTADO | **CI:** Lint ✓ Test ✓ Build ✓
**Commit:** `7ab3f03` — `fix(#391363): corrigir arredondamento de parcelas no replanejamento`

**Problema original:**
Plano de R$ 17.169,85 com 34 parcelas gerava apenas R$ 14.492,76 (faltando R$ 2.677,09).
A divisão `valorRestante / parcelasRestantes` produzia ponto flutuante e cada parcela salva com `.toFixed(2)` acumulava erro de centavos. Não havia validação de que a soma das parcelas batia com o principal.

**O que foi corrigido em `src/services/renegociacaoService.ts`:**

1. **Arredondamento controlado:** `valorParcelaRegular` agora usa `Math.floor(x * 100) / 100` (trunca centavos para baixo ao invés de arredondar, evitando soma > principal)
2. **Última parcela compensatória:** A última parcela é calculada como `principal - somaGerada` para absorver os centavos residuais, garantindo soma exata
3. **Validação de integridade no `salvarRenegociacao()`:** Antes de inserir no banco, verifica que `|soma(parcelas) - principal| <= R$ 0.02`. Se divergir, lança erro e impede gravação
4. **Validação preventiva no `validarRenegociacao()`:** Checa totalização antes de confirmar operação

**Resultado dos testes (22/22 passaram):**

| Teste | Resultado |
|---|---|
| R$ 17.169,85 / 34 parcelas (caso do bug) | ✅ Soma exata: R$ 17.169,85 |
| R$ 10.000 / 3 parcelas (divisão inexata) | ✅ R$ 3.333,33 + R$ 3.333,33 + R$ 3.333,34 |
| R$ 1.000,01 / 7 parcelas (centavos irracionais) | ✅ Soma exata |
| Dia 30 com fevereiro (overflow) | ✅ Fev usa dia 28, março volta a 30 |
| Dia 31 em meses de 30 dias | ✅ Abril/Junho usam dia 30 |
| 12 meses sem duplicata por overflow | ✅ 12 meses distintos |
| Entrada inteligente (max entre % e parcela) | ✅ R$ 1.000 |
| 60 parcelas com entrada R$ 5.000 | ✅ Soma exata: R$ 50.000 |
| 1 parcela (só entrada) | ✅ Valor = R$ 5.000 |
| 34 parcelas com dia 30 e overflow | ✅ 34 meses distintos, soma exata |
| Stress test: 100 cenários aleatórios | ✅ 0 falhas |

**Arquivos alterados:**
- `src/services/renegociacaoService.ts` — `calcularRenegociacao()`, `salvarRenegociacao()`, `validarRenegociacao()`
- `src/services/__tests__/renegociacaoService.test.ts` — 22 testes unitários (novo)
- `vite.config.ts` — configuração Vitest
- `package.json` — scripts `test` e `test:watch`
- `.github/workflows/ci.yml` — step de teste adicionado ao CI

---

### ✅ BUG #397137 — Boleto pago antes do replanejamento

**Status:** Correção aplicada (commit `4b97a84`), aguardando reteste de Isabelle.

---

## Resumo Executivo

| Categoria | Quantidade |
|---|---|
| Bugs confirmados no código | 16 |
| Bugs corrigidos e testados | 3 |
| Melhorias / Features | 5 |
| **Total** | **24** |

### Prioridade de Correção

| Bloco | Foco | Bugs | Qtd |
|---|---|---|---|
| **Bloco 1** | Críticos (afetam dinheiro/valores) | #399923, #399901, #399907, #399877, #399883, #400011, #399995 | 7 |
| **Bloco 2** | Fluxo bloqueado | #399885, #399891, #399911, #400009 | 4 |
| **Bloco 3** | UX/Display | #400007, #399903, #399915, #400001, #399997, #399905 | 6 |
| **Melhorias** | Features/UX | #399895, #399899, #399977, #399991, #384795 | 5 |

---

## BLOCO 1 — CRÍTICOS (afetam dinheiro/valores)

---

### BUG #399923 + #399901 — Desconto de comissão aplicado para todos os usuários + Valor divergente no convite extra

**Severidade:** Alta
**Arquivo:** `src/components/SelecaoParcelamentoLojinha.tsx` linha 52
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// Linha 52 — DEFAULT É 'comissao' PARA TODOS OS USUÁRIOS
const [tipoUsuario, setTipoUsuario] = useState<'comissao' | 'aderente' | 'nao_aderente' | 'plano_social'>('comissao');
```

Além disso, nas linhas 537-551, existe um seletor marcado como **"Debug - Provisório"** que permite ao usuário trocar o tipo manualmente. Em produção, o default `'comissao'` faz com que TODOS os usuários recebam preço de comissão (R$ 1.000 ao invés de R$ 1.200).

**Impacto:** Todos os usuários não-comissão pagam menos do que deveriam. Perda financeira direta.

**Correção:**
1. Buscar `tipo_participante` real do usuário logado na tabela `turma_participantes`
2. Alterar o default de `'comissao'` para `'aderente'` (valor mais seguro)
3. Remover o seletor de debug ou ocultá-lo atrás de uma flag de admin
4. Exemplo de implementação:
```typescript
// Buscar tipo real do usuário
useEffect(() => {
  const fetchTipoUsuario = async () => {
    const { data: { user } } = await supabase.auth.getUser();
    if (user && plano.turma_id) {
      const { data: participante } = await supabase
        .from('turma_participantes')
        .select('tipo_participante')
        .eq('user_id', user.id)
        .eq('turma_id', plano.turma_id)
        .single();
      if (participante?.tipo_participante) {
        const mapa: Record<string, string> = {
          'Formando': 'aderente',
          'Comissão': 'comissao',
          'Não Aderente': 'nao_aderente'
        };
        setTipoUsuario(mapa[participante.tipo_participante] || 'aderente');
      }
    }
  };
  fetchTipoUsuario();
}, [plano.turma_id]);
```

**Também afeta:** Bug #399901 (valor do 2º lote aparece R$ 1.000 ao invés de R$ 1.200) — mesma causa raiz. Adicionalmente, a opção de arrecadação alternativa habilitada para convites extras deve ser desabilitada com base nas regras do lote/plano.

---

### BUG #399907 — Erro na somatória dos valores (convite extra)

**Severidade:** Alta
**Arquivo:** `src/hooks/useCalculoParcelamento.ts` (lógica de desconto)
**Status:** Confirmado no código

**Causa raiz:**
O desconto por tipo de usuário é aplicado como valor fixo **total**, não **por unidade**. Quando se compra 5 convites a R$ 1.200 cada com desconto de R$ 200 por unidade (comissão):

- **Esperado:** 5 × (R$ 1.200 - R$ 200) = R$ 5.000
- **Gerado:** 5 × R$ 1.200 - R$ 200 = R$ 5.800

O desconto fixo é aplicado uma única vez sobre o total, ao invés de ser multiplicado pela quantidade.

**Correção:**
No `useCalculoParcelamento.ts`, na lógica de desconto por tipo de usuário, multiplicar o desconto fixo pela quantidade:
```typescript
case 'comissao':
  if (descontos.comissao) {
    desconto = descontos.comissao.tipo === 'valor_fixo'
      ? (descontos.comissao.valor || 0) * quantidade  // ← multiplicar por quantidade
      : valorComCorrecao * ((descontos.comissao.valor || 0) / 100);
  }
  break;
```

Aplicar a mesma correção para todos os tipos de usuário (`aderente`, `nao_aderente`, `plano_social`).

---

### BUG #399877 — Arrecadação alternativa não recalcula valor da parcela no replanejamento

**Severidade:** Alta
**Arquivo:** `src/services/renegociacaoService.ts` linhas 447-463
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// Linha 447-455 — descontoMensal é CALCULADO...
const descontoMensal = calculateMonthlyDiscount();

// Linha 463 — ...mas NUNCA é subtraído do valor da parcela!
const valorParcela = resumo.valorParcela;  // ← usa valor SEM desconto
```

O `descontoMensal` é calculado corretamente mas nunca aplicado. As parcelas são criadas com o valor cheio, ignorando a arrecadação alternativa selecionada.

**Correção:**
Na criação das parcelas normais (linha 606+), subtrair o desconto mensal:
```typescript
// Opção 1: Ajustar no loop de criação de parcelas
const valorParcelaAjustado = valorParcela - descontoMensal;

// Opção 2: Recalcular o principal antes de chamar calcularRenegociacao
const principalAjustado = principalReplanejar - valorArrecadacaoTotal;
```

Cada parcela criada deve incluir o campo `desconto_arrecadacao` com o valor do desconto mensal aplicado, e `valor_arrecadacao_mes` com o valor correspondente.

---

### BUG #399883 + #400011 — Parcelas geradas fora do período financeiro da turma (replanejamento + upgrade)

**Severidade:** Alta
**Arquivos:**
- `src/services/renegociacaoService.ts` linhas 309-316 (replanejamento)
- `src/services/mudancaPlanoService.ts` linhas 921-926 e 1012-1031 (upgrade)
**Status:** Confirmado no código

**Causa raiz (replanejamento):**
```typescript
// renegociacaoService.ts linha 314-316
} else {
  dataBase = new Date();  // ← usa HOJE, ignora período financeiro da turma
}
```
Quando `dataPrimeiraParcela` não é fornecida, o código usa a data de hoje. Se hoje é 11/03 e a turma só permite pagamentos a partir de 01/04, a primeira parcela é gerada para 16/03 (hoje + 5 dias úteis).

Além disso, permite selecionar 40 parcelas quando o limite da turma é 39 — não há validação do número máximo contra `dataUltimaParcela`.

**Causa raiz (upgrade):**
```typescript
// mudancaPlanoService.ts linhas 921-926
const dataPrimeiraParcela = new Date();  // ← usa HOJE
dataPrimeiraParcela.setDate(mudanca.dia_vencimento_novo || 10);
if (dataPrimeiraParcela <= new Date()) {
  dataPrimeiraParcela.setMonth(dataPrimeiraParcela.getMonth() + 1);
}
```
Mesmo problema: não consulta `dados_pagamento.dataPrimeiraParcela` nem `dados_pagamento.dataUltimaParcela` da turma.

No loop de criação (linhas 1012-1031), parcelas são geradas indefinidamente sem checar se ultrapassam `dataUltimaParcela`.

**Correção (replanejamento):**
```typescript
// 1. Receber dadosPagamento da turma como parâmetro
// 2. Calcular data base respeitando período financeiro
let dataBase: Date;
if (dataPrimeiraParcela) {
  const [ano, mes, dia] = dataPrimeiraParcela.split('-').map(Number);
  dataBase = new Date(ano, mes - 1, dia);
} else {
  const hoje = new Date();
  if (dadosPagamento?.dataPrimeiraParcela) {
    const dataInicioFinanceiro = new Date(dadosPagamento.dataPrimeiraParcela);
    dataBase = hoje > dataInicioFinanceiro ? hoje : dataInicioFinanceiro;
  } else {
    dataBase = hoje;
  }
}

// 3. Validar número máximo de parcelas
if (dadosPagamento?.dataUltimaParcela) {
  const dataFim = new Date(dadosPagamento.dataUltimaParcela);
  const mesesDisponiveis = diffMeses(dataBase, dataFim);
  if (numeroParcelas > mesesDisponiveis) {
    throw new Error(`Máximo de ${mesesDisponiveis} parcelas para este período`);
  }
}
```

**Correção (upgrade):**
```typescript
// 1. Buscar dados_pagamento da turma
const { data: turmaData } = await supabase
  .from('turmas')
  .select('dados_pagamento')
  .eq('id', mudanca.turma_id)
  .single();

// 2. Respeitar data inicial do período financeiro
const dataInicioFinanceiro = turmaData?.dados_pagamento?.dataPrimeiraParcela
  ? new Date(turmaData.dados_pagamento.dataPrimeiraParcela)
  : null;

if (dataInicioFinanceiro && dataPrimeiraParcela < dataInicioFinanceiro) {
  dataPrimeiraParcela = dataInicioFinanceiro;
}

// 3. No loop de parcelas, validar contra dataUltimaParcela
const dataUltima = turmaData?.dados_pagamento?.dataUltimaParcela
  ? new Date(turmaData.dados_pagamento.dataUltimaParcela)
  : null;

for (let i = 0; i < qtdParcelasNormais; i++) {
  const dataVencimento = adicionarMesesComDiaFixo(dataPrimeira, i, diaVencimentoConfig);
  if (dataUltima && dataVencimento > dataUltima) {
    console.warn(`Parcela ${i+2} ultrapassaria o período financeiro. Parando.`);
    break;
  }
  // ... criar parcela
}
```

---

### BUG #399995 — Erro no upgrade de plano (valores pagos não abatidos, débito inexplicado, parcelamento estendido indevido)

**Severidade:** Alta
**Arquivo:** `src/services/mudancaPlanoService.ts` linhas 540-577
**Status:** Confirmado no código

**Causa raiz:** Três problemas distintos:

**1. Débito inexplicado (linhas 569-571):**
```typescript
} else if (tipoResultado === 'DEBITO') {
  debitoParcelaAjuste = valorDebito;  // ← cria parcela sem explicação visual
}
```
O débito é aplicado mas o frontend não mostra de onde vem esse valor.

**2. Valores pagos não abatidos:**
O cálculo de crédito (linhas 551-568) funciona corretamente no service, mas o frontend pode não estar passando corretamente o `valorCredito` (soma dos valores já pagos no plano atual).

**3. Parcelamento estendido indevido:**
A configuração default pode incluir estendido quando não deveria. Ao abrir a tela de upgrade, não herda as condições de pagamento do plano atual.

**Correção:**
1. Adicionar campo de descrição na UI explicando a origem do débito
2. No preview de mudança, mostrar breakdown claro:
   - Valor do plano novo: R$ X
   - Crédito do plano atual (já pago): -R$ Y
   - Diferença a pagar: R$ Z
3. Pré-selecionar o mesmo número de parcelas e tipo de parcelamento do contrato atual
4. Na `previewMudancaPlano`, garantir que `valorCredito` = soma real dos pagamentos do contrato atual

---

## BLOCO 2 — FLUXO BLOQUEADO

---

### BUG #399885 — Lote agendado não ativou no horário programado

**Severidade:** Alta
**Arquivo:** `supabase/functions/manage-lotes-schedule/index.ts` linhas 129-134
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// Linhas 129-134 — lote pausado é IGNORADO COMPLETAMENTE
if (lote.status_venda === "pausado") {
  console.log(`[CRON] Lote "${lote.nome_lote}" está pausado. Ignorando ativação automática.`);
  return null;  // ← retorna antes de verificar o horário programado
}
```

Quando um lote é pausado, a verificação de horário agendado nunca é executada. Se o horário programado chegar enquanto o lote está pausado, ele nunca será ativado.

**Correção:**
Mover a verificação de `pausado` para DEPOIS da lógica de ativação por horário:
```typescript
// Verificar se há ativação programada PRIMEIRO
if (lote.data_inicio_venda) {
  const dataInicio = new Date(lote.data_inicio_venda);
  if (agora >= dataInicio && (lote.status_venda === 'inativo' || lote.status_venda === 'pausado')) {
    // Ativar o lote — o horário programado tem prioridade
    await supabase.from('lotes').update({ status_venda: 'ativo' }).eq('id', lote.id);
    return { ativado: true };
  }
}

// SÓ ENTÃO verificar se está pausado para outras operações
if (lote.status_venda === "pausado") {
  return null;
}
```

---

### BUG #399891 — Solicitação de rescisão não aparece no painel admin

**Severidade:** Alta
**Arquivo:** `src/pages/gestao-financeira/MinhasRescisoes.tsx` linhas 64-75
**Status:** Provável erro de FK

**Causa raiz:**
```typescript
// Linha 68 — FK explícita pode estar incorreta
contratos!rescission_requests_contrato_id_fkey(
  numero_contrato,
  planos(nome_plano),
  turmas(nome)
)
```

O nome da foreign key `rescission_requests_contrato_id_fkey` pode não corresponder ao nome real no schema do Supabase, causando falha silenciosa na query.

**Correção:**
1. Verificar o nome exato da FK no banco: `SELECT conname FROM pg_constraint WHERE conrelid = 'rescission_requests'::regclass;`
2. Simplificar a query removendo a FK explícita:
```typescript
const { data, error } = await supabase
  .from('rescission_requests')
  .select(`*, contratos(numero_contrato, planos(nome_plano), turmas(nome))`)
  .eq('user_id', user.id)
```
3. Adicionar tratamento de erro visível para diagnóstico

---

### BUG #399911 — Limite de compra de 5 convites não é cumulativo entre transações

**Severidade:** Alta
**Arquivo:** `src/components/SelecaoQuantidade.tsx` linhas 17-25
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// Linha 17 — usa limite raw sem verificar compras anteriores
const limiteUsuario = plano.lote.limite_por_usuario || 999;
// Linha 25
const maxQuantidade = Math.min(limiteUsuario, estoqueDisponivel);
```

O `limiteUsuario` é o limite total configurado no lote, mas não subtrai as quantidades já compradas pelo usuário em transações anteriores. Um usuário pode fazer compra de 3 + compra de 3 = 6 convites, ultrapassando o limite de 5.

**Correção:**
Receber como prop ou buscar do banco a quantidade já comprada:
```typescript
// No componente pai (Lojinha.tsx), antes de renderizar SelecaoQuantidade:
const { data: contratosExistentes } = await supabase
  .from('contratos')
  .select('quantidade')
  .eq('user_id', userId)
  .eq('lote_id', plano.lote.id)
  .in('status', ['ativo', 'pendente']);

const quantidadeJaComprada = contratosExistentes?.reduce(
  (sum, c) => sum + (c.quantidade || 1), 0
) || 0;

// No SelecaoQuantidade.tsx:
interface SelecaoQuantidadeProps {
  plano: PlanoComLote;
  quantidadeJaComprada?: number;  // ← novo prop
  onConfirmar: (quantidade: number) => void;
  onVoltar: () => void;
}

const limiteUsuario = plano.lote.limite_por_usuario || 999;
const limiteRestante = Math.max(0, limiteUsuario - (quantidadeJaComprada || 0));
const maxQuantidade = Math.min(limiteRestante, estoqueDisponivel);

// Se limiteRestante === 0, mostrar alerta e desabilitar botão
```

---

### BUG #400009 — Sistema permite upgrade de plano já transferido

**Severidade:** Alta
**Arquivo:** `src/pages/MinhasContratacoes.tsx` linhas 77-93
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// Linhas 77-93 — isUpgradeEnabled NÃO verifica transferências
const isUpgradeEnabled = (contrato: any): boolean => {
  if (!contrato?.planos) return false;
  if (!contrato.planos.alteracao) return false;
  if (contrato.planos.alteracao.upgrade !== true) return false;
  // ... verifica apenas regras de data
  // ❌ NÃO verifica se contrato está em transferência
};
```

Na linha 783: `hidden: contrato.status !== "Ativo"` — um contrato transferido que permanece com status "Ativo" (bug #400007) passa nessa checagem.

**Correção:**
```typescript
const isUpgradeEnabled = (contrato: any): boolean => {
  if (!contrato?.planos) return false;
  if (!contrato.planos.alteracao) return false;
  if (contrato.planos.alteracao.upgrade !== true) return false;

  // ← ADICIONAR: Bloquear se contrato está em transferência
  if (contrato.status === 'em_transferencia' || contrato.status === 'transferido') return false;

  // ← ADICIONAR: Verificar se há transferência pendente
  // (requer busca assíncrona ou campo denormalizado no contrato)

  const regras = contrato.planos.pagamento?.regras_upgrade;
  if (!regras || regras.length === 0) return true;
  // ...
};
```

Aplicar mesma correção em `isDowngradeEnabled()` (linhas 96-113).

---

## BLOCO 3 — UX / DISPLAY

---

### BUG #400007 — Status não atualiza na listagem geral após transferência

**Severidade:** Média
**Arquivo:** `src/components/TransferenceModal.tsx` linhas 197-206
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// Linha 197 — RPC cria transfer_request mas NÃO atualiza contratos.status
const { data, error } = await (supabase as any).rpc('create_transfer_request', { ... });
// ❌ Nenhum update em contratos.status após sucesso
```

O contrato permanece como "ativo" na listagem geral mesmo após a transferência ser aceita.

**Correção:**
Após RPC bem-sucedido:
```typescript
if (data?.success) {
  // Atualizar status do contrato para refletir transferência
  await supabase
    .from('contratos')
    .update({ status: 'em_transferencia' })
    .eq('id', contractData.id);
}
```

E no fluxo de aceitação (IncomingTransferModal), atualizar para `'transferido'`.

---

### BUG #399903 — Sistema marca aluno como inadimplente no dia do vencimento

**Severidade:** Alta
**Arquivo:** `supabase/functions/verificar-aptidao-financeira/index.ts` linha 246
**Status:** Análise necessária

**Código atual:**
```typescript
// Linha 246
if (dataVencimento < hoje) {  // parcela vencida = data ANTES de hoje
```

**Análise:** O operador `<` está tecnicamente **correto** — o aluno tem até o dia do vencimento para pagar. O problema real pode ser:

1. **Timezone:** A função calcula `nowBrasilia` (UTC-3) mas `data_vencimento` pode estar em UTC no banco, causando comparação incorreta
2. **Frontend:** O status de inadimplência pode ser mostrado no frontend por uma lógica diferente que usa `<=`

**Correção:**
1. Verificar formato de `data_vencimento` no banco (DATE vs TIMESTAMPTZ)
2. Se TIMESTAMPTZ, normalizar: `dataVencimento.setHours(23, 59, 59, 999)` — considerar vencido só após o fim do dia
3. Auditar o frontend para encontrar outras checagens de inadimplência

---

### BUG #399915 — Dashboard não atualizou valores/parcelas

**Severidade:** Média
**Arquivo:** `src/pages/Dashboard.tsx` linhas 60-149
**Status:** Confirmado no código

**Causa raiz:**
```typescript
// O useEffect depende apenas de turma, não de mudanças nos dados
useEffect(() => {
  fetchParcelas();
}, [turmaAtual?.id, turmaLoading]);
// ❌ Sem refresh quando operações são concluídas
```

**Correção:**
Opção A — Adicionar refresh counter:
```typescript
const [refreshKey, setRefreshKey] = useState(0);

useEffect(() => {
  fetchParcelas();
}, [turmaAtual?.id, turmaLoading, refreshKey]);

// Nos callbacks de fechar modais:
onClose={() => setRefreshKey(prev => prev + 1)}
```

Opção B — Real-time subscription:
```typescript
useEffect(() => {
  fetchParcelas();
  const sub = supabase
    .channel('parcelas-changes')
    .on('postgres_changes', { event: '*', schema: 'public', table: 'parcelas' }, fetchParcelas)
    .subscribe();
  return () => { sub.unsubscribe(); };
}, [turmaAtual?.id]);
```

---

### BUG #400001 — Termo de transferência não aparece

**Severidade:** Média
**Arquivo:** `src/components/TransferenceModal.tsx` linhas 547-553
**Status:** Confirmado — placeholder nunca implementado

**Código atual:**
```typescript
// Linhas 547-553 — PLACEHOLDER
<div className="bg-gray-100 border-2 border-dashed border-gray-300 rounded-lg p-8 text-center">
  <p className="text-gray-600 mb-4">Termo de Transferência de Titularidade</p>
  <p className="text-sm text-gray-500">Visualização do contrato PDF apareceria aqui</p>
</div>
```

**Correção:**
1. Criar template de termo de transferência (similar ao contrato de adesão)
2. Gerar PDF usando a infra existente (`contractVariableReplacer.ts` + `PDFViewer.tsx`)
3. Renderizar no step de assinatura:
```typescript
{pdfUrl ? (
  <PDFViewer url={pdfUrl} />
) : (
  <div>Carregando termo de transferência...</div>
)}
```

---

### BUG #399997 — Link que explica valores da transferência dá erro

**Severidade:** Baixa
**Arquivo:** `src/components/TransferenceModal.tsx` linha 525
**Status:** Confirmado

**Causa raiz:** URL aponta para domínio externo incorreto ou rota inexistente.

**Correção:**
Alterar para rota interna SPA:
```typescript
// De:
href="https://hub.lucasaugusto.dev/gestao-financeira/entenda-valores"
// Para:
href="/gestao-financeira/entenda-valores"
// Ou renderizar informações inline no próprio modal
```

---

### BUG #399905 — Não permitiu liberar manual a aptidão

**Severidade:** Média
**Arquivo:** `src/components/EditarAptidaoModal.tsx` + `supabase/functions/verificar-aptidao-financeira/index.ts`
**Status:** Parcialmente investigado

**Análise:**
O cron JÁ respeita `editado_manualmente` (linhas 152-162 do index.ts — faz skip correto). O problema pode ser:
1. O botão de editar aptidão está desabilitado no frontend (verificar condições de renderização)
2. RLS policy na tabela `aptidao_financeira_alunos` bloqueia UPDATE do admin
3. O upsert falha silenciosamente

**Correção:**
1. Verificar RLS: `SELECT * FROM pg_policies WHERE tablename = 'aptidao_financeira_alunos'`
2. Verificar no EditarAptidaoModal se há condição que desabilita o botão de salvar
3. Adicionar toast de erro visível no catch do upsert

---

## MELHORIAS / FEATURES (5 tarefas)

---

### #399895 — Habilitar aba histórico

**Tipo:** Feature | **Prazo:** 16/03/2026
**Arquivo provável:** `src/pages/backoffice/Historico.tsx` (já existe)

**O que fazer:**
1. Verificar se a página já está funcional
2. Adicionar rota no menu/sidebar se não estiver visível
3. Garantir que os dados de histórico estão sendo salvos corretamente

---

### #399899 — Visualização de pacotes disponíveis (mesmo inadimplente)

**Tipo:** Melhoria UX | **Prazo:** 16/03/2026

**O que fazer:**
1. No componente de listagem de lotes, mostrar lotes mesmo quando o usuário não é apto
2. Adicionar badge/tag explicando o motivo do bloqueio (ex: "Inadimplente — regularize para comprar")
3. Desabilitar botão de compra mas manter visualização

---

### #399977 — Melhoria didática (UX tela Meus Contratos)

**Tipo:** Melhoria UX | **Sem prazo**

**O que fazer:**
1. Reduzir destaque do código do contrato (menor, cor mais suave)
2. Mostrar apenas tipo de serviço na coluna "Título" (sem nome da turma)
3. Adicionar coluna "Quantidade" mostrando itens adquiridos

---

### #399991 — Parcela estimada deve refletir configuração real

**Tipo:** Melhoria | **Prazo:** 16/03/2026
**Arquivo provável:** `src/hooks/useValorMensalPlano.ts`

**O que fazer:**
1. Buscar configuração real do plano do aderente (diluição AA, estendido)
2. Calcular parcela estimada considerando essas configurações
3. Mostrar tooltip explicando o cálculo

---

### #384795 — Conferir restrições por aderente (quitação e inadimplência)

**Tipo:** Feature/Teste | **Prazo:** 17/03/2026

**O que fazer:**
1. Criar cenários de teste automatizados usando o CLI (`cli-anything-hub-portal-vibe`)
2. Validar restrições para diferentes estados: quitado, inadimplente, parcialmente pago
3. Documentar resultados para o time

---

## COMO EXECUTAR TESTES

O projeto possui um CLI de testes em `agent-harness/`:

```bash
# Instalação
cd agent-harness && pip install -e .

# Comandos disponíveis para validação
cli-anything-hub-portal-vibe turma list                           # Listar turmas
cli-anything-hub-portal-vibe adesao summary <turma_id>            # Resumo de adesões
cli-anything-hub-portal-vibe contrato list --turma-id <id>        # Listar contratos
cli-anything-hub-portal-vibe replanejamento simular <contrato_id> --parcelas 12
cli-anything-hub-portal-vibe rescisao simular <contrato_id> --valor-plano 10000
cli-anything-hub-portal-vibe upgrade simular <contrato_id> --valor-atual 8000 --valor-novo 12000

# Testes automatizados com browser
cli-anything-hub-portal-vibe adesao run FORMAE-172687 --headless --reports-dir ./reports
cli-anything-hub-portal-vibe replanejamento run user@email.com --parcelas 12 --headless
cli-anything-hub-portal-vibe upgrade run user@email.com --headless
cli-anything-hub-portal-vibe rescisao run user@email.com --headless
```

### Monitor (porta 3001)
```bash
curl http://localhost:3001/api/health          # Status do sistema
curl http://localhost:3001/api/client-errors    # Erros JavaScript do cliente
curl http://localhost:3001/api/events           # Eventos do sistema
```
