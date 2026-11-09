# ICMS da nota complementar no custo, guia para quem usa o BI

Por que o ICMS de uma nota complementar às vezes baixa o custo da peça e às
vezes não, e como o BI mostra cada caso.
Última revisão: 06/11/2026.

---

## 1. Em uma frase

**O crédito do ICMS complementar é tomado sempre do mesmo jeito, mas o Solution
só tira esse ICMS do custo da peça quando o custo da compra é reprocessado
depois que a nota complementar entra no sistema.**

Então duas compras iguais, com complementares iguais, podem ficar com custos
diferentes no estoque. A diferença não é fiscal: é a ordem em que as coisas
aconteceram no sistema.

---

## 2. O que é a nota complementar

Às vezes o fornecedor destaca ICMS a menos na nota de venda. Para corrigir, ele
emite depois uma **nota complementar de ICMS**: uma nota sem mercadoria e sem
valor a pagar, só com o ICMS que faltou. No Solution ela entra pela operação
**240, "Entrada de ICMS complementar"**, e fica ligada à nota de compra
original.

Como a compra é de custo "Valor Líquido", o ICMS que a empresa credita não deve
ficar no custo da peça. Por isso, em tese, o ICMS da complementar deveria
**baixar** o custo da compra original. Na prática, isso só acontece em um dos
casos abaixo.

---

## 3. Os quatro casos no BI

A coluna `obs_composicao` diz em qual caso a linha está.

| `obs_composicao` | O que aconteceu | O custo da peça |
|---|---|---|
| **Nota complementar: ICMS abatido do custo (custo reprocessado depois dela)** | Alguém rodou o reprocessamento do custo médio **depois** que a complementar foi lançada | Baixou: o ICMS saiu do custo |
| **Nota complementar: ICMS não abatido do custo (custo não reprocessado depois dela)** | O custo nunca foi reprocessado depois da complementar | Não mudou: ficou como no lançamento da compra |
| **Nota complementar de ICMS lançada sem ICMS: crédito não tomado, não altera o custo** | A nota do fornecedor tem ICMS, mas ela foi lançada no Solution com ICMS zero | Não mudou |
| **Nota complementar de valor, sem ICMS: não altera o custo** | É um complemento de preço, sem imposto | Não mudou |

O `custo_pela_composicao` do BI já segue essa regra. Ele só abate o ICMS da
complementar no primeiro caso, então a composição fecha com o custo do Solution
nos quatro.

---

## 4. Exemplo do ICMS abatido: NF 17990, Teresina

Compra de peças da Mutum, lançada em 02/06/2025. Item **8094848**, 6 unidades.

| Quando | O que aconteceu |
|---|---|
| 02/06/2025 | A NF **17990** é lançada. O custo do item é R$ 8.843,34 |
| 02/07/2025, 22:20 | Entra a complementar **NF 18042**, com R$ 2.401,79 de ICMS para este item |
| 03/07/2025, 07:43 | O custo médio do item é **reprocessado** |

| | Valor |
|---|---|
| Custo da compra, sem a complementar | 8.843,34 |
| (−) ICMS da complementar | 2.401,79 |
| **= Custo do Solution** | **6.441,55** |

O reprocessamento veio depois da complementar, então o ICMS saiu do custo.

---

## 5. Exemplo do ICMS não abatido: NF 63914, Balsas

Compra de peças da Mutum, lançada em 23/05/2025. Item **1100BR**, 1 unidade.

| Quando | O que aconteceu |
|---|---|
| 23/05/2025 | A NF **63914** é lançada. O custo do item é R$ 10.028,10 |
| 03/07/2025, 08:46 | Entra a complementar **NF 64062**, com R$ 2.187,03 de ICMS para este item |
| — | O custo do item **nunca é reprocessado** depois disso |

| | Valor |
|---|---|
| Custo do Solution | **10.028,10** |
| Custo se o ICMS da complementar tivesse sido abatido | 7.841,07 |

O crédito de R$ 2.187,03 foi tomado na contabilidade, exatamente como em
Teresina. Mas o custo da peça no estoque continua com esse valor dentro.

Repare na hora: o reprocessamento das notas de Teresina rodou às 07:43 do dia
03/07, e esta complementar só entrou às 08:46 do mesmo dia. Por uma hora ela
ficou de fora.

---

## 6. Exemplo da complementar de ICMS lançada sem ICMS: NF 26365, São Mateus do Maranhão

Compra de peças da Mutum, lançada em 30/05/2025. A complementar **NF 26450**
entrou em 31/07/2025. A nota emitida pelo fornecedor tem **R$ 1.083,06 de
ICMS**, mas ela foi lançada no Solution com ICMS **zero**: nem o livro fiscal
nem a contabilidade têm esse crédito. O custo das peças não mudou.

São **9 notas** assim, de Teresina e São Mateus do Maranhão, de jun–jul/2025.
Somando, são **R$ 2.339,99** de ICMS destacado que não foram creditados. É um
ponto para o fiscal conferir.

## 7. Exemplo do complemento de valor: NF 1105658, Marabá

Complemento de **R$ 238,72** no preço de um lubrificante comprado em abril de
2023 (operação 117, "Complementar de valores"). É uma diferença de preço, não
de ICMS, então não há ICMS para abater. São 3 notas assim, todas de 2023.

---

## 8. O tamanho de cada caso

| Caso | Notas de compra | ICMS da complementar |
|---|---|---|
| ICMS abatido do custo | 8 | 4.090,82 |
| ICMS não abatido do custo | 66 | 48.145,76 |
| Complementar de ICMS lançada sem ICMS | 9 | 2.339,99 na nota do fornecedor, zero no Solution |
| Complemento de valor | 3 | — |

- **O abatido** é tudo de Teresina, reprocessado na manhã de 03/07/2025.
- **O não abatido** é quase tudo de 2025, de Balsas, Teresina e São Mateus do
  Maranhão, mais 15 notas de Lagoa da Confusão de 2021.

---

## 9. O que fazer quando aparecer

**Para análise de preço e margem**, lembre que o sistema usa o custo do
Solution. No caso "não abatido", o custo da peça no estoque está **mais alto**
do que o custo líquido do crédito, na medida do ICMS da complementar.

**Para corrigir o custo**, o caminho é rodar o reprocessamento do custo médio
dos itens afetados. Isso é com a controladoria ou com o suporte do Solution. O
BI só aponta e não corrige nada no sistema.

**Para a complementar de ICMS lançada sem ICMS**, o assunto é fiscal: vale o
fiscal conferir se o crédito dessas complementares deveria ter sido tomado.

**O complemento de valor** não pede nenhuma ação.

---

## Dúvidas

A explicação de como a regra é calculada está na
[documentação técnica](ICMS_complementar_documentacao_tecnica.md). Se algum
caso não se encaixar no que está aqui, fale com o time de BI.
