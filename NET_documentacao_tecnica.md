# Compra NET, documentação técnica

Para quem dá manutenção na fato, no script SQL ou no modelo do Power BI.
Última revisão: 16/10/2026.

- **Fato oficial do NET:** `fcompra_com_net_2025.sql`
- **Variante de história completa:** `fcompra_com_net.sql` (calcula NET em toda a base e para todo fornecedor; serve para auditoria, **não** alimenta o indicador)
- **Banco:** PostgreSQL 13.10, ERP Solution, acesso somente leitura

---

## 1. O que o NET é, em termos de cálculo

Valor da mercadoria sob a ótica da **fábrica**, isto é, o que a CNH recebeu líquido dos impostos que ela mesma cobrou na venda. Não é custo contábil, não é base de crédito, não é valor da nota.

```
compra_net = vlr_produto_na_nota
           - ICMS_da_nota
           - pis_da_fabrica
           - cofins_da_fabrica
```

Na CTE `fcompra` isso é uma subtração simples, porque as quatro colunas já saem de `base` com o sinal da devolução aplicado:

```sql
CASE WHEN base.origem_piscofins_net IN ('fora da janela do NET',
                                        'Sem NET, não é CNH',
                                        'Sem NET, não é peças') THEN NULL
     ELSE base.vlr_produto_na_nota
            - COALESCE(base.ICMS_da_nota, 0)
            - COALESCE(base.pis_da_fabrica, 0)
            - COALESCE(base.cofins_da_fabrica, 0)
END as compra_net
```

### Por que parte de `e51vlrprod` e não de `e51totnota2`

Porque `totnota2 = vlrprod + IPI + ICMS ST` (conferido, diferença 0,00 em R$ 70 mi). IPI e ICMS ST **já estão fora** do `vlrprod`, então deduzi-los de novo seria contar duas vezes. E o **ICMS ST GR nem compõe o total da nota**: deduzi-lo tirava ~12% indevidamente (R$ 6,5 mi em 2025). ST da nota e ST GR são mutuamente exclusivos na base: 24.614 linhas só com GR contra 23.055 só com o da nota.

### Por que o PIS/COFINS não vem da `movext`

A `movext` guarda o que a **entrada** registrou, que é o nosso crédito, calculado pela nossa alíquota. No regime monofásico de autopeças a concessionária não toma crédito, então o Solution grava CST 70 ou 73 e **zero** em `e51vlrpis` / `e51vlrcofins`. O imposto da fábrica existiu, só não está na `movext`: está no XML da NF-e, em `notass_xml`.

A coluna `compra_net_pela_entrada` calcula a mesma fórmula com o PIS/COFINS da entrada, para dar como comparar os dois. Não use como NET.

---

## 2. Fontes

| tabela | papel |
|---|---|
| `movext` | a linha da fato: produto, valores, `e51ndetxml`, `e51_e37grusub` |
| `notass` | a nota: operação (`e22codope`), datas, fornecedor (`e62clifor_conta`) |
| `notass_xml` | o XML da NF-e. `e74tipo_xml = '09'` é a NF-e de entrada |
| `crp_contas` | o cadastro, usado para identificar fornecedor da CNH |
| `codope`, `estgruposub`, `cadite`, `filial`, `usuari` | de/para e descrições, iguais ao fcompra de produção |

`e74tipo_xml = '09'` é **dedução** pela cobertura e pelo conteúdo. Não há no banco tabela que descreva os tipos. Falta confirmar com o suporte do Solution.

---

## 3. Pipeline de CTEs

```
parametro              data de início do NET (um lugar só)
contas_cnh             quem é fornecedor da CNH (um lugar só)
escopo_notas           notas no escopo do fcompra, da janela, e só da CNH
  └─ xml_candidato     um documento por nota, lixo antes do prolog removido, versão escolhida
      └─ xml_valido         MATERIALIZED, só documento bem formado
          └─ piscofins_do_xml      MATERIALIZED, CAMADA 1: item a item
              ├─ piscofins_por_produto   CAMADA 2: agregado por (nota, cProd)
              └─ piscofins_por_nota      CAMADA 3: agregado por nota
dados_da_compra_na_nota_dev   igual ao fcompra de produção
base                   a linha, com os três LEFT JOIN das camadas + contas_cnh
fcompra                compra_net e as colunas de devolução
select final           ordem das colunas
```

