## Spring Flights Application

This is a demo application showcasing RSocket support in Spring.

This application is made of 3 modules:

* `radar-collector`, an app that provides information about radars and the aircraft signals they collect.
* `flight-tracker` and `flight-client`, an app that displays an interactive map with radars and aircrafts.

## SECTION 1 - No Gateway
To run without Gateway:
```
git checkout 70a8240
```

### Running the applications

First, run the collector application:

```
$ ./gradlew :radar-collector:build
$ java -jar radar-collector/build/libs/radar-collector-0.0.1-SNAPSHOT.jar
```

Then, run the tracker web application:
```
$ ./gradlew :flight-tracker:build
$ java -jar flight-tracker/build/libs/flight-tracker-0.0.1-SNAPSHOT.jar
```

The tracker application is available at `http://localhost:8080/index.html`

### Radar Collector

This application is providing information about radars (actually populated from list of airports):
their IATA code, location and aircraft signals recorded. The aircraft signals are randomly
generated and the list of radars is inserted in a MongoDB database.

The application starts an RSocket server with TCP transport, at `localhost:9898`.

You can get a list of airports located inside specific coordinates,
using the [rsocket-cli](https://github.com/rsocket/rsocket-cli):

```
rsocket-cli --stream --metadataFormat=message/x.rsocket.routing.v0 -m=locate.radars.within --dataFormat=json \
-i='{"first":{"lng": 3.878915, "lat": 46.409025}, "second": {"lng": 6.714843, "lat": 44.365644}}' tcp://localhost:9898/
```

You can also get a stream of aircraft locations for a given radar, with:

```
rsocket-cli --stream --metadataFormat=message/x.rsocket.routing.v0 -m=listen.radar.LYS --dataFormat=json -i "" tcp://localhost:9898/
```


### Flight Tracker

This application displays an interactive map showing radars - it is also concatenating
the streams of aircraft signals for the radars displayed on screen.

The application starts a WebFlux server at `localhost:8080`, with an RSocket over websocket endpoint on `/rsocket`.
The `flight-client` module builds the JavaScript client using [Leaflet](https://leafletjs.com/) and the the websocket client
from [rsocket-js](https://github.com/rsocket/rsocket-js/).

The browser will first locate all radars in the current view box; you can do the same on the CLI with:

```
rsocket-cli --stream --metadataFormat=message/x.rsocket.routing.v0 -m=locate.radars.within --dataFormat=json \
-i='{"viewBox": {"first":{"lng": 3.878915, "lat": 46.409025}, "second": {"lng": 6.714843, "lat": 44.365644}}, "maxRadars": 10}' ws://localhost:8080/rsocket
```

Once all the radars are retrieved, we can ask a merged stream of all aircrafts for those radars to the server.

```
rsocket-cli --stream --metadataFormat=message/x.rsocket.routing.v0 -m=locate.aircrafts.for --dataFormat=json \
-i='[{"iata":"LYS"}, {"iata":"CVF"}, {"iata":"NCY"}]' ws://localhost:8080/rsocket
```

The browser will perform such a request and update the aircrafts positions live.

The Leaflet map has a small number input (bottom left) which controls the reactive streams demand from the client.
Decreasing it significantly should make the server send less updates to the map. Increasing it back should
catch up with the updates.

Also, once the RSocket client is connected to the server, a bi-directionnal connection is established:
they're now both able to send requests (being a requester) and respond to those (being a responder).
Here, this demo shows how the JavaScript client can respond to requests sent by the server.

Sending the following request to the web server will make it send requests to all connected clients
to let them know that they should change their location to the selected radar:

```
curl -X POST localhost:8080/location/CDG
```

### Deploy to Cloud Foundry

Use the `cf` CLI to target your instance of Cloud Foundry.

Open the file called `manifest-vars.yml` and edit the domains as appropriate for your Cloud Foundry foundation. You can run the command `cf domains` to see the domains available to you.

Next, you need to provide flight-client with the correct URL to flight-tracker on Cloud Foundry (otherwise, the URL defaults to `ws://localhost:8080/rsocket`).
Run the following following script and enter the appropriate URL when prompted. 
You can copy/paste the sample value provided by the script - just change the domain as necessary.
```
$ ./config-client.sh
```

Build the radar-collector and flight-client apps.
Please note that even if you built the projects earlier in this exercise, you must re-build flight-client to incorporate the change in the last step.
```
./gradlew :radar-collector:build
./gradlew :flight-tracker:build
```

Push the apps to Cloud Foundry, providing `manifest-vars.yml` as input:
```
cf push --vars-file manifest-vars.yml
```

The tracker application is available at `http://flight-tracker.<YOUR-DOMAIN>/index.html`

## SECTION 2 - With Gateway
To run with Gateway:
```
git checkout master
```

### Run locally using Spring Cloud Gateway RSocket

In this mode, radar-collector and flight-tracker both behave as clients of the gateway application, meaning that they initiate a connection with the gateway when they start up. 
Flight-tracker does not create a connection directly with radar-collector.
The gateway will broker the communication between flight-tracker and radar-collector.

First, run the gateway application:
```
$ ./gradlew :radar-gateway:build
$ java -jar radar-gateway/build/libs/radar-gateway-0.0.1-SNAPSHOT.jar
```

Then, run two instances of the radar-collector application using a different profile for each.
The profiles determine which subset of [airports](radar-collector/src/main/resources/airports.json) each instance loads.
They also set a tag that gateway can use to route requests to the appropriate instance.
```
$ # Build radar-collector
$ ./gradlew :radar-collector:build

$ # In one terminal run:
$ java -jar radar-collector/build/libs/radar-collector-0.0.1-SNAPSHOT.jar --spring.profiles.active=civilian

$ # In a second terminal run:
$ java -jar radar-collector/build/libs/radar-collector-0.0.1-SNAPSHOT.jar --spring.profiles.active=military
```

Flight-tracker requests are enhanced with additional metadata that instructs the gateway on the instance of collector to which to route the message.
Run the tracker web application:
```
$ ./gradlew :flight-tracker:build
$ java -jar flight-tracker/build/libs/flight-tracker-0.0.1-SNAPSHOT.jar
```

The tracker application is available at `http://localhost:8080/index.html`
You can stop and start the military collector instance, refreshing the map in the browser in between stops and starts, to see that the military airport icon shows up when the military app instance is running.

### Deploy to Cloud Foundry with Public TCP Route to Gateway

Use the `cf` CLI to target your instance of Cloud Foundry.

Open the file called `manifest-vars.yml` and edit the domains as appropriate for your Cloud Foundry foundation. You can run the command `cf domains` to see the domains available to you.

Next, you need to provide flight-client with the correct URL to flight-tracker on Cloud Foundry (otherwise, the URL defaults to `ws://localhost:8080/rsocket`).
Run the following following script and enter the appropriate URL when prompted. 
You can copy/paste the sample value provided by the script - just change the domain as necessary.
```
$ ./config-client.sh
```

Build the radar-collector and flight-client apps.
Please note that even if you built the projects earlier in this exercise, you must re-build flight-client to incorporate the change in the last step.
```
./gradlew :radar-gateway:build
./gradlew :radar-collector:build
./gradlew :flight-tracker:build
```

Push the apps to Cloud Foundry, providing `manifest-vars.yml` as input:
```
cf push --vars-file manifest-vars.yml
```

Use `cf apps` to verify that only the tracker and gateway are routable, and that no route exists to the collector apps.
Note that the route to the tracker is only used by the Javascript client from the browser.
Like the collectors, the tracker also makes an outbound connection to the gateway. 

The tracker application is available at `http://flight-tracker.<YOUR-DOMAIN>/index.html`

You can use `cf stop radar-collector-military` and `cf start radar-collector-military`, refreshing the map in the browser in between stops and starts, to see that the military airport icon shows up when the military app instance is running.

### Deploy to Cloud Foundry with Internal TCP Route to Gateway

Follow the same step as in the previous section, but this time use a different manifest file as follows:
```
cf push -f manifest-internal.yml --vars-file manifest-vars.yml
```

Use `cf apps` to verify that only the tracker and gateway are routable, and that no route exists to the collector apps.
In addition, the route pointing to the gateway is on the apps.internal domain, meaning that the gateway itself is also not directly accessible from the internet.



--------
TO-DO:
- Set up controller to function as a server or client based solely on configuration
- Demo request through configuration instead of code