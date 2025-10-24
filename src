// === CẤU HÌNH BLYNK ===
#define BLYNK_TEMPLATE_ID "TMPL6ph8Zc7wt"
#define BLYNK_TEMPLATE_NAME "Bể cá thông minh"


#include <DallasTemperature.h>
#include <OneWire.h>
#include <Servo.h>
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>


// === CẤU HÌNH WIFI ===
char auth[] = "cKGrcCpV9pB5DOMvvMtNBsz7W1gVzcRB";  // Thay bằng Auth Token của bạn
char ssid[] = "Wifi_2.4G";         // Tên WiFi
char pass[] = "66668888";     // Mật khẩu WiFi


// === ĐỊNH NGHĨA CHÂN ===
// Cảm biến và relay
#define PH_SENSOR_PIN D6       // Cảm biến phao (mức nước)
#define RELAY_PUMP_PIN D5      // Relay kênh 1: bơm nước
#define RELAY_TEMP_PIN D4      // Relay kênh 2: nhiệt độ
#define ONE_WIRE_BUS D3        // Cảm biến nhiệt độ DS18B20


// Servo và nút nhấn cho ăn
#define SERVO_PIN D8           // Chân servo cho ăn (GPIO15)
#define BUTTON_PIN D7          // Chân nút nhấn cho ăn (GPIO13)


// === KHỞI TẠO ĐỐI TƯỢNG ===
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature DS18B20(&oneWire);
Servo feederServo;


// === CẤU HÌNH RELAY ===
#define RELAY_ON LOW          // Relay bật khi LOW (relay thường đóng)
#define RELAY_OFF HIGH        // Relay tắt khi HIGH


// === BIẾN TOÀN CỤC - NHIỆT ĐỘ ===
float tempC = 25.0;           // Nhiệt độ hiện tại
bool heaterOn = false;        // Trạng thái thiết bị sưởi
bool heaterManual = false;    // Chế độ thủ công cho nhiệt
const float setpoint = 22.0;      // Nhiệt độ mục tiêu (°C)
const float hysteresis = 1.0;     // Độ rộng vùng hysteresis (°C)


// === BIẾN TOÀN CỤC - BƠM NƯỚC ===
bool pumpOn = false;          // Trạng thái bơm
bool pumpManual = false;      // Chế độ thủ công cho bơm


// === BIẾN TOÀN CỤC - NÚT NHẤN CHO ĂN ===
unsigned long lastDebounceTime = 0;
const unsigned long debounceDelay = 50;  // Thời gian chống dội nút (ms)
bool lastButtonState = HIGH;


// === BIẾN TOÀN CỤC - TIMING ===
unsigned long lastSensorRead = 0;
unsigned long lastBlynkUpdate = 0;
const unsigned long SENSOR_INTERVAL = 1000;   // Đọc cảm biến mỗi 1s
const unsigned long BLYNK_INTERVAL = 2000;    // Cập nhật Blynk mỗi 2s


void setup() {
  Serial.begin(115200);
  Serial.println("=== HE THONG DIEU KHIEN BE CA TU DONG ===");
 
  // Khởi động cảm biến nhiệt độ
  DS18B20.begin();
 
  // Khởi động servo cho ăn
  feederServo.attach(SERVO_PIN);
  feederServo.write(0);   // Servo về vị trí ban đầu
 
  // Thiết lập chân I/O
  pinMode(PH_SENSOR_PIN, INPUT_PULLUP);  // Cảm biến phao
  pinMode(RELAY_PUMP_PIN, OUTPUT);       // Relay bơm
  pinMode(RELAY_TEMP_PIN, OUTPUT);       // Relay nhiệt
  pinMode(BUTTON_PIN, INPUT_PULLUP);     // Nút nhấn cho ăn
 
  // Tắt cả hai relay ban đầu
  digitalWrite(RELAY_PUMP_PIN, RELAY_OFF);  // Tắt bơm
  digitalWrite(RELAY_TEMP_PIN, RELAY_OFF);  // Tắt nhiệt
 
  // Kết nối WiFi và Blynk
  Serial.println(">> Dang ket noi WiFi...");
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.println(">> WiFi da ket noi!");
 
  Blynk.begin(auth, ssid, pass);
  Serial.println(">> Blynk da ket noi!");
 
  Serial.println(">> He thong da khoi dong thanh cong!");
  Serial.println(">> Nhan nut de cho ca an thu cong.");
  Serial.println("=====================================");
}


