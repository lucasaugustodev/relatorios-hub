# Documentação Financeira - Página de Seleção de Parcelamento

**Documento**: Análise de Variáveis, Cálculos e Regras de Negócio
**Sistema**: Hub Portal - Seleção de Parcelamento
**Data**: 31/10/2025
**Versão**: 1.0

---

## 1. VARIÁVEIS E CAMPOS DE ENTRADA

### 1.1 Variáveis de Controle do Sistema

| Variável | Tipo | Propósito Financeiro |
|----------|------|---------------------|
| `isRaffleEnabled` | boolean | Controla se a arrecadação alternativa (rifas/semestres) está ativada no cálculo |
| `parcelamento` | número | Quantidade de parcelas normais selecionadas pelo cliente |
| `parcelamentoEstendido` | número | Quantidade de parcelas estendidas via cartão (taxa adicional) |
| `diaVencimento` | 1-31 | Dia do mês para vencimento das parcelas subsequentes |
| `dataPrimeiraParcela` | data | Data da primeira parcela (formato AAAA-MM-DD) |
| `tipoUsuario` | string | Tipo do usuário: comissao, aderente, nao_aderente, plano_social |
| `loteAtivo` | objeto | Lote com valor base e tabela de descontos aplicável |

### 1.2 Inputs do Usuário

#### A. Data da Primeira Parcela
- **Formato**: YYYY-MM-DD
- **Entrada**: Calendário interativo
- **Validações**:
  - Deve estar entre `dataInicial` e `dataFinal` configuradas
  - Não pode ser fim de semana
  - Não pode ser feriado bancário
  - Limite máximo: `limiteDiasAdesao` dias após a primeira parcela configurada

#### B. Dia de Vencimento das Parcelas
- **Formato**: Número de 1 a 31
- **Aplicável**: Apenas quando houver mais de 1 parcela
- **Obrigatoriedade**: Obrigatório se `parcelamento > 1`
- **Comportamento**: Ajustado automaticamente se cair em fim de semana ou feriado

#### C. Quantidade de Parcelas Normais
- **Formato**: Número inteiro
- **Intervalo**: 1 até `maxParcelas`
- **Cálculo do Máximo**:
  ```
  Se configurado dataUltimaParcela:
    maxParcelas = MÍNIMO entre:
      - Meses disponíveis até dataUltimaParcela
      - Número máximo de parcelas configurado

  Senão:
    maxParcelas = numeroMaximoParcelas (padrão: 60)
  ```

#### D. Quantidade de Parcelas Estendidas (Cartão)
- **Formato**: Número inteiro
- **Intervalo**: 0 até `maxParcelasEstendido`
- **Condição**: Só ativo quando `parcelamento === maxParcelas`
- **Requisito**: Configuração `cartaoParcelamentoEstendido = true`

#### E. Porcentagem e Valores de Desconto
- **Desconto por Tipo de Usuário**: Configurado por lote
- **Desconto de Adesão**: Percentual ou valor fixo (se dentro do período)
- **Desconto à Vista**: Percentual ou valor fixo (apenas se 1 parcela sem arrecadação)

---

## 2. CÁLCULOS FINANCEIROS REALIZADOS

### 2.1 Correção Monetária

**Condição de Aplicação**: `selectedPlan?.financeiro?.temCorrecao === true`

**Fórmulas por Tipo**:

#### Tipo: VALOR FIXO
```
Correção Total = valorCorrecao × numberOfPeriods

Onde:
- valorCorrecao: valor configurado por período
- numberOfPeriods: quantidade de períodos desde dataInicio até hoje
```

#### Tipo: PERCENTUAL - Juros Simples
```
Correção Total = valorBase × (percentual / 100) × numberOfPeriods

Onde:
- valorBase: valor do lote ativo
- percentual: taxa configurada
- numberOfPeriods: quantidade de períodos
```

#### Tipo: PERCENTUAL - Juros Compostos
```
Correção Total = valorBase × [(1 + percentual/100)^numberOfPeriods - 1]

Onde:
- valorBase: valor do lote ativo
- percentual: taxa configurada
- numberOfPeriods: quantidade de períodos
```

