# kafka_oke

## Introdução

<b>Strimzi</b> oferece suporte a <b>Kafka</b> através de operadores do <b>kubernetes</b> para implantação e gerenciamentop dos componentes e dependências de Kafka no cluster.

Os operadores são extensões do k8s que permitem a comunidade de usuários estender os métodos de empacotamento, implantamentação e gerenciar um aplicativo Kubernetes. O sistema de extensão das APIs oferecidos pelo k8s através do CRD, é suportado a partir da versão 1.7; logo disponibilizado nativamente na <b>nuvem do Oracle Cloud e subsequentemente no Kubernetes em PaaS na Oracle (aka OKE)</b>. 

Os operadores Strimzi estendem a funcionalidade do Kubernetes, automatizando tarefas comuns e complexas relacionadas a uma implantação do Kafka. Ao implementar o conhecimento das operações do Kafka no código, as tarefas de administração do Kafka são simplificadas e diminuem a intervenção manual.

Strimzi fornece operadores para gerenciar um cluster Kafka em execução em um cluster k8s. Os operadores disponibilizados são: <b>operador de cluster (implanta e gerencia clusters Apache Kafka, Kafka Connect, Kafka MirrorMaker, Kafka Bridge, Kafka Exporter), operador de entidade, tópicos e usuário</b>.

Para saber mais, segue a referência: https://strimzi.io/docs/operators/master/overview.html

## Pré-Requisito

1. Cluster OKE criado no Oracle Cloud. <br> Para saber mais: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke.htm
2. Instalação do Oracle OCI CLI. <br> Para saber mais: https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm
3. Criação do kubeconfig para acesso ao cluster kubernetes no Oracle Cloud. As indicações e comando para criação são fornecidos após o provisionamento do Kubernetes na console web do Oracle Cloud  

# Instalação

1. <b>Inserir o chart do strimzi ao repositorio</b>
```
$ helm repo add strimzi https://strimzi.io/charts/
```

2. <b>Instale o chart no cluster k8s</b>
```
$ helm install strimzi-kafka strimzi/strimzi-kafka-operator
```

Será produzido uma saída na console similar a esta:

```
NAME: strimzi-kafka
LAST DEPLOYED: Wed Jan 13 09:44:56 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing strimzi-kafka-operator-0.20.1

To create a Kafka cluster refer to the following documentation.
```

3. <b>Liste os pods com o operator strimzi</b>
```
$ alias k=kubectl
$ k get po -l=name=strimzi-cluster-operator
```

4. <b>Crie o arquivo kafka.yaml com o conteudo abaixo descrito</b>

```
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 2.5.0
    replicas: 1
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      log.message.format.version: "2.5"
    storage:
      type: ephemeral    
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
```

5. <b>Crie os recursos (statefulset, pod, service) do kafka no kubernetes</b>
```
$ k create -f kafka.yaml
```

Para validar os recursos criados ao cluster no kubernetes execute
```
$ k get all -l=app.kubernetes.io/name=kafka
```

Será produzido uma saída na console similar a esta:
```
NAME                     READY   STATUS    RESTARTS   AGE
pod/my-cluster-kafka-0   1/1     Running   0          8m24s

NAME                                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/my-cluster-kafka-bootstrap   ClusterIP   10.96.138.84   <none>        9091/TCP,9092/TCP,9093/TCP   8m30s
service/my-cluster-kafka-brokers     ClusterIP   None           <none>        9091/TCP,9092/TCP,9093/TCP   8m30s

NAME                                READY   AGE
statefulset.apps/my-cluster-kafka   1/1     8m34s

```


## Exemplo de operações com o kafka

O nome (my-cluster) setado em metadata.name ao statesetful, será o nome do recurso, cluster e prefixo de nome aos outros objetos.
<br>
Para facilitar as operações criaremos uma variável local com este nome <br>

```
$ export KAFKA_CLUSTER_NAME=my-cluster
```

<b> Listar o tópico </b>
```
$ k exec -ti  my-cluster-kafka-0  -- bin/kafka-topics.sh --list --bootstrap-server $KAFKA_CLUSTER_NAME-kafka-bootstrap:9092

```
<br>

<b> Produzir mensagem </b>
```
$ k exec -ti  my-cluster-kafka-0  -- bin/kafka-console-producer.sh --broker-list $KAFKA_CLUSTER_NAME-kafka-bootstrap:9092 --topic topicosA
```

Será aberto o shell do producer com indicativo <b>(>)</b> na console. Cada mensagem registrada é ingressada por linha. Para sair do producer, Ctrl+C<br><br>
 

<b> Consumir mensagem produzidas anteriormente</b> 
```
$ k exec -ti  my-cluster-kafka-0 -- bin/kafka-console-consumer.sh --bootstrap-server $KAFKA_CLUSTER_NAME-kafka-bootstrap:9092 --topic topicosA --from-beginning
```
