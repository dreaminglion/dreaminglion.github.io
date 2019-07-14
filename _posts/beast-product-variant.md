---
layout: post
title:  "product variant"
date:   2016-11-18 23:00:00 +0800
categories: beast
---
[beast app 选择商品规格（这是个 app 下载链接哦，大家可以搜索四件套商品体验下）](http://static.thebeastshop.com/app/beast.apk)

<!--more-->

<video src="https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/beast-product-size.mp4" width="360" height="480" autoplay="true"  controls="controls"></video>


### struct `Spv`

- **id** `int` spv id
- **left** `int` 剩余库存量，这个只是一个参考值（比如用于界面展示），不能够用于严格的判断，因为库存量在不断变化
- **image** `URL` spv 图片
- **brand** 有修改
  - id `Int`
  - name `String`
- **price** `float`
- **rawPrice** `float`
- **minAmount** `int=1` minimum order amount 最小购买数量，默认为 1
- **discount** `Object=` **since 1.1**
  - appOnly `Boolean=false` 是否为 app 专享价，若为抢购价，则为 `false` **since 1.2**
  - vipLevel `Numbe=null` 是否为 vip 折扣，若为抢购价，则为 `null`
  - birthday `Boolean=false` 是否享受生日折扣，若在生日折扣时间内，且尚未使用过当年的生日折扣，则为 `true` **since 1.5**
- **quota** `Object=` 可购买限额 **since 1.5**
  - left `int` 当前用户目前还剩余的购买额度
  - total `int` 总限额，对于每个用户，最大购买的限额
- **flashSale** `Object=` **since 1.3** 如果不是闪购，或者闪购有效期之外，则为 `null`
  - expiresAt `timestamp` 单位秒，距离闪购结束的时间
  - soldOut `Boolean` 闪购限额已经售光。如果在闪购有效期内，且闪购限额已经售光，则商品不能购买，显示为 soldout。
- **presell** `Object=` 是否为预售商品，若不是预售，则为 `null`. **since 1.1**
  - deliverAt `timestamp` 预计发货时间
- **group** `Array.<Dimension.Choice>=` 规格维度的选项组合。在 cart 实体中，会不包含这个值
- **summary** `String` 一句话推荐
- **spvDesc** `String` spv 的描述
- **name** `String` 商品名
- **tax** `Object=` **since 1.2**
  - est `Float` 预估税费
  - included `Boolean` 是否已经包含税费
  - ratio `Float` 税率
  - typeName `String` 税收类型名 `"个人行邮税"`, or `"跨境综合税"`
- **deliveryDates** `Array.<String>` 配送日期列表
- **firstDeliveryDate** `String` 首次配送日期 格式 `X月X日`

### struct `Variant`

- dimensions `Array.<Dimension>` 若只有一个规格，该字段为空数组
- spvs `Array.<Spv>`


```json
{
  "pack" : {
    "count" : 2,
    "product" : {
      "brand" : {
        "name" : "THE BEAST"
      },
      "name" : "“用吻唤醒爱”永生花盒（复古粉）"
    },
    "variant" : 1
  },
  "selectAmount" : true,
  "variant" : {
    "spvs" : [
      {
        "group" : [
          0,
          0
        ],
        "id" : 1,
        "image" : "http://img.thebeastshop.com/imagePro/PROD001015716/PROD001015716_1_1468561146555.jpg",
        "left" : 100,
        "price" : 100,
        "rawPrice" : 500
      },
      {
        "group" : [
          0,
          1
        ],
        "id" : 2,
        "image" : "http://img.thebeastshop.com/imagePro/PROD001014690/PROD001014690_1_1467289174422.jpg",
        "left" : 0,
        "price" : 200,
        "rawPrice" : 300
      },
      {
        "group" : [
          1,
          1
        ],
        "id" : 3,
        "image" : "http://img.thebeastshop.com/imagePro/PROD001014690/PROD001014690_1_1467289174422.jpg",
        "left" : 10,
        "price" : 200,
        "rawPrice" : 300
      }
    ],
    "dimensions" : [
      {
        "choices" : [
          "26",
          "27"
        ],
        "name" : "尺码"
      },
      {
        "choices" : [
          "蓝色",
          "红色"
        ],
        "name" : "颜色"
      }
    ]
  }
}
```

## 是否弹出规格/数量选择 popover 的规则

- `dimensions` 为空，
  - `Spv::minAmount` 为 `1`，则不弹
  - 大于 `1`，则弹
- `dimensions` 不为空，弹

## variant 选择数据结构说明

本图对应上面 json 数据的部分情况。

<img src="https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/beast-product-size.jpg" width="360" height="480" />

1. dimensions 可以看出，这个商品有两组属性「尺码」、「颜色」对应的值「26、27」，「蓝色、红色」是可以进行选择的属性。
2. 按这个 2x2 的规格，全部规格是 4 种 group 的组合 「0，0」，「0，1」，「1，0」，「1，1」。
3. 但上文只给出了三组，及 「1，0」--> 「27，蓝色」 这组是没有的，没有这组的商品可以选择。
4. 图中第一组 dimensions 选择的尺码「26」，那么按照 spvs 颜色可以选择「蓝色、红色」但是 "group":[0,1] 的 left = 0 就是没有库存了，也是不可以选择。
5. 所以选择「26」的时候，可以选择「蓝色」，而「红色」是灰色的不可以选择。

## variant 选择规则

1. 由 dimensions 前端可以得到全部的 group 组合。
2. 过滤掉那些 spvs 中 left 小于最小购买数量的 group。
3. spvs 给出全部现有的规格商品，所有组合都没有规格始终是灰色的。
4. 当从上向下点击选中一个规格，拿到点击 group 的 index，开始遍历 spvs，得到对应全部 group 例：选中「26」得到 group 「0，0」，「0，1」但是「0，1」的库存不足，所以只有「0，0」可以选择。
5. 由步骤 4 得到选中「26」的时候，「0，1」是不可以选择的，所以「红色」就是灰色的，「蓝色」可以选择被选中。
6. 全部的 dimensions 有且只有一个规格选中，就完成了一个商品的选择，点击完成得到 group 对应的 id 就完成了选择。

如果大家觉得这个方案不错，我可以整理下代码，把这个组件在 [GitHub](https://github.com/dreaminglion) 开源一下。
