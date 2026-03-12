# Exemplo de Integração do ContractViewer na Página de Contratação

Este documento mostra como integrar o novo componente `ContractViewer` na página [Contratacao.tsx](src/pages/Contratacao.tsx).

## 🔄 Mudanças Necessárias

### 1. Importar o novo componente

Na linha 11 de [Contratacao.tsx](src/pages/Contratacao.tsx#L11), adicionar:

```tsx
import PDFViewer from "@/components/PDFViewer";
import ContractViewer from "@/components/ContractViewer"; // NOVO
```

### 2. Usar o hook atualizado

Na linha 147-155, o hook `useContractFile` agora retorna também `prepareVariableData`:

```tsx
const {
  contractContent,
  contractFile,
  contractFiles,
  loading: contractFileLoading,
  error: contractFileError,
  requiresOTP,
  prepareVariableData // NOVO
} = useContractFile(selectedPlanId);
```

### 3. Substituir o PDFViewer pelo ContractViewer

Na seção de visualização do contrato (linhas 1151-1171), substituir:

**ANTES:**
```tsx
) : contractFiles[currentContractIndex]?.url ? (
  // Visualizador de PDF do contrato atual
  <PDFViewer
    url={contractFiles[currentContractIndex].url}
    fileName={contractFiles[currentContractIndex].nome}
  />
) : (
```

**DEPOIS:**
```tsx
) : contractFiles[currentContractIndex]?.url ? (
  // Visualizador de contrato com variáveis substituídas
  <ContractViewer
    contractUrl={contractFiles[currentContractIndex].url}
    variableData={prepareVariableData({
      // Dados do cliente
      fullName: selectedOption === "self" ? userEmail.split('@')[0] : fullName,
      cpf: selectedOption === "self" ? "" : cpf,
      telefone: selectedOption === "self" ? "" : telefone,

      // Dados de endereço
      street,
      number,
      complement,
      neighborhood,
      city,
      state,

      // Dados do plano (você pode buscar do localStorage ou state)
      planDescription: "Plano Selecionado", // TODO: Buscar descrição real do plano
      planValue: "R$ 299,00/mês", // TODO: Buscar valor real do plano

      // Data de vencimento (se já tiver sido selecionada)
      dueDate: "Todo dia 10", // TODO: Buscar vencimento do localStorage
    })}
    fileName={contractFiles[currentContractIndex].nome}
  />
) : (
```

## 🎯 Código Completo da Seção de Contrato

Substitua a seção completa de visualização do contrato (linhas 1139-1172):

```tsx
<div className="w-full max-w-md lg:max-w-[800px] space-y-6 mb-6 lg:mb-8">
  {/* Contract Content */}
  {contractFileLoading ? (
    <div className="flex items-center justify-center py-8">
      <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
      <p className="ml-3 text-gray-600">Carregando contrato...</p>
    </div>
  ) : contractFileError ? (
    <div className="text-center py-8">
      <p className="text-red-600 mb-2">Erro ao carregar contrato</p>
      <p className="text-sm text-gray-500">{contractFileError}</p>
    </div>
  ) : contractFiles[currentContractIndex]?.url ? (
    // Visualizador de contrato com variáveis substituídas
    <ContractViewer
      contractUrl={contractFiles[currentContractIndex].url}
      variableData={prepareVariableData({
        // Dados do cliente
        fullName: selectedOption === "self" ? userEmail.split('@')[0] : fullName,
        cpf: selectedOption === "self" ? "" : cpf,
        telefone: selectedOption === "self" ? "" : telefone,

        // Dados de endereço
        street,
        number,
        complement,
        neighborhood,
        city,
        state,

        // Dados do plano
        planDescription: getPlanDescription(), // Criar função helper
        planValue: getPlanValue(), // Criar função helper

        // Data de vencimento
        dueDate: getDueDate(), // Criar função helper
      })}
      fileName={contractFiles[currentContractIndex].nome}
      maxHeight="500px"
    />
  ) : (
    // Fallback para contrato de template (texto)
    <div className="bg-white border border-gray-200 rounded-lg p-6 max-h-96 overflow-y-auto">
      <h3 className="text-lg font-bold text-gray-900 mb-4">
        {contractFiles[currentContractIndex]?.nome || 'CONTRATO DE PRESTAÇÃO DE SERVIÇOS'}
      </h3>
      <div className="text-sm text-gray-700 space-y-4 whitespace-pre-line">
        {replaceContractVariables(contractContent, {
          nomeCompleto: fullName,
          cpf: cpf,
          dataNascimento: dataNascimento,
          genero: genero
        })}
      </div>
    </div>
  )}

  {/* Sign Button */}
  <button
    onClick={handleContractSign}
    disabled={contractLoading}
    className="w-full bg-blue-600 text-white text-sm sm:text-base lg:text-[18px] font-normal rounded-[10px] h-12 sm:h-14 lg:h-16 hover:bg-blue-700 transition-colors disabled:opacity-50"
  >
    {contractLoading ? "Processando..." : "Assinar Contrato"}
  </button>

  {/* Back Button */}
  <button
    onClick={handleBack}
    className="w-full text-gray-600 text-sm sm:text-base lg:text-[16px] font-semibold h-12 sm:h-14 lg:h-16 hover:text-gray-800 transition-colors"
  >
    Voltar
  </button>
</div>
```

## 🔧 Funções Helper Recomendadas

Adicione estas funções no componente `Contratacao` para buscar os dados do plano:

```tsx
// Adicionar depois da linha 176 (após useEffect do planId)

// Buscar descrição do plano
const getPlanDescription = () => {
  // TODO: Implementar busca real do plano do Supabase
  const planData = localStorage.getItem('selectedPlanData');
  if (planData) {
    const plan = JSON.parse(planData);
    return `${plan.nome} - ${plan.descricao || ''}`;
  }
  return 'Plano Selecionado';
};

// Buscar valor do plano
const getPlanValue = () => {
  // TODO: Implementar busca real do plano do Supabase
  const planData = localStorage.getItem('selectedPlanData');
  if (planData) {
    const plan = JSON.parse(planData);
    return `R$ ${plan.valor?.toFixed(2) || '0,00'}/mês`;
  }
  return 'Valor a definir';
};

// Buscar data de vencimento
const getDueDate = () => {
  const dueDay = localStorage.getItem('selectedDueDate');
  if (dueDay) {
    return `Todo dia ${dueDay}`;
  }
  return 'A definir';
};
```

## 📦 Instalação de Dependências

Se ainda não tiver instalado, adicione as dependências necessárias:

```bash
npm install pdfjs-dist
npm install --save-dev @types/pdfjs-dist
```

## ✅ Checklist de Implementação

- [ ] Importar `ContractViewer` em [Contratacao.tsx](src/pages/Contratacao.tsx)
- [ ] Atualizar destructuring do `useContractFile` para incluir `prepareVariableData`
- [ ] Substituir `PDFViewer` por `ContractViewer` na seção de visualização
- [ ] Criar funções helper (`getPlanDescription`, `getPlanValue`, `getDueDate`)
- [ ] Salvar dados do plano no localStorage na página de seleção de plano
- [ ] Salvar data de vencimento no localStorage na página de parcelamento
- [ ] Testar com um PDF que contenha as variáveis marcadas com `@`

## 🎨 Resultado Esperado

Ao visualizar o contrato:

1. **PDF é carregado** automaticamente
2. **Texto é extraído** do PDF
3. **Variáveis são substituídas** pelos dados reais:
   - `@NOME_CLIENTE` → "João da Silva Santos"
   - `@RUA_CLIENTE` → "Rua das Flores"
   - `@CIDADE_CLIENTE` → "Blumenau"
   - etc.
4. **Contrato é exibido** como HTML formatado e estilizado
5. **Usuário pode baixar**:
   - PDF original (sem substituições)
   - HTML processado (com dados personalizados)

## ❓ Dúvidas Comuns

**P: E se eu quiser manter o visualizador de PDF original?**
R: Você pode manter ambos e adicionar um toggle para escolher entre "PDF Original" e "Contrato Processado"

**P: As variáveis são case-sensitive?**
R: Sim! Use exatamente `@NOME_CLIENTE` (maiúsculas) no PDF

**P: Posso adicionar novas variáveis?**
R: Sim! Basta:
1. Adicionar no PDF com `@`
2. Adicionar no `ContractVariableData` em [contractVariableReplacer.ts](src/services/contractVariableReplacer.ts)
3. Mapear na função `replaceContractVariables`

---

**Pronto!** Com essas mudanças, seus contratos PDF terão substituição automática de variáveis! 🎉
