# Exemplo de Contrato com Variáveis

Este documento mostra como deve ser formatado o contrato PDF original para que as variáveis sejam substituídas corretamente.

## 📄 Modelo de Contrato (para criar o PDF)

```
═══════════════════════════════════════════════════════════════════
                    CONTRATO DE PRESTAÇÃO DE SERVIÇOS
═══════════════════════════════════════════════════════════════════


CONTRATANTE

Pelo presente instrumento particular de Contrato de Prestação de Serviços,
o(a) CONTRATANTE @NOME_CLIENTE, portador(a) do CPF nº @CPFCNPJ_CLIENTE,
residente e domiciliado(a) à @RUA_CLIENTE, nº @NUM_CLIENTE, @COMPL_CLIENTE,
bairro @BAIRRO_CLIENTE, @CIDADE_CLIENTE/@UF_CLIENTE, telefone @TELCEL_CLIENTE,
doravante denominado simplesmente CONTRATANTE.


CONTRATADA

E de outro lado, HUB EDUCACIONAL LTDA, inscrita no CNPJ sob o nº XX.XXX.XXX/0001-XX,
com sede na Rua Exemplo, 1000, Centro, Blumenau/SC, doravante denominada
simplesmente CONTRATADA.


OBJETO DO CONTRATO

O presente contrato tem como objeto a prestação dos seguintes serviços educacionais:

@TODOS_PRODUTOS_SERVICOS


CLÁUSULA PRIMEIRA - DO VALOR E FORMA DE PAGAMENTO

1.1. O CONTRATANTE pagará à CONTRATADA o valor de @DESCR_TPCBR, sendo:

    - Forma de pagamento: Recorrência mensal
    - Data de vencimento: @VENC_CLIENTE_CBR

1.2. Em caso de atraso no pagamento, incidirão multa de 2% (dois por cento)
e juros de mora de 1% (um por cento) ao mês, calculados pro rata die.


CLÁUSULA SEGUNDA - DA VIGÊNCIA

2.1. O presente contrato entra em vigor na data de sua assinatura, em
@DT_ADESAO_WEB, e terá vigência conforme plano contratado.


CLÁUSULA TERCEIRA - DAS OBRIGAÇÕES DA CONTRATADA

3.1. Prestar os serviços educacionais conforme descrito no objeto do contrato.

3.2. Disponibilizar acesso à plataforma digital do HUB Portal.

3.3. Fornecer suporte técnico durante o horário comercial.


CLÁUSULA QUARTA - DAS OBRIGAÇÕES DO CONTRATANTE

4.1. Efetuar os pagamentos nas datas de vencimento estabelecidas.

4.2. Utilizar os serviços contratados de forma responsável e conforme
os termos de uso da plataforma.

4.3. Manter seus dados cadastrais atualizados.


CLÁUSULA QUINTA - DO CANCELAMENTO

5.1. O CONTRATANTE poderá solicitar o cancelamento do contrato a qualquer
momento, mediante comunicação formal à CONTRATADA.

5.2. O cancelamento será efetivado ao final do período já pago, sem
reembolso de valores proporcionais.


CLÁUSULA SEXTA - DA PROTEÇÃO DE DADOS

6.1. As partes comprometem-se a tratar os dados pessoais em conformidade
com a Lei Geral de Proteção de Dados (LGPD - Lei 13.709/2018).

6.2. Os dados do CONTRATANTE serão utilizados exclusivamente para a
execução deste contrato e conforme Política de Privacidade disponível
no site da CONTRATADA.


CLÁUSULA SÉTIMA - DO FORO

7.1. Fica eleito o foro da comarca de @CIDADE_CLIENTE/@UF_CLIENTE para
dirimir quaisquer dúvidas ou litígios oriundos do presente contrato,
com renúncia expressa de qualquer outro, por mais privilegiado que seja.


E, por estarem assim justos e contratados, firmam o presente instrumento.


@CIDADE_CLIENTE/@UF_CLIENTE, @DT_ADESAO_WEB


_________________________________
@NOME_CLIENTE
CPF: @CPFCNPJ_CLIENTE
CONTRATANTE


_________________________________
HUB EDUCACIONAL LTDA
CNPJ: XX.XXX.XXX/0001-XX
CONTRATADA
```

## 🔄 Após Processamento

