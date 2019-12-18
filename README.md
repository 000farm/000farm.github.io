000.farm = Technology + Aquaponics


## Considerations

Use commodity hardware.


## Techology Stack

- Arduino
- Raspberry Pi


## How It Works

#### Sensor Controller 

An Arduino is used as a sensor controller. All the sensors are connected to this device and are read from in a specific interval. 


#### Metrics Collector

A Raspberry Pi is connected via USB to the Arduino. This device collects the data from the Arduino Sensor Controller and make the data available for collection over a generic network.


##### Metrics Aggregator

A Raspberry Pi is used to aggregate and display collected metrics. This device polls the Metrics Collector to collect the metrics and stores them in its database. This device also has a dashboard service where users can view graphed metrics.



## Getting Started


#### Metrics Aggregator

Install Docker

```
curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh
sudo usermod -aG docker pi
```

Remove all Docker Images / Containers

```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -a -q)
```

Install Docker Network

```
docker network create obs
```

Create Persistent Storage

```
docker volume create prometheus-storage
docker volume create grafana-storage
```

Run Prometheus

```
docker run \
    -d --name prometheus \
    -p 9090:9090 \
    --net obs \
    -v /home/pi/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v prometheus-storage:/prometheus \
    --restart unless-stopped \
    prom/prometheus
```

Testing Config

```
global:
  scrape_interval:     15s
  external_labels:
    monitor: 'codelab-monitor'

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'aquapi'
    scrape_interval: 5s
    static_configs:
      - targets: ['aquapi:8080']
```

Run Grafana

```
docker run \
    -d --name grafana \
    -p 3000:3000 \
    --net obs \
    -v grafana-storage:/var/lib/grafana \
    --restart unless-stopped \
    grafana/grafana
```

#### Metrics Collector


Install Docker Network

```
docker network create aqua
```

Install Redis

```
docker run \
    -d --name redis \
    -p 6379:6379 \
    --net aqua \
    --restart unless-stopped \
    redis
```

Run serial_to_redis

```
docker run \
    -d --name serial_to_redis \
    --net aqua \
    --device=/dev/ttyACM0 \
    --restart unless-stopped \
    wfong/serial-to-redis
```

Run redis_to_prometheus

```
docker run \
    -d --name redis_to_prometheus \
    -p 8080:8080 \
    --net aqua \
    --restart unless-stopped \
    wfong/redis-to-prometheus
```

#### Sensors Controller

```
const int tempSensor=A1;
const int lightSensor=A2;

String tempC;
float vOut;
int lightValue;

void setup() {
  pinMode(tempSensor, INPUT);
  pinMode(lightSensor, INPUT);
  Serial.begin(9600);
}

void loop() {
  // Temp
  vOut = analogRead(tempSensor); // Double reads
  delay(100);
  vOut = analogRead(tempSensor);  
  tempC = ((vOut/1024.0)*5000)/10; // Convert to C? 
  Serial.println("sensor_temp:" + tempC);
  delay(500);
  
  // Light
  lightValue = analogRead(lightSensor); // Double reads
  delay(100);
  lightValue = analogRead(lightSensor);
  Serial.println("sensor_light:" + String(lightValue));
  delay(500);

  // Sleep
  delay(5000);
}

// Double reads: https://forum.arduino.cc/index.php?topic=69742.0

```