`escopo_notas` é o que **corta o custo**: o XML só é parseado para nota que o fcompra usa, dentro da janela e de fornecedor CNH. De 27.923 notas do escopo de 2025+ sobram 18.361.

---

## 4. Os três defeitos de `notass_xml` e as defesas

Isto é o que mais custou a descobrir. Não remova nenhuma das três defesas.

**Cada linha de `notass_xml` é um documento COMPLETO, não um pedaço.** 464.479 das ~464,5 mil linhas do tipo 09 têm `<nfeProc>` dentro, inclusive a segunda linha da mesma nota. Quando uma nota tem mais de uma linha, são **versões** da mesma NF-e (o `e62nfe_id` da `notass` bate com as duas), não pedaços. A primeira versão deste script agregava com `STRING_AGG` e grudava documento com documento, gerando `invalid XML document`. Só não aparecia com filtro de setembro porque a duplicidade é toda de 2016 a 2022.

| defeito | tamanho | defesa no SQL |
|---|---|---|
| mais de um documento por nota | 787 notas, todas até 2022 | `ROW_NUMBER() ... ORDER BY (btrim(e74chavenfe) <> '') DESC, e74codint DESC` e `WHERE versao = 1`. Prefere a importação nova, que tem a chave preenchida |
| documento truncado | 210 linhas, 139 cortadas em exatamente 30.000 caracteres (limite antigo do campo, hoje 1.024.000) | `AND x.e74xml LIKE '%</nfeProc>%'`. 195 notas só têm a versão cortada e ficam sem XML |
| BOM gravado como texto antes do prolog | 5 linhas (os bytes `EF BB BF` virados em três caracteres, mesmo com o banco em UTF8) | `regexp_replace(x.e74xml, '^[^<]+', '')`. Recupera a nota em vez de descartar |

Mais duas, por cima:

- **`AND x.e74notaint <> 0`**: `notaint = 0` é balde de XML sem nota vinculada, 34 mil linhas. A `notass` não tem `notaint` 0, então o join já as excluiria, mas o filtro é barato.
- **`xml_is_well_formed_document(xml)` em CTE `MATERIALIZED`**: é a última barreira. **Um** documento ruim aborta a query inteira, então nada vai para o `XMLPARSE` sem passar por ela. O `MATERIALIZED` existe para o planner não inverter a ordem e parsear antes de filtrar. O filtro `LIKE` sozinho **não** basta: os 5 do BOM passam por ele e a base inteira quebra.

Custo: a validação parseia o XML duas vezes. Vale o preço, porque a alternativa é a carga do BI falhar inteira.

---

## 5. As três camadas e a precedência

O join exato é por número de item: `movext.e51ndetxml = <det nItem>`. Só que **`e51ndetxml` só começou a ser gravado em 13/09/2025**; antes é zero em 100% das linhas. Sem camada de reserva o script não erraria, ele cairia calado no fallback da entrada, que no monofásico é zero, e o NET sairia inflado.

| camada | chave | precisão medida (2026, contra o gabarito do nItem) |
|---|---|---|
| 1, item | `px.nitem = mv.e51ndetxml` | exata, é o valor do item |
| 2, produto | `pp.cprod = upper(btrim(mv.e18codpro))` | o `cProd` casa em **92,6%** das linhas e, onde casa, o valor rateado bate com o exato **no centavo em 100,00%** |
| 3, nota | só `notaint` | sozinha, aplicada em tudo, recuperaria 98,5% do total mas acertaria só 30% das linhas no centavo |

Camadas 2 e 3 são a **mesma fórmula**, `imposto_do_XML / vProd_do_XML × vlrprod_da_linha`, mudando só o grão. Por isso nenhuma delas duplica valor quando o mesmo produto aparece em dois itens da nota, nem infla quando a `movext` tem menos linhas que o XML tem itens: cada linha leva a parte dela, proporcional ao próprio `vlrprod`.

