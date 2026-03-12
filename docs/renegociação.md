📘 DOCUMENTO COMPLETO – LÓGICA DA RENEGOCIAÇÃO DO CONTRATO DE FORMANDOS

(versão unificada e final)

A renegociação é um processo que reorganiza a dívida do aluno de forma clara, justa e sustentável, considerando:
    •    valores já vencidos
    •    valores que ainda vão vencer
    •    participação ou não na arrecadação alternativa (AA)
    •    uso ou não do parcelamento estendido
    •    percentual de entrada
    •    juros/multa acumulados

Esse documento descreve TUDO o que o sistema precisa entender e calcular.

⸻

🔵 1. IDENTIFICAÇÃO DO CENÁRIO ATUAL

(Primeiro, o sistema lê o estado real do contrato)

O sistema coleta cinco informações:

1.1. Mensalidades vencidas

É o valor base das parcelas mensais que ficaram para trás e ainda não foram pagas.

1.2. Arrecadação Alternativa (AA) vencida

Mesmo que o aluno saia da AA, AA vencida é principal não pago.
Entra sempre como dívida.

1.3. Mensalidades futuras (a vencer)

É a parte do plano que ainda vai vencer no futuro.

1.4. AA futura (a vencer)

E aqui entra a primeira VARIANTE importante:
    •    Se o aluno participa da AA, esse valor não entra no replanejamento.
    •    Se o aluno não participa, esse valor vai ser incorporado ao principal do plano e entra no replanejamento.
    •    Se o aluno entra na AA agora, tiramos esse valor do replanejamento.

1.5. Juros e multa

Aqui ficam SOMENTE os encargos gerados pelo atraso — tanto mensal quanto AA.

Esse valor NUNCA vira principal.
Sempre vira uma parcela única separada chamada “Acordo de Renegociação”.

⸻

🔵 2. DEFINIÇÃO DO PRINCIPAL QUE SERÁ REPLANEJADO

(Combina vencido + futuro, dependendo da configuração escolhida)

O sistema sempre começa com:

principal_vencido = mensal_vencido + aa_vencida

Agora entram as variantes da AA:

2.1. Se o aluno CONTINUA participando da AA
    •    AA futura fica fora do replanejamento
    •    mensal futuro entra

principal_futuro = mensal_futuro

2.2. Se o aluno PASSA A PARTICIPAR da AA
    •    AA futura também fica fora
    •    mensal futuro entra

principal_futuro = mensal_futuro

2.3. Se o aluno DEIXA de participar da AA
    •    AA futura entra junto com mensal futuro

principal_futuro = mensal_futuro + aa_futura

2.4. Total a replanejar (principal geral)

principal_replanejar = principal_vencido + principal_futuro

Esse é o valor que será dividido em parcelas normais (entrada + remanescentes).

⸻

🔵 3. CÁLCULO DA ENTRADA INTELIGENTE

(Entrada nunca pode ser menor do que a parcela. Ela se ajusta automaticamente.)

O sistema calcula dois valores:

3.1. Entrada Percentual

Exemplo:
    •    Entrada padrão: 20%
    •    principal vencido: R$ 2.000,00

entrada_percentual = 0.20 × principal_vencido
entrada_percentual = 400

3.2. Valor da nova parcela

Depende do uso do parcelamento estendido:

A) Sem estendido

valor_parcela = principal_replanejar / parcelas_normais

B) Estendido modo B (sem taxa embutida)

valor_parcela = principal_replanejar / (parcelas_normais + parcelas_estendidas)

C) Estendido modo A (taxas embutidas)

valor_parcela = (principal_replanejar + taxas) / total_parcelas

3.3. Entrada final

A entrada real será:

entrada_final = maior entre (entrada_percentual, valor_parcela)

Isso mantém:
    •    UX simples
    •    parcelas lineares
    •    menos distorção financeira
    •    consistência para o aluno

⸻

🔵 4. GERAÇÃO DAS NOVAS PARCELAS

(Sempre a mesma estrutura, independente da escolha do aluno)

O sistema gera:

4.1. Parcela 1 – Entrada

Tipo: Parcela Normal (Entrada)
Valor: entrada_final
Pertence ao plano, não é juros.

⸻

4.2. Parcelas Normais Remanescentes

Quantidade:

total_parcelas_gerais - 1

Todas têm o valor:

valor_parcela

Se houver estendido:
    •    primeiro vêm as parcelas normais
    •    depois as estendidas

⸻

4.3. Parcela Única – Acordo de Renegociação

Tipo: Acordo de Renegociação
Composto por:

juros + multa (mensal + AA)

Essa parcela é independente e não altera o plano.

⸻

🔵 5. REGRAS DO SISTEMA (TODA A LÓGICA OPERACIONAL)

5.1. Entrada
    •    sempre é principal
    •    precisa ser paga primeiro
    •    impede novos replanejamentos

⸻

5.2. AA vencida

Sempre vira principal vencido.
Nunca vira juros.

⸻

5.3. AA futura

Depende da escolha do aluno:

Situação    O que acontece
continua na AA    AA futura sai do plano e vira boletos
passa a participar    AA futura sai do plano e vira boletos
sai da AA    AA futura entra no replanejamento como principal


⸻

5.4. Estendido
    •    aumenta a quantidade de parcelas
    •    pode aumentar o valor (modo A)
    •    não muda juros/multa
    •    não muda AA vencida nem futura
    •    influencia a entrada final

⸻

5.5. Acordo de Renegociação
    •    sempre parcela única
    •    sempre só juros + multa
    •    pode gerar juros se atrasar
    •    nunca vira principal

⸻

🔵 6. EXEMPLO COMPLETO (PLANO DE R$ 10.000)

Cenário hipotético
    •    valor total do plano: R$ 10.000
    •    mensalidades: 10 × 1.000
    •    mensalidades vencidas: 2 × 1.000 → 2.000
    •    AA vencida: 0
    •    mensal futuro: R$ 8.000
    •    AA futura: R$ 2.000 (exemplo)
    •    juros/multa total: R$ 300
    •    entrada percentual: 20%

Aluno decide:
    •    sair da AA
    •    usar estendido modo B
    •    parcelar em 10 normais + 4 estendidas (14 total)

⸻

📌 Passo 1 — Calcular principal vencido

principal_vencido = 2.000 + 0 = 2.000

📌 Passo 2 — Como ele SAIU da AA, AA futura entra:

principal_futuro = 8.000 + 2.000 = 10.000

📌 Passo 3 — Total do principal replanejado:

principal_replanejar = 2.000 + 10.000 = 12.000

📌 Passo 4 — Valor da parcela (estendido modo B)

valor_parcela = 12.000 / 14 ≈ 857,14

📌 Passo 5 — Entrada percentual

entrada_percentual = 20% × 2.000 = 400

Como 400 < 857, a entrada precisa ser ajustada:

entrada_final = 857,14

📌 Passo 6 — Parcelas geradas
    1.    Entrada — 857,14
2–14. Parcelas normais/estendidas — 857,14
Acordo — 300

⸻

🔵 7. RESUMO SIMPLES DA LÓGICA

(para equipe comercial e atendimento)
    •    Toda renegociação parte do principal vencido + futuro.
    •    AA vencida vira principal.
    •    AA futura pode entrar ou sair do plano conforme a escolha do aluno.
    •    A entrada nunca é menor que o valor da parcela.
    •    Juros e multa sempre vão para uma parcela separada.
    •    O estendido aumenta a quantidade de parcelas e pode aumentar o valor (modo A).
    •    O aluno não pode replanejar novamente sem pagar a entrada.
