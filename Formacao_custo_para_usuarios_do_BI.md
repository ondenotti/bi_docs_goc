# Formação do custo das compras, guia para quem usa o BI

Como o BI monta o **Custo Médio Calculado** a partir das notas de compra, o
que cada linha da matriz significa, e por que ele é comparado com o custo do
Solution.
Última revisão: 10/11/2026.

---

## 1. Em uma frase

**O BI refaz, nota por nota, a conta que o Solution faz para dar entrada de
uma peça no estoque: parte do valor da nota, soma o que vira custo, tira os
impostos que a empresa recupera e chega ao custo. No fim, compara com o custo
que o Solution gravou.**

Se os dois números baterem, a formação do custo está explicada. Se não
baterem, existe alguma situação que o BI ainda não conhece, e ela deve ser
investigada.

---

## 2. Por que montar o custo de novo

O Solution grava, em cada linha da nota, o valor que entrou no estoque. É esse
valor que forma o custo médio da peça, que por sua vez é usado no preço de venda
e no CMV. Mas o Solution não mostra **de que** esse valor é feito.

A matriz do BI abre esse valor em partes: quanto é mercadoria, quanto é
imposto que ficou no custo, quanto é crédito que saiu, quanto é frete. Assim
fica possível responder perguntas como:

- Quanto do custo das compras do período é ICMS ST?
- Quanto de crédito de PIS e COFINS foi abatido?
- Quanto de frete entrou no estoque?
- O custo que está no estoque é o que deveria estar?

> **Sobre o nome:** "Custo Médio Calculado" e "Custo Médio Solution" são o
> **valor total** que entrou ou saiu do estoque no período, em reais. Não são o
> custo unitário da peça. O nome vem da coluna `custo_medio`, porque é esse
> valor que alimenta o custo médio de cada peça.

---

## 3. A matriz, linha a linha

A matriz tem três blocos:

1. **A formação do custo nas compras**, montada pela própria nota.
2. **As movimentações que entram pelo custo do Solution:** devolução, estorno
   e outras entradas.
3. **Os erros do Solution já mapeados.**

No fim vêm o resultado e duas linhas de conferência.

**Bloco 1: compras, montadas pela nota**

```text
(+) Compra
(+) ICMS ST GR
(-) ICMS
(-) PIS
(-) COFINS
(+) Frete
(-) ICMS Frete
(-) PIS Frete
(-) COFINS Frete
(-) ICMS NF Complementar
(-) ICMS Simples Nacional
```

**Bloco 2: movimentações pelo custo do Solution**

```text
(-) Custo da Devolução
(+) Custo do Estorno da Devolução
(+) Custo Outras Entradas
```

**Bloco 3: erros do Solution já mapeados**

```text
(+) Frete CTC Divergência - Erro Solution
(+) PIS/COFINS Troca CST
(+) PIS/COFINS Alterado Após Custeio
```

**Resultado e conferência**

```text
(=) Custo Médio Calculado                              soma dos três blocos
    Custo Médio Solution                               conferência
    ICMS NF Complementar (Não Reprocessado no Custo)   só informação
```

No BI as linhas aparecem em outra ordem: "Frete CTC Divergência" vem logo
depois de "ICMS Simples Nacional", e as duas de PIS/COFINS vêm depois de "Custo
Outras Entradas". A conta é a mesma. Aqui elas estão agrupadas pelo que
significam.

### Antes de tudo: quais notas entram

- **Só notas do agrupamento "Compras"** do tipo de lançamento.
- **Só linhas que mexem no estoque.** Compras de uso e consumo, energia
  elétrica e outras que não entram no estoque têm custo zero e ficam de fora de
  todas as linhas da matriz. Elas continuam no BI como compra, mas não formam
  custo de peça.

---

## 4. Bloco 1: as compras, montadas pela nota

Neste bloco entram **só as compras**, ou seja, as notas cujo custo o Solution
calcula pelo **Valor Líquido** da nota. Devoluções, estornos e outras
entradas ficam de fora dele e aparecem no bloco 2.

### (+) Compra

É o **total da linha na nota**, que já vem pronto do Solution:

```text
  valor dos produtos
- desconto
+ IPI
+ ICMS ST destacado na nota
+ frete e seguro destacados na própria nota
= Compra
```

O IPI e o ICMS ST entram no custo porque a empresa **não recupera** esses
impostos: eles são parte do preço da peça.

"Compra" é **só compra**. As notas de devolução, de estorno e de outras
entradas não entram nesta linha: elas estão no bloco 2, pelo custo do
Solution.

> O frete que aparece aqui é só o que o fornecedor cobrou **dentro da nota**.
> O frete cobrado à parte, por CT-e, é a linha "(+) Frete", mais abaixo.

