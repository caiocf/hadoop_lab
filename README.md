# Hadoop Lab com Docker Compose

Este projeto sobe um laboratorio Hadoop com Docker Compose usando imagens `bde2020` baseadas em Hadoop `3.2.1` e Java 8.

A topologia deste lab e:

- 1 `namenode`
- 3 `datanodes`
- 1 `resourcemanager`
- 3 `nodemanagers`
- 1 `historyserver`

O objetivo e disponibilizar um ambiente local simples para estudo de:

- HDFS
- YARN
- ResourceManager
- NodeManager
- History Server

## Arquivos do projeto

- `docker-compose.yaml`: define os servicos do cluster
- `hadoop.env`: concentra as variaveis de configuracao do Hadoop, HDFS, YARN e MapReduce
- [README-java-app.md](README-java-app.md): guia conceitual de como uma aplicacao Java externa poderia gravar dados no HDFS e submeter processamento para este lab

## Servicos que serao criados

- `namenode`: e o servico central do HDFS. Ele nao guarda o conteudo dos arquivos em si; ele guarda os metadados.
  Exemplo: sabe que o arquivo `/input/teste.txt` existe, em quais blocos ele foi dividido e em quais DataNodes esses blocos estao.
- `datanode1`, `datanode2`, `datanode3`: sao os workers de armazenamento do HDFS. Eles guardam os blocos reais dos arquivos.
  Exemplo: quando voce faz `hdfs dfs -put /tmp/teste.txt /input/`, o conteudo do arquivo vai para os DataNodes, nao para o NameNode.
- `resourcemanager`: e o coordenador do YARN. Ele decide onde um job vai rodar e distribui recursos do cluster, como memoria e CPU.
  Exemplo: quando voce executa um `wordcount`, o ResourceManager escolhe em quais NodeManagers as etapas do job podem rodar.
- `nodemanager1`, `nodemanager2`, `nodemanager3`: sao os workers de processamento do YARN. Cada NodeManager executa os containers e tarefas enviados pelo ResourceManager.
  Exemplo: no `wordcount`, um NodeManager pode executar a fase `map` e outro pode executar a fase `reduce`.
- `historyserver`: guarda e exibe o historico de jobs finalizados e seus logs agregados.
  Exemplo: depois que o `wordcount` termina, voce pode abrir a interface do History Server para ver o job concluido e consultar detalhes da execucao.

Em resumo:

- HDFS = `namenode` + `datanodes` para armazenamento distribuido
- YARN = `resourcemanager` + `nodemanagers` para processamento distribuido

## Pre-requisitos

- Docker Desktop instalado e em execucao
- Docker Compose v2 disponivel via `docker compose`
- Portas locais livres: `9870`, `9000`, `9864`, `9865`, `9866`, `8088`, `8042`, `8043`, `8044`, `8188`

## Como subir o ambiente

No diretorio do projeto, execute:

```powershell
docker compose -f .\docker-compose.yaml up -d
```

Para acompanhar os logs:

```powershell
docker compose -f .\docker-compose.yaml logs -f
```

Para parar os containers:

```powershell
docker compose -f .\docker-compose.yaml stop
```

Para remover os containers da stack:

```powershell
docker compose -f .\docker-compose.yaml down
```

Para remover tambem os volumes persistidos:

```powershell
docker compose -f .\docker-compose.yaml down -v
```

## Como acessar

Depois que os containers estiverem inicializados, os servicos ficam disponiveis nestes enderecos:

- NameNode UI: http://localhost:9870
- HDFS RPC / `fs.defaultFS`: `hdfs://localhost:9000`
- DataNode 1 UI: http://localhost:9864
- DataNode 2 UI: http://localhost:9865
- DataNode 3 UI: http://localhost:9866
- YARN ResourceManager: http://localhost:8088
- NodeManager 1 UI: http://localhost:8042
- NodeManager 2 UI: http://localhost:8043
- NodeManager 3 UI: http://localhost:8044
- History Server: http://localhost:8188

## Guia complementar

Se voce quiser entender como uma aplicacao Java externa poderia usar este cluster, consulte tambem:

- [README-java-app.md](README-java-app.md)

Esse documento complementar mostra, de forma conceitual:

- como uma app Java poderia gravar conteudo no HDFS via `namenode`
- como poderia submeter um processamento via `resourcemanager`
- como ler o resultado final de volta no HDFS

## Configuracao principal

O arquivo `hadoop.env` aplica algumas definicoes relevantes:

- `CORE_CONF_fs_defaultFS=hdfs://namenode:9000`
- `HDFS_CONF_dfs_webhdfs_enabled=true`
- `HDFS_CONF_dfs_permissions_enabled=false`
- `HDFS_CONF_dfs_blocksize=134217728` (`128 MB`)
- `HDFS_CONF_dfs_replication=3`
- agregacao de logs do YARN habilitada
- ResourceManager com persistencia de estado
- limites basicos de memoria e CPU para cada NodeManager

