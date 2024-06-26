﻿# Web scraping com Python e SQLserver de dados da prefeitura de Indaial
## Objetivo

O presente artigo tem como objetivo descrever um processo de web scraping utilizando para coletar dados de proposições de projetos de vereadores da camara de Indaial Santa catarina. O projeto utilizou três tecnologias fundamentais para hashtag analisededados: 
- Python para coletar e tratar as informações do site da câmara.
- SQLserver para armazenar os dados.
- PowerBI para visulualizar os dados.

## Introdução

Web scraping, também conhecido como extração de dados da web, é o nome dado ao processo de coleta de dados estruturados da web de maneira automatizada. É uma técnica que deve ser utilizada com caultela, apenas alguns sites permitem este tipo de técnica pois processo pode interferir na estabilidade do servidor e causar problemas indesejados. Para isso a regra de antes de usar web scraping é perguntar se existe uma API.

Python é uma ferramente excelente para este tipo de técnica pois a vasta biblioteca desta tecnologio possibilitam extrações e transformações incríveis.
O projeto em si foi realizado acompanhando o passo a passo da maratona de Maratona Python com Power BI do JOVIANO SILVEIRA proposto na mentoria Evolve Data do mentor João Oliveira.

O projeto envolve levantar informações de proposições na camara de vereadores da cidade de indaial Santa Catarina através do site da câmara. Os dados coletados serão enviados para um banco de dados no SQL server e posteriormente visualizados no Power BI. Através deste processo é possível observar os principais veradores propositores da cidade e acompanhar qual dos vereadores aprovaram mais proposições.

## Requisitos

Para realizar o processo é preciso ter instalado os seguintes programas na sua máquina, clicando no link tem o vídeo do Joviano explicando como instalar cada um deles.

- Python Versão 3.6.6
Confesso que tive dificuldades de instalação nessa parte então instalei o anaconda, o vídeo diz claramente para não fazer isso, porém eu sou adepto do que funciona e é prático. A versão 3.6.6 não parece estar mais disponível no site do Python e instalar as bibliotecas exigidas no curso não funciona dá erro.

## Instalar as bibliotecas:

Algumas devem ser instaladas nessa ordem
- python -m pip install pip==20.3.1
- pip install jupyterlab==2.2.9
- pip install numpy==1.18.3
- pip install pandas==1.1.5
- pip install matplotlib==3.3.3
- pip install seaborn==0.11.0
- pip install openpyxl==3.0.5
- pip install xlrd==1.2.0
- pip install folium==0.12.1

## Específicas para webscraping nesse projeto

- pip install beautifulsoup4==4.9.3
- pip install html5lib==1.1
- pip install requests==2.25.1
- pip install lxml==4.6.2
- SQLserver
- Power BI

## Mãos a obra

Instalados os softwares hora de começar a executar o projeto. Primeiro passo e como boa prática em Python é importar as bibliotecas necessárias para o projeto e devem ser feitas da seguinte maneira.

```python
import bs4
import pandas as pd
import requests
import ast
import time
from urllib.error import URLError, HTTPError
from urllib.request import Request, urlopen
import pyodbc
import urllib.request as urllib_request
from urllib.request import urlopen

```

Em seguida cria-se uma variável para inserir a url que iremos utilizar para o processo

```python
url = "http://api.willcode.tech/funcionarios/?USUARIO=USUARIO&SENHA=SENHA_SECRETA&ACAO=LISTAR-TODOS&SHOW_EMPTY"
response = requests.request("GET", url)
```

Informar o agente para ser utilizado no dicionario headers:
```python
agente = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
headers = {'User-Agent': agente}
```


Esta é a função utilizada para chamar a url que iremos utilizar já com os tratamentos de erros.

```python

def ConsultaWebB(url):
    try:
        req = Request(url, headers = headers)
        response = urlopen(req)
        return response.read()
    except:
        pass
```

Utilizando o beautiful soup para tornar a página mais utilizável
```python
def captura_html_pagina(url):
    html = ConsultaWebB(url)
    soup = bs4.BeautifulSoup(html, 'html.parser')
    return soup
```