### (+) ICMS ST GR

ICMS por **substituição tributária recolhido em guia** pela própria empresa,
fora da nota. É muito comum em peças compradas **de outro estado**: o
fornecedor não cobra o ICMS ST na nota, e a empresa recolhe esse imposto por
guia. Como ele não é recuperado, **soma** no custo, como o ICMS ST da nota. A
diferença é que ele não está no total da nota, por isso tem linha própria.

Quase sempre, uma linha tem um ou outro: ou o ICMS ST veio na nota (e já está
em "Compra"), ou foi recolhido em guia (e está aqui).

### (−) ICMS

O **crédito de ICMS** da compra, que a empresa recupera na apuração e por isso
sai do custo.

Atenção: é o ICMS **creditado**, não o ICMS destacado na nota. Quando a peça
tem ICMS ST, o ICMS destacado pelo fornecedor quase nunca gera crédito e, por
isso, fica no custo. Nas compras de 2025 e 2026, as linhas com ICMS ST
destacaram R$ 3,0 milhões de ICMS, e só R$ 30 mil viraram crédito.

### (−) PIS e (−) COFINS

O **crédito de PIS e COFINS** da compra. Só existe quando a linha tem direito
a crédito (CST 50 a 56). Peças monofásicas ou com alíquota zero (CST 70 a 75)
vêm com zero, e nada sai do custo.

### (+) Frete

O **frete do CT-e** ligado à nota, rateado entre as peças. É o frete que a
transportadora cobrou à parte. Ele **soma** no custo, porque é um gasto para a
peça chegar ao estoque.

### (−) ICMS Frete, (−) PIS Frete, (−) COFINS Frete

Os **créditos de impostos do CT-e**. Da mesma forma que na mercadoria, o que a
empresa recupera do frete sai do custo.

### (−) ICMS NF Complementar

Às vezes o fornecedor destaca ICMS a menos e depois emite uma **nota
complementar** só com o ICMS que faltou. O crédito dessa nota é tomado, mas o
Solution **só tira esse ICMS do custo da peça quando o custo é reprocessado
depois** que a complementar entra no sistema.

Esta linha mostra **só o ICMS complementar que o Solution de fato abateu** do
custo. O que não foi abatido está na última linha da matriz. Veja
[ICMS complementar no custo](ICMS_complementar_para_usuarios_do_BI.md).

### (−) ICMS Simples Nacional

Quando o fornecedor é do **Simples Nacional**, ele pode informar na nota um
valor de ICMS que o comprador tem direito a creditar. O Solution abate esse
crédito do custo, e a matriz também.

---

## 5. Exemplos do bloco 1

### Exemplo 1: uma peça com desconto e frete de CT-e, Gurupi

Nota **91976** (nota interna 581754), lançada em 13/03/2026. Item
**92123058**, 1 unidade, sem ICMS ST.

| Linha da matriz | Valor | De onde vem |
|---|---|---|
| (+) Compra | 8.390,50 | Produto 9.988,69 − desconto 1.598,19 |
| (−) ICMS | −335,62 | Crédito de ICMS (igual ao destacado, porque a peça não tem ST) |
| (−) PIS | −132,91 | Crédito de PIS |
| (−) COFINS | −612,17 | Crédito de COFINS |
| (+) Frete | 276,87 | Parte deste item no frete do CT-e |
| (−) ICMS Frete | −33,22 | Crédito de ICMS do frete |
| (−) PIS Frete | −4,57 | Crédito de PIS do frete |
| (−) COFINS Frete | −21,04 | Crédito de COFINS do frete |
| **(=) Custo Médio Calculado** | **7.527,84** | |
| Custo Médio Solution | 7.527,84 | Bate no centavo |

### Exemplo 2: ICMS ST recolhido em guia, Gurupi

Nota **3286411** (nota interna 464502), lançada em 02/01/2025. Item
**47459734**, 6 unidades.

| Linha da matriz | Valor | De onde vem |
|---|---|---|
| (+) Compra | 5.322,23 | Produtos 5.187,36 + IPI 134,87 |
| (+) ICMS ST GR | 1.477,13 | ST recolhida em guia, fora da nota |
| (−) ICMS | 0,00 | A nota destacou 212,68 de ICMS, mas a peça tem ST: **sem crédito** |
| (−) PIS | −82,08 | |
| (−) COFINS | −378,08 | |
| **(=) Custo Médio Calculado** | **6.339,20** | |
| Custo Médio Solution | 6.339,20 | Bate no centavo |

Repare no ICMS: os R$ 212,68 destacados na nota não saem do custo, porque não
viraram crédito. Se o BI usasse o ICMS destacado, o custo ficaria errado.

