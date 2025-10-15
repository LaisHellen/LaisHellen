# Revisão e otimização do fluxo de planilha automatizada
O objetivo do projeto foi substituir o fluxo da planilha Macro 2024/2025 por um modelo anual contínuo, eliminando a necessidade de troca de arquivos e reduzindo o consumo de armazenamento e processamento. Logo, realizei a revisão e otimização da planilha corporativa que reunia dados brutos e campos calculados de múltiplas origens (manuais e automatizadas), com consumo médio de 2,1 GB por hora durante a execução.

![image](https://prod-files-secure.s3.us-west-2.amazonaws.com/a905e310-3606-48e7-b760-c76cc029e59b/48fce9cf-80c5-440a-a25b-11170a5e286c/Captura_de_Tela_2025-01-10_as_10.01.25.png)

Para começar, mapeei toda a linhagem de dados manualmente, identificando dependências, origens e conexões ativas que impactavam o desempenho. Resumidamente descrito na imagem abaixo:
![fluxo leroy generico 2.png](attachment:9120df2e-ff65-4b8c-99c1-c9326d7315e9:fluxo_leroy_generico_2.png)

A partir desse diagnóstico, realizei a limpeza e reorganização do fluxo, o que reduziu significativamente o consumo de processamento e viabilizou a construção das quebras mensais e anuais desenvolvidas posteriormente em uma demanda complementar do mesmo projeto, facilitando análises comparativas e visuais.

![fluxo novo leroy generico 3.png](attachment:9fb8d044-c1ea-4816-8bee-49a182a07e52:fluxo_novo_leroy_generico_3.png)

No BigQuery do cliente, desenvolvi uma UDF em SQL e com uso de `REGEXP` para automatizar a classificação de campanhas, parecido com o seguinte código: 

```
CREATE OR REPLACE FUNCTION `projeto.dataset.nome_udf`(x ANY TYPE, y ANY TYPE)
OPTIONS (description="Classificação principal das Campanhas relevantes para a planilha Macro Performance através do nome da campanha(x) e origem/mídia(y)") AS (
CASE 
     WHEN
      REGEXP_CONTAINS(x,r'(?i).*nome_campanha|nome_campanha.*')
      AND REGEXP_CONTAINS(y,r'.*origem / midia.*')
      AND NOT REGEXP_CONTAINS(x,r'(?i).*nome_campanha_|_nome_campanha.*')
    THEN "Nome coluna"

      WHEN
      REGEXP_CONTAINS(x,r'(?i).*nome_campanha|nome_campanha.*')
      AND REGEXP_CONTAINS(y,r'.*origem / midia.*')
      AND NOT REGEXP_CONTAINS(x,r'(?i).*nome_campanha_|_nome_campanha.*')
    THEN "Nome coluna"
  ELSE "Não Classificado"
END
);
```

Também estruturei consultas SQL com uso de tabelas `PIVOT` e particionadas, o que diminuiu a carga sobre as planilhas e centralizou o tratamento de dados na camada do banco.

```
/* VIEW PARA CALCULAR O VALOR DAS MÉTRICAS DE ACORDO COM O DIA E A UDF PARA A PLANILHA DE MACRO PERFORMANCE */
SELECT *
FROM (
  SELECT
    Date AS Data,
    `projeto.dataset.nome_udf`(campaign, sourceMedium) AS tipo_campanhas,
    SUM(adCost) AS investimento,
    SUM(transactionRevenue) AS receita,
    SUM(transactions) AS transacoes,
    SUM(impressions) AS impressoes,
    SUM(sessions) AS sessoes,
    SUM(adClicks) AS cliques
  FROM
    `projeto.dataset.nome_view`
  GROUP BY
    Date, 
    tipo_campanhas
)
PIVOT (
  -- convertendo o tipo das campanhas em coluna para fazer a agregação por data
  SUM(investimento) AS investimento,
  SUM(receita) AS receita,
  SUM(transacoes) AS transacoes,
  SUM(impressoes) AS impressoes,
  SUM(sessoes) AS sessoes,
  SUM(cliques) AS cliques
  FOR tipo_campanhas IN ('Nome Coluna', 'Nome Coluna', 'Nome Coluna')
)
ORDER BY 
  Data;
```
E na planilha final, em Sheets, otimizei fórmulas utilizando funções como `ARRAYFORMULA`, `SE` e `QUERY`. 

Essa abaixo por exemplo, traz dados filtrados e renomeados de outra aba; se não houver dados, mostra 0 como resultado.

```
=ARRAYFORMULA(
  SE(
    QUERY(Fonte!A:Z; "SELECT Col1, Col2, Col3, Col4 LABEL Col1 'Nome1', Col2 'Nome2', Col3 'Nome3', Col4 'Nome4'") = "";
    0;
    QUERY(Fonte!A:Z; "SELECT Col1, Col2, Col3, Col4 LABEL Col1 'Nome1', Col2 'Nome2', Col3 'Nome3', Col4 'Nome4'")
  )
)
```
Resumo das modificações técnicas:



