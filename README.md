# FALLGUARD – PHÁT HIỆN TÉ NGÃ ESP32 + MPU6050 + MQTT

Bản mô phỏng học tập / prototype. Không phải thiết bị cảnh báo y tế đã được kiểm định.

## 1. Chức năng và kiến trúc

- ESP32 đọc MPU6050 qua I2C, mục tiêu 50 mẫu/giây, bằng tác vụ FreeRTOS riêng.
- Đọc gia tốc Ax/Ay/Az, tốc độ góc Gx/Gy/Gz; tính độ lớn gia tốc, roll, pitch và độ nghiêng so với tư thế đứng đã hiệu chuẩn.
- Serial Monitor hiển thị ở 115200 baud, mỗi khoảng 500 ms.
- MQTT gửi số đo, trạng thái phát hiện, nguồn dữ liệu và trạng thái LED/còi mỗi khoảng 500 ms.
- Web nhận MQTT qua WSS, hiển thị các thẻ HTML có class, hai biểu đồ, hai nút LED/còi ở Manual, nút xác nhận cảnh báo và hiệu chuẩn.
- Auto/Manual không tắt thuật toán phát hiện. Nếu xác nhận sự kiện, cảnh báo tự động có ưu tiên cao nhất.
- Phát hiện và cảnh báo cục bộ không phụ thuộc trang Web. Tác vụ lấy mẫu chạy riêng để giảm ảnh hưởng khi MQTT kết nối lại.
- Không gửi email, SMS hay tự gọi cứu hộ. Không có phần mềm App native; dashboard Web thích ứng màn hình điện thoại.

Không dùng tên, vị trí hoặc thông tin sức khỏe thật với broker công cộng của bản mô phỏng này.

## 2. Tệp dự án

| Tệp | Vai trò |
|---|---|
| README.html | Trang hướng dẫn nhanh, có nút sao chép code từng tệp |
| web/index.html | Dashboard Web |
| web/styles.css | Giao diện thích ứng PC/điện thoại |
| web/app.js | Nhận MQTT, vẽ biểu đồ và gửi lệnh có xác nhận |
| web/mqtt-wire.js | Giao thức MQTT 3.1.1 qua WebSocket, không dùng CDN |
| tests/ | Các kiểm tra thuật toán / giao thức / logic Web |

Giải nén toàn bộ rồi mở tutorial.html. Khi mở dashboard cục bộ, giữ bốn tệp trong thư mục web cạnh nhau.

## 3. Kết nối Wokwi

| Thành phần | Chân | ESP32 / kết nối |
|---|---|---|
| MPU6050 | VCC | 3V3 |
| MPU6050 | GND | GND |
| MPU6050 | SDA | GPIO21 |
| MPU6050 | SCL | GPIO22 |
| MPU6050 | AD0 | GND, chọn địa chỉ 0x68 |
| MPU6050 | XDA, XCL, INT | Để trống trong bản này |
| LED đỏ | Anode A | GPIO26 qua điện trở 220Ω |
| LED đỏ | Cathode C | GND |
| Buzzer piezo | Chân 2 (+) | GPIO27 |
| Buzzer piezo | Chân 1 (-) | GND |
| RESET / XÁC NHẬN | Hai chân khác hàng | GPIO25 và GND |
| TEST FALL | Hai chân khác hàng | GPIO32 và GND |

Hai nút dùng INPUT_PULLUP; nhấn = LOW. Nhấn giữ ít nhất 0,1 giây để qua bộ lọc nhấn 40 ms. Hai chân cùng một hàng của nút nhấn 4 chân nối sẵn với nhau, không dùng chúng như hai đầu công tắc.

MPU6050 mặc định 0x68. Nếu nối AD0 lên VCC thì địa chỉ đổi 0x69; muốn dùng cấu hình đó phải đổi MPU_ADDR trong code. Module thật phải có mức kéo lên I2C phù hợp logic 3,3 V; làm theo sơ đồ module cụ thể.

