# Ideia de aplicacao Java usando este Hadoop Lab

Este documento mostra, de forma conceitual, como uma aplicacao Java poderia usar este laboratorio Hadoop para:

- salvar conteudo no HDFS
- processar um relatorio com MapReduce
- ler o resultado de volta no HDFS

O objetivo aqui nao e entregar uma aplicacao pronta, mas dar ao leitor uma ideia clara de integracao.

## Visao geral

Uma aplicacao Java externa normalmente conversa com estes servicos:

- `namenode`: para operacoes de metadados e acesso ao HDFS
- `resourcemanager`: para submeter jobs MapReduce no YARN

Fluxo tipico:

1. a aplicacao grava um arquivo no HDFS
2. a aplicacao submete um job MapReduce
3. o YARN executa o job nos `nodemanagers`
4. a saida do job e gravada no HDFS
5. a aplicacao le o resultado final

## Servicos envolvidos

- `namenode`: ponto de entrada logico para HDFS
- `datanode1`, `datanode2`, `datanode3`: armazenam os blocos reais dos arquivos
- `resourcemanager`: coordena a execucao do job
- `nodemanager1`, `nodemanager2`, `nodemanager3`: executam as tarefas de processamento
- `historyserver`: permite consultar o historico do job depois da execucao

## Exemplo 1: salvar conteudo no HDFS

Servico usado: `namenode`

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

import java.io.BufferedWriter;
import java.io.OutputStreamWriter;
import java.nio.charset.StandardCharsets;

Configuration conf = new Configuration();
conf.set("fs.defaultFS", "hdfs://namenode:9000");

FileSystem fs = FileSystem.get(conf);
Path destino = new Path("/input/teste.txt");

try (BufferedWriter writer =
         new BufferedWriter(
             new OutputStreamWriter(fs.create(destino, true), StandardCharsets.UTF_8))) {
    writer.write("ola hadoop hadoop docker");
    writer.newLine();
}
```

O que esse trecho faz:

- conecta no HDFS pelo servico `namenode`
- cria o arquivo `/input/teste.txt`
- grava uma linha de texto

## Exemplo 2: processar um relatorio com MapReduce

Servicos usados:

- `namenode`
- `resourcemanager`

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.map.TokenCounterMapper;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.reduce.IntSumReducer;

Configuration conf = new Configuration();
conf.set("fs.defaultFS", "hdfs://namenode:9000");
conf.set("mapreduce.framework.name", "yarn");
conf.set("yarn.resourcemanager.address", "resourcemanager:8032");

Job job = Job.getInstance(conf, "relatorio-wordcount");
job.setJarByClass(RelatorioWordCountApp.class);

job.setMapperClass(TokenCounterMapper.class);
job.setReducerClass(IntSumReducer.class);

job.setOutputKeyClass(Text.class);
job.setOutputValueClass(IntWritable.class);

FileInputFormat.addInputPath(job, new Path("/input"));
FileOutputFormat.setOutputPath(job, new Path("/output-relatorio"));

boolean ok = job.waitForCompletion(true);
if (!ok) {
    throw new RuntimeException("Falha no processamento");
}
```

O que esse trecho faz:

- le os arquivos em `/input`
- executa um `wordcount`
- grava o resultado em `/output-relatorio`

## Exemplo 3: ler o resultado do relatorio

Servico usado: `namenode`

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;

Configuration conf = new Configuration();
conf.set("fs.defaultFS", "hdfs://namenode:9000");

FileSystem fs = FileSystem.get(conf);
Path resultado = new Path("/output-relatorio/part-r-00000");

try (BufferedReader reader =
         new BufferedReader(
             new InputStreamReader(fs.open(resultado), StandardCharsets.UTF_8))) {

    String linha;
    while ((linha = reader.readLine()) != null) {
        System.out.println(linha);
    }
}
```

Saida esperada para o exemplo usado neste lab:

```text
docker  1
hadoop  2
ola     1
```

## Leitura rapida da arquitetura

No HDFS:

- a aplicacao fala com o `namenode` para descobrir onde o arquivo deve ser criado ou lido
- o conteudo real e armazenado nos `datanodes`

No YARN:

- a aplicacao fala com o `resourcemanager` para submeter o job
- o `resourcemanager` distribui o processamento para os `nodemanagers`

## Dependencias Java

Em um projeto Java, normalmente voce precisaria das bibliotecas do ecossistema Hadoop, como:

- `hadoop-common`
- `hadoop-hdfs`
- `hadoop-mapreduce-client-core`
- `hadoop-yarn-client`

As versoes devem ser compativeis com o Hadoop usado no cluster.

Como este lab usa Hadoop `3.2.1`, o ideal e alinhar as dependencias Java com essa familia de versao.

## Observacao importante sobre rede

Os nomes de servico usados nos snippets:

- `namenode`
- `resourcemanager`

funcionam quando a aplicacao Java esta:

- na mesma rede Docker do cluster
- ou com resolucao DNS/hosts preparada para esses nomes

Se a aplicacao estiver realmente fora do ambiente Docker, normalmente voce precisara:

- expor as portas necessarias
- usar hostname/IP acessivel de fora
- ajustar a configuracao para esse endereco externo

## Resumo

Como ideia para o leitor, o fluxo de uma aplicacao Java seria:

1. escrever dados no HDFS via `namenode`
2. submeter um job via `resourcemanager`
3. deixar o cluster processar nos `nodemanagers`
4. ler o resultado final no HDFS

Este lab ja permite demonstrar essa arquitetura de forma didatica.
