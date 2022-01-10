## Aggregation
Aggregation คือ การประมวลผลข้อมูลหลาย record ให้มารวมกันในที่เดียว และจะ return ผลลัพธ์ออกมา โดยสามารถใช้ประโยชน์ดังนี้
- ใช้ในการ group หรือจับกลุ่มข้อมูลจากที่อื่นมารวมกัน
- ใช้ในการจัดการข้อมูล ดำเนินการข้อมูล และ return ค่าผลลัพธ์ออกไปเป็นชุดข้อมูลเดียว
- ใช้ในการหาผลลัพธ์ข้อมูลเช่น sum , average , count
- ใช้ในการวิเคราะห์การเปลี่ยนแปลงข้อมูลที่เกี่ยวกับการเวลา

การดำเนินการสามารถทำได้ดังนี้
- Aggregation Pipelines
- Single Purpose Aggregation Operations
- Map-reduce functions

### Aggregation Pipelines
Aggregation Pipelines เป็นส่วนหนึ่งในการประมวลผลของข้อมูล
- แต่ละ stage ในการดำเนินการของการ input ข้อมูล ตัวอย่างเช่น 
  - stage สามารถใช้ในการ filter ค่าของข้อมูลออกมาได้
  - ใช้จัดกลุ่มข้อมูล คำนวณค่าที่อยู่ใน document ออกมาได้
- ข้อมูลหรือ document ที่มาจาก stage หนึ่งจะถูกส่งต่อไปยัง stage ถัดไป
- aggregation pipeline จะสามารถ return ค่าผลลัพธ์สำหรับกลุ่มของข้อมูลได้ ตัวอย่างเช่น
  - return ค่าผลรวม , ค่า average , ค่า min , ค่า max  


#### ตัวอย่าง Aggregation Pipeline
```js
db.orders.insertMany( [
   { _id: 0, productName: "Steel beam", status: "new", quantity: 10 },
   { _id: 1, productName: "Steel beam", status: "urgent", quantity: 20 },
   { _id: 2, productName: "Steel beam", status: "urgent", quantity: 30 },
   { _id: 3, productName: "Iron rod", status: "new", quantity: 15 },
   { _id: 4, productName: "Iron rod", status: "urgent", quantity: 50 },
   { _id: 5, productName: "Iron rod", status: "urgent", quantity: 10 }
] )
```
ตัวอย่าง aggregation pipeline ที่รวมการทำงาน 2 stage เข้าไว้ด้วยกัน และ return total ค่า quantity ของ urgent orders แต่ละ product ออกมา

```js
db.orders.aggregate( [
   { $match: { status: "urgent" } },
   { $group: { _id: "$productName", sumQuantity: { $sum: "$quantity" } } }
] )
```

##### $match:
- ใช้ในการ filter ค่า status ที่ตรงกับ urgent

##### $group:
- ใช้จัดกลุ่มข้อมูล productName.
- ใช้ $sum ในการคำนวณหา total quantity ในแต่ละ productName ซึ่งจะถูกเก็บไว้ใน field ของ sumQuantity และจะ returned aggregation pipeline.

```js
    $min, $max หาค่าของที่น้อยที่สุด หรือมากที่สุดของ field ที่ระบุ
    $avg หาค่าเฉลี่ยของ field ที่ระบุ
    $addToSet จับ field ที่ระบุมารวมกันเพื่อสร้างเป็น array ขึ้นมาใหม่ โดยข้อมูลใน array นี้จะไม่ซ้ำกัน
    $first, $last กรณีข้อมูลเรียงมาก่อนหน้นานี้ สามารถหา document ตัวแรกสุด หรือตัวสุดท้ายมาใช้งานได้
    $push, จับ field ที่ระบุมารวมกันเพื่อสร้างเป็น array ใหม่โดยไม่คัดตัวที่ซ้ำกันออกไป
    $sum, หาผลรวมของข้อมูลในกลุ่ม (ตั้งแต่ MongoDB 3.2 $sum สามารถนำมาใช้ใน $project stage ได้ด้วย)
```