Utilizando a função para obter os elementos que farão parte do nosso banco de dados
```python
def Cabecalho(html):
    dt = html.find_all('dt')
    dd = html.find_all('dd')
    dic = {}
    for i in range (len(dt)):
        x = dt[i].get_text()
        y = dd[i].get_text()
        dic[x] = y
    return dic 
```


Utilizando a função para levar os dados do web scrapping para um dicionário:
```python
def Conteudo(proposicao, ano):
    url = 'https://www.legislador.com.br//LegisladorWEB.ASP?WCI=ProposicaoTexto&ID=3&TPProposicao=1&nrProposicao='+str(proposicao)+'&aaProposicao='+str(ano)
    html = captura_html_pagina(url)
    dic = Cabecalho(html)
    dic['Proposição'] = proposicao
    dic['Ano'] = ano
    dic['Texto'] = html.p.get_text()
    return dic
```


Utilizando o laço while para automatizar todo o processo de coleta de dados :
```python
def TabelaResultados (iniciar_em, quantidade, ano, erros_admissiveis, segundos_espera):
    ultima_consulta = iniciar_em + quantidade - 1
    
    erros = 0
    
    i = 1
    lista = []
    while iniciar_em <= ultima_consulta and erros <= erros_admissiveis:
        try:
            x = Conteudo(iniciar_em, ano)
            lista = lista + [Conteudo(iniciar_em, ano)]
        except:
            errors += 1
            pass
        time.sleep(segundos_espera)
        iniciar_em+=1
        i+=1
        
    return pd.DataFrame(lista)
```


No SQL server precisamos criar um banco e uma tabela com as colunas para que possamos inserir os dados via web scrapp e despois conectamos com o banco, o código para estabelecer uma conexão entre o python e o SQL é o seguinte:
```python
conn = pyodbc.connect('Trusted_Connection=yes', 
                      driver = '{ODBC Driver 17 for SQL Server}',
                      server = 'JEZANDRE\MSSQLSERVER2', 
                      database = 'ScrapJovi2')

query = '''
    select 
        * 
    from Proposicoes
'''
sql_query = pd.read_sql_query(query,conn)
sql_query

```

Através do seguinte código inseri os dados na tabela do SQL, e lembrem que tive que utilizar um comando para que não houvesse erro ao inserir os dados? Pois então o código tem que ser colocado antes da query ser executada que é o comando Set Language ‘Portuguese’
```python
base = pd.DataFrame(columns=['Reunião', 'Deliberação', 'Situação', 'Assunto', 'Autor', 'Proposição', 'Ano', 'Texto'])
teste = base.append(teste).fillna('')

conn = pyodbc.connect('Trusted_Connection=yes', 
                      driver = '{ODBC Driver 17 for SQL Server}',
                      server = 'JEZANDRE\MSSQLSERVER2', 
                      database = 'ScrapJovi2')
cursor = conn.cursor()

for index, row in teste.iterrows():
    cursor.execute('''
        Set Language 'Português'
    
        INSERT INTO Proposicoes (
            DataReuniao,
            DataDeliberacao,
            Situacao,
            Assunto,
            Autor,
            Proposicao,
            Ano,
            Texto
        ) 
        values(?,?,?,?,?,?,?,?)''',
        row['Reunião'], 
        row['Deliberação'], 
        row['Situação'], 
        row['Assunto'], 
        row['Autor'], 
        row['Proposição'], 
        row['Ano'], 
        row['Texto']
    )
    
conn.commit()
cursor.close()  
    
```

Transformando os passos acima para criar uma função de querys no sql
```python
def SQLSelect(query):
    conn = pyodbc.connect('Trusted_Connection=yes', 
                      driver = '{ODBC Driver 17 for SQL Server}',
                      server = 'JEZANDRE\MSSQLSERVER2', 
                      database = 'ScrapJovi2')
    out = pd.read_sql_query(query, conn)
    return ou
```