**Validações**:
- Aplicada apenas se data atual estiver entre `dataInicio` e `dataTermino`
- Periodicidade: diária, mensal ou anual

---

### 2.2 Cálculo Escalonado de Descontos

O sistema aplica descontos em **etapas sucessivas** (cada desconto é aplicado sobre o resultado do anterior):

#### ETAPA 0: Aplicação de Correção Monetária
```
valorComCorrecao = valorBase + valorCorreção
```

#### ETAPA 1: Desconto por Tipo de Usuário
```
Desconto baseado em:
- tipoUsuario === "comissao"      → usa descontos.comissao
- tipoUsuario === "aderente"      → usa descontos.aderente
- tipoUsuario === "nao_aderente"  → usa descontos.nao_aderente
- tipoUsuario === "plano_social"  → usa descontos.plano_social

Cada desconto pode ser:
- "valor_fixo": Desconto = valor configurado
- "porcentagem": Desconto = valorComCorrecao × (valor / 100)

valorEtapa1 = valorComCorrecao - desconto
```

#### ETAPA 2: Desconto de Adesão
```
Condições:
- descontoAdesao = true
- Data atual >= dataInicioDesconto
- Data atual <= dataTerminoDesconto

Se tipoDesconto = "PERCENTUAL":
  Desconto = valorEtapa1 × (percentualDesconto / 100)

Se tipoDesconto = "VALOR":
  Desconto = valorDesconto configurado

valorEtapa2 = valorEtapa1 - desconto
```

#### ETAPA 3: Desconto à Vista
```
Condições OBRIGATÓRIAS (todas devem ser verdadeiras):
- parcelamento === 1 (pagamento em 1 única parcela)
- parcelamentoEstendido === 0 (sem parcelas estendidas)
- isRaffleEnabled === false (sem arrecadação alternativa)
- Regra dentro do período válido

Se tipoDesconto = "PERCENTUAL":
  Desconto = valorEtapa2 × (percentualDesconto / 100)

Se tipoDesconto = "VALOR":
  Desconto = valorDesconto configurado

valorFinal = valorEtapa2 - desconto
```

**Valor Final do Plano com Descontos**:
```
getValorPlanoComDesconto() = valorFinal (após todas as 3 etapas)
```

---

### 2.3 Cálculo de Arrecadação Alternativa (Rifas/Semestres)

**Condições de Ativação**:
- `isRaffleEnabled === true` (toggle habilitado pelo usuário)
- `selectedPlan?.financeiro?.valorTotalArrecadacao > 0`
- `dataPrimeiraParcela` preenchida
- Se múltiplas parcelas: `diaVencimento` preenchido

**Fórmula**:
```
1. Filtrar semestres válidos:
   semestresValidos = semestres onde dataVencimento >= dataPrimeiraParcela

2. Calcular proporção:
   proporção = quantidade_semestresValidos / total_semestres

3. Calcular valor proporcional:
   valorArrecadacao = proporção × valorTotalArrecadacao

Função: getValorArrecadacaoValida()
```

**Desconto Mensal Aplicado**:
```
descontoMensal = valorArrecadacao / totalParcelas

Este valor é SUBTRAÍDO de cada parcela mensal.
```

---

### 2.4 Cálculo de Taxa de Antecipação (Parcelamento Estendido)

**Condições**:
- `isParcelamentoEstendidoAtivo === true`
- `taxaAntecipacao > 0` configurada no plano

**Fórmula - Progressão Aritmética**:
```
1. Calcular valor base da parcela (sem taxa):
   valorParcelaSemTaxa = valorPlano / (parcelamento + parcelamentoEstendido)

2. Calcular fator de progressão:
   fatorProgressao = qtdParcelasEstendidas × (qtdParcelasEstendidas + 1) / 2

3. Calcular taxa total de antecipação:
   taxaAntecipacaoTotal = valorParcelaSemTaxa × (taxaAntecipacao / 100) × fatorProgressao

Taxa Total aplicada sobre as parcelas estendidas.
```

**Explicação**:
- A taxa aumenta progressivamente a cada parcela estendida
- Fórmula simula custo crescente de antecipação de crédito

