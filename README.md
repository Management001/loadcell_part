# loadcell_part

```python
from bluetooth import *
import threading
from matplotlib import pyplot as plt
from matplotlib import animation
from mpl_toolkits.axes_grid1 import make_axes_locatable
import numpy as np
import time
import struct
import matplotlib
from matplotlib import font_manager
from datetime import datetime

# 사용자 및 환경 정보
USER_INFO = {
    "User": "Kang Jeongtae",
    "Age": 25,
    "Weight": 70,
    "Height": 175,
    "Gender": "Male",
    "Activity Level": "Regular Exercise"
}

ENVIRONMENT_INFO = {
    "Location": "Indoor",
    "Floor Type": "Wood",
    "Shoe Type": "Running Shoes",
    "Temperature": "22°C",
    "Humidity": "55%"
}

SENSOR_INFO = {
    "Sensor Model": "XYZ-1000",
    "Firmware Version": "v1.2.0",
    "Device ID": "123456789"
}

# 폰트 경로 설정
font_path = '/usr/share/fonts/truetype/msttcorefonts/times.ttf'
prop = font_manager.FontProperties(fname=font_path)
matplotlib.use("Qt5Agg")

global foot_total, foot_left, foot_right, client_sock1, client_sock2
foot_left = [0, 0, 0, 0]
foot_right = [0, 0, 0, 0]
max_retries = 10
retry_count = 0

def connect_device(mac_address):
    for _ in range(10):
        try:
            client_sock = BluetoothSocket(RFCOMM)
            client_sock.connect((mac_address, 1))
            print(f"Bluetooth connected to {mac_address}")
            return client_sock
        except BluetoothError as e:
            print(f"Bluetooth Error: {e}")
            time.sleep(1)
    print("Connection Timeout")
    sys.exit()

def initialize_log(file_name):
    with open(file_name, 'w') as f:
        f.write("# User Information\n")
        for key, value in USER_INFO.items():
            f.write(f"{key}: {value}\n")
        f.write("\n# Environment Information\n")
        for key, value in ENVIRONMENT_INFO.items():
            f.write(f"{key}: {value}\n")
        f.write("\n# Sensor Metadata\n")
        for key, value in SENSOR_INFO.items():
            f.write(f"{key}: {value}\n")
        f.write("\n# Measurement Data\n")
        f.write("# Timestamp | Side | Sensor 1 | Sensor 2 | Sensor 3 | Sensor 4\n")

def log_data(file_name, side, sensor_values):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open(file_name, 'a') as f:
        f.write(f"{timestamp} | {side} | {' | '.join(map(str, sensor_values))}\n")

def receive_data(sock, side, log_file):
    global foot_left, foot_right
    while True:
        try:
            data = sock.recv(1 + 4 * 4 + 1 + 1)  # 1(Header) + 16(float*4) + 1(Checksum) + 1(Footer)
            if len(data) != 19:
                print("Incomplete packet received")
                continue
            
            # Header/Footer 확인
            header, footer = data[0], data[-1]
            if header != 0xAA or footer != 0x55:
                print("Invalid Header/Footer")
                continue
            
            # 데이터와 체크섬 추출
            payload = data[1:17]  # 16 bytes (4 floats)
            received_checksum = data[17]
            
            # 체크섬 검증
            calculated_checksum = 0
            for b in payload:
                calculated_checksum ^= b
            
            if calculated_checksum != received_checksum:
                print("Checksum mismatch")
                continue
            
            # 4개의 float 데이터 언패킹
            sensor_values = struct.unpack('ffff', payload)
            if side == 'left':
                foot_left = sensor_values
            else:
                foot_right = sensor_values
            log_data(log_file, side, sensor_values)
            print(f"{side.capitalize()} Sensor Values: {sensor_values}")
        
        except Exception as e:
            print(f"Data receive error: {e}")
            break

log_file = "foot_pressure_log.txt"
initialize_log(log_file)

client_sock1 = connect_device("98:DA:60:04:CA:0C")
client_sock2 = connect_device("98:DA:60:07:D8:33")

device1 = threading.Thread(target=receive_data, args=(client_sock1, 'left', log_file))
device2 = threading.Thread(target=receive_data, args=(client_sock2, 'right', log_file))
device1.start()
device2.start()

fig = plt.figure()
plt.rc("font", family="times new roman")

# 왼쪽 센서 히트맵
ax1 = plt.subplot(221)
divider = make_axes_locatable(ax1)
cax1 = divider.append_axes('right', size='5%', pad=0.05)
line1 = ax1.matshow(np.zeros([2,2]), cmap='jet', vmin=0, vmax=30000)
ax1.axis('off')
plt.colorbar(line1, cax=cax1)

def animate_left(i):
    global foot_left
    list1 = np.array([[foot_left[0], foot_left[1]], [foot_left[2], foot_left[3]]])
    line1.set_array(list1)
    return line1

# 오른쪽 센서 히트맵
ax2 = plt.subplot(222)
divider = make_axes_locatable(ax2)
cax2 = divider.append_axes('right', size='5%', pad=0.05)
line2 = ax2.matshow(np.zeros([2,2]), cmap='jet', vmin=0, vmax=30000)
ax2.axis('off')
plt.colorbar(line2, cax=cax2)

def animate_right(i):
    global foot_right
    list2 = np.array([[foot_right[0], foot_right[1]], [foot_right[2], foot_right[3]]])
    line2.set_array(list2)
    return line2

anim1 = animation.FuncAnimation(fig, animate_left, frames=1, interval=100)
anim2 = animation.FuncAnimation(fig, animate_right, frames=1, interval=100)

plt.tight_layout()
plt.show()

```