void loop() {
  Blynk.run();  // Chạy Blynk
 
  // === XỬ LÝ THEO THỜI GIAN (non-blocking) ===
  if (millis() - lastSensorRead >= SENSOR_INTERVAL) {
    controlWaterLevel();      // Điều khiển mức nước
    controlTemperature();     // Điều khiển nhiệt độ
    lastSensorRead = millis();
  }
 
  // === CẬP NHẬT BLYNK ===
  if (millis() - lastBlynkUpdate >= BLYNK_INTERVAL) {
    updateBlynk();
    lastBlynkUpdate = millis();
  }
 
  // === XỬ LÝ NÚT NHẤN CHO ĂN ===
  handleFeedingButton();
 
  delay(10);  // Delay ngắn để hệ thống ổn định
}


// === CẬP NHẬT DỮ LIỆU LÊN BLYNK ===
void updateBlynk() {
  // V0 - Hiển thị nhiệt độ
  Blynk.virtualWrite(V0, tempC);
 
  // V1 - Trạng thái mực nước (0: đầy, 1: thấp)
  bool isWaterLow = digitalRead(PH_SENSOR_PIN) == LOW;
  Blynk.virtualWrite(V1, isWaterLow ? 1 : 0);
 
  // V2 - Trạng thái bơm nước (0: tắt, 1: bật)
  Blynk.virtualWrite(V2, pumpOn ? 1 : 0);
 
  // V3 - Thông báo trạng thái nhiệt độ
  String tempStatus;
  if (tempC < 22) {
    tempStatus = "Nước quá lạnh!";
  } else if (tempC > 28) {
    tempStatus = "Nước quá nóng!";
  } else {
    tempStatus = "Nhiệt độ bình thường";
  }
  Blynk.virtualWrite(V3, tempStatus);
}


// === BLYNK - ĐIỀU KHIỂN BƠM THỦ CÔNG (V4) ===
BLYNK_WRITE(V4) {
  int pumpControl = param.asInt();
 
  if (pumpControl == 1) {
    // Chế độ thủ công BẬT
    pumpManual = true;
    digitalWrite(RELAY_PUMP_PIN, RELAY_ON);
    pumpOn = true;
    Serial.println(">> [BLYNK] Bom nuoc BAT (thu cong)");
  } else {
    // Tắt chế độ thủ công - quay lại tự động
    pumpManual = false;
    Serial.println(">> [BLYNK] Chuyen sang che do tu dong cho bom nuoc");
   
    // Kiểm tra ngay lập tức và điều khiển theo mức nước
    bool isWaterLow = isWaterLowStable();
    if (isWaterLow) {
      digitalWrite(RELAY_PUMP_PIN, RELAY_ON);
      pumpOn = true;
      Serial.println(">> [BLYNK] Bom nuoc BAT (tu dong - nuoc thap)");
    } else {
      digitalWrite(RELAY_PUMP_PIN, RELAY_OFF);
      pumpOn = false;
      Serial.println(">> [BLYNK] Bom nuoc TAT (tu dong - nuoc day)");
    }
  }
}


// === BLYNK - ĐIỀU KHIỂN NHIỆT THỦ CÔNG (V5) ===
BLYNK_WRITE(V5) {
  int heaterControl = param.asInt();
 
  if (heaterControl == 1) {
    // Chế độ thủ công BẬT
    heaterManual = true;
    digitalWrite(RELAY_TEMP_PIN, RELAY_ON);
    heaterOn = true;
    Serial.println(">> [BLYNK] May suoi BAT (thu cong)");
  } else {
    // Tắt chế độ thủ công - quay lại tự động
    heaterManual = false;
    Serial.println(">> [BLYNK] Chuyen sang che do tu dong cho nhiet do");
   
    // Đọc nhiệt độ hiện tại và điều khiển ngay lập tức
    getTemperature();
    controlHeaterWithHysteresis(tempC);
   
    // Thông báo trạng thái sau khi điều khiển tự động
    if (heaterOn) {
      Serial.println(">> [BLYNK] May suoi BAT (tu dong - nhiet do thap)");
    } else {
      Serial.println(">> [BLYNK] May suoi TAT (tu dong - nhiet do cao)");
    }
  }
}


// === BLYNK - CHO CÁ ĂN TỰ ĐỘNG (V6) ===
BLYNK_WRITE(V6) {
  int feedControl = param.asInt();
 
  if (feedControl == 1) {
    feedFish();
    Serial.println(">> [BLYNK] Cho ca an tu dong qua Blynk");
   
    // Tự động reset nút về 0 sau khi cho ăn
    Blynk.virtualWrite(V6, 0);
  }
}


