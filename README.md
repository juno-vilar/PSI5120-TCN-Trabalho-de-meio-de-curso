# PSI5120-TCN-Trabalho-de-meio-de-curso
Trabalho de meio de curso da matéria PSI5120 - Tópicos de Computação em Nuvem de pós graduação oferecida na Escola Politécnica da USP.

## Horizontal Pod Autoscaler (HPA) do Kubernetes utilizando o Minikube
Este documento contém o roteiro para implantanção e experimentação de um servidor web do tipo php-apache que utiliza o autoescalonamento horizontal (Horizontal Pod Autoscaler), ferramenta crucial na automatização da gestão de microsserviços em nuvem. 
Com base em métricas como porcentagem de uso de CPU, quantidade de uso da memória, entre outras, o HPA consegue alocar horizontalmente as cargas demandadas por serviço, através da criação ou deleção de réplicas dos Pods ativos.
Neste repositório, o HPA é mostrado operando com Minikube em um cluster local.

## Pré-requisitos
- Ubuntu Server ou Desktop, versão 22.04 ou Windows 10
- Docker Desktop
- ``minikube``
- ``kubectl``

## Roteiro

1. Inicie o Minikube com:
```
minikube start --driver=docker
```
2. Adquira e habilite o metrics-server a partir dos comandos:
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/release/latest/download/components.yaml
minikube addons enable metrics-server
```
3. Crie e aplique o deployment e o service:
```
kubectl apply -f manifestos-yaml/php-apache.yaml
```
4. Crie o HPA:
```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
5. Gerar requisições para a atuação do HPA:
```
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```
   OBS: Rodar este código em um terminal alternativo.
6. Para observar o HPA agindo:
```
kubectl get hpa php-apache --watch
ou
kubectl get deployment php-apache
```
7. Para observar o autoscaling (autoscalonamento) em métricas múltiplas ou customizáveis:
```
kubectl get hpa php-apache -o yaml > manifestos-yaml/hpa-v2.yaml
```
  Possíveis métricas:
  * Memória:
``` YAML
metrics:
 - type: Resource
   resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```
  * Número de réplicas de Pods criados:
``` YAML
metrics:
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
```
  * Objetos - Métricas customizadas associadas a um objeto do cluster
``` YAML
metrics:
  - type: Object
    object:
      metric:
        name: request-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: <Número-a-dividir-pelos-Pods>
```
  * External - Métricas externas ao Kubernetes
``` YAML
metrics:
  - type: External
  external:
    metric:
      name: queue_messages_ready
      selector:
        matchLabels:
          queue: "worker_tasks"
    target:
      type: AverageValue
      averageValue: 30
```
## Limpeza
```
kubectl delete -f manifestos-yaml/
```

# HPA do Kubernets no AWS EKS
Esta parte do repositorio diz respeito a implantação de um servidor web do tipo php-apache com o HPA, porém agora em um cluster AWS EKS.

## Pré-requisitos
- Ubuntu Server ou Desktop, versão 22.04 ou Windows 10
- AWS CLI (``aws configure``)
- ``kubectl``
- ``eksctl``

## Roteiro
1. Crie o cluster EKS diretamente ou com um manifesto YAML:
```
eksctl create cluster --name cluster-eks --region region-code
eksctl create cluster -f manifestos-yaml/cluster-eks.yaml
```
2. Configure o kubectl para o Cluster EKS criado:
```
aws eks update-kubeconfig --name my-ekis-cluster --region us-east-1

```
3. Instale e habilite o metrics-server:
```
eksctl create addon --name metrics-server --cluster hpa-eks --region us-east-1 --force
kubectl -n kube-system rollout status deployment/metrics-server --timeout=5m
```
4. Crie e aplique o deployment e o serviço:
```
kubectl apply -f manifestos-yaml/php-apache.yaml
```
5. Crie aplique o HPA:
```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
6. Gerar requisições para a atuação do HPA:
```
kubectl run -it --rm load-generator --image=busybox --restart=Never -- /bin/sh -c 'while true; do wget -q -O- http://php-apache; done'
```
7. Para observar o HPA:
```
kubectl get hpa php-apache --watch
```

## Limpeza
```
kubectl delete -f manifestos-yaml/
kubectl delete deployment.apps/php-apache service/php-apache horizontalpodautoscaler.autoscaling/php-apache
eksctl delete cluster --name hpa-eks --region us-east-1
```
