# Otimização de processamento da dashboard de criativos no Looker

O cliente sinalizou que a dashboard de criativos no Looker estava demorando para carregar e pediu para investigarmos possíveis soluções.

* Recriei as tabelas fazendo particionamento e clusterização em SQL no Banco de dados do cliente, para diminuir o processamento e realizei as trocas para alimentar o Looker.
  
Exemplo de Código utilizado:

```
---- CRIAÇÃO DA TABELA COM CLUSTER E PARTIÇÃO ---

CREATE OR REPLACE TABLE
    `id_projeto.dataset.tabela_particionada`
  PARTITION BY 
    date
  CLUSTER BY 
    date,
    campaign_name
  AS
  SELECT 
	  * 
  FROM 
	  `id_projeto.dataset.view_base` 
```

* Recriei as consultas programadas e unifiquei as em apenas uma consulta, usando a função merge para alimentar as novas tabelas periodicamente conectadas na dashboard de criativos. Tudo numa mesma programação facilita economia de processamento.
    
    Exemplo de Código utilizado:
```
MERGE
    `id_projeto.dataset.tabela_particionada`
  USING(
    SELECT
      dimensões,
      metricas
    FROM
         `id_projeto.dataset.view_base` 
    WHERE
        date >= DATE_SUB(CURRENT_DATE(), INTERVAL 5 DAY)
  )
  
  ON
  FALSE
  WHEN NOT MATCHED BY SOURCE AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 5 DAY) THEN DELETE
  WHEN NOT MATCHED
  THEN

  INSERT 
    (
      dimensões,
      metricas
  )

  VALUES
    (
			dimensões,
      metricas
    )
;


MERGE
    `id_projeto.dataset.tabela_particionada`
  USING(
    SELECT
      dimensões,
      metricas
    FROM
         `id_projeto.dataset.view_base` 
    WHERE
        date >= DATE_SUB(CURRENT_DATE(), INTERVAL 5 DAY)
  )
  
  ON
  FALSE
  WHEN NOT MATCHED BY SOURCE AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 5 DAY) THEN DELETE
  WHEN NOT MATCHED
  THEN

  INSERT 
    (
      dimensões,
      metricas
  )

  VALUES
    (
			dimensões,
      metricas
    )
;
```

O Looker também possui uma personalização de janela de tempo de atualização, logo deixei para atualizar os dados a cada 12h, o espaço de tempo máximo que a plataforma deixava espaçar

Durante as implementações também foram inclusas etapas de validações que consistiam em comparar valores macro de tabelas originais e cópias, a fim de garantir integridade nas replicações e substituições, analisando o mesmo período, métrica e dimensão principal, além da quantidade de linhas que cada tabela gerava.

Exemplo de código usado para comparar:

```
SELECT 
	dimensão,
	SUM(metrica) AS metrica
FROM 
	`id_projeto.dataset.tabela_particionada` 
WHERE 
	date= "AAAA-MM-DD"
GROUP BY 
	dimensão


SELECT 
	dimensão,
	SUM(metrica) AS metrica
FROM 
	`id_projeto.dataset.view_base` 
WHERE 
	date= "AAAA-MM-DD"
GROUP BY 
	dimensão
```

Foi sinalizado ao cliente, algumas boas práticas que o mesmo podia fazer uso para ajudar no carregamento, como:

* visualizar dados no menor período de tempo possível, como diário ou semanal.
* Utilizar o recurso de atualizar dados que o Looker Studio para atualizar as páginas mais rapidamente.
* Caso as soluções técnicas implementas, em conjunto com as boas praticas não surtisse efeito pratico no manuseio da dashboard, poderiamos usar o recurso do Google Ads Transfer que na época era recente via de transferencia de dados entre plataformas, oferecido pelo GCP, que podria ajudar no gerenciamento de relatorio recorrente além de

Após o feedback positivo de que as medidas tomadas ajudaram a resolver o problema, foram excluidas as tabelas e visualizações deprecadas através do comando `DROP TABLE` e `DROP VIEW` em SQL, no dataset do cliente, para evitar custo de armazenamento, auxiliar em organização e otimizar processo de manutenção. 

Referências usadas durante a fase de estudo e planejamento

https://cloud.google.com/bigquery/docs/materialized-views-create?hl=pt-br#partition_cluster

https://cloud.google.com/bigquery/docs/reference/standard-sql/data-definition-language#creating_a_clustered_table_from_the_result_of_a_query
