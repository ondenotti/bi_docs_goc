# Compra NET, guia para quem usa o BI

O que o valor NET é, o que ele não é, e como ler os números sem tropeçar.
Última revisão: 16/10/2026.

---

## 1. Em uma frase

**NET é quanto a fábrica recebeu pela peça, limpo dos impostos que ela mesma cobrou na venda.**

É um número visto do lado de lá do balcão. A CNH equaliza os objetivos de retirada da concessionária em valor NET, e esse número não vem pronto do Solution: a gente calcula.

---

## 2. O que o NET não é

Esta é a parte que mais gera confusão, então vale deixar explícito.

**NET não é o custo da peça.** O custo é o que a peça pesa no nosso estoque e no nosso resultado. O NET é o que a fábrica faturou líquido. São duas perguntas diferentes e as duas respostas estão certas ao mesmo tempo.

**NET não é o valor da nota.** O valor total da nota tem IPI, ICMS ST e frete por dentro. O NET parte só do valor da mercadoria.

**NET não é base de crédito de imposto.** O crédito é assunto do nosso fiscal e segue outra regra, explicada no item 4.

Se alguém perguntar "por que o NET não fecha com o custo?", a resposta é: não deve fechar. Se fechasse, um dos dois estaria errado.

---

## 3. Como o número é montado

```
valor da mercadoria
  (-) ICMS destacado na nota
  (-) PIS que a fábrica cobrou
  (-) COFINS que a fábrica cobrou
  = COMPRA NET
```

Um exemplo real, de uma válvula comprada da CNH em setembro de 2026 (nota 4162553, produto 47735860, 9 unidades):

| | |
|---|---|
| valor da mercadoria | 28.954,62 |
| (−) ICMS da nota | 1.158,18 |
| (−) PIS que a fábrica cobrou (2,30% sobre 27.796,44) | 639,32 |
| (−) COFINS que a fábrica cobrou (10,80% sobre 27.796,44) | 3.002,02 |
| **= NET** | **24.155,10** |

Note três coisas nesse exemplo.

**O IPI e o ICMS ST não aparecem.** Não é esquecimento: eles não estão dentro do "valor da mercadoria", estão somados depois para formar o total da nota. Descontá-los seria descontar duas vezes.

**As alíquotas são 2,30% e 10,80%, não 1,65% e 7,60%.** Autopeça é regime monofásico: a fábrica paga uma alíquota maior porque ela concentra o imposto da cadeia inteira. Quem revende depois não paga mais nada.

**O custo médio dessa mesma linha é 37.286,03**, bem acima do NET e até acima do valor da mercadoria, porque ele carrega ICMS ST, IPI e frete. É o exemplo vivo do item 2: NET e custo não conversam.

---

## 4. O PIS/COFINS da nota não é o nosso crédito

Esse ponto confunde até quem é da área, e ele é o coração do indicador.

Em ICMS funciona como a intuição manda: o imposto destacado na nota é o crédito que a gente aproveita. **Em PIS/COFINS não.** A lei manda o comprador **calcular** o crédito aplicando a *própria* alíquota sobre o valor da compra, sem olhar o que o vendedor cobrou ou pagou.

Então, na mesma nota, convivem dois números diferentes e ambos corretos:

| | de quem é | quanto |
|---|---|---|
| PIS/COFINS que a fábrica cobrou | é o imposto **dela** | o que o NET usa |
| PIS/COFINS que o nosso fiscal creditou | é o crédito **nosso** | o que afeta o custo |

Na válvula do exemplo acima, a fábrica cobrou R$ 3.641,34 de PIS e COFINS e o nosso fiscal creditou **zero**. Está certo: autopeça monofásica não dá crédito na revenda, porque a cadeia já foi tributada na fábrica.

E acontece o contrário também. Numa compra de fornecedor pequeno, do Simples Nacional, a nota pode mostrar PIS/COFINS pequeno, ou nenhum, e o nosso fiscal ainda assim credita 1,65% e 7,60% sobre o valor da compra. Também está certo, e é expressamente permitido.

> **Resumo:** o PIS/COFINS impresso numa nota de compra é o débito do vendedor. Não é o nosso crédito e não limita o nosso crédito.

Por isso o indicador não usa o imposto registrado na entrada. Ele vai ler o imposto **no XML da nota**, que é onde está o que a fábrica realmente cobrou.

---

## 5. Onde o NET existe, e onde vem vazio

O NET só é calculado onde ele significa algo. Em todo o resto a coluna vem **vazia**.

**Só compras da CNH.** O NET é a ótica da fábrica. Aplicar essa ótica a um fornecedor que não é fábrica dá um número sem significado, não um número errado. Compra de distribuidor, de ferragista, de posto, de supermercado: sem NET.

**Só grupos de peças.** O indicador é de retirada de peças. Máquina fica fora.