## 4. Công thức và đơn vị

Gia tốc tổng:

A_g = sqrt(Ax_g² + Ay_g² + Az_g²)

Gia tốc gồm trọng lực: cảm biến đứng yên thường đọc khoảng 1g, không phải 0g. 1g = 9,80665 m/s². Ngưỡng 2,5g tương đương khoảng 24,52 m/s².

Roll = atan2(Ay, Az) × 180/π

Pitch = atan2(-Ax, sqrt(Ay² + Az²)) × 180/π

Tạo vector đơn vị u = a / |a|. Khi hiệu chuẩn, lưu vector u0 ở tư thế đứng. Độ nghiêng = acos(clamp(u · u0, -1, 1)) × 180/π.

Độ đổi hướng sau va đập được tính bằng góc giữa vector gia tốc gần tĩnh sau va đập với một vector gần 0,5 giây trước va đập. Lịch sử hợp lệ được chọn trong khoảng 0,5–1,5 giây trước đó; nếu không có, dùng tư thế chuẩn.

Chỉ hiển thị góc trên Web khi độ lớn gia tốc trong 0,8–1,2g và đã hiệu chuẩn. Khi đang va đập mạnh, góc từ gia tốc không đáng tin; Web hiển thị dấu —. Serial vẫn in giá trị góc gần tĩnh gần nhất kèm valid=0. Không lấy số góc đó để xác nhận tư thế trong lúc va đập.

**Gyro là tốc độ góc (°/s), không phải góc (°).** Kéo thanh trượt Rotation trong Wokwi không trực tiếp đặt một góc nghiêng. Góc trong bản này được ước lượng từ hướng trọng lực ở trạng thái gần tĩnh; không tích phân gyro để tạo yaw, không dùng bộ lọc hợp nhất tư thế đầy đủ.

MPU6050 được cấu hình ±8g, độ nhạy 4096 LSB/g; gyro ±500°/s, độ nhạy 65,5 LSB/(°/s). Đọc I2C trực tiếp bằng Wire để kiểm tra số byte và lỗi giao tiếp, không cần thư viện MPU6050 ngoài.

## 5. Thuật toán xác nhận sự kiện

### Hiệu chuẩn ban đầu

Giữ cảm biến đứng yên ở tư thế đại diện người đang đứng khoảng 2 giây. Wokwi giữ mặc định Ax=0, Ay=0, Az=1g, các gyro=0.

Cần khoảng 100 mẫu liên tục: tổng gia tốc 0,9–1,1g, gyro <8°/s và đổi hướng giữa hai mẫu ≤3°. Khi đạt yêu cầu, lưu tư thế chuẩn rồi chuyển READY. Đây là hiệu chuẩn tư thế, không phải hiệu chuẩn đầy đủ sai số cảm biến theo nhiều trục.

### Điều kiện báo động

1. **Va đập:** A_g ≥2,5g → chuyển VERIFY, lưu đỉnh va đập và hướng trước va đập.
2. **Tư thế sau va đập:** trong cửa sổ 2,5 giây, độ nghiêng so với chuẩn ≥60°, độ đổi hướng so với trước va đập ≥45°.
3. **Giữ tư thế:** các điều kiện trên liên tục ít nhất 1,2 giây, A_g trong 0,8–1,2g và độ lớn gyro <30°/s.
4. Nếu thỏa → ALARM, tăng bộ đếm sự kiện một lần, LED nhấp nháy, còi 1.800Hz kêu ngắt quãng.
5. Nếu không thỏa trong 2,5 giây → về READY, không báo động.

Nếu khoảng cách giữa hai mẫu vượt 100 ms, hủy quá trình xác minh đang có để không giả coi thời gian mất mẫu là bất động. Nếu không có mẫu hợp lệ quá 200 ms hoặc giao tiếp I2C lỗi, báo SENSOR_FAULT. Lỗi cảm biến không được xem là “không té ngã”.