---

### 2.5 Cálculo de Taxa de Administração do Cartão

**Tipos de Cobrança Configuráveis**:

#### 1. Valor Fixo
```
totalTaxas = valorFixo configurado para quantidade de parcelas
```

#### 2. Valor Percentual
```
totalTaxas = (valorParcelaBase × qtdParcelasEstendidas) × (percentual / 100)
```

#### 3. Valor Fixo + Percentual (Híbrido)
```
totalTaxas = valorFixo + [(valorParcelaBase × qtdParcelasEstendidas) × (percentual / 100)]
```

**Rateio da Taxa**:
```
taxaPorParcela = totalTaxas / (parcelamento + parcelamentoEstendido)

A taxa é DISTRIBUÍDA igualmente entre TODAS as parcelas (normais + estendidas).
```

---

### 2.6 Cálculo do Valor da Parcela

**Fórmula Principal**:
```
1. Obter valor total com descontos:
   valorTotal = getValorPlanoComDesconto()

2. Obter valor de arrecadação (se aplicável):
   valorArrecadacao = getValorArrecadacaoValida()

3. Calcular valor líquido a parcelar:
   valorComDesconto = valorTotal - valorArrecadacao

4. Calcular total de parcelas:
   totalParcelas = parcelamento +
                   (isParcelamentoEstendidoAtivo ? parcelamentoEstendido : 0)

5. Calcular valor unitário da parcela:
   valorParcela = ARREDONDAR(valorComDesconto / totalParcelas)
```

---

### 2.7 Cálculo de Máximo de Parcelas Disponíveis

**Algoritmo**:
```
Se NÃO tem dataUltimaParcela configurada:
  return numeroMaximoParcelas (padrão: 60)

Se TEM dataUltimaParcela:
  primeiraParcela = dataPrimeiraParcela OU dataSimulacao OU dataAtual

  diferençaAnos = dataUltimaParcela.ano - primeiraParcela.ano
  diferençaMeses = dataUltimaParcela.mês - primeiraParcela.mês

  mesesDisponiveis = (diferençaAnos × 12) + diferençaMeses + 1

  return MÍNIMO(mesesDisponiveis, numeroMaximoParcelas)
```

**Impacto Financeiro**:
- Limita quantidade máxima de parcelamento baseado em data final
- Garante que última parcela não ultrapasse deadline do plano

---

## 3. REGRAS DE NEGÓCIO E VALIDAÇÕES

### 3.1 Validações de Data

#### Data da Primeira Parcela
- **Obrigatória**: Sim
- **Validações**:
  - Deve estar entre `dataInicial` e `dataFinal` configuradas
  - Não pode ser sábado ou domingo
  - Não pode ser feriado bancário
  - Se data configurada já passou, usa data atual como referência
  - Limite: até `limiteDiasAdesao` dias após primeira parcela configurada

#### Dia de Vencimento
- **Obrigatória**: Apenas se `parcelamento > 1`
- **Validações**:
  - Se não especificado, usa o dia da primeira parcela
  - Automaticamente ajustado para próximo dia útil se cair em feriado/fim de semana

---

### 3.2 Regras de Parcelamento

#### Parcelamento Normal
- **Mínimo**: 1 parcela
- **Máximo**: Calculado dinamicamente baseado em `dataUltimaParcela`
- **Comportamento**: Se seleção exceder máximo, resetar automaticamente para máximo

#### Parcelamento Estendido
- **Habilitação**: Apenas quando `parcelamento === maxParcelas` (esgotou parcelas normais)
- **Requisito**: `cartaoParcelamentoEstendido === true` na configuração
- **Comportamento**:
  - Reset para 0 se reduzir quantidade de parcelamento normal
  - Auto-ajuste para máximo quando aumentar de 0

---

### 3.3 Regras de Aplicação de Descontos

#### 1. Desconto por Tipo de Usuário
- **Aplicação**: Sempre (primeira etapa)
- **Modalidades**: Valor fixo OU percentual
- **Fonte**: `loteAtivo.descontos[tipoUsuario]`

