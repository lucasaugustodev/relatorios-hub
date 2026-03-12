# Configuração de Variáveis em Contratos PDF

Este documento explica como configurar contratos PDF com variáveis que serão substituídas automaticamente pelos dados do cliente durante o processo de contratação.

## 📋 Variáveis Disponíveis

As variáveis devem ser marcadas no contrato PDF com o prefixo `@`. Durante o processamento, essas variáveis serão substituídas pelos dados reais do cliente.

### Dados do Cliente (origem: `/portal` - cadastro)

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `@NOME_CLIENTE` | Nome completo do cliente | João da Silva Santos |
| `@CPFCNPJ_CLIENTE` | CPF do cliente | 123.456.789-00 |
| `@TELCEL_CLIENTE` | Telefone celular do cliente | (47) 99999-8888 |

### Dados de Endereço (origem: `/contratacao`)

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `@RUA_CLIENTE` | Logradouro do endereço | Rua das Flores |
| `@NUM_CLIENTE` | Número do endereço | 123 |
| `@COMPL_CLIENTE` | Complemento do endereço | Apto 45 |
| `@BAIRRO_CLIENTE` | Bairro | Centro |
| `@CIDADE_CLIENTE` | Cidade | Blumenau |
| `@UF_CLIENTE` | Estado (UF) | SC |

### Dados do Plano (origem: seleção de plano)

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `@TODOS_PRODUTOS_SERVICOS` | Nome completo do plano, itens, eventos e descrição | Plano Premium - Inclui... |
| `@DESCR_TPCBR` | Descrição do valor do plano selecionado | R$ 299,00/mês |

### Dados de Pagamento (origem: `/selecao-parcelamento`)

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `@VENC_CLIENTE_CBR` | Data de vencimento escolhida | Todo dia 10 |

### Dados Automáticos

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `@DT_ADESAO_WEB` | Data atual (data de adesão) | 24/10/2025 |

## 🔧 Como Funciona

### 1. Preparação do PDF

Ao criar o contrato PDF no backoffice, use as variáveis no texto:

```
CONTRATO DE PRESTAÇÃO DE SERVIÇOS

Pelo presente instrumento particular, de um lado @NOME_CLIENTE,
portador do CPF @CPFCNPJ_CLIENTE, residente e domiciliado à
@RUA_CLIENTE, nº @NUM_CLIENTE, @COMPL_CLIENTE, bairro @BAIRRO_CLIENTE,
@CIDADE_CLIENTE/@UF_CLIENTE, doravante denominado CONTRATANTE...

O presente contrato tem como objeto a prestação dos seguintes serviços:
@TODOS_PRODUTOS_SERVICOS

Valor: @DESCR_TPCBR
Vencimento: @VENC_CLIENTE_CBR

Firmado em @CIDADE_CLIENTE/@UF_CLIENTE, aos @DT_ADESAO_WEB.
```

### 2. Processamento Automático

Durante a contratação:

1. O sistema **extrai o texto do PDF** usando `pdfjs-dist`
2. **Substitui todas as variáveis** pelos dados reais do cliente
3. **Formata o resultado como HTML** para visualização
4. **Exibe o contrato processado** com os dados personalizados

### 3. Componentes Utilizados

#### `ContractViewer`

Componente principal para visualização de contratos com variáveis substituídas:

```tsx
import ContractViewer from '@/components/ContractViewer';

<ContractViewer
  contractUrl={contractFile.url}
  variableData={{
    nomeCliente: "João da Silva",
    cpfCnpjCliente: "123.456.789-00",
    ruaCliente: "Rua das Flores",
    numCliente: "123",
    // ... outros dados
  }}
  fileName="contrato.pdf"
/>
```

#### Hook `useContractFile`

Busca contratos e prepara dados de variáveis:

```tsx
const {
  contractFiles,
  loading,
  error,
  prepareVariableData
} = useContractFile(planId);

// Preparar dados do formulário
const variableData = prepareVariableData({
  fullName: "João da Silva",
  cpf: "123.456.789-00",
  street: "Rua das Flores",
  number: "123",
  // ... outros dados
});
```

## 🎯 Exemplo de Uso Completo

### Na página de Contratação

```tsx
import { useState } from 'react';
import ContractViewer from '@/components/ContractViewer';
import { useContractFile } from '@/hooks/useContractFile';

function Contratacao() {
  const [formData, setFormData] = useState({
    fullName: '',
    cpf: '',
    telefone: '',
    street: '',
    number: '',
    complement: '',
    neighborhood: '',
    city: '',
    state: '',
  });

  const {
    contractFiles,
    prepareVariableData
  } = useContractFile(selectedPlanId);

  const variableData = prepareVariableData(formData);

  return (
    <ContractViewer
      contractUrl={contractFiles[0]?.url}
      variableData={variableData}
      fileName={contractFiles[0]?.nome}
    />
  );
}
```

## 📝 Observações Importantes

1. **Todas as variáveis devem começar com `@`** no PDF original
2. Se uma variável não for substituída, será exibida como `[DADO NÃO INFORMADO]`
3. O contrato processado pode ser baixado em formato HTML
4. O PDF original também fica disponível para download
5. O componente é responsivo e funciona em mobile/desktop

## 🔐 Segurança

- As variáveis são substituídas apenas no momento da visualização
- O PDF original permanece intacto no storage
- Cada usuário vê apenas seus próprios dados
- A substituição acontece no lado do cliente (navegador)

## 🚀 Próximos Passos

1. **Upload do contrato no Backoffice**: Use o componente `ContractUpload` para fazer upload do PDF com variáveis
2. **Teste as variáveis**: Certifique-se de que todas as variáveis estão corretas no PDF
3. **Visualize no processo de contratação**: O contrato será automaticamente processado e exibido com os dados do cliente

---

**Desenvolvido para Hub Portal** - Sistema de substituição automática de variáveis em contratos
