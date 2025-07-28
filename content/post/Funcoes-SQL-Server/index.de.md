---
title: "Funcoes SQL Server"
description: 
date: 2025-07-27T21:43:58-03:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
    - database
    - sql server
    - functions
categories:
    - Database
    - SQL Server
---



# Desvendando as Funções no SQL Server: Poder e Boas Práticas\!

Olá, pessoal\! Hoje vamos mergulhar no mundo das **funções no SQL Server**. Elas são ferramentas incrivelmente poderosas para encapsular lógica e reutilizar código, mas, como tudo no SQL, exigem atenção para não virarem um gargalo de performance.

## O que são Funções no SQL Server?

Uma função no SQL Server é um objeto de banco de dados que aceita parâmetros de entrada, executa um conjunto de ações (como consultas, cálculos, etc.) e retorna um único valor (escalar) ou uma tabela. Elas são similares a procedimentos armazenados, mas com uma diferença crucial: **funções podem ser usadas dentro de instruções `SELECT`**, `WHERE`, `HAVING`, e até mesmo dentro de outras funções, o que as torna extremamente versáteis.

Existem dois tipos principais de funções que podemos criar:

1.  **Funções Escalares (Scalar-valued Functions - T-SQL)**: Retornam um único valor de qualquer tipo de dado.
2.  **Funções com Valor de Tabela (Table-valued Functions - TVFs)**: Retornam uma tabela. Podem ser de duas formas:
      * **Inline Table-valued Functions (iTVFs)**: São mais simples e performáticas, pois essencialmente expandem a lógica da função diretamente na consulta chamadora, como uma view parametrizada.
      * **Multi-statement Table-valued Functions (mTVFs)**: Permitem lógica mais complexa, incluindo múltiplos comandos T-SQL, mas podem ter um impacto significativo na performance devido a otimizações menos agressivas e, muitas vezes, a necessidade de materialização de dados.

## Por Que Usar Funções?

  * **Reutilização de Código**: Escreva a lógica uma vez e use-a em vários lugares.
  * **Modularização**: Quebre problemas complexos em partes menores e mais gerenciáveis.
  * **Consistência**: Garanta que uma lógica de negócio específica seja aplicada da mesma forma em todo o banco de dados.
  * **Manutenção Simplificada**: Alterações na lógica precisam ser feitas em apenas um lugar.

## Criando Funções: Exemplos e Boas Práticas

Vamos a alguns exemplos e, mais importante, às boas práticas\!

### 1\. Função Escalar Simples

Imagine que você precisa calcular o valor total de um imposto sobre um preço.

```sql
CREATE FUNCTION dbo.CalcularImposto (@Preco DECIMAL(18, 2), @PercentualImposto DECIMAL(5, 2))
RETURNS DECIMAL(18, 2)
AS
BEGIN
    RETURN @Preco * (@PercentualImposto / 100);
END;
GO

-- Exemplo de uso
SELECT
    Produto,
    PrecoUnitario,
    dbo.CalcularImposto(PrecoUnitario, 10.00) AS ValorImposto,
    PrecoUnitario + dbo.CalcularImposto(PrecoUnitario, 10.00) AS PrecoTotal
FROM
    MinhaTabelaDeProdutos;
```

**Dica de Performance**: Funções escalares, especialmente as que realizam operações complexas ou acessam dados, podem levar a problemas de performance se chamadas para cada linha em grandes conjuntos de resultados (o temido **RBAR - Row By Agonizing Row**). Pense se a lógica não poderia ser resolvida diretamente na sua consulta principal ou se uma iTVF seria mais adequada.

### 2\. Função com Valor de Tabela (Inline TVF)

Se você precisa retornar um conjunto de resultados baseado em parâmetros, as iTVFs são suas melhores amigas para performance. Elas são tratadas pelo otimizador de consultas de forma muito eficiente.

```sql
CREATE FUNCTION dbo.ObterProdutosPorCategoria (@Categoria NVARCHAR(50))
RETURNS TABLE
AS
RETURN
(
    SELECT
        ProdutoID,
        NomeProduto,
        Preco
    FROM
        Produtos
    WHERE
        Categoria = @Categoria
);
GO

-- Exemplo de uso
SELECT
    p.ProdutoID,
    p.NomeProduto,
    p.Preco
FROM
    dbo.ObterProdutosPorCategoria('Eletrônicos') AS p
WHERE
    p.Preco > 500;
```