---

## 6. Bloco 2: movimentações pelo custo do Solution

Nestas três linhas o BI **não monta** o custo pela nota. Ele usa direto o
valor que o Solution tirou ou colocou no estoque, e **os impostos e o frete
dessas notas não entram** em nenhuma linha do bloco 1.

O motivo: nessas operações o Solution quase sempre usa o custo que a peça já
tinha no estoque, e não o valor da nota. Por isso a nota não explica o valor.
O que vale para o estoque é o que o Solution de fato movimentou, e esse é o
custo do Solution.

### (−) Custo da Devolução

A **devolução de compra**: a peça volta ao fornecedor e **sai** do estoque. O
valor é negativo.

A peça sai pelo custo com que entrou, não pelo valor da nota de devolução. Na
maioria das vezes é o custo da compra de origem. Quando a filial devolve tudo o
que tinha da peça, é o custo médio da filial. Os impostos destacados na nota de
devolução não mexem no estoque: a diferença vai para a contabilidade.

**Exemplo:** a compra **509515** de Lagoa da Confusão (20/06/2025) deu entrada
de uma peça por R$ 98.719,04. Ela foi devolvida pela nota interna **518195**
(28/07/2025). Pela nota de devolução, com o ICMS destacado nela, o custo daria
R$ 74.301,61. Mas o Solution baixou do estoque **R$ 98.719,04**, exatamente o
custo da entrada. É esse valor que a matriz usa.

### (+) Custo do Estorno da Devolução

O **estorno**: anula uma devolução ou uma saída, e a peça **volta** ao
estoque. O valor é positivo. É o espelho da devolução: devolveu, sai o custo;
estornou, o custo volta.

**Exemplo:** a nota interna **616744** de Teresina (11/08/2026) anulou uma nota
de saída. O item F0CV0D003F5ED vale R$ 11.320,17 na nota, com R$ 2.547,04 de
ICMS. Pela nota, o custo daria R$ 8.773,13. O estoque da peça estava zerado,
e o Solution devolveu a peça pelo último custo médio da filial, **R$
10.059,62**. É esse o valor da matriz.

### (+) Custo Outras Entradas

Entradas pelas operações **54 e 211 (Entrada de brindes/doações)** e **60
(Entrada de cestas de Natal)**. Em peças, essas operações não costumam ser
brinde ou doação. Quase sempre foram usadas para contornar algum problema de
lançamento. São raras e, até hoje, sempre entradas.

O Solution lança essas entradas pelo **custo médio da filial**, não pelo valor
da nota.

**Exemplo:** a NF **63628** (nota interna 601345), de Palmas, 30/05/2026, deu
entrada de uma peça que vale R$ 35.187,93 na nota. O estoque estava zerado e o
último custo médio era R$ 27.985,38. A peça entrou por **R$ 27.985,38**, e é
esse o valor da matriz.

Quando a peça nunca teve custo na filial, o custo médio é zero e a entrada
fica sem custo. Isso é o que acontece com a maioria das cestas e brindes.

---

## 7. Bloco 3: os erros do Solution já mapeados

Em alguns casos, algo mudou na nota **depois** que o Solution calculou o custo,
e o custo não foi recalculado. O custo que ficou no estoque está errado, mas é
o que o sistema usa. O BI então soma o erro como uma parcela própria. Assim o
Custo Médio Calculado bate com o do Solution, e o tamanho do erro fica à vista.

| Linha da matriz | O que aconteceu | Página com os detalhes |
|---|---|---|
| (+) Frete CTC Divergência - Erro Solution | O CT-e foi dividido com outra nota, ligado ou tirado da nota depois do custeio. **Positivo:** frete a mais no estoque. **Negativo:** frete a menos | [Frete do CT-e no custo](Frete_CTe_para_usuarios_do_BI.md) |
| (+) PIS/COFINS Troca CST | O CST de PIS/COFINS foi trocado de 73 para 53 depois do custeio. O crédito passou a existir, mas o custo ficou com o imposto dentro | [PIS/COFINS alterado depois do custeio](Troca_CST_PIS_COFINS_para_usuarios_do_BI.md) |
| (+) PIS/COFINS Alterado Após Custeio | O PIS/COFINS foi alterado de outro jeito depois do custeio: CST trocado de 53 para 73, valor recalculado ou preenchido depois. **Positivo:** custo mais caro que o correto. **Negativo:** mais barato | [PIS/COFINS alterado depois do custeio](Troca_CST_PIS_COFINS_para_usuarios_do_BI.md) |

Esses erros são raros e quase todos antigos. Em 2025 as três linhas são zero,
e em 2026 só existe um caso de frete.