#### 2. Desconto de Adesão
- **Aplicação**: Apenas dentro do período configurado
- **Etapa**: Após desconto de tipo de usuário (segunda etapa)
- **Modalidades**: Valor fixo OU percentual
- **Fonte**: Configuração do plano financeiro

#### 3. Desconto à Vista
- **Condições RÍGIDAS** (todas obrigatórias):
  - `parcelamento === 1` (1 parcela)
  - `parcelamentoEstendido === 0` (sem cartão)
  - `isRaffleEnabled === false` (sem arrecadação)
  - Data atual dentro do período válido da regra
- **Etapa**: Última (terceira etapa)
- **Modalidades**: Valor fixo OU percentual

---

### 3.4 Regras de Arrecadação Alternativa

#### Habilitação
- **Toggle**: `isRaffleEnabled === true`
- **Valor configurado**: `valorTotalArrecadacao > 0`
- **Datas obrigatórias**:
  - `dataPrimeiraParcela` preenchida
  - `diaVencimento` preenchido (se múltiplas parcelas)

#### Semestres Válidos
- **Critério**: `dataVencimento >= dataPrimeiraParcela`
- **Cálculo**: Proporcional por quantidade de semestres válidos

#### Impacto Financeiro
- **Reduz valor da parcela mensal**
- **Cria parcelas separadas de arrecadação** (com números negativos)

---

### 3.5 Ajuste Automático de Datas para Dias Úteis

**Algoritmo**:
```
Função ajustarParaDiaUtil(data):
  enquanto data for fim de semana OU feriado bancário:
    avançar data em 1 dia
  retornar data ajustada
```

**Aplicado em**:
- Data da primeira parcela
- Data de cada parcela subsequente
- Data de vencimento de arrecadações

---

### 3.6 Ordem de Aplicação dos Cálculos

```
Sequência Financeira:

1. Valor Base do Lote
   ↓
2. + Correção Monetária (se aplicável)
   ↓
3. - Desconto por Tipo de Usuário (ETAPA 1)
   ↓
4. - Desconto de Adesão (ETAPA 2, se dentro do período)
   ↓
5. - Desconto à Vista (ETAPA 3, se condições atendidas)
   ↓
6. = VALOR FINAL DO PLANO

Após determinação do valor final:

7. - Arrecadação Alternativa (se ativada)
   ↓
8. = VALOR LÍQUIDO A PAGAR
   ↓
9. ÷ Quantidade de Parcelas
   ↓
10. + Taxa de Antecipação (se parcelamento estendido)
    ↓
11. + Taxa de Administração do Cartão (se parcelamento estendido)
    ↓
12. = VALOR DA PARCELA FINAL
```

---

## 4. GERAÇÃO DO CRONOGRAMA DE PARCELAS

### 4.1 Estrutura de Cada Parcela

```
Objeto Parcela:
{
  numero: inteiro,           // 1, 2, 3... ou -1, -2... (arrecadação)
  data: Date,               // Data ajustada para dia útil
  valor: decimal,           // Valor em reais
  tipo: string,             // 'primeira', 'mensal', 'ultima', 'estendida', 'arrecadacao'
  categoria: string,        // 'normal', 'estendida', 'arrecadacao'
  descricao: string         // Descrição adicional (usado em arrecadação)
}
```

### 4.2 Algoritmo de Geração do Cronograma

#### ETAPA 1: Primeira Parcela
```
- Número: 1
- Data: dataPrimeiraParcela (ajustada para dia útil)
- Valor: valorParcela calculado
- Tipo: 'primeira'
- Categoria: 'normal'
```

#### ETAPA 2: Parcelas Normais (2 até quantidade selecionada)
```
Para cada parcela de 2 até parcelamento:
  - Número: 2, 3, 4...
  - Data: última data + 1 mês, usando diaVencimento
  - Se ultrapassar dataUltimaParcela: ajustar data
  - Ajustar para dia útil
  - Valor: valorParcela
  - Tipo: 'mensal' (ou 'ultima' se for a última)
  - Categoria: 'normal'
```

