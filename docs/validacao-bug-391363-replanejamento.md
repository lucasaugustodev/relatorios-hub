# Validacao Completa — Bug #391363 — Replanejamento Financeiro

**Data:** 2026-03-11
**Ferramenta:** cli-hub-portal-vibe + testes unitarios Node.js + Python
**Status:** CORRIGIDO E VALIDADO

---

## 1. Descricao do Bug

**Titulo:** Valores nao batem (contratado vs a pagar) no replanejamento financeiro
**Causa raiz:** A funcao `calcularRenegociacao()` em `src/services/renegociacaoService.ts` usava `round()` para arredondar o valor de cada parcela. Isso acumulava erros de centavos ao longo de muitas parcelas, resultando em soma das parcelas != valor do contrato.

**Exemplo concreto:**
- Principal: R$ 17.169,85 / 34 parcelas
- Logica antiga: soma = R$ 17.170,00 (diferenca de +R$ 0,15)
- Logica nova: soma = R$ 17.169,85 (diferenca de R$ 0,00)

## 2. Correcao Aplicada

### 2.1 Tres camadas de protecao

**Camada 1 — Truncar em vez de arredondar:**
```typescript
// ANTES (bugado):
const valorParcelaRegular = round(restante / parcelasRestantes, 2);

// DEPOIS (corrigido):
const valorParcelaRegularBruto = restante / parcelasRestantes;
const valorParcelaRegular = Math.floor(valorParcelaRegularBruto * 100) / 100;
```

**Camada 2 — Ajuste da ultima parcela:**
```typescript
let somaGerada = 0;
for (let i = 0; i < numeroParcelas; i++) {
  if (isEntrada) {
    valorParcela = Math.round(entradaFinal * 100) / 100;
  } else if (isUltimaParcela) {
    // Ultima parcela = principal - tudo que ja foi gerado
    valorParcela = Math.round((principalReplanejar - somaGerada) * 100) / 100;
  } else {
    valorParcela = valorParcelaRegular;
  }
  somaGerada += valorParcela;
}
```

**Camada 3 — Validacao antes de salvar no banco:**
```typescript
const somaParcelas = novasParcelas.reduce((acc, p) => acc + parseFloat(p.valor), 0);
const diferencaTotal = Math.abs(somaParcelas - resumo.valorTotal);
if (diferencaTotal > 0.02) {
  throw new Error('Erro de integridade: soma das parcelas nao bate com valor total');
}
```

### 2.2 Correcao de datas (overflow de dia)

Funcao `adicionarMesesComDiaFixo()` em `src/utils/dateUtils.ts`:
```typescript
// JavaScript new Date(2026, 1, 30) = 2 de marco (overflow!)
// Correcao: calcular ultimo dia do mes e usar min(diaFixo, ultimoDia)
const ultimoDiaDoMes = new Date(anoDestino, mesFinal + 1, 0).getDate();
const diaFinal = Math.min(diaFixo, ultimoDiaDoMes);
```

### 2.3 Arquivos modificados

| Arquivo | Alteracao |
|---------|-----------|
| `src/services/renegociacaoService.ts` | `calcularRenegociacao()` — floor + last-parcel + validacao |
| `src/utils/dateUtils.ts` | `adicionarMesesComDiaFixo()` — overflow de dia |
| `src/services/__tests__/renegociacaoService.test.ts` | 22 testes unitarios (Vitest) |
| `vite.config.ts` | Config do Vitest |
| `package.json` | Scripts `test` e `test:watch` |
| `.github/workflows/ci.yml` | Step de teste no CI |

---

## 3. Cenarios de Teste

### 3.1 Resumo Geral

| Metrica | Logica ANTIGA | Logica NOVA |
|---------|:------------:|:-----------:|
| Cenarios corretos (soma = principal) | 3/15 (20%) | **15/15 (100%)** |
| Datas validas (sem duplicatas) | N/A | **15/15 (100%)** |
| Valores positivos (> 0) | N/A | **15/15 (100%)** |
| Testes unitarios Node.js | N/A | **33/33 (100%)** |