ตัวอย่าง
```js
[
   { _id: 'Steel beam', sumQuantity: 50 },
   { _id: 'Iron rod', sumQuantity: 60 }
]
```

##### $project
ใช้สำหรับเลือก field ที่จะส่งไปยัง stage ถัดไป คล้ายกับที่คำสั่ง $match ที่เราใช้เลือก ข้อมูลหรือ document ที่จะส่งไปยัง stage ถัดไป โดยการที่เราเลือก document หรือเลือก field ก่อนที่จะส่งไปยัง stage ถัดไปนั้นส่งผลให้ขนาดของแต่ละ document ลดลงไปด้วย ทำให้ stage ถัดไป process document เฉพาะในส่วนที่ต้องใช้งานจริงๆ เท่านั้น และนี่ส่งผลดีต่อ performance ของระบบ

##### $unwind
ใช้สำหรับการแตกข้อมูล Array ออกมาแล้วจับคู่เข้ากับ document ที่เป็น owner เป็นรายตัวไป ยกตัวอย่างเช่น ถ้าเรามีข้อมูลแบบนี้

{ "_id" : 1, "item" : "ABC1", sizes: [ "S", "M", "L"] }
แล้วเราลอง $unwind แบบนี้

db.inventory.aggregate( [ { $unwind : "$sizes" } ] )
จะได้ผลลัพธ์ดังนี้

{ "_id" : 1, "item" : "ABC1", "sizes" : "S" }
{ "_id" : 1, "item" : "ABC1", "sizes" : "M" }
{ "_id" : 1, "item" : "ABC1", "sizes" : "L" }
ถ้า stage ถัดไปเรา group by size และ count เราก็จะทราบจำนวนว่าแต่ละ size นั้นมี item กี่ชิ้น ดังนี้

db.inventory.aggregate([
 { $unwind: '$sizes' },
 { $group: { _id: '$sizes', count: { $sum:1 } } }
])



#### การ Run Aggregation Pipeline
ในการรันจะใช้ 

```js
db.collection.aggregate() หรือ aggregate
```

#### การ Update ข้อมูลโดยใช้ Aggregation Pipeline

##### *findAndModify*  
```js
db.collection.findOneAndUpdate()  
db.collection.findAndModify()  
```
##### *update*  
```js
db.collection.updateOne()  
db.collection.updateMany()  
Bulk.find.update()  
Bulk.find.updateOne()  
Bulk.find.upsert()  
```

### Aggregation Pipeline Optimization

Aggregation Pipeline มีระยะการปรับให้เหมาะสมซึ่งพยายามปรับปรุง pipeline เพื่อประสิทธิภาพที่ดีขึ้น

#### ลำดับการเพิ่มประสิทธิภาพ (Pipeline Sequence Optimization)

##### *($project or $unset or $addFields or $set) + $match Sequence Optimization*

สำหรับ Aggregation Pipeline ($project หรือ $unset หรือ $addFields หรือ $set) ตามด้วย $match stage ตัว MongoDB จะย้าย filter ใน stages $match ที่ไม่ต้องการค่าที่คำนวณใน stages ก่อนไปที่ new $match stage ก่อนแสดงผลลัพธ์


##### *$sort + $match Sequence Optimization*
หากมีลำดับด้วย $sort ตามด้วย $match และ $match จะย้ายก่อน $sort เพื่อลดจำนวน object ที่จะเรียงลำดับ ตัวอย่างเช่น หาก pipeline ประกอบด้วย stages ต่อไปนี้

```js
{ $sort: { age : -1 } },
{ $match: { status: 'A' } }
```

การปรับให้เหมาะสม ตัวเพิ่มประสิทธิภาพจะเปลี่ยนลำดับการทำงานดังนี้