Inserindo os dados no sql server
```python
def SQLInsertProposicoes(TabelaProposicoes):
    
    base = pd.DataFrame(columns=['Reunião', 'Deliberação', 'Situação', 'Assunto', 'Autor', 'Proposição', 'Ano', 'Texto'])
    TabelaProposicoes = base.append(TabelaProposicoes).fillna('')

    conn = pyodbc.connect('Trusted_Connection=yes', 
                      driver = '{ODBC Driver 17 for SQL Server}',
                      server = 'JEZANDRE\MSSQLSERVER2', 
                      database = 'ScrapJovi2')

    cursor = conn.cursor()

    for index, row in TabelaProposicoes.iterrows():

        cursor.execute('''
            Set Language 'Português'

            INSERT INTO Proposicoes (
                DataReuniao,
                DataDeliberacao,
                Situacao,
                Assunto,
                Autor,
                Proposicao,
                Ano,
                Texto
            ) 
            values(?,?,?,?,?,?,?,?)''', 

            row['Reunião'], 
            row['Deliberação'], 
            row['Situação'], 
            row['Assunto'], 
            row['Autor'], 
            row['Proposição'], 
            row['Ano'], 
            row['Texto']

        )

    conn.commit()
    cursor.close()
```


Criando a função truncate
```python

def SQLTruncate(NomeTabela):
    
    conn = pyodbc.connect('Trusted_Connection=yes', 
                    driver = '{ODBC Driver 17 for SQL Server}',
                    server = 'JEZANDRE\MSSQLSERVER2', 
                    database = 'ScrapJovi2')
    cursor = conn.cursor()
    
    cursor.execute(f'''
                    TRUNCATE TABLE {NomeTabela}
                    ''')
    conn.commit()
    cursor.close()

```


Iniciando testes para a criação da função de inserção
```python

proposicao = 202
ano = 2021
dados = Conteudo(proposicao, ano)
tabela = pd.DataFrame([dados])
SQLInsertProposicoes(tabela)

```


O objetivo dessa parte é se caso ocorrer alguma eventualidade e o banco de dados precisar ser interrompido por algum motivo, quando o processo se reestabelecer ser possível continuar de onde parou.
```python

```

Se caso o campo estiver vazio a condicional starta o processo a partir do 1

```python
ano = 2021

dados_ano = SQLSelect(f'select Proposicao = max(Proposicao) from Proposicoes where Ano = {ano}')
ultima_proposicao = dados_ano['Proposicao'].loc[0]

proxima_proposicao = int(ultima_proposicao)+1

dados = Conteudo(proxima_proposicao, ano)
tabela = pd.DataFrame([dados])
SQLInsertProposicoes(tabela)
```

Criando a função para a inserção dos dados
```python
def InsereProximaProposicao(ano):

    dados_ano = SQLSelect(f'select Proposicao = max(Proposicao) from Proposicoes where Ano = {ano}')
    ultima_proposicao = dados_ano['Proposicao'].loc[0]
    
    if ultima_proposicao == None:
        proxima_proposicao = 1

    else:
        proxima_proposicao = int(ultima_proposicao) + 1 


    dados = Conteudo(proxima_proposicao,ano)
    tabela = pd.DataFrame([dados])
    SQLInsertProposicoes(tabela)
```

Definindo funções de busca e de gravação assim o processo está quase pronto para inicar a gravar as informações no sql
```python
def ChecaErro(url):
    html = captura_html_pagina(url)
    Erro = html.find_all(class_="col-10")
    if(Erro == None):
        return True
    else: 
        return False

def BuscaGravaDadosAno(ano, quantidade = 10, erros_admissiveis = 2, segundos_espera = 0):

    # erros
    erros = 0
    i = 1
    lista = []
    cont = 1

    while erros <= erros_admissiveis and cont <= quantidade:

        try:
            cont += 1
            url = 'https://www.legislador.com.br//LegisladorWEB.ASP?WCI=ProposicaoTexto&ID=3&TPProposicao=1&nrProposicao='+str(proposicao)+'&aaProposicao='+str(ano)
            if(ChecaErro(url)) == False:
                 InsereProximaProposicao(ano)
            else:
                erros += 1
                pass
                
        except:
            erros += 1
            pass

        time.sleep(segundos_espera)

        # carregamento incremental das variáveis
        i +=1


```