Isso permite demonstrar replicacao real de blocos no HDFS entre 3 DataNodes, ao mesmo tempo em que mantem o ambiente simples para laboratorio local.

Neste laboratorio:

- o tamanho padrao de bloco do HDFS foi definido como `128 MB`
- o fator de replicacao padrao foi definido como `3`

Na pratica, isso significa que:

- arquivos pequenos, como os exemplos deste tutorial, normalmente ocupam apenas 1 bloco
- arquivos maiores que `128 MB` passam a ser divididos em multiplos blocos
- cada bloco pode ser replicado em ate 3 DataNodes diferentes dentro deste cluster

## Tamanho do bloco HDFS neste projeto

O valor usado neste projeto e:

- `134217728` bytes
- `131072 KB`
- `128 MB`

Conceito breve:

- no HDFS, um arquivo e dividido em blocos
- cada bloco pode ser armazenado e replicado entre os DataNodes
- neste projeto, um arquivo so sera quebrado em mais de um bloco quando passar de `128 MB`

Exemplos praticos:

- um arquivo de `10 MB` normalmente fica em `1` bloco
- um arquivo de `128 MB` normalmente fica em `1` bloco
- um arquivo de `300 MB` normalmente fica em `3` blocos

Como aumentar ou diminuir:

- edite o arquivo `hadoop.env`
- altere a linha `HDFS_CONF_dfs_blocksize=134217728`
- o valor deve ser informado em bytes

Exemplos:

- `67108864` = `64 MB`
- `134217728` = `128 MB`
- `268435456` = `256 MB`

Depois da alteracao, recrie os containers:

```powershell
docker compose -f .\docker-compose.yaml up -d
```

Observacao importante:

- a mudanca vale para novos arquivos gravados depois da alteracao
- arquivos antigos continuam com o layout de blocos que ja tinham quando foram escritos

## Persistencia de dados

Os dados sao persistidos em volumes Docker:

- `hadoop_namenode`
- `hadoop_datanode1`
- `hadoop_datanode2`
- `hadoop_datanode3`
- `hadoop_historyserver`

Isso significa que reiniciar os containers normalmente nao apaga os dados. Se voce quiser um ambiente totalmente limpo, use `down -v`.

## Validacao rapida

Verificar se todos os containers subiram:

```powershell
docker compose -f .\docker-compose.yaml ps
```

Verificar os 3 DataNodes registrados no HDFS:

```powershell
docker exec namenode /opt/hadoop-3.2.1/bin/hdfs dfsadmin -report
```

Verificar os 3 NodeManagers no YARN:

```powershell
docker exec resourcemanager /opt/hadoop-3.2.1/bin/yarn node -list
```

Verificar o tamanho padrao de bloco e o fator de replicacao configurados:

```powershell
docker exec namenode /opt/hadoop-3.2.1/bin/hdfs getconf -confKey dfs.blocksize
docker exec namenode /opt/hadoop-3.2.1/bin/hdfs getconf -confKey dfs.replication
```

Interpretacao esperada neste projeto:

```text
134217728
3
```

Entrar no container do NameNode:

```powershell
docker exec -it namenode bash
```

Listar a raiz do HDFS:

```bash
hdfs dfs -ls /
```

Criar um diretorio de teste no HDFS:

```bash
hdfs dfs -mkdir -p /lab/teste
hdfs dfs -ls /lab
```

## Exemplo de teste no HDFS

Este exemplo valida se o cluster consegue criar diretorio, receber arquivo e ler o conteudo no HDFS.

1. Entre no container do NameNode:

```powershell
docker exec -it namenode bash
```

2. Dentro do container, execute:

```bash
hdfs dfs -mkdir -p /input
echo "ola hadoop hadoop docker" > /tmp/teste.txt
hdfs dfs -put -f /tmp/teste.txt /input/
hdfs dfs -ls /input
hdfs dfs -cat /input/teste.txt
hdfs dfs -stat %r /input/teste.txt
hdfs fsck /input/teste.txt -files -blocks -locations
```

3. Se tudo estiver correto, a saida final deve mostrar:

```text
ola hadoop hadoop docker
```

4. O comando `hdfs dfs -stat %r /input/teste.txt` deve retornar:

```text
3
```

5. O comando `hdfs fsck` deve indicar `replication=3`, `Live_repl=3` e `Status: HEALTHY`.

6. Se quiser consultar o tamanho padrao de bloco diretamente no cluster:

```bash
hdfs getconf -confKey dfs.blocksize
```

