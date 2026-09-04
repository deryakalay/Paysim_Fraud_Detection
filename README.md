# PaySim Dolandırıcılık Tespiti

## Proje Hakkında

Bu proje, **PaySim** veri seti kullanılarak mobil para transferlerinde dolandırıcılık işlemlerinin tespit edilmesine yönelik uçtan uca bir makine öğrenmesi çalışmasıdır.

PaySim, gerçek mobil para hizmetlerinde gözlemlenen işlem örüntülerinden yararlanılarak oluşturulmuş sentetik bir finansal işlem veri setidir. Orijinal veri seti:

- **6.362.620 işlem**
- **11 değişken**

içermektedir.

Veri setindeki dolandırıcılık işlemlerinin oranı yalnızca yaklaşık **%0,1291** seviyesindedir. Bu nedenle proje kapsamında yalnızca doğruluk metriğine odaklanılmamış; özellikle:

- Precision
- Recall
- F1-Score
- PR-AUC
- ROC-AUC

metrikleri değerlendirilmiştir.

Projede üç farklı yaklaşım karşılaştırılmıştır:

- Random Forest
- LightGBM
- Isolation Forest

Bunlara ek olarak feature engineering, zamansal veri ayrımı, veri sızıntısı analizi, hiperparametre optimizasyonu, karar eşiği optimizasyonu ve SHAP tabanlı model açıklanabilirliği uygulanmıştır.

---

## Problem Tanımı

Amaç, mobil para işlemlerini aşağıdaki iki sınıftan birine ayırmaktır:

- **0 — Normal işlem**
- **1 — Dolandırıcılık işlemi**

Veri setindeki en önemli problemlerden biri ciddi sınıf dengesizliğidir.

| Sınıf | İşlem Sayısı | Oran |
|---|---:|---:|
| Normal İşlem | 6.354.407 | %99,8709 |
| Dolandırıcılık | 8.213 | %0,1291 |

Bu dağılım yaklaşık olarak her **774 normal işleme karşılık 1 dolandırıcılık işlemi** anlamına gelmektedir.

Bu nedenle yalnızca accuracy kullanmak yanıltıcı olabilir. Model tüm işlemleri normal olarak tahmin etse bile yaklaşık %99,87 doğruluk elde edebilir.

---

## Veri Seti

PaySim veri seti başlangıçta:

- **6.362.620 satır**
- **11 sütun**

içermektedir.

Keşifsel veri analizi sonucunda dolandırıcılık işlemlerinin yalnızca aşağıdaki iki işlem türünde gerçekleştiği görülmüştür:

- `TRANSFER`
- `CASH_OUT`

Bu nedenle modelleme aşamasında yalnızca bu iki işlem türü kullanılmış ve yaklaşık **2,77 milyon işlemden** oluşan bir modelleme veri seti elde edilmiştir.

> Bu nedenle proje sonuçları tüm PaySim işlem tiplerinden ziyade özellikle TRANSFER ve CASH_OUT işlemleri kapsamında değerlendirilmelidir.

### Temel Değişkenler

| Değişken | Açıklama |
|---|---|
| `step` | İşlemin gerçekleştiği zaman adımı |
| `type` | İşlem tipi |
| `amount` | İşlem tutarı |
| `oldbalanceOrg` | Göndericinin işlem öncesi bakiyesi |
| `newbalanceOrig` | Göndericinin işlem sonrası bakiyesi |
| `oldbalanceDest` | Alıcının işlem öncesi bakiyesi |
| `newbalanceDest` | Alıcının işlem sonrası bakiyesi |
| `isFraud` | Hedef değişken |
| `isFlaggedFraud` | PaySim tarafından oluşturulan kural tabanlı fraud işareti |

`nameOrig` ve `nameDest` gibi yüksek kardinaliteli hesap kimlikleri doğrudan model girdisi olarak kullanılmamıştır.

`isFlaggedFraud` değişkeni de hedef değişkene oldukça yakın bir fraud sinyali taşıdığı için modelleme dışında bırakılmıştır.

---

## Proje Akışı

1. Verinin yüklenmesi ve temel kontroller
2. Keşifsel veri analizi
3. Sınıf dengesizliği analizi
4. İşlem tipi analizi
5. Feature engineering
6. Veri sızıntısı analizi
7. Zamansal train-validation-test ayrımı
8. Random Forest modeli
9. LightGBM modeli
10. Isolation Forest modeli
11. Optuna ile hiperparametre optimizasyonu
12. Karar eşiği optimizasyonu
13. Test seti değerlendirmesi
14. SHAP ile model açıklanabilirliği
15. Bulguların değerlendirilmesi

---

## Feature Engineering

