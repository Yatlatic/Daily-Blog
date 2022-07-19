---
layout: post
toc: true
title: "Cesium-不基于Timeline的轨迹回放"
categories: Cesium
author:
  - Yatlatic
---

项目要求不基于Cesium自带的Timeline进行轨迹回放

相对来说，使用Timeline可以利用Cesium自带的功能函数进行时间点与位置坐标的配对，利用CallBackProperty进行回调即可。

不基于Timeline的轨迹回放亦可以利用这种思想，即利用时间的变化来不断更新位置坐标，实现轨迹的回放

##  要点 1 : CallBack Property的回调原理

CallBackProperty回调的根本原理，是监听函数内部是否存在Change事件。若有值在变化，则每变化一次函数就会调用一次。

那么在不使用Cesium自带的SampleProperty属性的前提下，我们可以通过不断获取当前时间来满足需要有值的变化这一回调条件。

##  要点 2 : 如何构建时间与位置坐标的关系

先设置了一个初始时间initTime来获取用户点击时的时间，再设置一个currentTime来不断获取当前时间，它们的差值为deltaTime，通过设置speed*deltaTime来获取一个Distance，那么对于任意相邻的两个位置属性点，我们可以通过该线性变化的Distance来获取该两点连线Distance处的位置坐标，由此构建了所有属性点的轨迹连线上，任意位置坐标与时间的函数关系。

##  源代码
```
      this._animationCallback = function (e) {
      let points = that._positions
      // 相邻点彼此位置
      let start = Cesium.Cartographic.fromDegrees(points[that._startpoint].lon, points[that._startpoint].lat, points[that._startpoint].height)
      let end = Cesium.Cartographic.fromDegrees(points[that._endpoint].lon, points[that._endpoint].lat, points[that._endpoint].height)
      let geodesic = new Cesium.EllipsoidGeodesic();
      geodesic.setEndPoints(start, end);
      // 控制轨迹回放速度
      let speed = 1000;
      let currentTime = Date.now();
      let deltaTimeInSecond = (currentTime - that._initTime) / 1e3;
      // 计算公式
      let distance_tra = speed * deltaTimeInSecond
      // 如果距离小于已知两个点的距离
      if (points[that._startpoint].lon) {
        if (distance_tra < geodesic.surfaceDistance) {
          // 获取两个点连线上，距离起始点distance_tra距离点的坐标
          let otherPosition = geodesic.interpolateUsingSurfaceDistance(distance_tra)
          // 将该点坐标转换为地理坐标经纬度
          let point_x = Cesium.Math.toDegrees(otherPosition.longitude)
          let point_y = Cesium.Math.toDegrees(otherPosition.latitude)
          let point_height = 0
          // 高度的回放按照距离占总行程的比例来定义
          if (points[that._endpoint].height - points[that._startpoint].height > 0) {
            point_height = points[that._startpoint].height + ((distance_tra / geodesic.surfaceDistance) * (Math.abs(points[that._endpoint].height - points[that._startpoint].height)))
          } else {
            point_height = points[that._startpoint].height - ((distance_tra / geodesic.surfaceDistance) * (Math.abs(points[that._endpoint].height - points[that._startpoint].height)))
          }
          that._cumulativePosition.push(Cesium.Cartesian3.fromDegrees(point_x, point_y, point_height))
          return that._cumulativePosition;
        } else if (that._endpoint == points.length - 1 || that._stop == true) {
          // 到轨迹数组最后时通过赋值来停止回调
          // 压入末尾点避免距离问题导致的轨迹残缺
          that._cumulativePosition.push(Cesium.Cartesian3.fromDegrees(points[that._endpoint].lon, points[that._endpoint].lat, points[that._endpoint].height))
          that._entity.polyline.positions = that._cumulativePosition
        } else {
          that._cumulativePosition.push(Cesium.Cartesian3.fromDegrees(points[that._endpoint].lon, points[that._endpoint].lat, points[that._endpoint].height))
          that._cumulativePosition = _.uniqWith(that._cumulativePosition, _.isEqual)
          distance_tra = 0
          that._initTime = currentTime
          that._startpoint += 1
          that._endpoint += 1
        }
      }
    }
```

##  注意点 1 : 轨迹缺失问题

轨迹缺失问题主要是因为时间有最小单位，当速度过大时，Distance存在最小值，实际的显示效果中，回放的轨迹会越过某些位置坐标。

为此，我们可以在DisTance计算到一个位置坐标点处时，将该_endpoint点压入进公共数组_cumulativePosition中来避免轨迹缺失的问题。

##  注意点 2 : 如何停止回调

之前讲到CallBackProperty的回调是通过不断获取当前时间来实现的，可回调函数需要停止，时间却不会停止。

在这里通过将_cumulativePosition赋值给正在不断更新的轨迹来实现停止，即通过赋值来取消对CallBackProperty的调用