A saida esperada e:

```text
134217728
```

7. Para validar a distribuicao real de blocos em um arquivo, use:

```bash
hdfs fsck /input/teste.txt -files -blocks -locations
```

Esse comando mostra:

- quantos blocos o arquivo possui
- o fator de replicacao do arquivo
- em quais `DataNodes` cada bloco esta armazenado

Exemplo de interpretacao:

- `1 block(s)` significa que o arquivo foi armazenado em `1` bloco
- `Live_repl=3` significa que esse bloco tem `3` replicas ativas
- a lista dentro de `DatanodeInfoWithStorage[...]` mostra em quais `DataNodes` esse bloco foi gravado

Trecho tipico de saida:

```text
/input/teste.txt 25 bytes, replicated: replication=3, 1 block(s):  OK
0. ... len=25 Live_repl=3 [DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...]]
```

Arquivos pequenos podem aparecer com apenas `1` bloco, o que e esperado. Para ver varios blocos distribuidos entre os `DataNodes`, use um arquivo maior que `128 MB`.

## Exemplo complementar: arquivo de 400 MB e quebra em blocos

Se voce quiser enxergar a divisao real de um arquivo em varios blocos, pode gerar um arquivo maior que `128 MB`.

1. Entre no container do NameNode:

```powershell
docker exec -it namenode bash
```

2. Dentro do container, execute:

```bash
dd if=/dev/zero of=/tmp/arquivo-400mb.bin bs=1M count=400
hdfs dfs -mkdir -p /input-grande
hdfs dfs -put -f /tmp/arquivo-400mb.bin /input-grande/
hdfs fsck /input-grande/arquivo-400mb.bin -files -blocks -locations
```

3. Um resultado tipico sera parecido com:

```text
/input-grande/arquivo-400mb.bin 419430400 bytes, replicated: replication=3, 4 block(s):  OK
0. ... len=134217728 Live_repl=3 [DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...]]
1. ... len=134217728 Live_repl=3 [DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...]]
2. ... len=134217728 Live_repl=3 [DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...]]
3. ... len=16777216 Live_repl=3 [DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...], DatanodeInfoWithStorage[...]]
```

Como interpretar:

- `419430400 bytes` significa que o arquivo tem `400 MB`
- `replication=3` significa que cada bloco deve ter `3` replicas
- `4 block(s)` significa que o arquivo foi dividido em `4` blocos
- `0.`, `1.`, `2.`, `3.` representam o primeiro, segundo, terceiro e quarto bloco
- `len=134217728` significa que o bloco tem `128 MB`
- `len=16777216` significa que o ultimo bloco tem `16 MB`
- `Live_repl=3` significa que existem `3` replicas ativas daquele bloco
- `DatanodeInfoWithStorage[...]` mostra em quais `DataNodes` aquele bloco esta armazenado

Por que aparecem `4` blocos:

- bloco 1 = `128 MB`
- bloco 2 = `128 MB`
- bloco 3 = `128 MB`
- bloco 4 = `16 MB`

Somando:

- `128 + 128 + 128 + 16 = 400 MB`

Por que os mesmos 3 `DataNodes` aparecem em todos os blocos:

- este lab possui exatamente `3` `DataNodes`
- o fator de replicacao padrao tambem e `3`
- por isso, cada bloco acaba tendo `1` replica em cada `DataNode`

Observacao:

- em um cluster maior, com mais `DataNodes`, nem sempre todos os blocos ficariam exatamente no mesmo conjunto de maquinas
- neste lab, como existem apenas `3` `DataNodes`, a tendencia e cada bloco ficar replicado nos tres

## Exemplo de teste do MapReduce

Depois de confirmar que `/input/teste.txt` existe no HDFS, voce pode executar um `wordcount` para validar HDFS, YARN e History Server.

1. Verifique se os containers estao ativos:

```powershell
docker compose -f .\docker-compose.yaml ps
```

2. Confirme que o arquivo de entrada existe no HDFS:

```powershell
docker exec namenode hdfs dfs -ls /
docker exec namenode hdfs dfs -ls /input
docker exec namenode hdfs dfs -cat /input/teste.txt
```

3. Se `/input` ainda nao existir, recrie o teste do HDFS:

```powershell
docker exec -it namenode bash
```

Dentro do container:

```bash
hdfs dfs -mkdir -p /input
echo "ola hadoop hadoop docker" > /tmp/teste.txt
hdfs dfs -put -f /tmp/teste.txt /input/teste.txt
hdfs dfs -ls /input
hdfs dfs -cat /input/teste.txt
```

4. Remova a saida anterior do `wordcount`, caso exista:

```powershell
docker exec namenode hdfs dfs -rm -r -f /output-wordcount
```

5. Execute o job MapReduce:

```powershell
docker exec namenode hadoop jar /opt/hadoop-3.2.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar wordcount /input /output-wordcount
```

6. Leia o resultado no HDFS:

```powershell
docker exec namenode hdfs dfs -cat /output-wordcount/part-r-00000
```

7. A saida esperada deve ser parecida com:

```text
docker  1
hadoop  2
ola     1
```

## Como verificar pela interface web

Depois de executar o `wordcount`, voce pode validar o processamento pelas UIs do cluster:

- `http://localhost:9870`: abra `Datanodes` para confirmar os 3 workers do HDFS e `Utilities > Browse the file system` para conferir `/input` e `/output-wordcount`
- `http://localhost:8088`: veja a aplicacao em `Applications` e os 3 workers em `Nodes`
- `http://localhost:8188`: consulte o historico do job finalizado no History Server
- `http://localhost:9864`, `http://localhost:9865` e `http://localhost:9866`: acompanhe as UIs individuais dos 3 DataNodes
- `http://localhost:8042`, `http://localhost:8043` e `http://localhost:8044`: acompanhe as UIs individuais dos 3 NodeManagers

## Observacoes

- Este ambiente e voltado para estudo e desenvolvimento local, nao para producao.
- O cluster roda em um unico host Docker. Isso ajuda a demonstrar a arquitetura logica do Hadoop, mas nao substitui tolerancia real a falha entre maquinas fisicas.
- A inicializacao completa pode levar algum tempo na primeira execucao por causa do download das imagens e da formatacao inicial dos servicos.

## Problemas comuns

Se algum servico nao subir:

- confirme que o Docker Desktop esta rodando
- verifique se as portas publicadas nao estao em uso
- veja os logs com `docker compose -f .\docker-compose.yaml logs -f`

Se o `wordcount` falhar com `Input path does not exist: hdfs://namenode:9000/input`:

- o diretorio `/input` nao existe no HDFS
- recrie o arquivo com a secao `Exemplo de teste no HDFS`
- confirme com `docker exec namenode hdfs dfs -ls /input`

Se o download pelo `Browse Directory` do NameNode abrir um link interno do Docker e falhar, por exemplo:

```text
http://b64388d0e03a:9864/webhdfs/v1/input/teste.txt?op=OPEN&namenoderpcaddress=namenode:9000&offset=0
```

- esse comportamento e esperado em laboratorios Docker com WebHDFS
- o NameNode apenas redireciona o download para o DataNode que contem o bloco do arquivo
- o navegador passa a tentar acessar o hostname interno do container, que normalmente nao existe fora da rede Docker

Alternativas praticas neste lab:

- ler o arquivo no terminal com `docker exec namenode hdfs dfs -cat /input/teste.txt`
- baixar para o filesystem local do container com `docker exec namenode hdfs dfs -get /input/teste.txt /tmp/`
- copiar do container para o host com `docker cp namenode:/tmp/teste.txt .`

Solucao conceitual para acesso externo via navegador:

- os servicos do HDFS precisam anunciar nomes DNS ou enderecos acessiveis fora do Docker
- em especial, os `DataNodes` tambem precisam anunciar hostnames e portas externas corretas
- nao basta apenas o `namenode`, porque o download via WebHDFS termina no `DataNode`

Se voce estiver migrando de uma versao anterior deste lab com apenas 1 DataNode:

- metadados antigos podem manter o NameNode em `safe mode` ou causar erro de recuperacao no `ResourceManager`
- para recomecar limpo com a nova topologia, use `docker compose -f .\docker-compose.yaml down -v` e depois `docker compose -f .\docker-compose.yaml up -d`

Se quiser recriar tudo do zero:

```powershell
docker compose -f .\docker-compose.yaml down -v
docker compose -f .\docker-compose.yaml up -d
```

## Fontes oficiais

Como este laboratorio usa imagens baseadas em Hadoop `3.2.1`, os links abaixo priorizam a documentacao oficial dessa mesma versao quando aplicavel.

- Apache Hadoop 3.2.1 overview: https://hadoop.apache.org/docs/r3.2.1/
- Apache Hadoop 3.2.1 Single Node Setup: https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/SingleCluster.html
- Apache Hadoop 3.2.1 Cluster Setup: https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/ClusterSetup.html
- Apache Hadoop 3.2.1 Commands Guide: https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/CommandsManual.html
- Apache Hadoop 3.2.1 HDFS Users Guide: https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html
- Apache Hadoop 3.2.1 YARN overview: https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-site/YARN.html
- Apache Hadoop 3.2.1 YARN Commands: https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-site/YarnCommands.html
- Apache Hadoop 3.2.1 MapReduce Tutorial: https://hadoop.apache.org/docs/r3.2.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html