Cảnh báo ALARM được giữ đến khi người dùng nhấn RESET / Xác nhận trên mạch hoặc Web. Cảm biến lỗi sau khi báo động không tự xóa báo động. Bộ đếm sự kiện gồm cả TEST, chỉ lưu trong RAM và về 0 khi khởi động lại.

Dải ngưỡng này do dự án chọn để minh họa nguyên lý; không phải bộ ngưỡng được chứng nhận hoặc bảo đảm độ nhạy/đặc hiệu ngoài đời. Có thể bỏ sót ngã nhẹ, trượt ngồi, hoặc ngã không đổi tư thế đủ lớn; có thể báo nhầm khi làm rơi thiết bị hoặc va chạm rồi nằm xuống. Cảm biến phải gắn chắc, thường ở thân/eo; không thể áp dụng trực tiếp các ngưỡng này cho mọi vị trí đeo.

### Auto / Manual

- Auto: khi chưa có ALARM, LED và còi tắt. Khi ALARM, LED chớp theo chu kỳ 0,5 giây, còi kêu 350 ms mỗi giây.
- Manual: hai nút Web bật/tắt LED và còi độc lập; cả hai có thể cùng bật. Tự trở lại Auto sau 60 giây không có lệnh mới.
- Phát hiện té ngã luôn chạy trong Manual. ALARM tự động ghi đè đầu ra Manual; không thể tắt riêng còi bằng nút Manual khi ALARM. Dùng Xác nhận để xóa cảnh báo.
- Lỗi cảm biến tắt các yêu cầu Manual, LED chớp ngắn mỗi 1,5 giây để báo lỗi, còi không kêu. Nếu ALARM đã được giữ từ trước thì cảnh báo đó tiếp tục.
- Các trường led/buzzer trên MQTT là yêu cầu hoạt động logic. Trong ALARM chúng luôn true, còn chân GPIO nhấp nháy/kêu ngắt quãng theo nhịp; Web mô tả đúng kiểu hoạt động thay vì nháy theo từng gói.

## 6. Chạy dự án trên Wokwi

1. Mở https://wokwi.com/projects/new/esp32.
2. Dán nội dung wokwi/sketch.ino thay toàn bộ code mẫu của tab sketch.ino.
3. Dán nội dung wokwi/diagram.json thay nội dung tab diagram.json. Mạch sẽ tự hiện.
4. Menu cạnh các tab → New file… → tạo chính xác FallDetector.h; dán nội dung tệp cùng tên.
5. Tạo libraries.txt với hai dòng:

```
PubSubClient
ArduinoJson
```

Hoặc dùng Library Manager thêm hai thư viện tương ứng. Wire và WiFi có sẵn trong ESP32. Không cần Adafruit_MPU6050, không cần Blynk.

6. Bấm tam giác xanh Start the simulation; chờ hàng đợi biên dịch.
7. Serial Monitor baud 115200. Giữ MPU ở giá trị mặc định khoảng 2 giây mô phỏng.
8. Tìm trạng thái READY, A=1.00g, tilt=0.0deg, valid=1 và MQTT connecting... OK.
9. Mở dashboard. Giữ base topic mặc định như bên dưới, chờ hai gói mới để hiện ESP32 trực tuyến.
10. Lưu dự án vào tài khoản Wokwi của bạn nếu muốn có đường dẫn dùng lại.

Bản đã chạy dùng ESP32 DevKit V1, MPU6050 I2C 0x68. Khi bắt đầu, có thể thấy trạng thái CALIBRATING trước READY; đây là bình thường.

## 7. Thử nhanh bằng nút TEST

Đây là cách lặp lại bài thử thuận tiện, không yêu cầu chỉnh nhiều thanh trượt trong 2,5 giây.

