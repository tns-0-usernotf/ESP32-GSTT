#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Ticker.h>
#include <WiFi.h>
#include <HTTPClient.h>

// ==========================================
// 1. CẤU HÌNH WIFI & THINGSPEAK (ĐIỀN THÔNG TIN CỦA BẠN VÀO ĐÂY)
// ==========================================
const char* ssid = "TEN_WIFI_CUA_BAN";
const char* password = "MAT_KHAU_WIFI";
String writeAPIKey = "DIEN_WRITE_API_KEY_CUA_BAN_VAO_DAY"; 

// ==========================================
// 2. CẤU HÌNH CHÂN CẮM (PIN MAPPING)
// ==========================================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDRESS  0x3C

#define SOIL_PIN      34   
#define BUZZER_PIN    25   
#define LED_ESP_PIN   2    
#define LED_OUT_PIN   26   

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
Adafruit_MPU6050 mpu;
Ticker blinkTicker;

// --- Ngưỡng cảnh báo Thiên tai ---
const int LANDSLIDE_THRESHOLD = 80;        
const float EARTHQUAKE_THRESHOLD = 2.0;    

// --- Biến thời gian (Không dùng delay) ---
unsigned long lastDisplayTime = 0;
unsigned long lastThingSpeakTime = 0;
const int DISPLAY_INTERVAL = 500;          // Cập nhật OLED 0.5s/lần
const int THINGSPEAK_INTERVAL = 20000;     // Đẩy dữ liệu lên web 20s/lần (ThingSpeak giới hạn 15s/lần)

// ==========================================
// 3. CÁC HÀM PHỤ TRỢ
// ==========================================
void toggleLED() {
  digitalWrite(LED_ESP_PIN, !digitalRead(LED_ESP_PIN));
}

void triggerAlarm(String errorMessage) {
  blinkTicker.detach();
  Serial.println(errorMessage);
  unsigned long startMillis = millis();
  while (millis() - startMillis < 10000) {
    digitalWrite(LED_ESP_PIN, HIGH); digitalWrite(BUZZER_PIN, HIGH); delay(500);
    digitalWrite(LED_ESP_PIN, LOW);  digitalWrite(BUZZER_PIN, LOW);  delay(500);
  }
  while (1) { delay(10); } 
}

// ==========================================
// 4. HÀM SETUP (KHỞI ĐỘNG)
// ==========================================
void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT); pinMode(LED_ESP_PIN, OUTPUT);
  pinMode(LED_OUT_PIN, OUTPUT); pinMode(SOIL_PIN, INPUT);

  blinkTicker.attach(0.05, toggleLED); 

  Wire.begin();
  Wire.setClock(800000); 

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) triggerAlarm("Loi: OLED");
  
  // --- KẾT NỐI WIFI CÓ HIỂN THỊ LÊN OLED ---
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 10);
  display.print("Connecting WiFi...");
  display.display();

  WiFi.begin(ssid, password);
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) { // Thử kết nối 10 giây
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  display.clearDisplay();
  display.setCursor(0, 10);
  if (WiFi.status() == WL_CONNECTED) {
    display.println("WiFi: CONNECTED!");
    Serial.println("\nWiFi Connected. IP: " + WiFi.localIP().toString());
  } else {
    display.println("WiFi: OFFLINE MODE");
    Serial.println("\nWiFi Failed. Running Offline Mode.");
  }
  display.display();
  delay(1500);

  // --- KIỂM TRA MPU6050 ---
  if (!mpu.begin()) triggerAlarm("Loi: MPU6050");
  mpu.setAccelerometerRange(MPU6050_RANGE_4_G); 
  mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);   

  blinkTicker.detach(); 
  digitalWrite(LED_ESP_PIN, LOW);

  for (int i = 0; i < 5; i++) {
    digitalWrite(BUZZER_PIN, HIGH); delay(500);
    digitalWrite(BUZZER_PIN, LOW);  delay(500);
  }
}

// ==========================================
// 5. HÀM LOOP (CHẠY CHÍNH)
// ==========================================
void loop() {
  // --- A. ĐỌC CẢM BIẾN ---
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);
  float totalAccel = sqrt(a.acceleration.x * a.acceleration.x + a.acceleration.y * a.acceleration.y + a.acceleration.z * a.acceleration.z);
  float vibration = abs(totalAccel - 9.8);
  bool isEarthquake = (vibration > EARTHQUAKE_THRESHOLD);

  int soilValue = analogRead(SOIL_PIN);
  int soilMoisturePercent = map(soilValue, 4095, 1000, 0, 100);
  if (soilMoisturePercent < 0) soilMoisturePercent = 0;
  if (soilMoisturePercent > 100) soilMoisturePercent = 100;
  bool isLandslideRisk = (soilMoisturePercent > LANDSLIDE_THRESHOLD);

  // --- B. ĐIỀU KHIỂN ĐÈN/CÒI ---
  if (isLandslideRisk) digitalWrite(LED_OUT_PIN, HIGH); 
  else digitalWrite(LED_OUT_PIN, LOW);  

  if (isEarthquake) {
    digitalWrite(BUZZER_PIN, HIGH); digitalWrite(LED_ESP_PIN, HIGH);  
  } else {
    digitalWrite(BUZZER_PIN, LOW); digitalWrite(LED_ESP_PIN, LOW);
  }

  // --- C. HIỂN THỊ OLED ---
  if (millis() - lastDisplayTime >= DISPLAY_INTERVAL) {
    lastDisplayTime = millis();
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    
    // Trạng thái WiFi (Góc trên bên phải)
    display.setCursor(110, 0);
    if(WiFi.status() == WL_CONNECTED) display.print("WIFI");
    else display.print("OFF");

    display.setCursor(0, 0);
    if (isEarthquake) display.println("!! DONG DAT !!");
    else if (isLandslideRisk) display.println("!! SAT LO !!");
    else display.println("AN TOAN");
    
    display.setCursor(0, 25);
    display.print("Do am dat: "); display.print(soilMoisturePercent); display.println("%");
    
    display.setCursor(0, 40);
    display.print("Chan dong: "); display.print(vibration, 1); display.println(" m/s2");
    
    display.display();
  }

  // --- D. ĐẨY DỮ LIỆU LÊN THINGSPEAK (MỖI 20 GIÂY) ---
  if (millis() - lastThingSpeakTime >= THINGSPEAK_INTERVAL) {
    lastThingSpeakTime = millis();
    
    // Chỉ gửi nếu có mạng
    if (WiFi.status() == WL_CONNECTED) {
      HTTPClient http;
      
      // Tạo đường link API
      String url = "http://api.thingspeak.com/update?api_key=" + writeAPIKey + 
                   "&field1=" + String(soilMoisturePercent) + 
                   "&field2=" + String(vibration, 2) + 
                   "&field3=" + String(isLandslideRisk ? 1 : 0) + 
                   "&field4=" + String(isEarthquake ? 1 : 0);
      
      http.begin(url);
      int httpCode = http.GET(); // Thực hiện lệnh gửi
      
      if (httpCode > 0) {
        Serial.print("Da gui ThingSpeak. Ma HTTP: ");
        Serial.println(httpCode); // Code 200 là thành công
      } else {
        Serial.println("Loi gui ThingSpeak!");
      }
      http.end(); // Đóng kết nối để tiết kiệm RAM
    }
  }
}
