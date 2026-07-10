Environmental Monitor
Environmental Monitor
The Environmental Monitor is a modular ESP32‑based sensing system designed to track real‑time environmental conditions including temperature, humidity, pressure, and gas concentration. It uses BMP280/BME280 sensors for atmospheric data and an MQ gas sensor for air‑quality detection. The project is built on a breadboard prototype and can be expanded into a PCB or enclosure.
This repository contains the firmware, wiring documentation, and setup instructions needed to run the monitor.

🚀 Features
ESP32 Microcontroller

Wi‑Fi capable

Fast sensor polling

Stable 3.3V operation

Environmental Sensors

BME280 – Temperature, humidity, pressure

BMP280 – Temperature, pressure

MQ Gas Sensor – Air‑quality / gas concentration (analog)

Breadboard Prototype

Easy to modify

Clear wiring layout

Supports additional modules (OLED, buzzer, LEDs)

Modular Firmware

Separate drivers for each sensor

Expandable architecture

Clean, readable code structure

📁 Project Structure
/src
  main.cpp
  bme280.cpp
  bmp280.cpp
  mq_sensor.cpp
  display.cpp

/include
  bme280.h
  bmp280.h
  mq_sensor.h
  display.h

/docs
  wiring_diagram.png
  sensor_notes.md

platformio.ini
README.md

 Component	              Purpose
ESP32 Dev Board     	    Main controller
BME280	                  Temp, humidity, pressure
BMP280	                  Temp, pressure
MQ Gas Sensor	            Gas concentration (analog)
Breadboard	              Prototype wiring
Jumper Wires	            Connections

🧪 Wiring Overview
BME280 / BMP280 (I²C Mode)
VIN → 3.3V

GND → GND

SCL → GPIO 22

SDA → GPIO 21

MQ Gas Sensor (Analog Mode)
VCC → 5V

GND → GND

A0 → GPIO 34 (ADC input)

Notes
MQ sensors require 5V heating, but output is analog and safe for ESP32 ADC.

Use a voltage divider if your MQ module outputs >3.3V on A0.

BME280 and BMP280 can share the same I²C bus.

. Install PlatformIO
PlatformIO is recommended for building and uploading firmware.

PlatformIO‑ready ESP32 code template
This template assumes:

ESP32

BME280 + BMP280 on I²C (GPIO 21/22)

MQ gas sensor on ADC (GPIO 34)

Output via Serial (for ingestion daemon) and optionally Wi‑Fi later.

platformio.ini
[env:esp32-env-monitor]
platform = espressif32
board = esp32dev
framework = arduino

monitor_speed = 115200

lib_deps =
  adafruit/Adafruit BMP280 Library
  adafruit/Adafruit BME280 Library

src/main.cpp 

#include <Arduino.h>
#include <Wire.h>
#include <Adafruit_BME280.h>
#include <Adafruit_BMP280.h>

#define I2C_SDA 21
#define I2C_SCL 22

#define MQ_PIN 34

Adafruit_BME280 bme;
Adafruit_BMP280 bmp;

bool hasBME = false;
bool hasBMP = false;

void setupSensors() {
  Wire.begin(I2C_SDA, I2C_SCL);

  if (bme.begin(0x76)) {
    hasBME = true;
    Serial.println("BME280 detected");
  } else {
    Serial.println("BME280 not found");
  }

  if (bmp.begin(0x77)) {
    hasBMP = true;
    Serial.println("BMP280 detected");
  } else {
    Serial.println("BMP280 not found");
  }

  analogReadResolution(12); // ESP32 ADC: 0–4095
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("Environmental Monitor starting...");
  setupSensors();
}

void loop() {
  float temperature = NAN;
  float humidity = NAN;
  float pressure = NAN;
  int   gas_raw   = analogRead(MQ_PIN);

  if (hasBME) {
    temperature = bme.readTemperature();
    humidity    = bme.readHumidity();
    pressure    = bme.readPressure() / 100.0F; // hPa
  } else if (hasBMP) {
    temperature = bmp.readTemperature();
    pressure    = bmp.readPressure() / 100.0F;
  }

  // Simple CSV line for ingestion daemon:
  // timestamp handled on server side
  Serial.print("ENV,");
  Serial.print(temperature); Serial.print(",");
  Serial.print(humidity);    Serial.print(",");
  Serial.print(pressure);    Serial.print(",");
  Serial.print(gas_raw);
  Serial.println();

  delay(2000); // 2s sampling
}


QuestDB ingestion pipeline (Linux + C daemon)
Install QuestDB
Download and run QuestDB on your Debian VM:

sudo apt install questdb
sudo systemctl enable questdb
sudo systemctl start questdb

Deploy the Ingestion Daemon
Compile:
gcc ingest_daemon.c -o ingest

Install as a systemd service:

sudo cp ingest.service /etc/systemd/system/
sudo systemctl enable ingest
sudo systemctl start ingest

The daemon will continuously ingest sensor readings into QuestDB.

QuestDB table design
In QuestDB, create a table for your sensor data:
CREATE TABLE env_monitor (
    ts        TIMESTAMP,
    temperature DOUBLE,
    humidity    DOUBLE,
    pressure    DOUBLE,
    gas_raw     INT
) TIMESTAMP(ts);