```js
{ $match: { status: 'A' } },
{ $sort: { age : -1 } }
```
##### *$redact + $match Sequence Optimization*
หากมี stage $redact ตามด้วย stage $match การรวมบางครั้งสามารถเพิ่มส่วนของ stage $match ก่อน stage $redact หาก $match ที่เพิ่มอยู่ที่จุด start ของ pipeline กา aggregation สามารถใช้ index ในการ query collection เพื่อจำกัดจำนวนของข้อมูล ที่เข้าสู่ pipeline  

```js
{ $redact: { $cond: { if: { $eq: [ "$level", 5 ] }, then: "$$PRUNE", else: "$$DESCEND" } } },
{ $match: { year: 2014, category: { $ne: "Z" } } }
```

การเพิ่มประสิทธิภาพสามารถเพิ่ม $match stage เดียวกันก่อน $redact stage

```js
{ $match: { year: 2014 } },
{ $redact: { $cond: { if: { $eq: [ "$level", 5 ] }, then: "$$PRUNE", else: "$$DESCEND" } } },
{ $match: { year: 2014, category: { $ne: "Z" } } }
```

##### *$project/$unset + $skip Sequence Optimization*
หากมีลำดับด้วย $project หรือ $unset ตามด้วย $skip ตัว $skip จะย้ายก่อน $project

```js
{ $sort: { age : -1 } },
{ $project: { status: 1, name: 1 } },
{ $skip: 5 }
```
สามารถปรับเปลี่ยนลำดับการทำงานได้ดังนี้

```js
{ $sort: { age : -1 } },
{ $skip: 5 },
{ $project: { status: 1, name: 1 } }
```

##### การเพิ่มประสิทธิภาพโดยการรวม Pipeline (Pipeline Coalescence Optimization)
When possible, the optimization phase coalesces a pipeline stage into its predecessor. Generally, coalescence occurs after any sequence reordering optimization.
การปรับให้เหมาะสมจะรวม phase pipeline เข้ากับ stage ก่อนหน้า โดยทั่วไปการรวมตัวกันจะเกิดขึ้นหลังจากการเรียงลำดับใหม่แล้ว

##### *$sort + $limit Coalescence*
เมื่อ $sort มาก่อน $limit การเพิ่มประสิทธิภาพสามารถรวม $limit เข้ากับ $sort หากไม่มี stage ที่รบกวนแก้ไขจำนวนข้อมูล , MongoDB จะไม่รวม $limit เข้ากับ $sort หากมี stage pipeline ที่เปลี่ยนจำนวนข้อมูลระหว่างขั้นตอน $sort และ $limit

```js
{ $sort : { age : -1 } },
{ $project : { age : 1, status : 1, name : 1 } },
{ $limit: 5 }
```

สามารถปรับเปลี่ยนลำดับการทำงานได้ดังนี้

```js
{
    "$sort" : {
       "sortKey" : {
          "age" : -1
       },
       "limit" : NumberLong(5)
    }
},
{ "$project" : {
         "age" : 1,
         "status" : 1,
         "name" : 1
  }
}
```

##### *$limit + $limit Coalescence*
หากมี $limit ตามด้วย $limit ทั้งสอง stage สามารถรวมกันเป็น $limit เดียว แต่จะมีจะนวนได้น้อยกว่าค่าทั้ง 2 ตัว 

```js
{ $limit: 100 },
{ $limit: 10 }
```

การรวมกันส่งผลให้ ค่า limit จะได้มากสุดเท่าตัวที่ต่ำสุดเท่านั้น

```js
{ $limit: 10 }
```

##### *$skip + $skip Coalescence*
หากมี $skip ตามด้วย $skip ต่อกัน ทั้งสองขั้นตอนสามารถรวมกันเป็น $skip เดียวได้ โดยที่จำนวนการข้ามคือผลรวม limit ของจำนวนการข้ามทั้ง 2 รวมกัน