**Por que iTVF é melhor que mTVF na maioria dos casos?** O otimizador de consultas pode "inliner" (expandir) a lógica da iTVF diretamente na consulta chamadora, permitindo otimizações mais abrangentes e uso eficaz de índices. mTVFs, por outro lado, muitas vezes materializam seus resultados em um `worktable`, o que pode impedir boas otimizações.

### 3\. Função com Valor de Tabela (Multi-statement TVF) - Cuidado\!

Embora úteis para lógica complexa que envolve múltiplas etapas ou variáveis, as mTVFs devem ser usadas com cautela devido ao seu potencial impacto na performance.

```sql
-- CUIDADO: Este é um exemplo de mTVF, geralmente evite se uma iTVF for possível.
CREATE FUNCTION dbo.ObterRelatorioVendasMensal (@Ano INT, @Mes INT)
RETURNS @Relatorio TABLE
(
    ProdutoID INT,
    NomeProduto NVARCHAR(255),
    TotalVendas DECIMAL(18, 2),
    MediaVendasDiaria DECIMAL(18, 2)
)
AS
BEGIN
    INSERT INTO @Relatorio (ProdutoID, NomeProduto, TotalVendas)
    SELECT
        p.ProdutoID,
        p.NomeProduto,
        SUM(iv.Quantidade * iv.PrecoUnitario)
    FROM
        Produtos p
    INNER JOIN
        ItensVenda iv ON p.ProdutoID = iv.ProdutoID
    INNER JOIN
        Vendas v ON iv.VendaID = v.VendaID
    WHERE
        YEAR(v.DataVenda) = @Ano AND MONTH(v.DataVenda) = @Mes
    GROUP BY
        p.ProdutoID, p.NomeProduto;

    UPDATE @Relatorio
    SET MediaVendasDiaria = TotalVendas / DAY(EOMONTH(CAST(CAST(@Ano AS VARCHAR) + '-' + CAST(@Mes AS VARCHAR) + '-01' AS DATE)));

    RETURN;
END;
GO

-- Exemplo de uso
SELECT * FROM dbo.ObterRelatorioVendasMensal(2023, 7);
```

**Quando usar mTVF?** Somente quando a lógica é realmente complexa e não pode ser expressa de forma concisa em uma iTVF, e após considerar outras alternativas como views, CTEs ou procedures que retornam resultados.

## Boas Práticas Essenciais

1.  **Evite Funções Escalares em Colunas de Grandes Tabelas em Cláusulas `WHERE` ou `JOIN`**: Isso impede o uso de índices e força um scan na tabela. Por exemplo, `WHERE dbo.MinhaFuncao(Coluna) = 'Valor'` é uma receita para problemas de performance.
2.  **Prefira iTVFs a mTVFs**: Sempre que possível, estruture sua lógica para se encaixar em uma Inline Table-valued Function para aproveitar as otimizações do SQL Server.
3.  **Mantenha a Lógica Simples**: Funções complexas demais podem se tornar difíceis de depurar e manter, além de potenciais problemas de performance.
4.  **Use Schemas Adequados**: Sempre prefixe suas funções com o schema apropriado (e.g., `dbo.MinhaFuncao`).
5.  **Entenda o Impacto na Otimização**: Funções podem "esconder" a intenção do otimizador de consultas, levando a planos de execução subótimos. Use `SET STATISTICS IO ON` e `SET STATISTICS TIME ON`, e analise o plano de execução para entender o que está acontecendo.
6.  **Teste, Teste, Teste\!**: Teste suas funções com diferentes volumes de dados e cenários para garantir que elas performem bem e retornem os resultados esperados.



As funções são uma parte fundamental do arsenal de qualquer desenvolvedor SQL Server. Usadas corretamente, elas podem aumentar drasticamente a eficiência e a manutenibilidade do seu código. Mas lembre-se: com grande poder vêm grandes responsabilidades de otimização\!

Você já teve algum desafio interessante com funções no SQL Server? Compartilhe sua experiência nos comentários\!