```cpp
#include <HX711.h>

/*
 Example using the SparkFun HX711 breakout board with a scale
 By: Nathan Seidle
 SparkFun Electronics
 Date: November 19th, 2014
 License: This code is public domain but you buy me a beer if you use this and we meet someday (Beerware license).

 This is the calibration sketch. Use it to determine the calibration_factor that the main example uses. It also
 outputs the zero_factor useful for projects that have a permanent mass on the scale in between power cycles.

 Setup your scale and start the sketch WITHOUT a weight on the scale
 Once readings are displayed place the weight on the scale
 Press +/- or a/z to adjust the calibration_factor until the output readings match the known weight
 Use this calibration_factor on the example sketch

 This example assumes pounds (lbs). If you prefer kilograms, change the Serial.print(" lbs"); line to kg. The
 calibration factor will be significantly different but it will be linearly related to lbs (1 lbs = 0.453592 kg).

 Your calibration factor may be very positive or very negative. It all depends on the setup of your scale system
 and the direction the sensors deflect from zero state
 This example code uses bogde's excellent library: https://github.com/bogde/HX711
 bogde's library is released under a GNU GENERAL PUBLIC LICENSE
 Arduino pin 2 -> HX711 CLK
 3 -> DOUT
 5V -> VCC
 GND -> GND

 Most any pin on the Arduino Uno will be compatible with DOUT/CLK.

 The HX711 board can be powered from 2.7V to 5V so the Arduino 5V power should be fine.

*/
// ============================== BLE 사용을 위한 헤더 선언 및 PIN 설정
#include <SoftwareSerial.h> // 소프트웨어를 통해 직렬 통신(Serial Communication)을 구현하는 라이브러리
#include <String.h> // 문자열 처리 라이브러리

SoftwareSerial bluetooth(2, 3); // 블루투스 모듈을 선언합니다.
// ============================== BLE

#include "HX711.h"

HX711 scale[4]; //HX711 scale(DOUT, CLK);
//double calibration_factor[4] = {-18350, -17550, -17350, -16550}; //-7050 worked for my 440lb max scale setup (back up!!!)
double calibration_factor[4] = {-15350, -15050, -14850, -14850}; //-7050 worked for my 440lb max scale setup (60kg set)
double loadcell_data[4] = {0};

void setup() {
  Serial.begin(9600);
  bluetooth.begin(9600);

  // scale's DOUT and SCK begin roof ( scale[0] <DOUT> 4, <SCK> 5 -> ... -> scale[3] 10, 11 )
  for(int begin_scales = 0; begin_scales < 4; begin_scales++ ){
    scale[begin_scales].begin((begin_scales+2)*2 /*LOADCELL_DOUT_PIN*/ , (begin_scales+2)*2+1 /*LOADCELL_SCK_PIN*/ );
  }

  // massage 1 // WILL EDIT... 
  Serial.println("HX711 calibration sketch");
  Serial.println("Remove all weight from scale");
  Serial.println("After readings begin, place known weight on scale");
  Serial.println("Press + or a to increase calibration factor");
  Serial.println("Press - or z to decrease calibration factor");

  // scale's scale data reset roof
  for(int reset_scales = 0; reset_scales < 4; reset_scales++){
    scale[reset_scales].set_scale(calibration_factor[reset_scales]);
    scale[reset_scales].tare(); //Reset the scale to 0
  }

  long zero_factor[4] = {0};
  for(int view = 0 ; view < 4 ; view++ ){
    zero_factor[view]=scale[view].read_average();
    Serial.print("Zero factor ["); Serial.print(view); Serial.print("] : "); //This can be used to remove the need to tare the scale. Useful in permanent scale projects.
    Serial.println(zero_factor[view]);
  }
  // delay(150);
}

void GetLoadcellData(void){ // 로드셀 데이터 불러오기
  for(int cali = 0 ; cali < 4 ; cali++){
    loadcell_data[cali] = scale[cali].get_units()*0.453592; // 로드셀 데이터를 kg으로 환산하는 코드 밎 출력
    if(loadcell_data[cali]<0) loadcell_data[cali]=-loadcell_data[cali]; // 음수 값을 양수로 바꾸어주는 연산을 진행. 
  }
}

void ViewLoadcellPlotter(){ // 측정된 load 데이터를 IDE에서 플로터 형태로 모니터에 출력
  for(int serial_plotter = 0 ; serial_plotter < 4 ; serial_plotter++){
    Serial.print((loadcell_data[serial_plotter]));
    Serial.print(",");
  }
  Serial.println("");
}

/*
void BluetoothDataSend(void){ // [BLE] 블루투스로 FSR 데이터 값 보내기.
  // bluetooth.println(String(int(loadcell_data[0]*1000))+"@"+String(int(loadcell_data[1]*1000))+"@"+String(int(loadcell_data[2]*1000))+"@"+String(int(loadcell_data[3]*1000))+"@"); // 문자형으로 데이터를 보내고 있음.

  char dataToSend[50]; // 충분히 큰 배열 선언
  for(int i = 0; i < 4; i++) {
    sprintf(dataToSend + strlen(dataToSend), "%d@", int(loadcell_data[i] * 1000));
  }
  bluetooth.println(dataToSend);
  
}
*/

/*
void BluetoothDataSend(void) { 
  // BLE로 Raw Data 전송
  for (int i = 0; i < 4; i++) {
    float value = loadcell_data[i]; // 현재 센서 값
    bluetooth.write((uint8_t*)&value, sizeof(float)); // float 데이터를 Raw 바이너리로 전송
  }
}
*/

void BluetoothDataSend(void) { 
  uint8_t buffer[sizeof(float) * 4 + 2]; // 4개의 float + Header + Footer

  buffer[0] = 0xAA; // Header
  memcpy(&buffer[1], &loadcell_data[0], sizeof(float) * 4); // float 데이터 복사
  buffer[sizeof(float) * 4 + 1] = 0x55; // Footer

  // 버퍼를 한 번에 전송
  bluetooth.write(buffer, sizeof(buffer));
}


void BluetoothDataSend(void) { 
  uint8_t header = 0xAA;  // 패킷 시작
  uint8_t footer = 0x55;  // 패킷 끝

  bluetooth.write(&header, 1);  // 헤더 전송
  
  for (int i = 0; i < 4; i++) {
    float value = loadcell_data[i];
    bluetooth.write((uint8_t*)&value, sizeof(float));  // float 데이터를 RAW로 전송
  }

  uint8_t checksum = 0;
  for (int i = 0; i < 4; i++) {
    checksum ^= (uint8_t)(loadcell_data[i]);  // 간단한 XOR 체크섬
  }
  bluetooth.write(&checksum, 1);  // 체크섬 전송
  
  bluetooth.write(&footer, 1);  // 푸터 전송
}

void readLoadCell(int index) {
  loadcell_data[index] = scale[index].get_units()*0.453592;
}

int loopCount = 0;
unsigned long timeStart;
unsigned long timeEnd;

unsigned long lastReadTime[4] = {0, 0, 0, 0};
const unsigned long readInterval = 1; // 1ms 간격으로 데이터 읽기

void loop() {
  timeStart = millis();
  // GetLoadcellData(); // ============================== Loadcell 데이터 불러오기
  for (int i = 0; i < 4; i++) {
    if (millis() - lastReadTime[i] > readInterval) {
      readLoadCell(i);
      lastReadTime[i] = millis();
    }
  }
  timeEnd = millis();
  Serial.print(timeEnd-timeStart);
  Serial.println(" <- GetLoadcellData");

  // timeStart = millis();
  // ViewLoadcellPlotter(); // ============================== Loadcell 데이터를 Serial Plotter로 출력 함수
  // timeEnd = millis();
  // Serial.print(timeEnd-timeStart);
  // Serial.println(" <- ViewLoadcellPlotter");

  timeStart = millis();
  BluetoothDataSend(); // ============================== BLE 데이터 전송 코드
  timeEnd = millis();
  Serial.print(timeEnd-timeStart);
  Serial.println(" <- BluetoothDataSend");
}
```
