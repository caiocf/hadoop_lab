# Hadoop Lab com Docker Compose v1

Este projeto sobe um laboratorio Hadoop enxuto com Docker Compose usando imagens `bde2020` baseadas em Hadoop `3.2.1` e Java 8.

O objetivo e disponibilizar um ambiente local simples para estudo de:

- HDFS
- YARN
- ResourceManager
- NodeManager
- History Server

## Arquivos do projeto

- `docker-compose.yaml`: define os servicos do cluster
- `hadoop.env`: concentra as variaveis de configuracao do Hadoop, HDFS, YARN e MapReduce

## Servicos que serao criados

- `namenode`: metadados do HDFS e interface web
- `datanode`: armazenamento de blocos do HDFS
- `resourcemanager`: gerenciamento de recursos e jobs do YARN
- `nodemanager`: execucao de containers e tarefas do YARN
- `historyserver`: historico e logs agregados de aplicacoes

## Pre-requisitos

- Docker Desktop instalado e em execucao
- Docker Compose v2 disponivel via `docker compose`
- Portas locais livres: `9870`, `9000`, `9864`, `8088`, `8042`, `8188`

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
- DataNode UI: http://localhost:9864
- YARN ResourceManager: http://localhost:8088
- NodeManager UI: http://localhost:8042
- History Server: http://localhost:8188

## Configuracao principal

O arquivo `hadoop.env` aplica algumas definicoes relevantes:

- `CORE_CONF_fs_defaultFS=hdfs://namenode:9000`
- `HDFS_CONF_dfs_webhdfs_enabled=true`
- `HDFS_CONF_dfs_permissions_enabled=false`
- agregacao de logs do YARN habilitada
- ResourceManager com persistencia de estado
- limites basicos de memoria e CPU para o NodeManager

Isso torna o ambiente mais simples para laboratorio local e reduz atrito com permissoes no HDFS.

## Persistencia de dados

Os dados sao persistidos em volumes Docker:

- `hadoop_namenode`
- `hadoop_datanode`
- `hadoop_historyserver`

Isso significa que reiniciar os containers normalmente nao apaga os dados. Se voce quiser um ambiente totalmente limpo, use `down -v`.

## Validacao rapida

Verificar se todos os containers subiram:

```powershell
docker compose -f .\docker-compose.yaml ps
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
```

3. Se tudo estiver correto, a saida final deve mostrar:

```text
ola hadoop hadoop docker
```

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

- `http://localhost:9870`: abra `Utilities > Browse the file system` e confira `/input` e `/output-wordcount`
- `http://localhost:8088`: veja a aplicacao em `Applications` no ResourceManager
- `http://localhost:8188`: consulte o historico do job finalizado no History Server

## Observacoes

- Este ambiente e voltado para estudo e desenvolvimento local, nao para producao.
- O cluster e pequeno e roda em um unico host Docker.
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
