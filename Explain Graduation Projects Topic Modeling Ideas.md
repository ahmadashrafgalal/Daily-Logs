# 📄 ملخص: Event-Driven Integration في مشروع التخرج (Topic Modeling)

## فهرس المحتويات

- [📄 ملخص: Event-Driven Integration في مشروع التخرج (Topic Modeling)](#-ملخص-event-driven-integration-في-مشروع-التخرج-topic-modeling)
  - [فهرس المحتويات](#فهرس-المحتويات)
  - [مقدمة](#مقدمة)
  - [Event Table Pattern](#event-table-pattern)
  - [Polling](#polling)
  - [Polling ذكي: last\_id tracking](#polling-ذكي-last_id-tracking)
  - [PostgreSQL SKIP LOCKED](#postgresql-skip-locked)
  - [LISTEN / NOTIFY](#listen--notify)
  - [Message Broker](#message-broker)
  - [Change Data Capture (CDC)](#change-data-capture-cdc)
  - [Idempotency](#idempotency)
  - [Derived Systems](#derived-systems)
  - [Batch vs Stream Processing](#batch-vs-stream-processing)
  - [Lambda Architecture](#lambda-architecture)
  - [Unifying Batch \& Stream](#unifying-batch--stream)
  - [نصائح عملية لمشروع التخرج](#نصائح-عملية-لمشروع-التخرج)

---

## مقدمة

في مشروع التخرج، عندنا نظام Topic Modeling.
السيناريو:

* عندنا بيانات بتجيلنا من Internship System.
* أي إضافة أو تعديل في البيانات (INSERT / UPDATE) محتاج تبعت notification للـ Topic Modeling Service.
* الـ Topic Modeling Service ممكن يكون Worker بلغة مختلفة (Python) عن Backend.

الفكرة الأساسية: **فصل الأنظمة وربطها بطريقة event-driven**، بحيث كل System يبقى مستقل، scalable، وقابل للصيانة.

---

## Event Table Pattern

**الوصف:**
إنشاء جدول events (مثلاً `domain_events`) لتخزين كل الأحداث المهمة في النظام.

**ميزات:**

* يقلل coupling بين Backend و Topic Modeling Worker
* ممكن يكون المصدر الوحيد للأحداث (source of truth)
* يسمح بعمل replay للأحداث القديمة بسهولة

**عيوب:**

* لو استعملت polling naive ممكن يزيد الحمل على الـ DB
* لو مش معمولة idempotency ممكن worker يعالج نفس الحدث مرتين

**طريقة التنفيذ:**

```sql
CREATE TABLE domain_events (
    id SERIAL PRIMARY KEY,
    event_type TEXT,
    payload JSONB,
    processed_at TIMESTAMP NULL
);
```

Backend بعد أي تغيير:

```sql
INSERT INTO domain_events (event_type, payload) VALUES ('internship_added', '{"id": 123}');
```

---

## Polling

**الوصف:**
الـ Worker يسأل الـ DB كل فترة (مثلاً 2–5 ثواني) عن الأحداث الجديدة.

**ميزات:**

* بسيط جدًا للتنفيذ
* مناسب لمشاريع صغيرة / متوسط الحمل

**عيوب:**

* لو الجدول ضخم بدون index → scan كامل → بطء
* لو فيه آلاف workers → ضغط على DB

**طريقة التنفيذ:**

```python
SELECT * FROM domain_events
WHERE processed = false
ORDER BY id
LIMIT 50;
```

---

## Polling ذكي: last_id tracking

**الوصف:**
الـ Worker يحتفظ بـ `last_processed_id` ويجلب الأحداث الأكبر منه.

**ميزات:**

* أقل تحميل على DB
* أسرع وأقرب real-time من polling عادي

**عيوب:**

* لازم Worker يحفظ آخر id بشكل دائم (local file أو DB)

**طريقة التنفيذ:**

```sql
SELECT * FROM domain_events
WHERE id > last_processed_id
ORDER BY id;
```

---

## PostgreSQL SKIP LOCKED

**الوصف:**
ميزة في PostgreSQL تسمح للـ Worker بمعالجة أحداث بدون صراع مع Workers آخرين.

**ميزات:**

* تمنع double processing
* تدعم عدة workers تعمل بالتوازي
* scalable أكثر من polling عادي

**عيوب:**

* خاص بـ PostgreSQL
* مش بديل كامل لـ message broker

**طريقة التنفيذ:**

```sql
SELECT * FROM domain_events
WHERE processed_at IS NULL
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

---

## LISTEN / NOTIFY

**الوصف:**
PostgreSQL يمكنه إرسال تنبيهات للـ Worker بمجرد إضافة حدث جديد، بدون polling.

**ميزات:**

* شبه real-time
* يقلل الحمل على DB
* لا تحتاج نظام message broker خارجي

**عيوب:**

* notifications مش durable → لو worker offline → ممكن تضيع
* لازم Worker يعمل SELECT على Event Table

**طريقة التنفيذ (Python):**

```python
import psycopg2
import select

conn = psycopg2.connect("dbname=mydb user=postgres")
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

cur = conn.cursor()
cur.execute("LISTEN new_event;")

while True:
    select.select([conn], [], [])
    conn.poll()
    while conn.notifies:
        notify = conn.notifies.pop(0)
        print("Got notification!")
        # اعمل SELECT على domain_events
```

---

## Message Broker

**الوصف:**
استخدام نظام خارجي مثل Kafka أو RabbitMQ لنقل الأحداث بين Backend و Workers.

**ميزات:**

* durable → لا تفقد الأحداث
* scalable → تدعم آلاف events في الثانية
* decouples Systems → كل service مستقل
* يدعم retry / replay / exactly-once semantics

**عيوب:**

* معقد أكثر من LISTEN / NOTIFY
* يتطلب setup خارجي و maintenance

**طريقة التنفيذ:**

1. Backend يرسل الرسائل إلى Topic في Kafka:

```python
producer.send("internship_events", key=id, value=event_payload)
```

2. Worker يشترك في Topic ويقرأ الرسائل:

```python
for message in consumer:
    process(message.value)
```

---

## Change Data Capture (CDC)

**الوصف:**
آلية لمراقبة تغييرات الـ DB وإرسالها كـ events.

**ميزات:**

* لا يحتاج تعديل كبير على الـ application logic
* يضمن consistency مع قاعدة البيانات
* ممكن يدمج مع message broker لتوزيع الأحداث

**عيوب:**

* أحيانًا يتطلب setup معقد للـ DB
* حجم الأحداث ممكن يكون كبير

**طريقة التنفيذ:**

* PostgreSQL → Logical Replication / Debezium
* أي تحديث على جدول محدد → يتحول لـ event في Kafka

---

## Idempotency

**الوصف:**
ضمان أن معالجة نفس الحدث أكثر من مرة لا تسبب مشاكل.

**ميزات:**

* يحمي من double processing
* مهم جدًا للـ distributed systems

**طريقة التنفيذ:**

```python
if event.id in processed_events_cache:
    continue  # ignore
else:
    process_event(event)
```

---

## Derived Systems

**الوصف:**
نظام فرعي يبنى على البيانات الأصلية ويحدث نفسه من الأحداث (مثلاً Topic Modeling Service يبني topics من domain_events).

**ميزات:**

* يفصل المعالجة عن الـ source system
* يسمح بإعادة المعالجة (replay) بسهولة
* scalable

**عيوب:**

* latency أعلى (asynchronous)
* لازم تصميم جيد للـ dataflows

**طريقة التنفيذ:**

* كل Worker يقرأ من Event Table / Message Broker
* يحول البيانات ويخزنها في نظامه الخاص (DB أو cache أو ML model)

---

## Batch vs Stream Processing

| المعيار         | Batch                | Stream                        |
| --------------- | -------------------- | ----------------------------- |
| البيانات        | finite               | unbounded                     |
| delay           | عالي                 | منخفض                         |
| إعادة المعالجة  | سهل                  | أصعب                          |
| fault tolerance | idempotent functions | managed state + checkpointing |

**نصيحة:**

* Stream → updates سريعة
* Batch → إعادة معالجة البيانات القديمة عند تعديل schema أو model

---

## Lambda Architecture

**الوصف:**
نظام مزيج بين Batch و Stream:

* Stream → updates سريعة، approximate
* Batch → reprocessing → accurate

**عيوب:**

* صعوبة صيانة logic مرتين
* merge النتائج صعب للـ joins المعقدة

---

## Unifying Batch & Stream

**الوصف:**
نظام واحد يقدر يعمل:

* Replay للأحداث القديمة
* Stream processing للأحداث الجديدة
* Exactly-once semantics

**أمثلة:** Apache Flink, Apache Beam

**ميزات:**

* يقلل التعقيد مقارنة بالـ Lambda
* يسمح بمرونة كبيرة في إعادة المعالجة وتحديث الـ derived systems

---

## نصائح عملية لمشروع التخرج

1. **ابدأ بالـ Event Table** → source of truth.
2. **استعمل last_id tracking** → أفضل من polling naive.
3. **لـ PostgreSQL فقط**: استخدم SKIP LOCKED للتوازي.
4. **لو عايز real-time:** LISTEN / NOTIFY مع SELECT على Event Table.
5. **لو حجم الأحداث كبير:** استخدم Message Broker (Kafka/RabbitMQ).
6. **لو عايز تتأكد من consistency مع DB:** استخدم CDC + Message Broker.
7. **اعمل idempotency** → مهم جدًا.
8. **صمم Derived System كو Worker مستقل** → Python أو أي لغة تانية.
9. **استخدم batch + stream حسب الحاجة** → reprocessing مهم عند تغيير schema أو model.