> **Ao ratear, o denominador é o `vProd`, nunca o `vBC` do PIS.** O `vBC` vem zero em muitos itens e ratear por ele infla o imposto em **47%**.

### A precedência decide pelo casamento, não pelo valor

```sql
CASE WHEN px.nitem IS NOT NULL AND px.cst_pis IS NOT NULL
          THEN COALESCE(px.vpis, 0)
     WHEN pp.vprod > 0 AND pp.itens_com_cst_pis > 0
          THEN COALESCE(round(mv.e51vlrprod * pp.vpis / pp.vprod, 2), 0)
     WHEN pn.vprod > 0 AND pn.itens_com_cst_pis > 0
          THEN COALESCE(round(mv.e51vlrprod * pn.vpis / pn.vprod, 2), 0)
     ELSE mv.e51vlrpis END
```

Regra: **CST declarado e sem valor vale ZERO**, porque CST 04, 06, 07 e 08 são operação em que o vendedor não cobrou contribuição nenhuma. Quem **não** tem CST em lugar nenhum cai na entrada, nunca em zero, senão fornecedor pequeno sem XML importado deduziria imposto nenhum e o NET sairia inflado.

Duas armadilhas que essa forma resolve e que você reintroduz se voltar para `COALESCE`:

1. **`COALESCE` desce de camada quando o valor é nulo.** Item isento que casou pelo nItem ganhava rateio. Era o caso dos espaguetes CST 07 na nota 4252.
2. **Com `CASE`, o primeiro `WHEN` que bate vence mesmo devolvendo nulo.** Por isso as camadas 2 e 3 precisam do `itens_com_cst_pis` (um `count(cst_pis)` nas agregações): sem ele, grupo de produto sem valor nenhum devolveria vazio em vez de descer para a camada seguinte.

O `origem_piscofins_net` **espelha exatamente essa precedência**. Se mexer em uma, mexa na outra, senão a coluna mente sobre qual camada alimentou a linha.

Impacto da correção de precedência, medido na variante de história completa, mesmo snapshot, 365.399 linhas: PIS 24.518.223,69 → 24.421.254,90, COFINS 116.800.448,26 → 116.355.513,30, NET 2.381.015.216,18 → 2.381.557.119,93. O NET sobe exatamente o imposto que deixou de ser deduzido, R$ 541.903,75.

---

## 6. `origem_piscofins_net`, os seis valores

Esta é a coluna de auditoria do indicador. A ordem abaixo é a ordem do `CASE`: o primeiro que bate vence.

| valor | condição | significado | linhas hoje | NET |
|---|---|---|---|---|
| `Sem NET, não é CNH` | `cnh.t310conta IS NULL` | fornecedor fora do grupo CNH. NET não se aplica | 111.981 | vazio |
| `Sem NET, não é peças` | `COALESCE(mv.e51_e37grusub, -1) NOT BETWEEN 400 AND 799` | é CNH, mas é máquina ou outro grupo | 3.761 | vazio |
| `fora da janela do NET` | `mv.e51datal < (SELECT inicio_net FROM parametro)` | peça da CNH anterior a 01/01/2025 | 183.026 | vazio |
| `XML item (nItem)` | `px.nitem IS NOT NULL AND px.cst_pis IS NOT NULL` | leitura exata do item do XML | 41.012 | 28.860.507,06 |
| `XML produto (cProd)` | `pp.vprod > 0 AND pp.itens_com_cst_pis > 0` | rateio no grupo do produto. Exato onde o `cProd` casa | 25.575 | 22.276.341,91 |
| `XML rateio da nota` | `pn.vprod > 0 AND pn.itens_com_cst_pis > 0` | alíquota efetiva da nota. É a única aproximação de verdade | 39 | 147.397,03 |
| `entrada (sem XML)` | o `ELSE` | nota CNH de peça sem XML aproveitável. Usa o PIS/COFINS da **entrada** | 10 | 10.508,06 |