### 3.2 Tabela Completa de Cenarios

| # | ID | Descricao | Principal | Parcelas | ANTIGO (soma / diff) | NOVO (soma / diff) | Datas | Status |
|---|-----|-----------|-----------|----------|---------------------|---------------------|-------|--------|
| 1 | BUG-391363-ORIGINAL | Caso original do bug | R$ 17.169,85 | 34 | R$ 17.170,00 / +R$ 0,15 | R$ 17.169,85 / R$ 0,00 | 34/34 | PASS |
| 2 | BUG-391363-DIA30 | Bug original com dia 30 (overflow fev) | R$ 17.169,85 | 34 | R$ 17.170,00 / +R$ 0,15 | R$ 17.169,85 / R$ 0,00 | 34/34 | PASS |
| 3 | BUG-391363-ENTRADA | Bug original com entrada 10% | R$ 17.169,85 | 34 | R$ 17.170,00 / +R$ 0,15 | R$ 17.169,85 / R$ 0,00 | 34/34 | PASS |
| 4 | DIVISAO-INEXATA-1 | R$ 10.000 / 3 (classica) | R$ 10.000,00 | 3 | R$ 9.999,99 / -R$ 0,01 | R$ 10.000,00 / R$ 0,00 | 3/3 | PASS |
| 5 | DIVISAO-INEXATA-2 | R$ 1.000,01 / 7 (irracionais) | R$ 1.000,01 | 7 | R$ 1.000,02 / +R$ 0,01 | R$ 1.000,01 / R$ 0,00 | 7/7 | PASS |
| 6 | MUITAS-PARCELAS | 60 parcelas + entrada R$ 5.000 | R$ 50.000,00 | 60 | R$ 49.999,89 / -R$ 0,11 | R$ 50.000,00 / R$ 0,00 | 60/60 | PASS |
| 7 | PARCELA-UNICA | 1 parcela (so entrada) | R$ 5.000,00 | 1 | R$ 5.000,00 / R$ 0,00 | R$ 5.000,00 / R$ 0,00 | 1/1 | PASS |
| 8 | ENTRADA-ALTA | Entrada R$ 5.000 | R$ 10.000,00 | 10 | R$ 10.000,04 / +R$ 0,04 | R$ 10.000,00 / R$ 0,00 | 10/10 | PASS |
| 9 | ENTRADA-PEQUENA | Entrada R$ 100 < parcela base | R$ 10.000,00 | 10 | R$ 10.000,00 / R$ 0,00 | R$ 10.000,00 / R$ 0,00 | 10/10 | PASS |
| 10 | DIA31-OVERFLOW | Dia 31 (abr/jun = 30 dias) | R$ 10.000,00 | 6 | R$ 10.000,02 / +R$ 0,02 | R$ 10.000,00 / R$ 0,00 | 6/6 | PASS |
| 11 | DIA30-FEVEREIRO | Dia 30 (fev = 28) | R$ 10.000,00 | 12 | R$ 9.999,96 / -R$ 0,04 | R$ 10.000,00 / R$ 0,00 | 12/12 | PASS |
| 12 | DIA29-NAO-BISSEXTO | Dia 29 em 2027 (fev = 28) | R$ 10.000,00 | 3 | R$ 9.999,99 / -R$ 0,01 | R$ 10.000,00 / R$ 0,00 | 3/3 | PASS |
| 13 | VALOR-GRANDE | Valor alto | R$ 123.456,78 | 48 | R$ 123.456,96 / +R$ 0,18 | R$ 123.456,78 / R$ 0,00 | 48/48 | PASS |
| 14 | VALOR-PEQUENO | Parcela minima R$ 50 | R$ 150,00 | 3 | R$ 150,00 / R$ 0,00 | R$ 150,00 / R$ 0,00 | 3/3 | PASS |
| 15 | STRESS-CENTAVOS | Maximo stress de centavos | R$ 99.999,99 | 37 | R$ 99.999,90 / -R$ 0,09 | R$ 99.999,99 / R$ 0,00 | 37/37 | PASS |