You can do this via QuestDB’s web console.

 ingestion daemon (reads Serial, writes to QuestDB)
Assumptions:

ESP32 connected via USB as /dev/ttyUSB0

QuestDB running locally on http://localhost:9000

We’ll use the Inserts via HTTP (/exec) for simplicity.

ingest_daemon.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include <fcntl.h>
#include <termios.h>
#include <curl/curl.h>

#define SERIAL_PORT "/dev/ttyUSB0"
#define BAUDRATE B115200

int open_serial(const char *port) {
    int fd = open(port, O_RDONLY | O_NOCTTY);
    if (fd < 0) {
        perror("open_serial");
        return -1;
    }

    struct termios tty;
    memset(&tty, 0, sizeof tty);

    if (tcgetattr(fd, &tty) != 0) {
        perror("tcgetattr");
        close(fd);
        return -1;
    }

    cfsetospeed(&tty, BAUDRATE);
    cfsetispeed(&tty, BAUDRATE);

    tty.c_cflag |= (CLOCAL | CREAD);
    tty.c_cflag &= ~CSIZE;
    tty.c_cflag |= CS8;
    tty.c_cflag &= ~PARENB;
    tty.c_cflag &= ~CSTOPB;
    tty.c_cflag &= ~CRTSCTS;

    tty.c_lflag &= ~(ICANON | ECHO | ECHOE | ISIG);
    tty.c_iflag &= ~(IXON | IXOFF | IXANY);
    tty.c_oflag &= ~OPOST;

    tty.c_cc[VMIN]  = 1;
    tty.c_cc[VTIME] = 0;

    if (tcsetattr(fd, TCSANOW, &tty) != 0) {
        perror("tcsetattr");
        close(fd);
        return -1;
    }

    return fd;
}

void send_to_questdb(double temperature, double humidity, double pressure, int gas_raw) {
    CURL *curl = curl_easy_init();
    if (!curl) return;

    char query[512];
    snprintf(query, sizeof(query),
             "INSERT INTO env_monitor "
             "(ts, temperature, humidity, pressure, gas_raw) "
             "VALUES (now(), %f, %f, %f, %d);",
             temperature, humidity, pressure, gas_raw);

    curl_easy_setopt(curl, CURLOPT_URL, "http://localhost:9000/exec");
    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, query);

    CURLcode res = curl_easy_perform(curl);
    if (res != CURLE_OK) {
        fprintf(stderr, "QuestDB error: %s\n", curl_easy_strerror(res));
    }

    curl_easy_cleanup(curl);
}

int main() {
    int fd = open_serial(SERIAL_PORT);
    if (fd < 0) {
        fprintf(stderr, "Failed to open serial port\n");
        return 1;
    }

    curl_global_init(CURL_GLOBAL_DEFAULT);

    char buf[256];
    char line[256];
    int idx = 0;

    while (1) {
        int n = read(fd, buf, sizeof(buf));
        if (n > 0) {
            for (int i = 0; i < n; i++) {
                char c = buf[i];
                if (c == '\n') {
                    line[idx] = '\0';
                    idx = 0;

                    // Expect: ENV,temp,hum,press,gas
                    if (strncmp(line, "ENV,", 4) == 0) {
                        double temp = 0, hum = 0, press = 0;
                        int gas = 0;

                        // Simple parsing
                        sscanf(line, "ENV,%lf,%lf,%lf,%d",
                               &temp, &hum, &press, &gas);

                        send_to_questdb(temp, hum, press, gas);
                    }
                } else if (idx < (int)sizeof(line) - 1) {
                    line[idx++] = c;
                }
            }
        } else {
            usleep(10000);
        }
    }

    curl_global_cleanup();
    close(fd);
    return 0;
}

Build and run daemon
Install dependencies (Debian):

sudo apt install build-essential libcurl4-openssl-dev
gcc ingest_daemon.c -o ingest -lcurl
./ingest

systemd service (optional but recommended)
Create /etc/systemd/system/env_ingest.service:

[Unit]
Description=Environmental Monitor Ingestion Daemon
After=network.target

[Service]
ExecStart=/usr/local/bin/ingest
Restart=always
User=youruser

[Install]
WantedBy=multi-user.target
Then:

sudo cp ingest /usr/local/bin/
sudo systemctl enable env_ingest.service
sudo systemctl start env_ingest.service

Grafana dashboard (QuestDB as data source)
High‑level steps:

Add data source

Type: PostgreSQL (QuestDB’s PG wire)

Host: localhost:8812

Database: qdb

User: admin (or configured)

SSL: disabled (local)

Create panel query

Example query:
SELECT
  ts,
  temperature,
  humidity,
  pressure,
  gas_raw
FROM env_monitor
WHERE $__timeFilter(ts)
ORDER BY ts;

Build panels:

Line chart: temperature vs time

Line chart: humidity vs time

Line chart: pressure vs time

Bar/line: gas_raw vs time

Add thresholds/alerts for gas_raw and temperature.