#### ETAPA 3: Parcelas Estendidas (se ativas)
```
Para cada parcela estendida:
  - Número: continua sequência (ex: 61, 62...)
  - Data: mesma data da última parcela normal
  - Valor: valorParcela + taxas rateadas
  - Tipo: 'estendida'
  - Categoria: 'estendida'
```

#### ETAPA 4: Parcelas de Arrecadação (se ativas)
```
Para cada semestre válido:
  - Número: -1, -2, -3... (negativo para identificação)
  - Data: dataVencimento do semestre (ajustada para dia útil)
  - Valor: proporção do valorTotalArrecadacao / qtd_semestres_validos
  - Tipo: 'arrecadacao'
  - Categoria: 'arrecadacao'
  - Descrição: nome do semestre
```

---

## 5. PERSISTÊNCIA E FLUXO DE DADOS

### 5.1 Dados Salvos na Tabela `contratos`

```
Campos Financeiros Principais:

- valor_total: getValorPlanoComDesconto()
  (Valor final após todos os descontos)

- valor_arrecadacao_alternativa: getValorArrecadacaoValida()
  (Valor total de rifas/semestres aplicável)

- valor_liquido: valor_total - valor_arrecadacao_alternativa
  (Valor efetivamente parcelado)

- numero_parcelas: quantidade de parcelas normais

- valor_parcela: valor unitário de cada parcela

- data_primeira_parcela: data selecionada pelo usuário

- dia_vencimento: dia do mês para vencimentos subsequentes

- tem_parcelamento_estendido: boolean

- numero_parcelas_estendido: quantidade de parcelas via cartão

- valor_parcela_estendido: valor unitário das parcelas estendidas

- tem_arrecadacao_alternativa: boolean

- valor_mensal_arrecadacao: desconto aplicado mensalmente

Campos Auxiliares:
- user_id, turma_id, plano_id, lote_id, categoria_id
- numero_contrato (gerado automaticamente)
- status: 'ativo'
- assinado: false (inicialmente)
- dados_configuracao: JSON com plano, eventos, itens gerais e cronograma
```

### 5.2 Dados Salvos na Tabela `parcelas`

```
Para cada item no cronograma:

- contrato_id: referência ao contrato criado
- user_id: usuário responsável
- numero_parcela: número sequencial (positivo ou negativo)
- tipo_parcela: 'normal' | 'estendido' | 'arrecadacao'
- valor: valor calculado da parcela
- data_vencimento: data formatada para SQL (YYYY-MM-DD)
- status: 'pendente'
- desconto_arrecadacao: valor do desconto mensal (se aplicável)
- valor_arrecadacao_mes: valor de arrecadação do mês (se aplicável)
- observacoes: informações adicionais
```

---

## 6. CASOS ESPECIAIS E CONSIDERAÇÕES

### 6.1 Limite de Dias para Adesão

Se configurado `limiteDiasAdesao`:
- Calcula data limite = primeira parcela configurada + limite de dias
- Valida se `dataPrimeiraParcela` não ultrapassa esse limite
- Exibe mensagem de erro se ultrapassar

### 6.2 Ajuste de Data Última Parcela

Quando `dataUltimaParcela` está configurada:
- Limita quantidade máxima de parcelas
- Ajusta automaticamente cronograma para não ultrapassar
- Se usuário tentar selecionar mais parcelas, sistema reseta para máximo

### 6.3 Proteção de Dados Sensíveis

Antes de mostrar valores finais:
- Requer `dataPrimeiraParcela` preenchida
- Se múltiplas parcelas: requer `diaVencimento` preenchido
- Evita cálculos incompletos ou incorretos

### 6.4 Arredondamentos

- Valores de parcelas: arredondados para 2 casas decimais
- Diferenças de arredondamento: ajustadas na última parcela
- Garante que soma das parcelas = valor total

---

## 7. INTEGRAÇÕES COM BANCO DE DADOS

### Consultas Realizadas (READ)

1. **Buscar Lote Ativo**
   - Tabela: `lotes`
   - Filtros: plano_id, disponivel=true, status_venda='ativo'
   - Retorna: valor, descontos

