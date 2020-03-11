# Wordpress for Kubernetes

## Disclamer
1. Meu minikube estava desatualizado, e não percebi, o que levou a problemas ao tentar usar o Nginx ingress controller para comunicar com o servicor FCGI via Ingress. Depois de muito debug, atualizei o minikube para a versão mais nova (1.17), o que ocasionou o problema 2.
2. O operator mysql da oracle quebrou na virada de versão 1.16+ porque essa parou de suportar versões apps/v1beta1 utilizada no projeto pra criar o cluster como discutido na [issue](https://github.com/oracle/mysql-operator/issues/307). Há um PR aberto pra concertar isso.
3. O minikube tem muitas peculiaridades com relação a exposição de serviços, depois de muitos problemas com isso resolvi levantar um cluster a Azure.

## Projeto

Com o objetivo de ser totalmente declarativo foi adotado o kustomize, facilitando assim o deploy de toda a infraestrutura.

Com um pocuo mais de tempo estruturando poderiamos chegar ao ponto de executar: `kubectl -k .` e toda a infraestrutura seria levantada, ou até deixar isso a cargo de uma solução de CD.

Poderíamos ainda replicar esse ambiente com pouquíssimo esforço usando o conceito do kustomize de overlays.

### Executando o projeto

O projeto foi executado no AKS com a versão do kubernetes 1.14.8.

NOTE: todos os comandos assumem que vocês está na pasta raiz do projeto.

Primeiramente instale o ingress controller executando:
```
kubectl -k ingress-controller
```

Agora vamos levantar o banco MySQL
```
kubectl -k mysql-cluster
```

NOTE: essa foi a única parte que não foi possível usar o kustomize. Por algum motivo desconhecido o deployment não consegue completar pela falta de um secret quando é rodado pelo kustomize. Há alguns issues relacionados sem solução.

E agora instalar o cert manager (a flag validate=false foi usada porque nos manifestos de deployment oficiais dele está com problema de compatibilidade com versões anteriores a 1.15.6, mas não é um problema quebrante).
```
 kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.13.1/cert-manager.yaml
```

Para criar um certificate issuer:
```
kubectl -k cert-manager
```

Edite o arquivo `ingress.yaml` e substitua o host por um domínio que você tenha acesso de administrador, depois crie um A record para ele apontando para o IP externo do balanceador de carga do congress controller. Você pode verificar isso executando:
```
kubectl get svc -n ingress-controller
```

Agora podemos criar nosso wordpress com HTTPS:
```
kubectl -k wordpress
```

A infra de logs pode ser criada primeiro instalando o elk-operator:
```
kubectl -k elk-operator
```

E depois execuntando:
```
kubectl -k logs
```

### Wordpress

Para o Wordpress foi escolhida a imagem do Dockerhub `wordpress:php7.4-fpm`, por ser a versão mais nova com FPM. A versão Alpine não foi utilizada por motivos de não haver uma boa manutenção na imagem, o que poderia abrir brechas de segurança na imagem.

Como webserver foi utilizado o NGINX ingress controller, instalado via helm no cluster, também presente no projeto gerado através do comando 
```
helm template nginx stable/nginx-ingress --namespace nginx-ingress-controller
```

O NGINX foi escolhido por ter uma boa performance, ser amplamente utilizado pela comunidade, ser muito customizável e ter suporte para FastCGI.

Houveram 2 problemas com a execução dessa tarefa
1. O FastCGI tem um módulo de segurança do lado do servidor que impede de arquivos estáticos como CSS e HTML serem acessados por scripts. Depois de muita procura não consegui achar um workarround razoável que não necessitasse desativasse essa feature de segurança. Aúnica razoável foi [essa](https://robertbasic.com/blog/php-fpm-security-limit-extensions-issue/), mas por algum motivo que não consegui desvendar a tempo, algumas annotations do Ingress não estavam surtindo efeito na geração do `nginx.conf`.
2. Não consegui nenhuma forma razoável de ter acesso de administrador de um domínio, o que impossibilitou fazer o requisito de terminação SSL. Se tivesse conseguido isso faria através das annotations do próprio NGINX ingress controller.
3. Tentei aidna uma outra abordagem, colocar um container nginx como sidecar do wordpress com diferentes regras de roteamento, inclusive compartilhando o volume `/var/www/html`, para arquivos que não fossem `.php` serem servidos diretamente pelo nginx, mas sem sucesso, os scripts continuavam sendo bloqueados.

### Comunicação das equipes

Considerando que a empresa já utiliza o Slack como ferramenta de comunicação interna, criaria um canal privado e convidaria os integrantes da equipe terceira como convidados e apenas com permissão a esse canal.

Se apenas a ferramenta de texto não fosse suficiente poeriamos implementar uma ferramenta de tickets, como Freshdesk, que tem boa integração com o slack. Dessa forma poderiamos metrificar todas as entregas da equipe terceirizada, sem burocratizar demais o dia a dia.

### Gitflow

Para esse projeto, utilizaria o flow GitOps, visto que o código em si é basicamente apenas infraestrutura, o GitOps pode nos ser muito útil facilitando a montagem de uma infra gradual com ambiente de Dev, QA e produção transicionando entre eles apenas a pasta de temas do Wordpress, que seria também versionada no Git.
Mudanças aos temas seriam apenas permitidas no ambiente de desenvolvimento, no momento de publish o desenvolvedor do site iria acessar uma plataforma de CD e clicar num botão de publicar, com isso uma pipeline seria iniciada fazendo o backup do banco de dados e do diretório `/var/www/html`, esses backups seriam remontados no ambiente de QA, que tem banco e diretório `/var/www/html` em modo de apenas leitura. Também na plataforma de CD no momento da aprovação seria iniciada uma pipeline igual a primeira para publicar o site no ambiente de produção, também em formato de apenas leitura, mas com uma infraestrutura mais robusta.

### Banco de dados

O grande desafio de rodar aplicações statefull (aplicações que guardam estado) em containers é justamente o seu maior benefício, ser efêmero. Aplicações desse tipo precisam de persistência, em especial banco de dados necessitam disso, para não perder seus dados. Podemos mitigar isso mapeando um volume na máquina que ele está rodando, mas isso ainda não é totalmente garantia de que não haverá perda de dados, visto que no Kubernetes as máquinas também são efêmeras, e de modo geral pode-se rodá-las até mesmo sem disco. Então no caso de necessitar rodar o banco de dados no Kubernetes em produção eu usaria um PersistantVolume, que as principais provedoras de cloud tem implementações dessa abstração do kubernetes, incluindo a Azure com o AzureFiles, dessa forma poderíamos garantir a persisência dos dados mesmo que o pod ou o nó sejam perdidos e também, com a ajuda de um bom operator, seja um terceiro, open source ou até mesmo desenvolvido internamente, que facilita a manipulação/gestão do cluster. Além disso faria um deploy de alta disponibilidade com 3 réplicas do banco 1 master 2 réplicas, com Pod Anti-afinity para garantir que 2 pods do sejam iniciados na mesma máquina e faria ainda backups diários em um sistema de arquivos (como um bucket), e usaria o mysql-router como "loadbalancer" do cluster.

Caso rodar o banco dentro do Kubernetes não seja um requisito, com certeza optaria por um banco de dados gerenciado pela provedora de cloud, principalmente com uma equipe pequena sem especialistas em banco de dados.

### Informações sensíveis

Para armazenar informações sensíveis, como certificados, senhas, etc. de forma segura faria uso de um vault, como o Azure Key Vault e montaria eles em segredos do Kubernetes pelo [gerador de segredos do Kustomize](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/secretGeneratorPlugin.md), dessa forma pra acessar o segredo a pessoa teria que ter acesso ao Key Vault, eles não precisariam ficar no repositório do git e um deploy em outra infraestrutura continuaria declarativo, sem a necessidade de uma ação imperativa de aplicar o secret manualmente, apenas das credenciais da Azure do colaborador, ou seja, caso o colaborador tenha as credenciais para acessar o secret, ele poderia fazer o deploy de toda a infraestrutura com apenas um comando (isso também se aplica a um CD).

### Gerenciamento de logs

Para colher logs foi escolhido a stack ELK, por ser extensível, flexível, simples manutenção, além de o Kibana oferecer possíbilidade de integração com SSO, o que nos daria a possibilidade de integrá-lo com o LDAP da empresa, facilitando também o complience e concessão de acessos aos recursos.

Foi usado o novo operator da Elastic para rodar o cluster de elasticsearch e o Kibana.

O operator do Elasticsearch teve seus manifestos gerados via `helm template`.

Não houve tempo de fazer o deploy/configuração do logstash e do Filebeat, pois os problemas com o minikube e com o FastCGI consumiram boa parte do tempo.