**Só de 01/01/2025 para frente.** Antes dessa data o ERP não gravava a informação que liga cada linha da nossa compra ao item correspondente dentro do XML. Dá para estimar, mas preferimos não publicar estimativa como se fosse leitura. Então o histórico anterior vem vazio, de propósito.

### O que isso significa na prática

- **Não compare o total de NET com o total de compras.** São populações diferentes: compras é tudo, NET é só peça da CNH de 2025 para cá.
- Se um visual de NET aparecer vazio, confira primeiro o período e o fornecedor antes de abrir chamado.

---

## 6. Um mês fechado, para dar escala

Setembro de 2026, peças da CNH, 2.610 linhas:

| | |
|---|---|
| valor da mercadoria | 2.214.294,78 |
| (−) ICMS da nota | 104.469,51 |
| (−) PIS/COFINS que a fábrica cobrou | 243.053,99 |
| **= NET** | **1.866.771,28** |

Para comparar: se usássemos o PIS/COFINS registrado na nossa entrada em vez do da fábrica, o número sairia **2.022.365,13**, ou seja **7,7% maior**. A diferença é essa: a fábrica cobrou R$ 243 mil de PIS e COFINS e a nossa entrada registrou R$ 87 mil, **2,8 vezes menos**, porque boa parte das peças é monofásica e não gera crédito.

Esses R$ 155 mil de diferença num único mês são a razão de existir do trabalho todo de ler o XML.

Nesse mês, **100% das linhas** tiveram o imposto lido do XML: 2.604 pelo item exato e 6 pelo código do produto. Nenhuma linha precisou de rateio ou de estimativa.

---

## 7. A coluna que explica cada número: `origem_piscofins_net`

É uma coluna interna, não exibida ao consumidor final do BI, mas vale entender a lógica. Toda linha diz de onde veio o imposto que entrou no cálculo dela. É a coluna que permite auditar o indicador sem abrir o banco.

**Os três rótulos que dizem "não tem NET":**

| rótulo | significado |
|---|---|
| `Sem NET, não é CNH` | fornecedor fora do grupo CNH |
| `Sem NET, não é peças` | é compra da CNH, mas é máquina ou outro grupo |
| `fora da janela do NET` | é peça da CNH, mas anterior a 01/01/2025 |

**Os quatro rótulos que dizem como o imposto foi obtido**, do mais preciso para o menos:

| rótulo | como foi obtido | confiança |
|---|---|---|
| `XML item (nItem)` | leu o imposto do item exato dentro do XML da nota | é o valor do documento |
| `XML produto (cProd)` | casou o código do produto com o do XML | na prática exato: onde o código casa, bate no centavo |
| `XML rateio da nota` | rateou o imposto da nota entre as linhas pelo valor | aproximado |
| `entrada (sem XML)` | a nota não tem XML aproveitável, usou o imposto da entrada | é a única linha realmente estimada |

### Como está hoje

De todas as linhas que têm NET, **99,93% vêm das duas primeiras formas**, ou seja, de leitura do documento. O rateio responde por 0,29% do valor e o uso do imposto da entrada por 0,02%, que são 10 linhas de notas da CNH que chegaram sem XML importado.

Em outras palavras: o NET hoje é praticamente todo leitura de documento fiscal, não estimativa. Se precisar defender o número numa reunião, esse é o argumento, e a coluna de origem é a prova.

---

## 8. O que ainda não está fechado

Honestidade sobre o estágio: **temos um cálculo coerente, não uma reconciliação.**

Nada no número acima foi conferido contra um extrato da própria CNH. A validação que decide é essa: pegar um mês de extrato de retirada da fábrica, com o NET que eles reconhecem, e comparar. Até isso acontecer, o indicador serve para acompanhar tendência, comparar meses e orientar decisão interna, mas não para contestar um número da fábrica.

Um ponto menor, também aberto:

- Dez linhas de peça da CNH chegaram sem XML e usam o imposto da entrada. Valor pequeno, e dá para pedir o XML ao fiscal e fechar.

---

## 9. Perguntas rápidas

**Como é tratado devoluções de compra no valor NET?**
Devolução entra no NET com sinal negativo, é a mesma lógica se analisar em valor de compra normal.

**Posso somar NET com compra ou com custo no mesmo visual?**
Pode mostrar lado a lado, não pode somar. São óticas diferentes: NET é o lado da fábrica, custo é o nosso.

**Ao analisar a base de dados do BI, por que a alíquota aparece em branco em algumas linhas?**
As colunas de CST e alíquota vêm do item exato do XML. Nas linhas obtidas por código de produto ou por rateio elas não existem, mesmo com o imposto preenchido. O valor está certo; só o detalhe da alíquota é que não se aplica.

**Uma peça da CNH apareceu como `Sem NET, não é peças`. É erro?**
Provavelmente é cadastro: o produto está em grupo fora da faixa de peças. Vale verificar o grupo do item, não o cálculo.

---

Dúvida sobre metodologia, cálculo ou manutenção: ver `NET_documentacao_tecnica.md`.
