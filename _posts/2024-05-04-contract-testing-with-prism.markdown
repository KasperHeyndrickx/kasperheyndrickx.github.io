---
layout: post
title:  "Contract testing with Prism"
date:   2024-05-04 11:42:34 +0100
categories: testing prism python
---

## The difficulties of testing with external systems

All systems interact with external components at some point. This can be a database, another microservice from your own team, a microservice from another team, or even a service outside of your company. Most interactions with databases can be tested by spinning up a [database testcontainer](https://java.testcontainers.org/modules/databases/). Other microservices maintained by your own team are also relatively easy. You're well aware of how they're deployed, how they behave and what dependencies they have. In most cases you can deploy these in a generic testcontainer. This leaves us with external services, these are usually the most challenging. There are a few common approaches here:

- Deploying the external service in a provided testcontainer
- Deploying the external service in a generic testcontainer
- Mocking the service
- Testing with a live sandbox instance
- Not testing the interaction directly

The first option is preferred, but not always available. The next best option would be to deploy the service in a generic testcontainer. This is assuming you have access to the docker image of this service, which isn't always the case. Also, the service might depend on other services too. Trying to deploy all this can quickly become too complicated.

The most common approach is to mock the external service. But there's one major problem with this: the behavior of your mock might not match the actual behavior of the external service. Also, if you decide to use a newer api version, it's highly unlikely that you check if the behavior of your mocks still matches reality. And finally, it's just a lot of work. Manually writing the expected output for each interaction becomes tiring real quick.

So what's the alternative?

## Stoplight Prism

[Stoplight Prism](https://github.com/stoplightio/prism) is a tool to generate a mock API based on an [OpenAPI specification](https://spec.openapis.org/oas/latest.html). I prefer to also run this in container, so that my test can spin up the prism service by itself. Let's take a look how this would work in a simple python example:

_`post_example.py`_

```python
import requests

def add_item(url: str, item: str) -> bool:
    path = "/api/items"
    headers = {"Content-Type": "application/json"}
    data = {"name": item}
    response = requests.post(f"{url}{path}", 
                             headers=headers, 
                             json=data)
    return response.status_code == 200
```

The method we want to test (`add_item(url, item)`) makes a POST request to an external service. We want to test that our interaction with this external service is correct. Is the URL correct? Is the body correct? Do we correctly make the request? We could do this by writing a mock, but as mentioned before, our mocks might be wrong. Let's see how we can use the provided openAPI specification to create a mock service:

_`resources/api.yaml`_

```yaml
openapi: 3.0.0
info:
  title: Items API
  version: 1.0.0
paths:
  /api/items:
    post:
      summary: Add a new item
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
      responses:
        '200':
          description: Successful operation
```

_`test_post_example.py`_

```python
import pytest
import os
from testcontainers.core.container import DockerContainer
from testcontainers.core.waiting_utils import wait_for_logs


class TestService:
    @pytest.fixture(scope="module", autouse=True)
    def service_container(self, request):
        prism = DockerContainer("stoplight/prism:4", init=True)
        prism.with_volume_mapping(os.getcwd() + '/resources', '/apis') \
            .with_exposed_ports(4010) \
            .with_command("mock -v 'debug' -h 0.0.0.0 /apis/api.yaml") \
            .start()
        wait_for_logs(prism, "Prism is listening on http://0.0.0.0:4010")

        def remove_container():
            prism.stop()

        request.addfinalizer(remove_container)
        return prism
        
    def test_add_item(self, service_container):
        url = "http://localhost:" + str(service_container.get_exposed_port(4010))
        assert add_item(url, 'test') == True
```

The test itself is defined in `test_add_item(self, service_container)`. Before this method is called, a Prism container spins up. The OpenAPI yaml configuration is passed to the container by attaching it in a volume. We wait until the container is ready (by waiting for logs), and then continue with the actual test.

Now when we call `add_item()` in the test, it will send the request to the Prism container, which validates the validity of the request. If our request doesn't match the API specification, we'll know about it!

## Specifying the response

The way our test is set up now gives us no control over the response of Prism. This might be enough in some cases, but sooner or later you'll want to test some logic that depends on the values returned by the external service. Let's take a look at what that might look like in practise. Let's consider an API that returns books based on their title. The response includes a 'status' field that can be "IN STOCK" or "SOLD OUT". We use this API in a function that returns 'True' if the book is in stock, and 'False' if the book is Sold out.

Here's what our method looks like:

```python
import requests

def in_stock(url: str, book_name: str, headers: dict) -> bool:
    response = requests.get(url + '/api/books', 
                            params={'name': book_name}, 
                            headers=headers)
    data = response.json()
    return data['status'] == 'IN STOCK'
```

We cannot make any assumptions as to what our Prism mock server will return. Therefore we cannot guarantee that this method will return True or False when we test it. We need a way to tell Prism what to respond to a certain request. To make this work, we need to add predefined examples to the OpenAPI specification. Prism allows us to define which example to return depending on a value passed through the header. Notice that out function definition now allows us to pass headers to the request.

```yaml
openapi: 3.0.0
info:
  title: Bookstore API
  version: 1.0.0
paths:
  /api/books:
    get:
      parameters:
        - name: name
          in: query
          required: true
          schema:
            type: string
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  name:
                    type: string
                  status:
                    type: string
                    enum:
                      - IN STOCK
                      - SOLD OUT
              examples:
                inStockExample:
                  value:
                    name: "The Great Gatsby"
                    status: "IN STOCK"
                  summary: Example of a book in stock
                soldOutExample:
                  value:
                    name: "1984"
                    status: "SOLD OUT"
                  summary: Example of a sold out book
```

```python
class TestService:
    @pytest.fixture(scope="module", autouse=True)
    def service_container(self, request):
        # omitted, see previous example
        
    def test_in_stock(self, service_container):
        url = "http://localhost:" + str(service_container.get_exposed_port(4010))
        assert True == in_stock(url, 'The Great Gatsby', 
                                {"prefer": "example=inStockExample"})

    def test_sold_out(self, service_container):
        url = "http://localhost:" + str(service_container.get_exposed_port(4010))
        assert False == in_stock(url, '1984', 
                                 {"prefer": "example=soldOutExample"})
```

The downside here is that we need to modify our production code so that we can inject headers in the test. This is not always desired, and sometimes it's not even possible. The OpenAPI specification also needs to include examples that we can use in our tests. In most cases you'll have to add these manually. So if the external service updates their API, you cannot just swap the specification and be done with it.

## Injecting headers and adding examples

What if we don't want or can't inject headers into our HTTP request? Stoplight has an example of how to implement a proxy server to work around this issue in [this repository](https://github.com/stoplightio/ExampleChooserPrismProxy). It's a service that sits between your test and the Prism mock server. This proxy looks at your request and adds the right 'prefer' header, based on predefined logic. It then forwards this query to Prism, and returns the result as-is.

Adding the examples to the OpenAPI spec seems a bit more problematic. It's not ideal to modify the specification directly, because then we cannot easily update it. At the time of writing, I haven't found a nice solution to add examples to OpenAPI specifications. It's not that difficult to write your own, but it comes with the added problem that Prism doesn't validate examples to the spec.

It's perfectly possible to create an example that doesn't match the defined response format. Currently I use Prism alongside traditional mocks. I use Prism to verify that my interaction with the API matches the API specification, and I use mocks to test my logic based on different potential responses.