1. Để MPU ở mặc định (0,0,1g), gyro=0; chờ READY.
2. Nhấn giữ nút TEST FALL màu xanh dương khoảng 0,1–0,2 giây.
3. ESP32 tạo chuỗi dữ liệu thử trong 5 giây: đứng yên → 0,2g trong 120ms → va đập 3g trong 60ms → nằm nghiêng 90°, giữ yên.
4. Serial ghi nguồn TEST_FALL. Sau va đập và đủ 1,2 giây tư thế ổn định, chuyển ALARM; LED đỏ nhấp nháy và còi kêu ngắt quãng.
5. Hết 5 giây tự trở về dữ liệu MPU6050, nhưng báo động vẫn giữ. MQTT có alarm_source=TEST_FALL để Web vẫn ghi rõ cảnh báo do TEST.
6. Nhấn RESET màu xanh lá hoặc Xác nhận trên Web để tắt báo động.

Nút TEST thay đầu vào thuật toán bằng chuỗi dữ liệu trong phần mềm; không làm phần cứng tự chuyển động. Nó kiểm tra thuật toán, đầu ra và đường hiển thị, không chứng minh khả năng phát hiện trên cơ thể thật. Khi dùng làm thiết bị theo dõi thật, đặt ALLOW_TEST_INPUT=false và bỏ nút TEST khỏi giao diện vận hành.

Các lệnh Serial (gõ vào ô Serial Monitor rồi Enter):

| Lệnh | Kết quả dự kiến |
|---|---|
| TEST_FALL | Chuỗi va đập + nghiêng + giữ yên → ALARM |
| TEST_BUMP | Va đập rồi trở lại đứng → không ALARM |
| TEST_TILT | Nghiêng dần 90° nhưng không va đập → không ALARM |
| RESET | Xóa cảnh báo, về Auto và đọc MPU thật |
| LIVE | Tương tự RESET, dừng TEST |
| CALIBRATE | Lấy lại tư thế chuẩn; chỉ khi không ALARM và cảm biến hợp lệ |

Web có ba nút TEST tương đương. TEST không nhận nếu chưa hiệu chuẩn, đang ALARM hoặc đang chạy một TEST khác. Trở về mặc định trước mỗi bài thử để diễn giải kết quả rõ ràng.

## 8. Thử dữ liệu cảm biến bằng thanh trượt Wokwi

Bấm MPU6050 để mở các thanh trượt. Gia tốc từng trục trên giao diện Wokwi có thể giới hạn ±2g; tổng ba trục vẫn có thể lớn hơn 2,5g. Firmware đã chọn dải đo ±8g để không bị cắt ở 2g mỗi trục.

| Tình huống | Ax, Ay, Az (g) | Kết quả |
|---|---|---|
| Đứng yên, hiệu chuẩn | 0, 0, 1 | A≈1g, tilt≈0°, READY |
| Nằm nghiêng chỉ bằng đổi hướng | 1, 0, 0 | A≈1g, tilt≈90°; không có va đập nên không ALARM |
| Va đập lớn đa trục | 2, 2, 1 | A≈3g → VERIFY |
| Sau va đập chuyển nhanh sang nằm | 1, 0, 0 | Nếu thao tác đủ nhanh và giữ ổn định trong cửa sổ, ALARM |
| Quay gyro X=90°/s | Ax/Ay/Az giữ gần 1g | Serial gyro≈90°/s; đây không phải đặt góc =90° |

Thao tác tay có thể vượt cửa sổ 2,5 giây và không báo động. Dùng TEST_FALL để có chuỗi thử định thời chính xác. Không tăng ngưỡng thời gian chỉ để che việc thao tác chậm nếu đang trình bày thuật toán cho chuyển động thật.

## 9. MQTT và dashboard

| Mục | Giá trị mặc định |
|---|---|
| Wi-Fi mô phỏng | Wokwi-GUEST, mật khẩu rỗng |
| Broker | broker.hivemq.com |
| ESP32 | TCP 1883 |
| Web | wss://broker.hivemq.com:8884/mqtt |
| Base topic | fallguard/hang-20261015-f4b628 |
| /state | ESP32 publish JSON, retained |
| /availability | online/offline, retained và Last Will |
| /cmd | Web gửi JSON, KHÔNG retained |