// === ĐIỀU KHIỂN MỨC NƯỚC ===
void controlWaterLevel() {
  // Nếu đang ở chế độ thủ công, không tự động điều khiển
  if (pumpManual) {
    return;
  }
 
  bool isWaterLow = isWaterLowStable();
 
  if (isWaterLow) {
    digitalWrite(RELAY_PUMP_PIN, RELAY_ON);   // Bật bơm
    pumpOn = true;
    Serial.println(">> [NUOC] MUC NUOC THAP - BOM DANG CHAY (tu dong)");
  } else {
    digitalWrite(RELAY_PUMP_PIN, RELAY_OFF);  // Tắt bơm
    pumpOn = false;
    Serial.println(">> [NUOC] MUC NUOC DAY - BOM DUNG (tu dong)");
  }
}


// === ĐIỀU KHIỂN NHIỆT ĐỘ ===
void controlTemperature() {
  getTemperature();
 
  Serial.print(">> [NHIET] Nhiet do hien tai: ");
  Serial.print(tempC);
  Serial.print(" °C (Muc tieu: ");
  Serial.print(setpoint);
  Serial.println(" °C)");
 
  // Nếu không ở chế độ thủ công, điều khiển tự động
  if (!heaterManual) {
    controlHeaterWithHysteresis(tempC);
  }
}


// === XỬ LÝ NÚT NHẤN CHO ĂN ===
void handleFeedingButton() {
  bool currentButtonState = digitalRead(BUTTON_PIN);
 
  // Phát hiện nút được nhấn (biến trạng thái từ HIGH sang LOW)
  if (lastButtonState == HIGH && currentButtonState == LOW) {
    if (millis() - lastDebounceTime > debounceDelay) {
      feedFish();
      lastDebounceTime = millis();
    }
  }
 
  lastButtonState = currentButtonState;
}


// === CHỐNG NHIỄU KHI ĐỌC CẢM BIẾN PHAO ===
bool isWaterLowStable() {
  int lowCount = 0;
 
  for (int i = 0; i < 10; i++) {
    if (digitalRead(PH_SENSOR_PIN) == LOW) {
      lowCount++;
    }
    delay(5);
  }
 
  return lowCount >= 7;  // Cần ít nhất 7/10 lần đọc LOW
}


// === ĐỌC NHIỆT ĐỘ CÓ TIMEOUT AN TOÀN ===
void getTemperature() {
  unsigned long startTime = millis();
 
  do {
    DS18B20.requestTemperatures();
    tempC = DS18B20.getTempCByIndex(0);
   
    if (tempC == -127.0) {
      delay(100);
    }
  } while (tempC == -127.0 && millis() - startTime < 2000);
 
  // Xử lý lỗi cảm biến
  if (tempC == -127.0 || tempC < -50 || tempC > 80) {
    Serial.println("!! [LOI] CAM BIEN NHIET DO DS18B20 BI LOI");
    tempC = 25.0; // Gán giá trị giả định an toàn
  }
}


// === ĐIỀU KHIỂN RELAY NHIỆT VỚI HYSTERESIS ===
void controlHeaterWithHysteresis(float currentTemp) {
  float lowerThreshold = setpoint - hysteresis;  // 26°C
  float upperThreshold = setpoint + hysteresis;  // 28°C
 
  if (!heaterOn && currentTemp < lowerThreshold) {
    digitalWrite(RELAY_TEMP_PIN, RELAY_ON);  // Bật thiết bị nhiệt
    heaterOn = true;
    Serial.print(">> [NHIET] Nhiet do THAP (");
    Serial.print(currentTemp);
    Serial.println(" °C) - May suoi BAT (tu dong)");
   
  } else if (heaterOn && currentTemp > upperThreshold) {
    digitalWrite(RELAY_TEMP_PIN, RELAY_OFF); // Tắt thiết bị nhiệt
    heaterOn = false;
    Serial.print(">> [NHIET] Nhiet do CAO (");
    Serial.print(currentTemp);
    Serial.println(" °C) - May suoi TAT (tu dong)");
  }
}


// === HÀM CHO CÁ ĂN BẰNG SERVO ===
void feedFish() {
  Serial.println(">> [CHO AN] Bat dau cho ca an...");
 
  // Quay servo để thả thức ăn
  feederServo.write(90);   // Quay servo tới góc thả thức ăn
  delay(800);              // Đợi servo quay và thức ăn rơi
 
  // Quay servo về vị trí ban đầu
  feederServo.write(0);    
  delay(500);              // Đợi servo về vị trí
 
  Serial.println(">> [CHO AN] Hoan tat cho an ca!");
 
  // Thông báo lên Blynk (notification)
  Blynk.logEvent("fish_fed", "Đã cho cá ăn thành công!");
 
  Serial.println("=====================================");
}


// === BLYNK - KẾT NỐI LẠI ===
BLYNK_CONNECTED() {
  Serial.println(">> Blynk da ket noi lai!");
  // Đồng bộ trạng thái khi kết nối lại
  updateBlynk();
}