Com os dados de exemplo:
- Nome: João da Silva Santos
- CPF: 123.456.789-00
- Endereço: Rua das Flores, 123, Apto 45, Centro, Blumenau/SC
- Telefone: (47) 99999-8888
- Plano: Plano Premium - Acesso completo à plataforma
- Valor: R$ 299,00/mês
- Vencimento: Todo dia 10
- Data: 24/10/2025

O contrato ficaria assim:

```
═══════════════════════════════════════════════════════════════════
                    CONTRATO DE PRESTAÇÃO DE SERVIÇOS
═══════════════════════════════════════════════════════════════════


CONTRATANTE

Pelo presente instrumento particular de Contrato de Prestação de Serviços,
o(a) CONTRATANTE João da Silva Santos, portador(a) do CPF nº 123.456.789-00,
residente e domiciliado(a) à Rua das Flores, nº 123, Apto 45,
bairro Centro, Blumenau/SC, telefone (47) 99999-8888,
doravante denominado simplesmente CONTRATANTE.


CONTRATADA

E de outro lado, HUB EDUCACIONAL LTDA, inscrita no CNPJ sob o nº XX.XXX.XXX/0001-XX,
com sede na Rua Exemplo, 1000, Centro, Blumenau/SC, doravante denominada
simplesmente CONTRATADA.


OBJETO DO CONTRATO

O presente contrato tem como objeto a prestação dos seguintes serviços educacionais:

Plano Premium - Acesso completo à plataforma, incluindo todos os cursos,
eventos ao vivo, mentorias em grupo e conteúdo exclusivo.


CLÁUSULA PRIMEIRA - DO VALOR E FORMA DE PAGAMENTO

1.1. O CONTRATANTE pagará à CONTRATADA o valor de R$ 299,00/mês, sendo:

    - Forma de pagamento: Recorrência mensal
    - Data de vencimento: Todo dia 10

[...]

@CIDADE_CLIENTE/@UF_CLIENTE, 24/10/2025


_________________________________
João da Silva Santos
CPF: 123.456.789-00
CONTRATANTE
```

## 📝 Dicas para Criar o PDF

### 1. Use um editor de PDF ou Word

Crie o documento no Word com as variáveis `@VARIAVEL` e exporte como PDF:

1. Escreva todo o contrato no Word
2. Coloque as variáveis com `@` onde precisar
3. Salve como PDF
4. Faça upload no backoffice

### 2. Formatação Recomendada

- **Títulos em MAIÚSCULAS** (serão automaticamente formatados como `<h3>`)
- **Parágrafos separados** por linha em branco (melhora a legibilidade)
- **Alinhamento justificado** (fica mais profissional)
- **Fonte serifada** como Times New Roman ou Georgia

### 3. Validação

Antes de fazer upload, verifique:

✅ Todas as variáveis começam com `@`
✅ Não há espaços entre `@` e o nome da variável
✅ Os nomes das variáveis estão em MAIÚSCULAS
✅ As variáveis estão escritas corretamente (sem typos)

## 🎨 Visualização no Sistema

Quando o contrato for processado, o sistema irá:

1. **Extrair o texto** do PDF preservando a estrutura
2. **Identificar títulos** (texto em maiúsculas)
3. **Identificar parágrafos** (texto normal)
4. **Substituir variáveis** pelos dados reais
5. **Aplicar estilos CSS** para melhor visualização:
   - Fonte serifada (Georgia)
   - Espaçamento adequado
   - Títulos destacados
   - Parágrafos com recuo

## 🔍 Exemplo Visual da Transformação

**Antes (no PDF):**
```
O CONTRATANTE @NOME_CLIENTE, CPF @CPFCNPJ_CLIENTE,
residente em @CIDADE_CLIENTE/@UF_CLIENTE...
```

**Depois (visualizado no sistema):**
```
O CONTRATANTE João da Silva Santos, CPF 123.456.789-00,
residente em Blumenau/SC...
```

## 💾 Salvar o Contrato Processado

O usuário poderá:

1. **Visualizar** o contrato com dados personalizados (HTML)
2. **Baixar** o contrato processado (HTML)
3. **Baixar** o contrato original (PDF)
4. **Assinar** digitalmente após revisão

---

**Template disponível em**: [Google Docs](link-do-template) ou [Word Download](link-do-word)