```js
{ $skip: 5 },
{ $skip: 2 }
```

หลังจากการรวมกัน

```js
{ $skip: 7 }
```

##### *$match + $match Coalescence*
เมื่อ $match ตามด้วย $match ต่อกัน ทั้งสองขั้นตอนสามารถรวมกันเป็น $match เดียวรวมเงื่อนไขกับ $and

```js
{ $match: { year: 2014 } },
{ $match: { status: "A" } }
```

หลังจากการรวมกัน

```js
{ $match: { $and: [ { "year" : 2014 }, { "status" : "A" } ] } }
```

##### *$lookup + $unwind Coalescence*
เมื่อ $unwind ตามด้วย $lookup ต่อกัน และ $unwind ทำงานบนฟิลด์ as ของ $lookup จะสามารถรวม $unwind เข้ากับ $lookup เพื่อหลีกเลี่ยงการสร้างเอกสารขนาดใหญ่

```js
{
  $lookup: {
    from: "otherCollection",
    as: "resultingArray",
    localField: "x",
    foreignField: "y"
  }
},
{ $unwind: "$resultingArray"}
```

หลังจากการรวมกัน

```js
{
  $lookup: {
    from: "otherCollection",
    as: "resultingArray",
    localField: "x",
    foreignField: "y",
    unwinding: { preserveNullAndEmptyArrays: false }
  }
}
```

#### Aggregation Pipeline Limits
##### ข้อจำกัดขนาดผลลัพธ์ (Result Size Restrictions)
คำสั่ง aggregation สามารถ return cursor หรือเก็บผลลัพธ์ในคอลเล็กชันได้ ข้อมูลแต่ละชุดในชุดผลลัพธ์ต้องมีขนาดเอกสาร BSON ขนาด 16 MB หากเอกสารใดเกินขีดจำกัดขนาดเอกสาร BSON การรวมจะทำให้เกิดข้อผิดพลาด ขีดจำกัดใช้กับเอกสารที่ return เท่านั้น ระหว่างการประมวลผล pipline ข้อมูลอาจมีขนาดเกินนี้ db.collection.aggregate() จะ return cursor โดยเป็นค่า default

##### ข้อจำกัดของจำนวน Stage (Number of Stages Restrictions)
MongoDB จะให้มี pipeline มากสุดที่ 1000 

##### ข้อจำกัดของหน่วยความจำ (Memory Restrictions)
pipeline stage จะมี limit อยู่ที่ 100 MB เท่านั้น หากเกินจะเกิดข้อผิดพลาด

##### Single Purpose Aggregation Operations
MongoDB จัดเตรียม db.collection.estimatedDocumentCount(), db.collection.count() และ db.collection.distinct() 
การดำเนินการทั้งหมดเหล่านี้ในการ aggregation จากคอลเลกชันเดียว แม้ว่าการดำเนินการเหล่านี้จะทำให้เข้าถึงกระบวนการรวมทั่วไปได้ง่าย แต่ก็ขาดความยืดหยุ่นและความสามารถของ Aggregation Pipeline

![image](https://user-images.githubusercontent.com/53977686/148652662-21d0476d-e872-407d-be25-cc60de8391a5.png)

#### Map-Reduce

Map-reduce สามารถเขียนใหม่ได้โดยใช้ตัวดำเนินการ Aggregation Pipeline เช่น $group, $merge และอื่นๆ

สำหรับการใช้ Map-reduce ต้องกใช้ฟังก์ชันที่กำหนดเอง MongoDB จัดเตรียมตัวดำเนินการ Aggregation $accumulator และ $function ใช้ตัวดำเนินการเหล่านี้เพื่อกำหนดนิพจน์การรวมแบบกำหนดเองใน JavaScript

เนื้อหาเพิ่มเติม
https://docs.mongodb.com/manual/aggregation/