---

## 8. O resultado e as duas linhas de conferência

### (=) Custo Médio Calculado

A soma de todas as linhas acima. É o custo que o BI consegue **explicar**.

### Custo Médio Solution

O valor que o Solution gravou no estoque. **Os dois devem ser iguais.** Uma
diferença expressiva quer dizer que apareceu uma situação nova, que o BI ainda
não conhece. Nesse caso, avise o time de BI para investigar.

Sobras de centavos são normais. Elas vêm de arredondamento do Solution.

### ICMS NF Complementar (Não Reprocessado no Custo)

**Não faz parte da conta.** Fica abaixo do resultado só para informação.

É o ICMS de notas complementares que **deveria** ter saído do custo da peça,
mas não saiu, porque o custo nunca foi reprocessado depois que a complementar
entrou. O crédito foi tomado na contabilidade, mas o estoque ficou com esse
valor dentro. **O custo dessas peças no estoque está mais alto do que o
correto, nesse valor.**

O valor aparece **negativo**, como a linha "(−) ICMS NF Complementar": é o
quanto o custo teria caído se o crédito tivesse sido abatido.

---

## 9. O tamanho, ano a ano

Toda a base, todas as filiais e todas as áreas, só linhas que mexem no estoque.

| Linha da matriz | 2024 | 2025 |
|---|---|---|
| (+) Compra | 529.965.659,24 | 414.655.878,43 |
| (+) ICMS ST GR | 4.330.066,60 | 4.415.477,81 |
| (−) ICMS | −17.742.495,96 | −12.860.591,40 |
| (−) PIS | −739.011,22 | −927.375,89 |
| (−) COFINS | −3.404.555,26 | −4.272.992,30 |
| (+) Frete | 890.026,72 | 656.175,33 |
| (−) ICMS Frete | −60.605,10 | −41.096,18 |
| (−) PIS Frete | −14.662,40 | −10.827,21 |
| (−) COFINS Frete | −67.533,29 | −49.869,33 |
| (−) ICMS NF Complementar | 0,00 | −4.090,82 |
| (−) ICMS Simples Nacional | −361,03 | −182,20 |
| (+) Frete CTC Divergência - Erro Solution | −8.524,57 | 0,00 |
| (−) Custo da Devolução | −17.903.813,05 | −20.891.117,25 |
| (+) Custo do Estorno da Devolução | 25.712,01 | 80.442,21 |
| (+) Custo Outras Entradas | 0,00 | 0,00 |
| (+) PIS/COFINS Troca CST | 25.896,69 | 0,00 |
| (+) PIS/COFINS Alterado Após Custeio | 6.293,10 | 0,00 |
| **(=) Custo Médio Calculado** | **495.302.092,48** | **380.749.831,20** |
| **Custo Médio Solution** | **495.284.684,52** | **380.749.831,20** |
| ICMS NF Complementar (Não Reprocessado no Custo) | 0,00 | −34.687,90 |

- **Em 2025 os dois batem no centavo.**
- **Em 2024 sobram R$ 17.407,96.** Quase tudo vem de uma única nota de Balsas
  (R$ 17.420,00), ainda em investigação. É exatamente o tipo de caso que a
  comparação serve para mostrar.

Desde 2019, mais de 99,9% das linhas batem. Os anos de 2016 a 2018, do começo
do uso do Solution, têm diferenças que ainda não foram investigadas.

---

## 10. Como usar no dia a dia

**Para saber de que é feito o custo do período:** leia o bloco 1. Ele mostra
quanto da compra virou custo e quanto foi recuperado em impostos.

**Para saber quanto as compras colocaram no estoque no período:** use o
resultado, que soma os três blocos. As devoluções tiram custo, os estornos e
as outras entradas colocam. As vendas e as outras saídas não estão nesta
matriz.

**Para achar problemas:** compare o Custo Médio Calculado com o Custo Médio
Solution. No nível da nota ou do item, a coluna `custo_diferenca_composicao`
mostra a diferença, e a coluna `obs_composicao` diz o motivo, quando ele já é
conhecido.

**Para saber se o estoque está com custo errado:** olhe o bloco 3 e a linha de
ICMS NF Complementar (Não Reprocessado no Custo). São valores que estão no
estoque, mas não deveriam estar, ou que deveriam ter saído do custo e não
saíram. Corrigir o custo no Solution é com a controladoria ou com o suporte do
Solution. O BI só aponta.

---

## Dúvidas

A explicação de como cada linha é calculada está na
[documentação técnica](Formacao_custo_documentacao_tecnica.md). Se algum
número não se encaixar no que está aqui, fale com o time de BI.
