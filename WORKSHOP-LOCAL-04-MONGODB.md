**Event Driven Architecture with Quarkus, Kafka, and Kubernetets**  

# Step 4 - Persistence with Hibernate Panache and MogoDB

## MongoDB with Hibernate Panache and Kafka with Microprofile Reactive Messaging

If you have cloned the "j4k-workshop-solution" project you can check a tag that corresponds to this step with the following command:

```shellscript

git checkout step-03

```

Yes, the tag is "step-03," and this is Step 4.

![Checkout Solution Tag](images/04-01.png)


NOTE: For the next sections you will need Docker and Docker Compose to run Kafka and MongoDB

Create a Docker Compose file named, "docker-compose.yaml" with the following contents (it doesn't have to be in your working directory):

```yaml
version: '3'

services:

  database:
    image:  'mongo'
    environment:
      - MONGO_INITDB_DATABASE=coffeeshopdb
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=redhat-20
    volumes:
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    ports:
      - '27017-27019:27017-27019'

  zookeeper:
    image: strimzi/kafka:0.11.4-kafka-2.1.0
    command: [
      "sh", "-c",
      "bin/zookeeper-server-start.sh config/zookeeper.properties"
    ]
    ports:
      - "2181:2181"
    environment:
      LOG_DIR: /tmp/logs

  kafka:
    image: strimzi/kafka:0.11.4-kafka-2.1.0
    command: [
      "sh", "-c",
      "bin/kafka-server-start.sh config/server.properties --override listeners=$${KAFKA_LISTENERS} --override advertised.listeners=$${KAFKA_ADVERTISED_LISTENERS} --override zookeeper.connect=$${KAFKA_ZOOKEEPER_CONNECT}"
    ]
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      LOG_DIR: "/tmp/logs"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```

And a file to initialize MongoDB correctly called "init-mongo.js" with the following contents:

```javascript
db.createUser({
        user:"coffeeshop-user",
        pwd:"redhat-20",
        roles:[
            {
                role:"readWrite",
                db:"coffeeshopdb"
            }
        ]
    }
)

```

Start it with:

```shell script

docker-compose up

```

## Persisting our Order with Hibernate Panache

[Hibernate Panache](https://quarkus.io/guides/hibernate-orm-panache)

First we will add the necessary extensions to our pom. We need the Hibernate Panache and Quarkus JUnit5 Mockito extensions:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-mongodb-panache</artifactId>
    </dependency>
    <dependency>
         <groupId>io.quarkus</groupId>
         <artifactId>quarkus-junit5-mockito</artifactId>
         <scope>test</scope>
   </dependency>
```
We also need to add the connection string to our application.properties file:

```properties
# Datasource
%dev.quarkus.mongodb.database = coffeeshopdb
%dev.quarkus.mongodb.connection-string = mongodb://coffeeshop-user:redhat-20@localhost:27017/coffeeshopdb
%dev.quarkus.log.category."io.quarkus.mongodb.panache.runtime".level=DEBUG
```

### MongoDB and Hibernate Panache

Update the FavFoodOrder by making it extend PanacheMongoEntity:

```java
public class FavFoodOrder extends PanacheMongoEntity {
    ...
}
```

The complete class now looks like:

```java
package org.j4k.workshops.quarkus.coffeeshop.favfood.domain;

import io.quarkus.mongodb.panache.PanacheMongoEntity;

import java.util.List;
import java.util.Objects;
import java.util.StringJoiner;

public class FavFoodOrder extends PanacheMongoEntity {

    String customerName;

    String orderId;

    List<FavFoodLineItem> favFoodLineItems;

    public FavFoodOrder() {
    }

    public FavFoodOrder(String customerName, String orderId, List<FavFoodLineItem> favFoodLineItems) {
        this.customerName = customerName;
        this.orderId = orderId;
        this.favFoodLineItems = favFoodLineItems;
    }

    @Override
    public String toString() {
        return new StringJoiner(", ", FavFoodOrder.class.getSimpleName() + "[", "]")
                .add("customerName='" + customerName + "'")
                .add("orderId='" + orderId + "'")
                .add("lineItems=" + favFoodLineItems)
                .toString();
    }

    @Override
    public boolean equals(Object o) {

        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        FavFoodOrder that = (FavFoodOrder) o;
        return Objects.equals(customerName, that.customerName) &&
                Objects.equals(orderId, that.orderId) &&
                Objects.equals(favFoodLineItems, that.favFoodLineItems);
    }

    @Override
    public int hashCode() {
        return Objects.hash(customerName, orderId, favFoodLineItems);
    }

    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }

    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    public List<FavFoodLineItem> getLineItems() {
        return favFoodLineItems;
    }

    public void setLineItems(List<FavFoodLineItem> favFoodLineItems) {
        this.favFoodLineItems = favFoodLineItems;
    }
}
```

That's all we need to do!

We will be using the Repository Pattern so create a new interface in the infrastructure package:

```java
package org.j4k.workshops.quarkus.coffeeshop.infrastructure;

import io.quarkus.mongodb.panache.PanacheMongoRepository;
import org.j4k.workshops.quarkus.coffeeshop.favfood.domain.FavFoodOrder;

public class FavFoodOrderRepository implements PanacheMongoRepository<FavFoodOrder> {
}

```

And we're done!

We will let the ApiResource handle persistence so we need to inject the FavFoodOrderRepository into the class:

```java

package org.j4k.workshops.quarkus.coffeeshop.infrastructure;

import org.j4k.workshops.quarkus.coffeeshop.favfood.domain.FavFoodOrder;
import org.j4k.workshops.quarkus.coffeeshop.domain.OrderInCommand;
import org.j4k.workshops.quarkus.coffeeshop.favfood.domain.FavFoodOrderHandler;
import org.j4k.workshops.quarkus.coffeeshop.favfood.infrastructure.FavFoodOrderRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.inject.Inject;
import javax.ws.rs.POST;
import javax.ws.rs.Path;
import javax.ws.rs.core.Response;

@Path("/api")
public class ApiResource {

    Logger logger = LoggerFactory.getLogger(ApiResource.class);

    @Inject
    KafkaService kafkaService;

    @Inject
    FavFoodOrderRepository repository;

    @POST
    @Path("/favfood")
    public Response acceptFavFoodOrder(final FavFoodOrder favFoodOrder) {

        logger.debug("received {}", favFoodOrder);
        repository.persist(favFoodOrder);

        OrderInCommand orderInCommand = FavFoodOrderHandler.handleOrder(favFoodOrder);

        logger.debug("sending {}", orderInCommand);
        kafkaService.placeOrders(orderInCommand);

        return Response.accepted(favFoodOrder).build();
    }
}
```

### Mocking the Repository

To properly test the ApiResource we need to mock the Repository class.  QuarksTest makes it easy to mock out injected objects.  Just add the following:

```java

    @InjectMock
    FavFoodOrderRepository repository;

```

The complete class:

```java
package org.j4k.workshops.quarkus.coffeeshop;

import io.quarkus.test.common.QuarkusTestResource;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.mockito.InjectMock;
import org.j4k.workshops.quarkus.coffeeshop.favfood.infrastructure.FavFoodOrderRepository;
import org.junit.jupiter.api.Test;

import javax.ws.rs.core.MediaType;

import static io.restassured.RestAssured.given;

@QuarkusTest @QuarkusTestResource(KafkaTestResource.class)
public class ApiResourceTest {

    final String json = "{\"customerName\":\"Lemmy\",\"orderId\":\"cdc07f8d-698e-43d9-8cd7-095dccace575\",\"favFoodLineItems\":[{\"item\":\"COFFEE_BLACK\",\"itemId\":\"0eb0f0e6-d071-464e-8624-23195c8f9e37\",\"quantity\":1}]}";

    @InjectMock
    FavFoodOrderRepository repository;

    @Test
    public void tetFavFoodEndpoint(){

        given()
                .when()
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .body(json)
                .post("/api/favfood")
                .then()
                .statusCode(202);

    };


}
```

Now the test should pass.