**99,93% das linhas com NET vêm de camada 1 ou 2**, que são exatas. O rateio sobrou com 0,29% do valor e o fallback da entrada com 0,02%.

Note que a ordem põe os motivos **permanentes** (fornecedor, grupo) antes do **temporal** (janela), porque a janela é escolha nossa e não natureza da linha. Com isso `Sem NET, não é peças` cobre toda a história da CNH fora de peças, não só 2025+.

---

## 7. As dez colunas do NET

| coluna | o que é | preenchida |
|---|---|---|
| `cst_pis_saida_fabrica` | CST do PIS no XML | só camada 1 |
| `cst_cofins_saida_fabrica` | CST do COFINS no XML | só camada 1 |
| `aliq_pis_fabrica` | `pPIS` do item | só camada 1 |
| `aliq_cofins_fabrica` | `pCOFINS` do item | só camada 1 |
| `base_piscofins_xml` | `vBC` do PIS do item | só camada 1 |
| `pis_da_fabrica` | PIS que a fábrica cobrou, com sinal de devolução | camadas 1, 2, 3 e entrada |
| `cofins_da_fabrica` | idem COFINS | camadas 1, 2, 3 e entrada |
| `compra_net` | a conta do item 1 | só onde há NET |
| `compra_net_pela_entrada` | a mesma conta com o PIS/COFINS da entrada, para comparar | todas as linhas |
| `origem_piscofins_net` | de onde saiu o PIS/COFINS, item 6 | todas as linhas |

As cinco primeiras são do **item** do XML, então no histórico e nas camadas 2 e 3 vêm em branco mesmo com o PIS/COFINS preenchido. Visual de alíquota fica vazio fora da camada 1.

---

## 8. Pontos de manutenção

Três lugares, e dois deles têm par fora do script.

**1. A janela, na CTE `parametro`.**

```sql
WITH parametro AS (
    SELECT DATE '2025-01-01' AS inicio_net
),
```

Mudar ali muda tudo: o corte do parse em `escopo_notas`, o cálculo e o rótulo. O filtro de `escopo_notas` tem **um mês de folga para trás** porque ele é na data da nota (`e62datal`) e o do NET é na data do lançamento (`e51datal`): nota de dezembro lançada em janeiro perderia o XML sem essa folga.

**2. Quem é CNH, na CTE `contas_cnh`.**

```sql
contas_cnh AS (
    SELECT DISTINCT t310conta
      FROM crp_contas
     WHERE t310tipo_conta_integrada = 'F'
       AND UPPER(t310nome) LIKE '%CNH%'
),
```

> **Esta mesma regra vive na dimensão de contas do Power BI. Mexeu aqui, mexa lá**, senão fato e dimensão discordam sobre quem é CNH.

O `DISTINCT` é defensivo: hoje `t310conta` é único (24.190 de 24.190) e sem ele, se um dia deixar de ser, o join multiplicaria linha da fato.

Conferido em 14/10/2026: a regra pega 12 contas, todas CNH INDUSTRIAL, mais o BANCO CNH INDUSTRIAL CAPITAL, que não tem nenhuma linha de compra. **FPT INDUSTRIAL** e **IVECO**, que são do grupo CNH Industrial e o `%CNH%` não pega, têm **zero linha** no escopo. E não existe fornecedor cadastrado como NEW HOLLAND (só um cliente, a associação de distribuidores). A regra por nome é segura **neste cadastro**, mas é segurança por sorte do cadastro, não por desenho: se cadastrarem a fábrica com outro nome, o NET dela desaparece sem erro nenhum. Vale uma conferência periódica.

**3. A faixa de peças, `BETWEEN 400 AND 799`.**

Aparece em dois lugares: no cálculo (teste positivo) e no rótulo (`COALESCE(..., -1) NOT BETWEEN`, para que subgrupo nulo caia fora em vez de escapar para o cálculo). A faixa vale para `e01codigo` 1 (GDO) e 2 (Cotrama) e não haverá outra, por isso a regra não olha a empresa.