Modelin fraud davranışlarını daha iyi öğrenebilmesi amacıyla çeşitli yeni değişkenler oluşturulmuştur.

Örnek olarak:

- İşlem tutarının gönderici bakiyesine oranı
- Gönderici ve alıcı bakiye ilişkileri
- Bakiye farkları
- Logaritmik işlem tutarı
- Zaman tabanlı değişkenler
- Saat bilgisi
- Gün bilgisi
- Döngüsel zaman değişkenleri
- Sıfır bakiye göstergeleri
- Yuvarlak tutar göstergeleri

oluşturulmuştur.

Özellikle bir işlemin tamamlanmasından sonra ortaya çıkan bilgilerin gerçek zamanlı fraud modeli içerisinde kullanılmamasına dikkat edilmiştir.

Bu yaklaşım veri sızıntısı riskini azaltmak için uygulanmıştır.

---

## Zamansal Train / Validation / Test Ayrımı

Veri seti rastgele bölünmek yerine işlemlerin zaman sıralaması dikkate alınarak ayrılmıştır.

| Veri Seti | İşlem Sayısı |
|---|---:|
| Train | 2.653.729 |
| Validation | 78.701 |
| Test | 37.979 |

Bu yöntem gerçek hayattaki kullanım senaryosuna daha yakındır.

Model geçmiş işlemler üzerinden eğitilir ve daha sonraki zaman dönemlerinde gerçekleşen işlemler üzerinde test edilir.

Fraud oranlarının dönemler arasında ciddi biçimde değiştiği görülmüştür:

- Train fraud oranı: yaklaşık **%0,2178**
- Test fraud oranı: yaklaşık **%3,2966**

Bu durum eğitim ve test dönemleri arasında bir dağılım değişimi olduğunu göstermektedir.

---

## Kullanılan Modeller

### Random Forest

Random Forest, birden fazla karar ağacını bir araya getiren ensemble learning yaklaşımı olarak kullanılmıştır.

Test setinde en yüksek PR-AUC sonucunu elde eden model olmuştur.

### LightGBM

LightGBM, büyük ölçekli tabular veri setleri için uygun bir gradient boosting algoritması olarak kullanılmıştır.

Modelin hiperparametreleri **Optuna** ile optimize edilmiştir.

Ayrıca karar eşiği validation veri seti üzerinden ayrıca değerlendirilmiştir.

### Isolation Forest

Isolation Forest, denetimsiz anomaly detection yaklaşımı olarak kullanılmıştır.

Bu sayede fraud detection probleminde supervised modeller ile anomaly detection yaklaşımının performansı karşılaştırılmıştır.

---

## Model Performansı

Test setinde elde edilen temel sonuçlar:

| Model | PR-AUC | ROC-AUC |
|---|---:|---:|
| **Random Forest** | **0.9995** | **1.0000** |
| LightGBM | 0.9948 | 0.9998 |

Random Forest test setinde en yüksek PR-AUC skoruna ulaşmıştır.

LightGBM de oldukça yüksek performans göstermiştir.

Isolation Forest ise supervised modellere kıyasla daha düşük performans üretmiştir.

Bu sonuç, bu veri setinde yalnızca anomali tespiti yaklaşımının fraud işlemlerini ayırt etmek için yeterli olmadığını göstermektedir.

---

## Karar Eşiği Optimizasyonu

Fraud detection problemlerinde modelin hangi olasılık değerinden sonra bir işlemi fraud olarak kabul edeceği önemli bir karardır.

LightGBM için validation seti üzerinden F1 skorunu maksimize eden karar eşiği:

**0.9973**

olarak bulunmuştur.

Ancak validation veri setinde optimize edilen bu eşik test setinde varsayılan **0.50** eşiğine kıyasla daha yüksek performans üretmemiştir.

Bu sonuç önemli bir noktaya işaret etmektedir:

> Bir dönem için optimize edilen karar eşiği, fraud davranışları ve sınıf dağılımı zaman içerisinde değiştiğinde sonraki dönemlerde aynı performansı göstermeyebilir.

Gerçek bir fraud detection sisteminde karar eşiklerinin düzenli olarak izlenmesi ve yeniden kalibre edilmesi gerekebilir.

---

## Veri Sızıntısı Deneyi

Projede ayrıca işlem tamamlandıktan sonra ortaya çıkan bazı değişkenlerin modele eklenmesiyle ayrı bir deney gerçekleştirilmiştir.

Bu özellikler modele dahil edildiğinde:

- **PR-AUC: 1.0000**
- **ROC-AUC: 1.0000**

sonuçları elde edilmiştir.

Ancak bu performans daha iyi bir gerçek zamanlı fraud modeli anlamına gelmemektedir.

Aksine, işlem sonrası ortaya çıkan değişkenlerin hedef değişken hakkında doğrudan veya çok güçlü bilgi taşıması model performansını yapay olarak artırabilir.

Bu nedenle söz konusu değişkenler nihai operasyonel modelden çıkarılmıştır.

---

## SHAP ile Model Açıklanabilirliği

LightGBM modelinin hangi değişkenlerden ne ölçüde etkilendiğini analiz etmek amacıyla SHAP kullanılmıştır.

En etkili üç özellik:

| Sıra | Özellik | Ortalama \|SHAP\| |
|---:|---|---:|
| 1 | `amount_ratio` | 0.8730 |
| 2 | `step` | 0.4986 |
| 3 | `oldbalanceOrg` | 0.3850 |

`amount_ratio`, işlem tutarının göndericinin mevcut bakiyesine oranını ifade etmektedir.

Bu değişkenin yüksek öneme sahip olması, yalnızca işlem tutarının değil, işlem tutarı ile hesap bakiyesi arasındaki ilişkinin de fraud tespitinde önemli bir sinyal taşıdığını göstermektedir.

`step` değişkeninin yüksek önemi ise fraud davranışlarının zaman içerisinde değişebileceği bulgusunu desteklemektedir.

---

## Temel Bulgular

- PaySim veri setindeki fraud oranı yalnızca yaklaşık **%0,1291** seviyesindedir.
- Fraud işlemleri yalnızca `TRANSFER` ve `CASH_OUT` işlem tiplerinde görülmektedir.
- Sınıf dengesizliği nedeniyle accuracy tek başına yeterli bir değerlendirme metriği değildir.
- Random Forest test setinde **0.9995 PR-AUC** ile en yüksek sonucu elde etmiştir.
- LightGBM test PR-AUC sonucu **0.9948** olarak ölçülmüştür.
- Supervised modeller Isolation Forest yaklaşımından daha başarılı sonuç vermiştir.
- Eğitim ve test dönemlerindeki fraud oranları arasında ciddi farklılık bulunmaktadır.
- İşlem sonrası bakiye değişkenlerinin kullanılması veri sızıntısına yol açabilmektedir.
- Validation veri setinde optimize edilen karar eşiği test döneminde aynı başarıyı göstermemiştir.
- SHAP analizinde `amount_ratio`, `step` ve `oldbalanceOrg` değişkenleri en etkili özellikler arasında yer almıştır.

---

## Değerlendirme Metrikleri

### Precision

Modelin fraud olarak tahmin ettiği işlemlerin ne kadarının gerçekten fraud olduğunu ölçer.

Yüksek precision, daha az yanlış fraud alarmı anlamına gelir.

### Recall

Gerçek fraud işlemlerinin ne kadarının model tarafından tespit edildiğini gösterir.

Finansal dolandırıcılık problemlerinde tespit edilemeyen fraud işlemlerinin maliyeti yüksek olabileceği için recall önemli bir metriktir.

### F1-Score

Precision ve Recall değerlerinin harmonik ortalamasıdır.

### ROC-AUC

Modelin fraud işlemlerini normal işlemlerden ayırma kabiliyetini genel olarak değerlendirir.

### PR-AUC

Precision ve Recall arasındaki performansı farklı karar eşikleri üzerinden değerlendirir.

Fraud sınıfının çok az olduğu dengesiz veri setlerinde özellikle anlamlı bir metriktir.

Bu nedenle proje kapsamında **PR-AUC temel model karşılaştırma metriklerinden biri olarak kullanılmıştır.**

---

## Projenin Sınırlılıkları

Bu çalışmanın bazı önemli sınırlılıkları bulunmaktadır:

1. PaySim gerçek işlem verisi değil, **sentetik bir veri setidir**.
2. Modelleme yalnızca `TRANSFER` ve `CASH_OUT` işlemleri üzerinde gerçekleştirilmiştir.
3. Hesap kimlikleri kullanılarak müşteri geçmişi veya network tabanlı özellikler oluşturulmamıştır.
4. Gerçek fraud sistemlerinde cihaz bilgisi, IP adresi, konum, müşteri profili ve geçmiş işlem davranışları gibi ek veriler kullanılabilir.
5. Eğitim ve test dönemlerindeki fraud oranları önemli ölçüde farklıdır.
6. Validation döneminde belirlenen karar eşiğinin ilerleyen dönemlerde yeniden kalibre edilmesi gerekebilir.

---

## Kullanılan Teknolojiler

- Python
- Pandas
- NumPy
- Scikit-learn
- LightGBM
- Optuna
- SHAP
- Matplotlib
- Seaborn

---

## Repo Yapısı

```text
paysim-fraud-detection/
│
├── README.md
├── paysim_fraud_detection.ipynb
├── requirements.txt
│
└── data/
    └── README.md
