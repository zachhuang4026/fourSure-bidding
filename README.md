# Deployment Instruction
# Team: FourSure!

- All the containers are upload to Docker Hub (Due to SVN storage limitaion)
- So before running below instructions, please make sure to log-in into Dokcer Hub

### Log in into Docker Hub
```shell script
docker login
```

## Creating/Configuring Containers with Docker Network
### Set up a network for the bidding service (ebay)
```shell script
docker network create --subnet=172.20.0.0/16 ebay
```

### Create FlaskServer Container
```shell script
docker run -it -p 5000:5000 --net ebay --ip 172.20.0.2 --name FlaskServer alpine
```
### Create API Gateway Container
```shell script
docker run -it -p 80:80 --net ebay --ip 172.20.0.3 --name apiGateway alpine
```

### Create AccountService Container
```shell script
docker run -p 5001:5001 --net ebay --ip 172.20.0.4 --name AccountService -e POSTGRES_PASSWORD=secret_password -d postgres:15.1-alpine
```

### Inspect network -> Should show all attached Docker containers
```shell script
docker network inspect ebay
```

## Run the Web Server Container
```shell script
docker run -it -p 5000:5000 --net ebay --ip 172.20.0.2 --name FlaskServer adamlim1/flask_server:latest
cd webService
python3 app.py
```

## Run the API Gateway Container
```shell script
docker run -it -p 80:80 --net ebay --ip 172.20.0.3 --name apiGateway adamlim1/api_gateway:latest
cd apiGateway
python3 app.py
```

## Run the Item Service Container
- Run the Item DB
```shell script
docker pull zachhuang4026/itemdb:1.1
docker run -p 27017:27017 --net ebay --ip 172.20.0.6 --name itemDB zachhuang4026/itemdb:1.1
```

- Run the Item Server
```shell script
docker pull zachhuang4026/itemserver
docker run -it -p 8080:8080 --net ebay --ip 172.20.0.5 --name itemServer zachhuang4026/itemserver
```

## Run the Account Service Container
- Run the container 
```shell script
docker run -it -p 5001:5001 --net ebay --ip 172.20.0.4 --name AccountService adamlim1/account_service:latest
cd accountService
```

- Create Account DB
```shell script
psql --username postgres
create database accounts; # Enter accounts db via \c accounts
exit
python3 db_setup.py
```

- Run the Flask app
```shell script
python3 app.py
```

## Run the RabbitMQ Container
```shell script
docker run -d --hostname ebay-rabbitmq --name ebayRabbitMQ --net ebay --ip 172.20.0.7 -p 15672:15672 -p 5672:5672 rabbitmq:3-management
```

## Run the Shopping Service Container
- Run the container
```shell script
docker run --name Shopping -e POSTGRES_PASSWORD=mysecret -p 3306:3306 -p 5432:5432 --net ebay --ip 172.20.0.8 -d hannahchiu88/shopping
docker exec -it Shopping /bin/bash
```

- Create Shopping DB
```shell script
psql --username postgres
create database shopping;
\c shopping;
CREATE TABLE shoppingCart (id varchar(36) primary key, userId varchar(36), itemId varchar(36));
CREATE TABLE watchlist (id varchar(36) primary key, userId varchar(36), itemId varchar(36));
\q
export CLASSPATH=/opt/rabbitmq-java-client/amqp-client-5.6.0.jar:/opt/json-simple-1.1.1.jar:/opt/postgresql-42.2.8.jar:/opt/rabbitmq-java-client/slf4j-api-1.7.25.jar:/opt/rabbitmq-java-client/slf4j-simple-1.7.25.jar:.
cd /src
java RPCServer 172.20.0.7
```

## Run the Notify Service container
```shell script
docker run -it --name Notification --net ebay --ip 172.20.0.9 hannahchiu88/notification /bin/bash
source venv/bin/activate
cd notifyService/src/
python app.py
```

## Run the Auction Service container
- Create Auction DB
```shell script
docker pull yichiehchen/auctionservicepostgres:latest
docker run -d --name auctionServicePostgres --net ebay --ip 172.20.0.11 -e POSTGRES_PASSWORD=abc123 -p 2345:2345 yichiehchen/auctionservicepostgres:latest
docker exec -it auctionServicePostgres /bin/bash
```

- Create Auction DB
```shell script
psql -U postgres
create database auction;
\q
```

- Populate Auction DB
```shell script
psql -U postgres -d auction -W
abc123
\i auction.sql
\q
```

- Update port config
```shell script
vi /var/lib/postgresql/data/postgresql.conf
```

```
# find the line “#port = 5432”, uncomment the line and change it to “port = 2345” 
# (see below, this section is closer to the beginning of the file):

#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'
					# comma-separated list of addresses;
					# defaults to 'localhost'; use '*' for all
					# (change requires restart)
#port = 5432				# (change requires restart)
max_connections = 100			# (change requires restart)
```

- Restart the container
```shell script
docker restart auctionServicePostgres
docker exec -it auctionServicePostgres /bin/bash
psql -U postgres -d auction -p 2345 -W
abc123
```

- Delete the data
```shell script
DELETE FROM bids;
DELETE FROM auctions;
```

- Run the server (open a new terminal)
```shell script
docker pull yichiehchen/auctionservice:latest
docker run -di --hostname auction-service --name auctionService --net ebay --ip 172.20.0.10 yichiehchen/auctionservice:latest
docker exec -it auctionService /bin/bash
```

- Run the server
```shell script
java -jar auctionService-1.0.jar 172.20.0.11 172.20.0.7 172.20.0.3:80
```

- Open another new terminal
```shell script
docker exec -it auctionService /bin/bash
java -jar startEndAuctions-1.0.jar 172.20.0.11 172.20.0.7 172.20.0.3:80
```

## Initial Auction data (Requires API Gateway and Item Service to be running)
```shell script
docker exec -it apiGateway /bin/sh
cd apiGateway
python3 initialize_ebay.py
```
#EOF 