Conferido nas compras da CNH de 2025+: nos grupos superiores 400, 500, 600 e 700 a faixa pega 100% das linhas, e o grupo 7000, que o de/para de área também chama de Peças, **não aparece em nenhuma compra da CNH**. O que fica fora são os grupos 1000 a 1700, que é o Comercial.

O filtro de peças **não** entra em `escopo_notas` de propósito: ele é de linha, e a `movext` de uma nota pode misturar peça e máquina. Cortar a nota perderia a peça que viajou junto, e seriam 635 notas de 18.361, 3,5% do parse.

---

## 9. Testes de regressão

Rode os três sempre que mexer no script.

**1. Contagem de linhas, que é o teste do join.** A fato tem de entregar exatamente o mesmo número de linhas e de `pk_id` distintos que o fcompra de produção. Qualquer camada nova pode multiplicar linha, e esse teste pega.

```sql
SELECT count(*), count(DISTINCT pk_id) FROM ( <fcompra de produção> ) z;
SELECT count(*), count(DISTINCT pk_id) FROM ( <fcompra_com_net_2025>  ) z;
```

Rode os dois **na mesma transação `REPEATABLE READ`**. A base é viva: em um dia de trabalho ela cresceu de 365.180 para 365.404 linhas, e comparar snapshots diferentes já me fez perseguir uma duplicação que não existia.

**2. O valor não muda onde não deveria.** Ao acrescentar um corte, o NET da população que **sobra** tem de ficar idêntico ao centavo:

```sql
-- exemplo do corte de peças
sum(compra_net) FILTER (WHERE <é peça>)  -- antes e depois: 51.294.754,06
```

**3. Setembro/2026, o mês de referência.** Dois conjuntos de números servem de marco:

- **Leitura do XML, todas as notas do escopo** (sem os cortes de CNH e peças): 2.866 itens, 707 notas, PIS 110.787,69, COFINS 521.805,65. Foi com estes que se provou que a correção do `STRING_AGG` não mexeu em nada.
- **A fato como está hoje, peças da CNH**: 2.610 linhas, vlrprod 2.214.294,78, ICMS 104.469,51, PIS/COFINS da fábrica 243.053,99, PIS/COFINS da entrada 87.460,14, **NET 1.866.771,28** contra 2.022.365,13 pela ótica da entrada (7,7% maior). Origem: 2.604 linhas por `nItem` e 6 por `cProd`, nenhuma por rateio ou entrada.

Para rodar o mês, descomente a janela de teste em `base` (`AND mv.e51datal BETWEEN ...`). Atenção: na fato o `escopo_notas` **não** tem mais toggle de janela de teste, porque o filtro dele agora vem da CTE `parametro`. Então um teste de um mês ainda parseia o XML de todo o período da janela; é esperado, e é por isso que a query de um mês custa ~25s em vez de 1s.

---

### Linha de conferência ponta a ponta

Uma linha real para validar o pipeline inteiro depois de qualquer mexida. Nota **4162553**, produto **47735860** (VALVULA), 9 unidades, setembro/2026, fornecedor CNH:

| campo | valor esperado | de onde vem |
|---|---|---|
| `vlr_produto_na_nota` | 28.954,62 | `movext.e51vlrprod` |
| `ICMS_da_nota` | 1.158,18 | `movext.e51vlicms_danota` |
| `cod_cst` (ICMS) | 10 | `movext.e51cst` |
| `cst_pis_saida_fabrica` | 02 | XML, `imposto/PIS/*/CST` |
| `aliq_pis_fabrica` / `aliq_cofins_fabrica` | 2,3000 / 10,8000 | XML, monofásico de autopeça |
| `base_piscofins_xml` | 27.796,44 | XML, `vBC`. Confere: 28.954,62 − 1.158,18 |
| `pis_da_fabrica` | 639,32 | XML, `vPIS`. Confere: 27.796,44 × 2,30% |
| `cofins_da_fabrica` | 3.002,02 | XML, `vCOFINS`. Confere: 27.796,44 × 10,80% |
| `pis` / `cofins` (entrada) | 0,00 / 0,00 | `movext`, zerado pelo monofásico |
| `custo_medio` | 37.286,03 | `movext.e51vtotal`, carrega ST, IPI e frete |
| `compra_net` | **24.155,10** | 28.954,62 − 1.158,18 − 639,32 − 3.002,02 |
| `compra_net_pela_entrada` | 27.796,44 | a mesma conta com o imposto da entrada |
| `origem_piscofins_net` | `XML item (nItem)` | casou pelo `e51ndetxml` |