Base topic phải giống BASE_TOPIC trong sketch. Mỗi nhóm dùng mã riêng để tránh trộn dữ liệu. Topic không phải mật khẩu; broker công cộng không có phân quyền cho mô hình này.

Web tự nối broker. Kết nối broker thành công chưa có nghĩa ESP32 trực tuyến: cần hai gói state có thời gian tăng trong cùng session. Dữ liệu cũ trên broker không mở khóa nút điều khiển. Quá 8 giây thực không nhận gói mới thì khóa điều khiển, vẫn đánh dấu các số cũ rõ ràng.

Chọn Thủ công, đợi ESP32 xác nhận rồi dùng Bật LED/Bật còi. Web không tự đổi trạng thái thành “đã bật” trước xác nhận. Sau 10 giây không nhận xác nhận, báo chưa có xác nhận. Phiên MQTT được đổi ngẫu nhiên mỗi lần ESP32 kết nối lại để bỏ lệnh retained từ kết nối trước. Đây không phải cơ chế xác thực an toàn.

Hai biểu đồ giữ tối đa 120 mẫu trong phiên mở trang. Đồ thị gia tốc có điểm cam là đỉnh giữa hai bản tin để đỉnh xung ngắn không mất hoàn toàn khi chỉ gửi mỗi 500ms. Góc được để trống trong giai đoạn động mạnh; không nối các điểm qua mẫu góc không hợp lệ.

Ví dụ lệnh bật LED, chỉ trong Manual:

```json
{"id":"fg-unique-id","session":"1234567890abcdef","action":"led","value":true}
```

Phải dùng session thật vừa nhận, không dùng chuỗi ví dụ. Các action hỗ trợ:

| action | value |
|---|---|
| mode | AUTO / MANUAL |
| led | true / false |
| buzzer | true / false |
| ack | true |
| calibrate | true |
| test | FALL / BUMP / TILT |

Không dùng cùng topic này cho dữ liệu vườn cây. Hai dự án độc lập.

## 10. Deploy dashboard lên GitHub Pages

Workflow `.github/workflows/deploy-pages.yml` tự publish riêng thư mục `web/`; firmware, test và tài liệu không được đưa lên Pages.

1. Tạo repository GitHub và push toàn bộ dự án lên nhánh `main` hoặc `master`.
2. Trong repository, vào **Settings → Pages** và chọn **GitHub Actions** ở mục Build and deployment.
3. Mở tab **Actions**, chạy workflow **Deploy dashboard to GitHub Pages** (hoặc push một thay đổi trong `web/`).
4. Chờ workflow hoàn tất; URL dashboard được hiển thị trong job deploy. URL thường có dạng `https://<owner>.github.io/<repository>/`.
5. Mở URL đó qua HTTPS, giữ broker mặc định `wss://broker.hivemq.com:8884/mqtt` và xác nhận Base topic trùng với `BASE_TOPIC` trong `wokwi/sketch.ino`.
6. Chạy Wokwi, chờ Serial báo `MQTT connecting... OK`, rồi chờ dashboard nhận hai bản tin liên tiếp để hiện **ESP32 trực tuyến**.

Dashboard tự kết nối MQTT khi mở trang. Deploy trang web không khởi động ESP32/Wokwi và không tạo dữ liệu nếu thiết bị chưa publish. Trình duyệt và thiết bị phải cùng broker, cùng base topic; topic mặc định hiện tại là `fallguard/hang-20261015-f4b628`.

GitHub Pages chỉ phục vụ dashboard tĩnh. Broker HiveMQ ở cấu hình mặc định là broker công cộng không có ACL: topic không phải mật khẩu, người khác có thể xem hoặc gửi dữ liệu/lệnh. Chỉ dùng dữ liệu mô phỏng, không gửi dữ liệu cá nhân; với triển khai thật cần broker riêng có TLS, xác thực và ACL. Không chia sẻ lệnh điều khiển trên topic công cộng.