### 3.3 Detalhamento — Caso Original do Bug (#1)

**R$ 17.169,85 / 34 parcelas, dia 15, inicio 2026-04-15**

| Parcela | Valor | Data | Tipo |
|---------|-------|------|------|
| 1 | R$ 505,00 | 2026-04-15 | Entrada |
| 2 | R$ 504,99 | 2026-05-15 | Regular |
| 3 | R$ 504,99 | 2026-06-15 | Regular |
| ... | R$ 504,99 | ... | Regular |
| 33 | R$ 504,99 | 2028-12-15 | Regular |
| 34 | R$ 505,17 | 2029-01-15 | Ultima (ajustada) |

- **Entrada inteligente:** max(0, 17169.85/34) = R$ 505,00
- **Parcela regular (truncada):** floor((17169.85 - 505.00) / 33 * 100) / 100 = R$ 504,99
- **Ultima parcela:** 17169.85 - (505.00 + 504.99 * 32) = R$ 505,17
- **Soma total:** 505.00 + (504.99 * 32) + 505.17 = **R$ 17.169,85** (exato)

### 3.4 Detalhamento — Datas com Overflow

#### Cenario DIA31-OVERFLOW (dia 31, inicio 2026-01-31)

| Parcela | Data | Dia | Ajuste |
|---------|------|-----|--------|
| 1 | 2026-01-31 | 31 | - |
| 2 | 2026-02-28 | 28 | Ajustado (fev tem 28 dias) |
| 3 | 2026-03-31 | 31 | - |
| 4 | 2026-04-30 | 30 | Ajustado (abr tem 30 dias) |
| 5 | 2026-05-31 | 31 | - |
| 6 | 2026-06-30 | 30 | Ajustado (jun tem 30 dias) |

#### Cenario DIA30-FEVEREIRO (dia 30, inicio 2026-01-30, 12 parcelas)

| Parcela | Data | Dia | Ajuste |
|---------|------|-----|--------|
| 1 | 2026-01-30 | 30 | - |
| 2 | 2026-02-28 | 28 | Ajustado (fev tem 28 dias) |
| 3 | 2026-03-30 | 30 | - |
| 4-12 | ...30 | 30 | Volta ao dia original |

#### Cenario BUG-391363-DIA30 (dia 30, 34 parcelas, inicio 2026-04-30)

| Parcela | Data | Ajuste |
|---------|------|--------|
| 1 | 2026-04-30 | - |
| 2-10 | 2026-05-30 ... 2027-01-30 | - |
| 11 | 2027-02-28 | Ajustado (fev 2027 tem 28 dias) |
| 12-22 | 2027-03-30 ... 2028-01-30 | - |
| 23 | 2028-02-29 | Ajustado (fev 2028 bissexto tem 29 dias) |
| 24-34 | 2028-03-30 ... 2029-01-30 | - |

**Resultado:** 34 meses unicos, nenhuma parcela perdida por overflow de data.

### 3.5 Entrada Inteligente

| Cenario | Principal | Entrada solicitada | Parcela base (P/N) | Entrada final | Regra |
|---------|-----------|-------------------|---------------------|---------------|-------|
| BUG-391363 | R$ 17.169,85 | R$ 0 | R$ 505,00 | R$ 505,00 | max(0, 505) = 505 |
| ENTRADA-PEQUENA | R$ 10.000 | R$ 100 | R$ 1.000 | R$ 1.000 | max(100, 1000) = 1000 |
| ENTRADA-ALTA | R$ 10.000 | R$ 5.000 | R$ 1.000 | R$ 5.000 | max(5000, 1000) = 5000 |

---

## 4. Testes Unitarios (Node.js / Vitest)

### 4.1 Resultados

