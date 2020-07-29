# Materialized Views

As visualizações materializadas são pré-computadas e armazenam em cache os resultados de uma consulta periodicamente para aumentar o desempenho e a eficiência. O BigQuery utiliza resultados pré-computados de visualizações materializadas e, sempre que possível, lê somente alterações delta da tabela base para computar resultados atualizados

# Funções de Analíticas no SQL

Uma função analítica calcula valores em um grupo de linhas e retorna um único resultado para _cada_ linha. Isso não ocorre em uma função agregada, que retorna um único resultado para _um grupo inteiro_ de linhas.

Ele inclui uma cláusula `OVER`, que define uma janela de linhas ao redor da linha que está sendo avaliada. Para cada linha, o resultado da função analítica é calculado com a janela de linhas selecionada como entrada, fazendo, possivelmente, a agregação.

Com as funções analíticas, é possível calcular médias móveis, classificar itens, calcular somas cumulativas e realizar outras análises.

Saiba mais: [https://cloud.google.com/bigquery/docs/reference/standard-sql/analytic-function-concepts#numbering_function_concepts](https://cloud.google.com/bigquery/docs/reference/standard-sql/analytic-function-concepts#numbering_function_concepts)

# Exemplo de Caso de uso
Por exemplo, temos uma tabela de endereço  que é carregada diariamente, e para cada endereço temos 1 ID associado a ela. Precisamos garantir que esse endereço(id) na nossa visão seja o ultima versão que foi carregada da origem. Para isso utilizar materialized views e funçoes analiticas (Partition By , over ROW NUMBER() ) para garantir a ultima versão do dado 


## Carga dia 1:

Ainda não existe dados na tabela, será carregada pela primeira vez no Big Query.

Dados tabela dia 1:
	


## Carga dia 2

Novos endereços inseridos, porem nenhuma atualização dos dados já existentes no Big Query (novos ids)

Dados tabela dia 2:


## Carga dia 3

Os registros inseridos no dia 1 teve alteração na origem, e serão carregados hoje (dia 3) pro Big Query.
Os id de endereço serão duplicado, sendo um antigo e um novo.

Dados Tabela dia 3 :

## Views Deduplicadoras

Para Garantir uma visão com os dados do endereço mais atualizado, criamos uma view materializada.
Exemplo de Query :

	WITH
	  x AS (
	  SELECT
	    *,
	    DATE(_PARTITIONTIME) AS data_ingestao,
	    CASE
	      WHEN CAST(last_modified_date AS STRING) IS NULL THEN creation_date
	    ELSE
	    last_modified_date
	  END
	    AS data_alteracao
	  FROM
	    `raw.people_addresses_teste` )
	SELECT
	  *,
	FROM (
	  SELECT
	    ROW_NUMBER() OVER(PARTITION BY id ORDER BY id, data_alteracao desc, data_ingestao desc) rownumber,
	    *
	  FROM
	    x )
	WHERE
	  rownumber = 1	

## Visão com View Deduplicadora
Obs: Lembrando que com a View materializada não faremos full scan toda vez que ela é consumida, ela computa os resultados anteriores e compara com os novos da tabela de onde ela está consumindo.

Visão deduplicadora - Somente ultima versão dos dados:

