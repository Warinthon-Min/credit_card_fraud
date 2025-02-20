# credit_card_fraud
This dataset consists of credit card transactions in the western United States.

## **การวิเคราะห์อัตราการทุจริตตามประเภทการซื้อ**
ในโปรเจคนี้ เราจะทำการวิเคราะห์ข้อมูลการทำธุรกรรมทางบัตรเครดิตที่มีการทุจริต โดยใช้เทคนิคต่างๆ เช่น การแยกกลุ่มข้อมูล, การตรวจสอบค่า Missing, การปรับสมดุลของข้อมูล (upsampling) และการสร้างโมเดล Logistic Regression เพื่อทำนายการทุจริต

## **ขั้นตอนที่ 1: การโหลดข้อมูลและตรวจสอบค่าขาดหาย**
   **-โหลดข้อมูลจากไฟล์ credit_card_fraud.csv โดยใช้ฟังก์ชัน read_csv() จาก tidyverse**
  ```r
   ccf <- read_csv('credit_card_fraud.csv', show_col_types = FALSE)
  ```
  **-ตรวจสอบค่า missing ของคอลัมน์ is_fraud ซึ่งระบุว่ามีการทุจริตหรือไม่**
   ```r
  sum(is.na(ccf$is_fraud))  # Count total Null values

   ```
  **-ตรวจสอบการกระจายของข้อมูลในคอลัมน์ is_fraud ซึ่งจะบอกจำนวนของธุรกรรมที่ทุจริต (1) และไม่ทุจริต (0)**
    ```r
    table(ccf$is_fraud)
    ```

## **ขั้นตอนที่ 2: การคำนวณอัตราส่วนการทุจริต**
  **-คำนวณอัตราส่วนการทุจริต (fraud-to-non-fraud ratio) โดยการหารจำนวนธุรกรรมที่ทุจริต (1) ด้วยจำนวนธุรกรรมที่ไม่ทุจริต (0)**
   ```r
  # Calculate fraud ratio
  is_frauds <- sum(ccf$is_fraud == 1)
  not_frauds <- sum(ccf$is_fraud == 0)
  ratio <- is_frauds / not_frauds
  print(ratio)  # Check fraud-to-non-fraud ratio
   ```
  **-แสดงผลอัตราส่วนการทุจริต**
  [1] 0.00527492
 
  **-จากการคำนวณอัตราส่วนการทุจริตที่แสดงผลเป็น 0.00527492 หมายความว่า:**
  สำหรับธุรกรรมทั้งหมดในชุดข้อมูลนี้ อัตราส่วนการทุจริต (fraud) คือ 0.53% (ประมาณ 0.53% ของธุรกรรมทั้งหมดเป็นการทุจริต) ตัวเลขนี้แสดงให้เห็นว่าข้อมูลในชุดนี้มีอัตราการทุจริตที่ต่ำมากเมื่อเทียบกับธุรกรรมที่ไม่ทุจริต

**เนื่องจากข้อมูลการทุจริตมีจำนวนน้อย จึงจำเป็นต้องใช้วิธีการจัดการข้อมูลที่ไม่สมดุล (เช่น การทำ upsampling) เพื่อให้โมเดลสามารถเรียนรู้การทุจริตได้ดีขึ้นในภายหลังค่ะ**
## **ขั้นตอนที่ 3: การปรับสมดุลของข้อมูล (Upsampling)**
  
  - ใช้ฟังก์ชัน upSample() จากแพ็กเกจ caret เพื่อทำการเพิ่มจำนวนข้อมูลที่ไม่ทุจริตให้สมดุลกับข้อมูลที่ทุจริต
  - เปลี่ยนชื่อคอลัมน์ Class เป็น is_fraud เพื่อความสะดวกในการใช้งาน
  - เปลี่ยนค่าของ is_fraud เป็น factor โดยแทนค่าด้วย "No" (ไม่ทุจริต) และ "Yes" (ทุจริต)
   
   ```r
  # Apply upsampling
  set.seed(42)
  ccf_over <- upSample(x = ccf %>% select(-is_fraud), y = as.factor(ccf$is_fraud))
  nrow(ccf_over)
  # rename column Class to is_fraud
  ccf_over <- ccf_over %>%
  rename(is_fraud = Class)
  # convert objtect variable to factor
  ccf_over$is_fraud <- factor(ccf_over$is_fraud, levels = c(0, 1), labels = c("No", "Yes"))
  # check the result
  table(ccf_over$is_fraud)  # Should return counts for "No" and "Yes"
   ```
## **ขั้นตอนที่ 4: การจัดการกับข้อมูลเวลาและการคำนวณอายุ**
- เปลี่ยนข้อมูลในคอลัมน์ dob (วันเกิด) เป็นประเภท Date เพื่อใช้ในการคำนวณอายุ
- คำนวณอายุของผู้ถือบัตรโดยการคำนวณจำนวนวันตั้งแต่วันเกิดจนถึงปัจจุบัน แล้วแปลงเป็นปี
- คำนวณระยะเวลาที่ใช้ในการทำธุรกรรม (time_in_seconds) จากข้อมูล trans_date_trans_time
- แปลงข้อมูล
   ```r
   # Convert to Date time format
  ccf_over$dob <- as.Date(ccf_over$dob, format = "%Y-%m-%d")

  # creat age colums in data set
  ccf_over$age <- as.numeric(difftime(Sys.Date(), ccf_over$dob, units = "days")) / 365

  ccf_over$trans_date_trans_time <- as.POSIXct(ccf_over$trans_date_trans_time, 
											 format="%Y-%m-%d %H:%M:%S")

  # Convert HH:MM:SS into total seconds
  ccf_over$time_in_seconds <- as.numeric(format(ccf_over$trans_date_trans_time, "%H")) * 3600 + 
                            as.numeric(format(ccf_over$trans_date_trans_time, "%M")) * 60 + 
                            as.numeric(format(ccf_over$trans_date_trans_time, "%S"))
  # convert charactor variables to factors
  ccf_over$merchant <- as.factor(ccf_over$merchant)
  ccf_over$category <- as.factor(ccf_over$category)
  ccf_over$city <- as.factor(ccf_over$city)
  ccf_over$state <- as.factor(ccf_over$state)
  ccf_over$job <- as.factor(ccf_over$job)
  ```
## ขั้นตอนที่ 5: การแยกข้อมูลเป็นชุดฝึกสอน (Training set) และชุดทดสอบ (Test set)
- ใช้ฟังก์ชัน split_data() เพื่อสุ่มแบ่งข้อมูลเป็นชุดฝึกสอน (70%) และชุดทดสอบ (30%)
- ตรวจสอบการกระจายของข้อมูลในทั้งชุดฝึกสอนและชุดทดสอบ เพื่อให้แน่ใจว่าอัตราการทุจริตในทั้งสองชุดไม่แตกต่างกันมาก
     ```r
  # split data
  split_data <- function(data) {
  set.seed(42)
  n <- nrow(data)
  id <- sample(1:n, size=0.7*n)
  train_df <- data[id,]
  test_df <- data[-id,] 
  return( list(train_ccf=train_df,test_ccf=test_df))
  }
prep_ccf <- split_data(ccf_over)
sum(is.na(prep_ccf))
table(prep_ccf$train_ccf$is_fraud)
table(prep_ccf$test_ccf$is_fraud)
     ```
## **ขั้นตอนที่ 6: การสร้างและฝึกสอนโมเดล Logistic Regression**
- ใช้ฟังก์ชัน train() จากแพ็กเกจ caret เพื่อฝึกสอนโมเดล Logistic Regression โดยใช้ตัวแปร amt (จำนวนเงิน) และ time_in_seconds (เวลาทำธุรกรรมในหน่วยวินาที)
- ใช้เทคนิค Repeated k-fold cross-validation (5-fold) ในการฝึกสอนโมเดล
 ```r
## logistic regression method = "glm"
set.seed(42)
## repeated k-fold cv
ctrl <- trainControl(method = "cv",
                     number = 5) # k

# Train Logistic Regression
logit_model <- train(is_fraud ~ amt+time_in_seconds,
                     data = prep_ccf$train_ccf, 
                     method = "glm", 
                     family = binomial, 
                     metric = "Accuracy", 
                     trControl = ctrl)

# Check Model
summary(logit_model)
## final model
logit_model$finalModel
```
## **ขั้นตอนที่ 7: การทำนายและประเมินผล**
- ใช้โมเดลที่ฝึกสอนมาแล้วในการทำนายผลลัพธ์ในชุดฝึกสอน (train dataset) และชุดทดสอบ (test dataset)
- สร้าง confusion matrix เพื่อประเมินผลการทำนายทั้งในชุดฝึกสอนและชุดทดสอบ
- คำนวณ Precision, Recall และ F1-score เพื่อประเมินประสิทธิภาพของโมเดลในการทำนายการทุจริต
```r
# Generate Predictions train dataset
predict_train <- predict(logit_model,prep_ccf$train_ccf)

# Create the confusion matrix
conf_matrix_train <- confusionMatrix(factor(predict_train), factor(prep_ccf$train_ccf$is_fraud))

# Print the confusion matrix
cat("Confusion Matrix for Train Data:\n")
print(conf_matrix_train)

# Generate Predictions test dataset
predict_test <- predict(logit_model, prep_ccf$test_ccf)

# Get predicted probabilities for the test dataset
pred_prob_test <- predict(logit_model, prep_ccf$test_ccf, type = "prob")

# Apply the threshold to classify transactions as fraud if probability >= 0.5
predicted_fraud_test <- ifelse(pred_prob_test$Yes >= 0.5 ,"Yes","No")

# Create the confusion matrix
conf_matrix_test <- confusionMatrix(factor(predicted_fraud_test), factor(prep_ccf$test_ccf$is_fraud))

# Print the confusion matrix
cat("Confusion Matrix for test Data:\n")
print(conf_matrix_test)


# Convert confusion matrix from list to matrix
conf_matrix_matrix <- conf_matrix_test$table

# Print the matrix
print(conf_matrix_matrix)

precision <- conf_matrix_matrix[2,2]/(conf_matrix_matrix[2,1]+conf_matrix_matrix[2,2])
cat("Precision : \n",precision,"\n")

recall <- conf_matrix_matrix[2,2]/(conf_matrix_matrix[1,2]+conf_matrix_matrix[2,2])
cat("recall: \n",recall,"\n")

F1_score <- 2*(precision*recall)/(precision+recall)
cat("F1 score : \n",F1_score)
```
## ขั้นตอนที่ 8 อ่านค่าผลลัพธิ์ที่ได้

Call:
NULL

Coefficients:
                  Estimate Std. Error z value Pr(>|z|)    
(Intercept)     -1.288e+00  7.928e-03 -162.42   <2e-16 ***
amt              7.634e-03  2.794e-05  273.21   <2e-16 ***
time_in_seconds -4.113e-06  1.320e-07  -31.17   <2e-16 ***

Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

(Dispersion parameter for binomial family taken to be 1)

Null deviance: 655653  on 472953  degrees of freedom
Residual deviance: 408576  on 472951  degrees of freedom
AIC: 408582

Number of Fisher Scoring iterations: 7

**1. ค่าคงที่ (Intercept**)
- ค่าคงที่ (Intercept) = -1.288
- หมายความว่า หากค่าของ amt และ time_in_seconds เท่ากับศูนย์ ค่าความน่าจะเป็นของการทุจริต (fraud) จะเป็นค่า 1 (ค่าที่ได้จากฟังก์ชันลอจิสติก) ที่ต่ำที่สุด
- การที่ค่าคงที่มีค่าติดลบแสดงว่าโมเดลจะมีแนวโน้มที่จะทำนายเป็น "ไม่ทุจริต" มากกว่า "ทุจริต" เมื่อไม่ได้มีการระบุข้อมูลเพิ่มเติม

**2. ค่า amt (จำนวนเงิน)**
- ค่าพารามิเตอร์ของ amt = 7.634e-03 (หรือ 0.007634)
- ค่านี้บ่งชี้ว่า เมื่อจำนวนเงิน (amt) เพิ่มขึ้น 1 หน่วย (เช่น 1 บาท) โอกาสที่การทุจริตจะเกิดขึ้นจะเพิ่มขึ้นเล็กน้อย
- ค่าพี (Pr(>|z|)) < 2e-16 แสดงว่า amt มีความสัมพันธ์ที่มีนัยสำคัญทางสถิติในการทำนายการทุจริต

**3. ค่า time_in_seconds (เวลาที่ใช้ในการทำธุรกรรม)**
- ค่าพารามิเตอร์ของ time_in_seconds = -4.113e-06 หมายความว่า ถ้าเวลาที่ใช้ในการทำธุรกรรมเพิ่มขึ้น 1 วินาที ความน่าจะเป็นที่ธุรกรรมจะเป็นการทุจริตจะลดลงเล็กน้อย
- ค่าพี (Pr(>|z|)) < 2e-16 แสดงว่า time_in_seconds มีความสัมพันธ์ที่มีนัยสำคัญทางสถิติในการทำนายการทุจริต

**4. Deviance**
- Null deviance = 655653 หมายถึง โมเดลที่ไม่รวมตัวแปรใด ๆ (โมเดลที่ไม่พิจารณาปัจจัยใด ๆ) ให้ค่า deviance ที่สูง
- Residual deviance = 408576 หมายถึง โมเดลที่ได้จากการฝึกสอนโดยใช้ตัวแปร amt และ time_in_seconds โดยลดลงจากค่า deviance เดิม แสดงว่าโมเดลที่ฝึกมีประสิทธิภาพในการทำนายการทุจริตได้ดีกว่าโมเดลที่ไม่ใช้ตัวแปรเหล่านี้
- AIC (Akaike Information Criterion) = 408582 ใช้ในการประเมินว่าโมเดลนี้เหมาะสมแค่ไหน ค่า AIC ที่ต่ำกว่าจะบ่งชี้ว่าโมเดลเหมาะสมมากกว่า

**5. จำนวนการวนรอบ (Iterations)**
- Number of Fisher Scoring iterations = 7 หมายความว่าโมเดลต้องทำการคำนวณ 7 รอบเพื่อหาค่าพารามิเตอร์ที่ดีที่สุด

**การตีความ:**
   โมเดล Logistic Regression นี้ใช้ตัวแปร amt (จำนวนเงิน) และ time_in_seconds (เวลาที่ใช้ในการทำธุรกรรม) ในการทำนายการทุจริต amt มีผลบวกในการเพิ่มความน่าจะเป็นของการทุจริต, ขณะที่ time_in_seconds มีผลลบในการทำนายการทุจริต ค่า AIC และการลดค่า Residual deviance แสดงว่าโมเดลนี้มีประสิทธิภาพในการทำนายการทุจริตที่ดีกว่าโมเดลที่ไม่พิจารณาปัจจัยใด ๆ

Confusion Matrix for Train Data:
Confusion Matrix and Statistics

          Reference
Prediction     No    Yes
       No  223643  59073
       Yes  12515 177723
                                          
               Accuracy : 0.8486          
                 95% CI : (0.8476, 0.8497)
    No Information Rate : 0.5007          
    P-Value [Acc > NIR] : < 2.2e-16       
                                          
                  Kappa : 0.6974          
                                          
 Mcnemar's Test P-Value : < 2.2e-16       
                                          
            Sensitivity : 0.9470          
            Specificity : 0.7505          
         Pos Pred Value : 0.7911          
         Neg Pred Value : 0.9342          
             Prevalence : 0.4993          
         Detection Rate : 0.4729          
   Detection Prevalence : 0.5978          
      Balanced Accuracy : 0.8488          
                                          
       'Positive' Class : No              
                                          
Confusion Matrix for test Data:
Confusion Matrix and Statistics

          Reference
Prediction    No   Yes
       No  96301 25575
       Yes  5366 75454
                                          
               Accuracy : 0.8474          
                 95% CI : (0.8458, 0.8489)
    No Information Rate : 0.5016          
    P-Value [Acc > NIR] : < 2.2e-16       
                                          
                  Kappa : 0.6945          
                                          
 Mcnemar's Test P-Value : < 2.2e-16       
                                          
            Sensitivity : 0.9472          
            Specificity : 0.7469          
         Pos Pred Value : 0.7902          
         Neg Pred Value : 0.9336          
             Prevalence : 0.5016          
         Detection Rate : 0.4751          
   Detection Prevalence : 0.6013          
      Balanced Accuracy : 0.8470          
                                          
       'Positive' Class : No              
                                          
          Reference
Prediction    No   Yes
       No  96301 25575
       Yes  5366 75454
Precision : 
 0.9336055 
recall: 
 0.7468549 
F1 score : 
 0.8298533

**ผลการคำนวณ Confusion Matrix สำหรับทั้งชุดข้อมูลการฝึก (train data) และชุดข้อมูลทดสอบ (test data) ขอสรุปผลดังนี้:**

**1. Confusion Matrix for Train Data**
- Accuracy: 84.86%
- Sensitivity (Recall): 94.70%
หมายถึง ค่าความสามารถของโมเดลในการทำนายธุรกรรมทุจริตได้ถูกต้อง (มีการทำนายเป็น "Yes" สำหรับการทุจริตอย่างแม่นยำ)
- Specificity: 75.05%
หมายถึง ค่าความสามารถของโมเดลในการทำนายธุรกรรมที่ไม่ทุจริตได้ถูกต้อง (มีการทำนายเป็น "No" สำหรับธุรกรรมที่ไม่ทุจริต)
- Positive Predictive Value (Precision): 79.11%
หมายถึง ค่าความสามารถของโมเดลในการทำนายธุรกรรมทุจริตที่เป็นจริง (ธุรกรรมที่ทำนายว่าเป็นทุจริตและมีการทุจริตจริง)
- Negative Predictive Value: 93.42%
หมายถึง ค่าความสามารถของโมเดลในการทำนายธุรกรรมที่ไม่ทุจริตที่เป็นจริง
- F1 score: 0.8299
เป็นค่าคะแนนที่ใช้ในการประเมินความสมดุลระหว่าง Precision และ Recall

**2. Confusion Matrix for Test Data**
- Accuracy: 84.74%
- Sensitivity (Recall): 94.72%
- Specificity: 74.69%
- Positive Predictive Value (Precision): 79.02%
- Negative Predictive Value: 93.36%
- F1 score: 0.8299

**3. การตีความผลลัพธ์:**
- ความแม่นยำ (Accuracy) ของโมเดลอยู่ที่ประมาณ 84.7% ซึ่งถือว่าอยู่ในระดับที่ดี
- ความสามารถในการตรวจจับการทุจริต (Recall/Sensitivity) สูงถึง 94.7%, หมายถึงโมเดลสามารถทำนายธุรกรรมที่ทุจริตได้ถูกต้องมาก
- Precision หรือความสามารถในการทำนายว่าเป็นทุจริตจริง ๆ มีค่าประมาณ 79.1% (train data) และ 79.0% (test data)
- F1 score อยู่ที่ 0.8299 ซึ่งเป็นค่าคะแนนที่ดีในการแสดงถึงสมดุลระหว่าง Precision และ Recall
- Specificity ที่ 75% ใน train data และ 74.7% ใน test data หมายถึงโมเดลมีความสามารถในการตรวจจับธุรกรรมที่ไม่ทุจริตได้ค่อนข้างดี

**4. สรุป:**
โมเดลนี้มีความแม่นยำสูงในการทำนายธุรกรรมที่ทุจริตและธุรกรรมที่ไม่ทุจริต และมีการคำนวณ Precision และ Recall ที่ดี โดยเฉพาะการตรวจจับการทุจริต (Recall) ซึ่งสูงกว่า 94% ทั้งในชุดฝึกและชุดทดสอบ
F1 score ที่สูงแสดงถึงความสมดุลระหว่าง Precision และ Recall ที่ดี ผลลัพธ์นี้ช่วยยืนยันว่าโมเดล Logistic Regression สามารถใช้งานได้ดีในการตรวจจับการทุจริตในธุรกรรมบัตรเครดิต

[visit to see full version this link](https://www.datacamp.com/datalab/w/9287fa1d-6582-442c-a9f3-ed29229949ef/edit)