## 11. Lỗi thường gặp

| Hiện tượng | Xử lý |
|---|---|
| FallDetector.h: No such file | Tạo tab đúng tên, phân biệt chữ hoa/thường; không đặt .h.txt |
| Không tìm thấy PubSubClient / ArduinoJson | Kiểm tra libraries.txt hoặc Library Manager |
| SENSOR_FAULT / số không đổi | Kiểm tra MPU VCC/GND, SDA21/SCL22, AD0 nối GND, địa chỉ 0x68 |
| CALIBRATING không kết thúc | Đặt Ax=0 Ay=0 Az=1, tất cả gyro=0, giữ yên 2 giây mô phỏng |
| Đổi Rotation nhưng góc không đổi như mong đợi | Rotation là °/s; đổi hướng vector gia tốc để mô phỏng tư thế gần tĩnh |
| Chỉ nghiêng 90° mà không báo | Đúng: cần cả va đập trước đó và các điều kiện thời gian |
| Chỉ gõ/va cảm biến mà không báo | Đúng nếu không có tư thế sau va đập phù hợp |
| TEST bị từ chối | Đợi READY, xác nhận cảnh báo cũ, đợi TEST trước kết thúc |
| Sau TEST hết 5 giây còi vẫn kêu | ALARM được giữ; bấm RESET/Xác nhận |
| Bấm tắt còi Manual khi ALARM không được | Cảnh báo tự động có ưu tiên cao hơn; dùng nút Xác nhận |
| Web có MQTT nhưng không ESP32 trực tuyến | Chạy Wokwi, xem MQTT OK, kiểm tra cùng base topic và chờ 2 gói |
| Nút bị mờ | Cần thiết bị mới trực tuyến, cảm biến hợp lệ, hiệu chuẩn xong; LED/còi cần Manual và không ALARM |
| Dữ liệu MQTT đến chậm | Wokwi có thể chạy chậm hơn thời gian thực; kiểm tra đồng hồ mô phỏng và Internet |
| Mất broker | Auto/đọc cảm biến vẫn chạy cục bộ; broker công cộng không bảo đảm uptime |
| Trễ/mất mẫu >100ms | Xem max_sample_gap_ms và missed_samples; mô hình có thể bỏ lỡ xung, cần đánh giá hiệu năng/FIFO trước dùng thật |

## 12. Khi làm phần cứng

Dùng mạch nguồn ổn định, gắn cảm biến chắc vào vị trí đã chọn, bảo vệ dây và vỏ. Piezo nhỏ có thể điều khiển bằng GPIO theo thông số linh kiện; loa điện trở thấp/còi dòng lớn phải dùng transistor hoặc driver, không đấu trực tiếp GPIO. Kiểm tra giới hạn dòng và cấp nguồn theo module thực tế.

Dùng broker riêng có TLS, tài khoản và ACL cho triển khai thật; thêm giám sát pin, mất kết nối, hỏng cảm biến, thời gian phát hiện và đường xác nhận hỗ trợ độc lập. Bản hiện tại không có lưu sự kiện bền vững, thông báo tới người chăm sóc hoặc bảo đảm thông báo được nhận.

Không tự ngã, làm người khác ngã hoặc tháo bỏ phương án hỗ trợ sẵn có để thử thiết bị. Sử dụng chuỗi dữ liệu mô phỏng. Đánh giá với dữ liệu chuyển động có gán nhãn và quy trình phù hợp trước khi đưa ra bất kỳ tuyên bố về độ nhạy hoặc độ đặc hiệu.

## 13. Lience


- Bagalà và cộng sự (2012), đánh giá thuật toán trên té ngã thực tế: https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0037062

Nguồn nghiên cứu giúp giải thích nguyên lý và giới hạn; không phải bằng chứng kiểm định bộ ngưỡng hoặc sản phẩm này.