Enfim a última etapa ainda estou enfrentando alguns problemas pois o web scrap está incorporando dados vazios desnecessários, parece que o mesmo identifica dados e não continua o processo
```python
ano_fim = 1996
ano_ini = 1996
anos = ano_fim - ano_ini + 1
for i in range(anos):
    print(i+ano_ini)

 
list(range(1996,1998+1))
ano_inicial = 2006
ano_final = 2021

for i in list(range(ano_inicial, ano_final+1)):
    print('Iniciando gravação dos dados do ano: ',i)
    try:
        BuscaGravaDadosAno(i, quantidade = 3000)
    except:
        pass
print('Inserção finalizada 😁')    
```

Depois de vários erros finalmente consegui terminar o Web Scrap rodando com dois dias aproximadamente. O código precisa ser aperfeiçoado. Mas isso não me impediu de conseguir coletar os dados e gerar um bom artigo e um bom Dashboard que postarei posteriormente.

## Conectando o Power BI no banco de dados

Para importar os dados utilizei o de um SELECT. Como o processo levou muitos dados desnecessários e vazios em data e outros campos utilizei o seguinte comando SQL para que no momento de importação viessem somente os dados que me interessavam:
```SQL
SELECT * FROM Proposicoes 
WHERE DataReuniao <> '1900-01-01'"
```

Assim somente os campos que possuíssem data de reunião registrada seriam importados.
Após este passo segui com o tratamento de dados utilizando o power query. Criei uma dimensão calendário utilizando os passos do Joviano, os quais consistem em criar duas Listas vazias e posteriormente utilizar as seguintes fórmulas. A primeira para inserir a data máxima:


E a segunda para inserir a data mínima:

```M
let
    Fonte = Date.AddYears(
Date.StartOfYear(vDataHoje),
-26)
in
    Fonte

```

Em seguida defini o dCalendario com a seguinte codificação M:

```M
let
    vDataIni = vDataMinima,
    vDataFim = Date.EndOfYear(vDataHoje),
    dias = Duration.Days(vDataFim - vDataIni)+1,
    lista_datas = List.Dates(vDataIni, dias, #duration(1,0,0,0)),
    TabelaDatas = Table.FromList(lista_datas, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Personalização Adicionada" = Table.AddColumn(TabelaDatas, "Datas", each 
            Table.FromRecords({[
                Mes = Date.Month([Column1]),
                AnoNum = Date.Year([Column1]),
                Ano_Mes = Date.ToText([Column1], "yyyy-MM")
            ]})
    ),
    #"Datas Expandido" = Table.ExpandTableColumn(#"Personalização Adicionada", "Datas", {"Mes", "AnoNum", "Ano_Mes"}, {"Datas.Mes", "Datas.AnoNum", "Datas.Ano_Mes"}),
    #"Tipo Alterado" = Table.TransformColumnTypes(#"Datas Expandido",{{"Datas.Ano_Mes", type text}, {"Column1", type date}, {"Datas.Mes", Int64.Type}, {"Datas.AnoNum", Int64.Type}}),
    #"Colunas Renomeadas" = Table.RenameColumns(#"Tipo Alterado",{{"Column1", "Data"}, {"Datas.Mes", "MesNum"}, {"Datas.AnoNum", "AnoNum"}, {"Datas.Ano_Mes", "Ano_Mes"}}),
    #"Linhas Classificadas" = Table.Sort(#"Colunas Renomeadas",{{"Data", Order.Descending}})

in
    #"Linhas Classificadas"

```

O próximo passo foi tratar as informações:
Na coluna Autor foi removida a palvara vereador, os espaços e os pontos finais. Além disso Alguns erros na importação do web sacrap foram corrigida. Algumas informação vieram como “Diversos” e essa palavra foi removida.

