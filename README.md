# AzureDataFactoryCOVID	

<br />Implementa uma solução de Engenharia de dados para insights da COVID19. O projeto utiliza exclusivamente a Azure, com o Data Factory como principal componente. A extração ocorre em diferentes fontes utilizando conectores e scripts Json. 
Já a transformação usa HDInsights, Databricks e Data Factory que também é responsável pelo carregamento, orquestração, automatização e monitoramento dos pipelines e data flows.<br />
Dados do Centro Europeu de Controle de Doenças (ECDC): casos confirmados, mortalidade, hospitalização (ICU cases), testagem e EUROSTAT: população por idade.

## Arquitetura
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/Arquitetura.JPG"/>
</p>

### Recursos utilizados no Azure:
* Blob Storage para armazenamento de logs, scripts e tudo o que não for enviado ao data lake.
* Data Lake Gen2 para armazenar dados brutos, processados e de pesquisa.
* Azure SQL Database e Server para gerenciamento geral.
* Data Factory para ingestão, transformação, cópia a base de dados SQL e orquestração.
* HDInsights e DataBricks como soluções alternativas de transformação.
* Azure Monitor e Log Analytics para monitoração dos pipelines, flows e triggers.
* Power BI para exibição de relatórios.

### Preparo preliminar do ambiente
* Criar uma conta Azure, uma subscription e um resource group usado como um contêiner do projeto.
* Criar duas contas de armazenamento conforme mencionado acima.
* Criar um SQL Database e um servidor caso não exista.
* Criar um Data Factory.
* Criar um dashboard com os recursos do projeto.

### ETL
#### Extração e ingestão
A ingestão dos dados ocorre de duas formas distintas, uma utilizando um conector de http sem armazenamento e a outra salvando os dados de população diretamente no blob usando scripts Json. A atividade copy do Data Factory é usada em ambos.
Para a construção do primeiro pipeline é necessário construir datasets e linked services.
<br />O pipeline usa algumas atividades: checa se o arquivo existe, verifica requisitos conhecidos como o número de colunas entre outros e usa um condicional que copia o arquivo e o deleta do armazenamento se verdadeiro ou envia um email informando a ausência caso falso.<br />
Um trigger de evento, que ativa o pipeline sempre que o arquivo chega, é usado para que o fluxo seja automático.

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/pl1.JPG"/>
</p>

O segundo pipeline também usa a atividade de cópia, porém com um conteúdo dinâmico escrito em JASON que contém a URL absoluta, relativa e o nome dos quatro arquivos que serão armazenados.
Um trigger de agendamento é adicionado ao Pipeline.

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/pl2.JPG"/>
</p>

#### Transformação
A primeira transformação se dá no do arquivo cases and deaths. Para isso, é necessário criar um Dataflow. A primeira coisa a fazer é inserir a fonte de dados.
Neste caso o próximo item é um filtro, que retira os demais continentes do processo. Usamos o Select para alterar o esquema, como a retirada da coluna continente que não é mais relevante. O Pivot foi usado para transformar importantes dados de linha em coluna. O Lookup transformou dados de código dos países e os inseriu. Por fim o Sink, que transforma todo o Dataflow em arquivo. 
Um novo pipeline é feito para rodar o Dataflow, conforme abaixo:  

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/df1.JPG"/>
</p>

O próximo dataflow de admissões hospitalares gera duas saídas distintas, arquivos semanais e diários. O início da transformação acontece basicamente como no anterior,
o Conditional Split, baseado em uma condição da coluna "indicator" distribui os dados entre dois flows, o diário e o semanal. O Join permite juntar os fluxos de dados em um, assim é possível melhorar a nomenclatura das datas.
O Sort que ordenou a coluna "reported_year_week" de forma descendente e "country" ascendente. Como no anterior, o sink gera a saída.
Novamente um novo pipeline é feito para rodar o dataflow, conforme abaixo:  

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/df2.JPG"/>
</p>

A fim de melhor explorar as ferramentas do Azure, o HDInsights é usado na próxima transformação. Para isso foi necessário criar uma managed identity. Exigência para registrar o recurso. 
O cluster criado foi do tipo Hadoop.
Os dados do arquivo teste foram transformados via script. Informações não contempladas nas transformações anteriores como o número de testes em execução e os casos confirmados foram levados em conta.
Três bancos de dados foram criados, um para manter as informações de pesquisa, um de dados brutos e outro de dados processados. De forma conclusiva, uma tabela é criada com as transformações desejadas, a exemplo do código do país.
Por fim um pipeline no Data Flow é criado usando o script Hive como referência que gera o arquivo. 

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/hdi.JPG"/>
</p>

O último tipo de transformação se deu no Databricks. Desta vez o arquivo de população será transformado. O arquivo continha informações sobre a população total e o percentual enquadrado por faixas etárias.
Cria-se o recurso Databricks, um cluster e um Active Directory. Neste último um app, que contém as chaves client, tenant e secret usadas na transformação.
No workspace do Databricks, um script Python é carregado fazendo as devidas montagens. A transformação em si, é também um script Python que mudará o ano, país, código e faixas etárias.
Um pipeline é construído no Dataflow com o cluster ligado para as devidas transformações.

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/dtb1.JPG"/>
</p>

#### Carregamento para o Azure SQL
Os arquivos cases and deaths, hospital admissions e testing data devidamente transformados são enviados via atividade de cópia do Dataflow para o SQL do Azure conforme exemplo abaixo:

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/tb1.JPG"/>
</p>

## Orquestração, automatização e monitoramento
Por fim é criado um pipeline pai automatizado por triggers sucessivas e sequenciais que executam os demais pipelines como orquestrador.

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/orq.png"/>
</p>

Foram criados alertas no monitor do Data Factory quando algum dos pipelines tiver falha. Um monitor de sucesso de pipelines das últimas 24h foi inserido no dashboard para acompanhamento.

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/mt.JPG"/>
</p>

## Relatórios
O Power BI foi usado para gerar o relatório dos arquivos tratados. 

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/0001.jpg"/>
</p>

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/0002.jpg"/>
</p>

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureDataFactoryCOVID/blob/main/factory/0003.jpg"/>
</p>

