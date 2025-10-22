# Revisão e otimização do fluxo de planilha automatizada

O objetivo do projeto foi substituir o fluxo da planilha Macro 2024/2025 por um modelo anual contínuo, eliminando a necessidade de troca de arquivos e reduzindo o consumo de armazenamento e processamento. Logo, realizei a revisão e otimização da planilha corporativa que reunia dados brutos e campos calculados de múltiplas origens (manuais e automatizadas), com consumo médio de 2,1 GB por hora durante a execução.

![image](https://github.com/LaisHellen/LaisHellen/blob/main/otimizacao_planilha1.png)

Para começar, mapeei toda a linhagem de dados manualmente, identificando dependências, origens e conexões ativas que impactavam o desempenho. Resumidamente descrito na imagem abaixo:
![image](https://github.com/LaisHellen/LaisHellen/blob/main/otimizacao_planilha2.png)

A partir desse diagnóstico, realizei a limpeza e reorganização do fluxo, o que reduziu significativamente o consumo de processamento e viabilizou a construção das quebras mensais e anuais desenvolvidas posteriormente em uma demanda complementar do mesmo projeto, facilitando análises comparativas e visuais.

![image](https://github.com/LaisHellen/LaisHellen/blob/main/otimizacao_planilha3.png)

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

| | **Antes** | **Depois** |
| :--- | :--- | :--- |
| **Fontes Conectadas** | 4 fontes automatizadas sem partição + 5 fontes de dados manuais | 3 tabelas particionadas |
| **Quantidade Extrações** | 11 automatizadas | 5 automatizadas + 5 de dados manuais normalizado |
| **Quantidade de views totais** | 23 | 14 |
| **Views conectadas à planilha final** | 6 views | 3 tabelas particionadas |
| **Processamento (views)** | 831,6 MB | 1 Consulta programada: 78,31 MB |
| **Processamento (planilha final)** | 2.1 GB/hora | 372,45 KB/hora |

Além disso, propus melhorias técnicas para manter a eficiência do fluxo, incluindo:

- Revisão e exclusão de dados anteriores a 2023
- Ampliação da automação de origens manuais
- Criação de Cloud Function para limpeza anual automática
- Padronização e documentação das colunas manuais
- Evolução da camada visual com dashboards e comparativos

Por fim, devido à outros fluxos de trabalho, organizei o cronograma de migração dos arquivos para a nova estrutura e deixei documentado o plano de continuidade do projeto para outro analista concluir o projeto.
