# credit_card_fraud
This dataset consists of credit card transactions in the western United States.
## **การวิเคราะห์อัตราการทุจริตตามประเภทการซื้อ**
ในโปรเจคนี้ เราจะทำการวิเคราะห์ข้อมูลการทำธุรกรรมทางบัตรเครดิตที่มีการทุจริต โดยใช้เทคนิคต่างๆ เช่น การแยกกลุ่มข้อมูล, การตรวจสอบค่า Missing, การปรับสมดุลของข้อมูล (upsampling) และการสร้างโมเดล Logistic Regression เพื่อทำนายการทุจริต
## **ขั้นตอนที่ 1: การโหลดข้อมูลและตรวจสอบค่าขาดหาย**
   **โหลดข้อมูลจากไฟล์ credit_card_fraud.csv โดยใช้ฟังก์ชัน read_csv() จาก tidyverse**
   r
   ccf <- read_csv('credit_card_fraud.csv', show_col_types = FALSE)
  '''
  -**ตรวจสอบค่า missing ของคอลัมน์ is_fraud ซึ่งระบุว่ามีการทุจริตหรือไม่**
  -**ตรวจสอบการกระจายของข้อมูลในคอลัมน์ is_fraud ซึ่งจะบอกจำนวนของธุรกรรมที่ทุจริต (1) และไม่ทุจริต (0)**