Essa linha exercita tudo que importa: a escolha do documento, a leitura do item, o monofásico com entrada zerada, e a diferença de R$ 3.641,34 entre as duas óticas numa única linha.

---

## 10. Desempenho

O custo é o parse do XML, não a `movext`. Medições de servidor do mesmo dia, base inteira: 28s, 33s, 34s, 38s para a fato. **A variação da carga do servidor é maior que o efeito dos filtros**, então não atribua ganho a mudanças pequenas sem repetir a medição.

O que de fato pesa:

- o parse: ~18 mil notas na fato, contra ~80 mil na variante de história completa (50s só a leitura do XML)
- a transferência das ~365 mil linhas para o Power Query, que **nenhum** filtro de NET reduz, porque o número de linhas da fato não muda

Se precisar de ganho real, o caminho é **materializar `piscofins_do_xml` em tabela** e atualizar só o incremental. Aí o parse sai da carga. Esbarra em uma restrição: o Solution é banco de terceiro, somente leitura, então a tabela teria de viver no Senior ou em esquema nosso.

---

## 11. Limitações conhecidas

1. **Não há reconciliação.** Hoje existe um cálculo coerente, não um número conferido contra a fábrica. Falta um extrato de retirada da CNH de um mês com o NET que eles reconhecem.
2. **`e74tipo_xml = '09'`** é dedução, não confirmação do suporte do Solution.
3. **CST 04 na cadeia.** CST 04 é revenda monofásica a alíquota zero: zero é certo para "o que *este* vendedor cobrou", mas sob a ótica da fábrica alguém pagou 2,30/10,80 lá atrás e não capturamos. Para o indicador não pesa, porque a CNH é fabricante e o CST dela vem com valor.
4. **4 itens com CST 01 sem valor no XML** (R$ 730,37 de vProd) são preenchimento errado do emitente e viram zero. Subestima o imposto em uns R$ 12.
5. **228 linhas com CST ICMS 90 sem alíquota no XML** (setembro), ainda não tratadas.
6. **10 linhas de peça da CNH sem XML aproveitável** usam o PIS/COFINS da entrada. R$ 10,5 mil de NET, 0,02%. Dá para listar e pedir o XML ao fiscal.
7. **A regra de CNH é por nome.** Ver o ponto 2 da seção 8.

---

## 12. Histórico de correções

| quando | o que | por que importa |
|---|---|---|
| 13/10/2026 | saiu o `STRING_AGG`, entraram as três defesas de `notass_xml` | sem filtro de data o script estourava com `invalid XML document` |
| 13/10/2026 | entraram as camadas 2 e 3 | o `e51ndetxml` só existe de 13/09/2025 em diante; sem elas 2023 e 2024 caíam calados no fallback |
| 14/10/2026 | precedência pelo casamento e não pelo valor | item isento ganhava rateio |
| 14/10/2026 | NET só para fornecedor da CNH | ótica da fábrica aplicada a quem não é fábrica vira ficção: fornecedor do Simples escreve PIS/COFINS de enfeite no XML |
| 14/10/2026 | NET só para peças, 400 a 799 | máquina era 85% do valor do NET com 635 linhas, e o indicador é de retirada de peças |

Detalhe de encoding: os arquivos `.sql` são UTF-8 e os rótulos de `origem_piscofins_net` têm acento, igual ao `'Contábil'` que já existia no fcompra. Se abrir em editor que assuma outra página de código, os acentos quebram; salve sempre em UTF-8.
