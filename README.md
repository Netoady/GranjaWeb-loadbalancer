# Atividade 2 - Criação do Load Balancer para Recursos da Aplicação Front-End (GranjaWeb)

**Disciplina:** Gerência e Configuração de Serviços para Internet (GCSI)  
**Aluno:** Francisco Barbosa da Silva Neto  
**Ambiente:** Docker (Ubuntu e Nginx)

---

## Objetivo
Documentar a configuração de um Load Balancer com Nginx utilizando Docker para distribuir requisições entre 5 nós (no1 a no5) servindo a aplicação front-end em React (GranjaWeb), aplicando o algoritmo Round Robin e realizando o repasse dos cabeçalhos de identificação de IP das requisições.

---

## Estrutura do Projeto

```bash
GranjaWeb-loadbalancer/
├── build/
├── default.conf
├── nginx.conf
└── README.md
```

## Pré-requisitos (Preparação do Ambiente)
Para iniciar, é necessário ter o Docker em execução no host e os arquivos compilados do front-end GranjaWeb copiados para a pasta build/ na raiz do projeto:

```bash
# A partir da pasta do projeto GranjaWeb, gerar o build e copiar para build/
npm run build
mkdir -p ~/Documentos/GCSI/GranjaWeb-loadbalancer/build
cp -r dist/* ~/Documentos/GCSI/GranjaWeb-loadbalancer/build/
```
Os arquivos nginx.conf e default.conf ficam na raiz de ~/Documentos/GCSI/GranjaWeb-loadbalancer e são montados no contêiner do Load Balancer via volume (-v).

---

## Passo 1: Criando a Rede Docker

Crie uma rede bridge dedicada para que o Load Balancer (granjaweb-lb) e os 5 nós (no1 a no5) se comuniquem pelos nomes dos contêineres:

```bash
docker network create --driver bridge granjaweb-lb-net
```

---

## Passo 2: Subindo os 5 Nós do Front-end (no1 a no5)

Suba os 5 contêineres na rede granjaweb-lb-net, montando a pasta build/ do GranjaWeb e adicionando o cabeçalho X-Backend-Server para identificar qual nó respondeu a cada requisição:

```bash
for i in 1 2 3 4 5; do
  docker run -d \
    --name "no${i}" \
    --network granjaweb-lb-net \
    -v "$(pwd)/build:/usr/share/nginx/html:ro" \
    nginx:1.31.8-alpine \
    /bin/sh -c "printf 'server {\n    listen 80;\n    location / {\n        root /usr/share/nginx/html;\n        index index.html;\n        try_files \$uri \$uri/ /index.html;\n    }\n    add_header X-Backend-Server \$hostname always;\n}\n' > /etc/nginx/conf.d/default.conf && exec nginx -g 'daemon off;'"
done
```

---

## Passo 3: Configurando e Subindo o Load Balancer (granjaweb-lb)

O nginx.conf define o bloco upstream com os 5 nós (no1:80 a no5:80) utilizando o algoritmo Round Robin (padrão do Nginx) e o formato de log com os IPs da requisição ($remote_addr, $http_x_real_ip, $http_x_forwarded_for) e o nó de destino ($upstream_addr).

O default.conf encaminha as requisições via proxy_pass para o upstream repassando os cabeçalhos Host, X-Real-IP, X-Forwarded-For e X-Forwarded-Proto.

Suba o contêiner do Load Balancer na porta 8080 montando nginx.conf e default.conf:

```bash
docker run -d \
  --name granjaweb-lb \
  --network granjaweb-lb-net \
  -p 8080:80 \
  -v "$(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro" \
  -v "$(pwd)/default.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginx:1.31.8-alpine
```

---

## Passo 4: Verificando e Testando o Load Balancer

Confira se os 6 contêineres (granjaweb-lb e no1 a no5) estão em execução:

```bash
docker ps
```
Teste a distribuição em Round Robin enviando múltiplas requisições para http://localhost:8080/:

```bash
for i in {1..10}; do curl -s -I http://localhost:8080/ | grep -iE "HTTP/|X-Backend-Server"; done
```
Confira os logs do Load Balancer para validar o encaminhamento para os nós e o registro/repasse dos IPs:

```bash
docker logs granjaweb-lb
```

---


## Comandos de Gerenciamento

```bash
# Parar os contêineres
docker stop granjaweb-lb no1 no2 no3 no4 no5

# Iniciar novamente os contêineres
docker start no1 no2 no3 no4 no5 granjaweb-lb

# Remover contêineres e a rede
docker rm -f granjaweb-lb no1 no2 no3 no4 no5
docker network rm granjaweb-lb-net
```