2. **Buscar Turma**
   - Tabela: `turmas`
   - Filtro: código da turma

3. **Buscar Plano**
   - Tabela: `planos`
   - Retorna: categoria_id, configurações financeiras

4. **Buscar Categoria Principal**
   - Tabela: `categorias`
   - Filtro: turma_id, is_principal=true

5. **Buscar Usuário Autenticado**
   - Supabase Auth: getUser()

### Inserções Realizadas (CREATE)

1. **Criar Contrato**
   - Tabela: `contratos`
   - Retorna: ID do contrato criado

2. **Criar Parcelas (Lote)**
   - Tabela: `parcelas`
   - Inserção em batch de todas as parcelas

### Deleções (DELETE)

1. **Deletar Contrato (em caso de erro)**
   - Tabela: `contratos`
   - Ocorre se falhar criação de parcelas

---

## 8. RESUMO EXECUTIVO PARA FINANCEIRO

### Objetivo da Página
Permitir que o usuário:
1. Selecione datas de pagamento
2. Escolha quantidade de parcelas (normais + estendidas)
3. Visualize cálculo completo de descontos
4. Confirme cronograma de pagamentos
5. Gere contrato e parcelas no sistema

### Responsabilidades Financeiras
1. **Cálculo de Valor Final**: Aplicação sequencial de correção monetária e descontos
2. **Geração de Cronograma**: Criação de calendário de vencimentos com ajustes para dias úteis
3. **Persistência de Dados**: Salvamento de contrato e parcelas no banco de dados
4. **Validações**: Garantia de integridade de datas, valores e regras de negócio
5. **Transparência**: Exibição detalhada de todos os descontos e taxas aplicados

### Fluxo de Trabalho
```
Usuário seleciona:
  → Tipo de usuário (comissão, aderente, etc.)
  → Data da primeira parcela
  → Quantidade de parcelas
  → Dia de vencimento (se múltiplas parcelas)
  → Arrecadação alternativa (opcional)
  → Parcelamento estendido (opcional)
     ↓
Sistema calcula:
  → Correção monetária
  → Descontos em cascata (tipo usuário → adesão → à vista)
  → Valor líquido (após arrecadação)
  → Valor unitário da parcela
  → Taxas de cartão (se aplicável)
     ↓
Sistema gera:
  → Cronograma completo de parcelas
  → Ajustes para dias úteis
  → Parcelas de arrecadação (se aplicável)
     ↓
Sistema persiste:
  → Contrato com todos os dados financeiros
  → Parcelas individuais com datas e valores
     ↓
Usuário é redirecionado para:
  → Página de contratação para assinatura
```

### Pontos de Atenção Financeira

1. **Descontos Escalonados**: Cada desconto é aplicado sobre o resultado do anterior (não sobre valor original)
2. **Arrecadação Proporcional**: Rifas/semestres são calculadas proporcionalmente aos semestres válidos
3. **Taxa Progressiva**: Taxa de antecipação aumenta progressivamente em parcelamento estendido
4. **Ajustes de Data**: Todas as datas são ajustadas para dias úteis automaticamente
5. **Validação de Limite**: Sistema impede parcelamento além de data limite configurada
6. **Proteção de Dados**: Valores só são exibidos com datas obrigatórias preenchidas

---

## 9. GLOSSÁRIO DE TERMOS

- **Lote Ativo**: Grupo de valores e descontos disponível para venda
- **Parcelamento Normal**: Parcelas mensais padrão (até máximo configurado)
- **Parcelamento Estendido**: Parcelas adicionais via cartão com taxa de antecipação
- **Arrecadação Alternativa**: Sistema de rifas/semestres que reduz valor da parcela mensal
- **Correção Monetária**: Atualização do valor base por juros simples ou compostos
- **Desconto em Cascata**: Aplicação sucessiva de descontos (um sobre o outro)
- **Dia Útil**: Dia que não é sábado, domingo ou feriado bancário
- **Taxa de Antecipação**: Taxa cobrada sobre parcelas estendidas (progressiva)
- **Taxa de Administração**: Taxa do cartão de crédito (fixa, percentual ou híbrida)

---

**Fim do Documento**
