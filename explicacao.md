<!-- Dockerfiles -->

<!-- products/Dockerfile -->
- Usa a imagem do node, na versão 18, o WORKDIR define o diretorio no qual vai ser copiado o "index.js" do computador para o container, o RUN vai fazer as instalações das dependencias (express) e package.json, EXPOSE está dizendo que o products vai aparecer na porta 3001, e o CMD é o comando que vai rodar o arquivo dentro do container.

<!-- orders/Dockerfile -->
- Usa a imagem do python, versão 3.9, no mesmo sentido que anteriormente WORKDIR define diretorio e o COPY vai copiar o app.py pro diretorio, RUN vai fazer instalações das dependencias do python que seria o flask, redis, a conexão com o db do mysql e os requests, EXPOSE para exportar a porta em que irá aparecer, no caso a 3002 e o CMD vai rodar o arquivo dentro do container.

<!-- payments/Dockerfile -->
- Usa a imagem do PHP CLI versão 8.0, mesma coisa, WORKDIR define a pasta APP e COPY copia o index.php, nesse caso não há RUN pois não tem instalação de pacote adicional, é somente o PHP puro, EXPOSE na porta 3003 e o CMD vai rodar dentro do container.

<!-- docker-compose.yml -->
<!-- Services -->
- Products: API de produtos, vai criar uma imagem a partir do dockerfile na pasta de origem que é a ./products, o ports mapeia a porta exposta com a do container no 3001. Está no network `prova-devops` compartilhando a rede com as demais API's.

- Orders: API de pedidos, vai criar uma imagem a partir do dockerfile na pasta de origem que é a ./orders, o "depends on" diz que os mencionados precisam iniciar primeiro, são eles: products, db e redis.  E o ports mapeia a porta exposta com a do container no 3002. Está no network `prova-devops`  compartilhando a rede com as demais API's.

- Payments: API PHP que consulta a de pedidos, vai criar uma imagem a partir do dockerfile na pasta de origem que é a ./payments, o "depends on" diz que o mencionado precisa iniciar primeiro que é a API de Orders , o ports mapeia a porta exposta com a do container no 3003. Está no network `prova-devops`  compartilhando a rede com as demais API's.

- DB: Banco MySQL com credenciais root/example, tem o "restart:always" para reiniciar caso caia, tem o usuario e senha sendo puxadas do arquivo .env e também utiliza a network `prova-devops` e está na porta 3306:3306

- Redis: Serviço Redis usado como cache, utilizando a ultima imagem do redis, está na porta 6379:6379 e também está compartilhando a rede `prova-devops`.

<!-- Networks -->
- Todos os serviços estão compartilhando a mesma rede internamente, que no caso é a `prova-devops`. É uma rede virtual do tipo Bridge

<!-- Execução -->
- `docker-compose up --build` sobe todos os serviços

- Requisição para `/payment` em `localhost:3003` retorna JSON com dados do pedido e status de pagamento

Testes realizados:
- Docker ps para validar o UP dos containers: 

PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS          PORTS                               NAMES
29abc45fef36   prova-devops-payments   "docker-php-entrypoi…"   11 minutes ago   Up 11 minutes   0.0.0.0:3003->3003/tcp              prova-devops-payments-1
cdc7b13c7127   prova-devops-orders     "python app.py"          11 minutes ago   Up 11 minutes   0.0.0.0:3002->3002/tcp              prova-devops-orders-1
4232a5314b28   prova-devops-products   "docker-entrypoint.s…"   11 minutes ago   Up 11 minutes   0.0.0.0:3001->3001/tcp              prova-devops-products-1
d02b6d430acc   mysql:5.7               "docker-entrypoint.s…"   11 minutes ago   Up 11 minutes   0.0.0.0:3306->3306/tcp, 33060/tcp   prova-devops-db-1
e71ec5d02001   redis:latest            "docker-entrypoint.s…"   11 minutes ago   Up 11 minutes   0.0.0.0:6379->6379/tcp              prova-devops-redis-1

- Verificação dos payments: 

PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> Invoke-WebRequest -Uri http://localhost:3003/payment | Select-Object -ExpandProperty Content
{"status":"paid","order":{"order_id":101,"product_id":1,"quantity":2,"total_price":6000}}

- Verificação do DB: 

PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> docker ps --filter "name=db" --format "{{.ID}}"
d02b6d430acc
PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> docker exec -it d02b6d430acc mysql -uroot -pexample ecommerce
mysql: [Warning] Using a password on the command line interface can be insecure.
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
Server version: 5.7.44 MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SELECT * FROM orders;
+----+------------+----------+-------------+
| id | product_id | quantity | total_price |
+----+------------+----------+-------------+
|  1 |          1 |        2 |        6000 |
|  2 |          1 |        2 |        6000 |
+----+------------+----------+-------------+
2 rows in set (0.00 sec)

mysql> exit;

- Verificação dos produtos em cache pelo Redis: 

PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> docker ps --filter "name=redis" --format "{{.ID}}"
e71ec5d02001
PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> docker exec -it e71ec5d02001 redis-cli
127.0.0.1:6379> keys *
1) "product"
127.0.0.1:6379> get product
"{'id': 1, 'name': 'Notebook', 'price': 3000}"
127.0.0.1:6379>

- Verificação das orders: 

PS C:\Users\MATHEUSSANTOSDEOLIVE\downloads\Prova-DevOps> Invoke-WebRequest -Uri http://localhost:3002/order | Select-Object -ExpandProperty Content
{"order_id":101,"product_id":1,"quantity":2,"total_price":6000}