```M
let
    Fonte = Sql.Database("JEZANDRE\MSSQLSERVER2", "ScrapJovi2", [Query="Select * from Proposicoes#(lf)Where DataReuniao <> '1900-01-01'"]),
    #"Tipo Alterado" = Table.TransformColumnTypes(Fonte,{{"DataReuniao", type date}, {"DataDeliberacao", type date}}),
    #"Linhas Filtradas" = Table.SelectRows(#"Tipo Alterado", each [DataReuniao] >= vDataMinima and [DataReuniao] <= vDataHoje),
    #"Valor Substituído" = Table.ReplaceValue(#"Linhas Filtradas","Vereador","",Replacer.ReplaceText,{"Autor"}),
    #"Valor Substituído1" = Table.ReplaceValue(#"Valor Substituído",".","",Replacer.ReplaceText,{"Autor"}),
    #"Texto Aparado" = Table.TransformColumns(#"Valor Substituído1",{{"Autor", Text.Trim, type text}}),
    #"Índice Adicionado" = Table.AddIndexColumn(#"Texto Aparado", "Linha", 1, 1, Int64.Type),
    #"Valor Substituído2" = Table.ReplaceValue(#"Índice Adicionado","Diversos","",Replacer.ReplaceText,{"Autor"})
in
    #"Valor Substituído2"

```


O próximo passo é pra mim talvez um dos mais interessantes. Alguns projetos possuíam mais de um autor, o que poderia implicar em uma contagem errada de proposições totais. Para corrigir isso utilizei a opção referência.
Depois criei uma coluna índicie iniciando de 1.
Selecionando as colunas autor e Indice exclui as demais colunas.
Em seguida dividi o nome dos autores em linhas, separando pelo delimitador.
Utilizei a virgula e marquei a opção avançada para selecionar dividir por linhas ao invés de colunas.
Assim o mesmo vereador possuiria um índice que corresponde ao projeto em que ele trabalhou em conjunto com outro vereador.
Depois disso criei outra referencia, porém agora para criar uma dimensão de autores, assim teríamos apenas os nomes dos vereadores sem repetições.

Por fim é só identificar o tipo de dados e corrigir alguns nomes.
O próximo passo é utilizar o power pivot para criar o relacionamento entre as tabelas da seguindo maneiras:

```M
let
    Fonte = fD_Proposicoes[[Linha],[Autor]],
    #"Dividir Coluna por Delimitador" = Table.ExpandListColumn(Table.TransformColumns(Fonte, {{"Autor", Splitter.SplitTextByDelimiter(",", QuoteStyle.Csv), let itemType = (type nullable text) meta [Serialized.Text = true] in type {itemType}}}), "Autor"),
    #"Tipo Alterado" = Table.TransformColumnTypes(#"Dividir Coluna por Delimitador",{{"Autor", type text}}),
    #"Texto Aparado" = Table.TransformColumns(#"Tipo Alterado",{{"Autor", Text.Trim, type text}})
in
    #"Texto Aparado"

```
E a partir daí o próximo passo é criar as medidas e visualizações:
A Primeira medida elaborada foi o cálculo do total de proposições, que era simplesmente contar o total de linhas da tabela fato:

![image](https://github.com/Jezandre/Webscrapping_Indaial/assets/63671761/24fe578c-f04e-431f-b337-d0177f779f4a)


O próximo passo foi calcular as proposições aprovadas:
```dax
3 Proposições Aprovadas = 
CALCULATE(
    [2 Proposições],
    fD_Proposicoes[Situacao]="Proposição Aprovada"
)

```
Os dados para análise servem como avaliação de desempenho dos vereadores, Na primeira aba utilizei códigos e filtros para avaliarmos as relações entre proposições em pauta e proposições aprovadas da qual foi utilizada a seguinte função DAX:
```dax
4 Percentual de Proposições aprovadas = 
DIVIDE(
    [3 Proposições Aprovadas],
    COUNTROWS(
        fD_Proposicoes
    )
)

```


Por fim elaborei um background no FIGMA, ferramenta que amo utilizar:

![image](https://github.com/Jezandre/Webscrapping_Indaial/assets/63671761/8c5bfc03-1878-4dc7-bd41-0d9ac2040dcd)

E Assim ficou o Dashboard:

## Proposições Indaial

[Veja o relatório no Power BI](https://app.powerbi.com/view?r=eyJrIjoiNGJjMTM2ZjAtZjlhMC00NDU4LThmMWUtYWQzOGJiZjMxOWI3IiwidCI6ImI0NGQxYWIzLTFhYjAtNDVlNS04MzU1LWMzNmQzMzYzNGEyYSJ9)