```
33 passed, 0 failed

Testes executados:
  - R$ 17.169,85 / 34: 34 parcelas geradas
  - R$ 17.169,85 / 34: soma = principal (diff=-0.0000)
  - R$ 10.000 / 3: 3 parcelas
  - R$ 10.000 / 3: soma = principal
  - R$ 1.000,01 / 7: soma = principal
  - R$ 17.169,85 + entrada R$ 3.000: soma = principal
  - Parcela unica: 1 parcela
  - Parcela unica: valor = 5000
  - R$ 50.000 / 60: 60 parcelas
  - R$ 50.000 / 60: soma = principal
  - Dia 30 fev->28
  - Dia 30 mar volta ao 30
  - Dia 31 abr->30
  - Dia 31 jun->30
  - Dia 29 fev 2027->28
  - Nenhum mes duplicado (12 meses unicos)
  - Entrada inteligente: max(100, 10000/10) = 1000
  - Entrada alta respeitada: 5000
  - Primeira parcela = entrada
  - Bug dia 30: soma = R$ 17.169,85
  - Bug dia 30: 34 meses unicos
  - Todos os valores > 0
  - Contagem parcelas: N=1, 2, 3, 5, 10, 12, 24, 34, 36, 48, 60 (todos OK)
```

### 4.2 Arquivo de teste

`src/services/__tests__/renegociacaoService.test.ts` — 22 testes Vitest cobrindo:
- Integridade: soma das parcelas = principal (6 cenarios)
- Contagem de parcelas (11 valores de N)
- Overflow de datas (4 cenarios com fev, abr, jun)
- Entrada inteligente (2 cenarios)
- Caso especifico do bug #391363 (3 cenarios)

---

## 5. Comparacao Python CLI (cli-hub-portal-vibe) vs TypeScript (web)

### 5.1 Divergencia encontrada no CLI

O `replanejamento.py` do CLI (`cli-hub-portal-vibe/cli_anything/hub_portal_vibe/core/replanejamento.py`, linha 97) **ainda usa a logica antiga**:

```python
valor_parcela = round(restante / n_rest, 2) if n_rest > 0 else restante
```

Isso significa que o CLI de simulacao tambem apresenta o bug de centavos. A funcao `arredondar_parcelas()` em `financeiro_base.py` (linha 252) ja usa floor + ajuste de ultima parcela corretamente, mas NAO e usada no `simular_replanejamento()`.

### 5.2 Recomendacao

Atualizar `simular_replanejamento()` no CLI para usar `arredondar_parcelas()` de `financeiro_base.py`, garantindo paridade com a correcao do TypeScript.

---

## 6. Nota sobre Teste E2E (Playwright)

O teste E2E via Playwright (adesao run + replanejamento run) nao pode ser executado neste ambiente por falta de dependencias do sistema (libatk, libgtk, etc). Para validacao E2E:

1. Instalar deps do Playwright: `playwright install-deps chromium`
2. Criar contrato: `cli-anything-hub-portal-vibe adesao run FORMAE-172687 --headless --parcelas 34`
3. Testar replanejamento: `cli-anything-hub-portal-vibe replanejamento run <email> --headless --parcelas 12`
4. Verificar no banco: `cli-anything-hub-portal-vibe --json contrato info <id>`

---

## 7. Conclusao

| Verificacao | Resultado |
|-------------|-----------|
| Soma das parcelas = valor contratado | **100% (15/15 cenarios)** |
| Nenhuma data duplicada | **100% (15/15 cenarios)** |
| Nenhum valor negativo ou zero | **100% (15/15 cenarios)** |
| Datas com overflow tratadas corretamente | **100% (fev 28/29, abr 30, jun 30)** |
| Entrada inteligente funciona | **100%** |
| Validacao de integridade antes de salvar | **Implementada** |
| Testes unitarios | **33/33 passed** |
| Logica antiga falhava | **12/15 cenarios com erro** |
| Logica nova funciona | **15/15 cenarios corretos** |

**Bug #391363 esta CORRIGIDO e VALIDADO.**
