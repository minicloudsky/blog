---
title: 力导向图
date: 2020-07-09 12:59:55
categories: 前端
tags:
- 前端

---

## force-graph 力导向图属性总结

### [力导向图](https://github.com/vasturiano/force-graph)

<!-- more -->

1. 基础容器

```html
<div id="graph"></div>
```

2. 基础节点数据

   ```json
   const gData = {
               "nodes": [
                   {
                       "id": "mysql",
                       "group": 1
                   },
                   {
                       "id": "redis",
                       "group": 1
                   },
                   {
                       "id": "hive",
                       "group": 2
                   },
                   {
                       "id": "impala",
                       "group": 2
                   }, {
                       "id": "hdfs",
                       "group": 3
                   }, {
                       "id": "张三",
                       "group": 3
                   }, {
                       "id": "李四",
                       "group": 4
                   }, {
                       "id": "王五",
                       "group": 4
                   }, {
                       "id": "赵六",
                       "group": 5
                   }, {
                       "id": "田七",
                       "group": 5
                   }, {
                       "id": "nccp",
                       "group": 6
                   }, {
                       "id": "kafka",
                       "group": 6
                   }
               ],
               "links": [
                   {
                       "source": "mysql",
                       "target": "张三",
                       "value": "持股"
                   },
                   {
                       "source": "redis",
                       "target": "kafka",
                       "value": "归属于"
                   },
                   {
                       "source": "张三",
                       "target": "kafka",
                       "value": "来源于"
                   },
                   {
                       "source": "hive",
                       "target": "nccp",
                       "value": "拥有"
                   }, {
                       "source": "impala",
                       "target": "nccp",
                       "value": "导出到"
                   }, {
                       "source": "李四",
                       "target": "hive",
                       "value": "管理"
                   }, {
                       "source": "王五",
                       "target": "redis",
                       "value": "负责"
                   }, {
                       "source": "赵六",
                       "target": "李四",
                       "value": "管理"
                   }, {
                       "source": "田七",
                       "target": "impala",
                       "value": "负责"
                   }, {
                       "source": "kafka",
                       "target": "李四",
                       "value": "被管理"
                   }, {
                       "source": "田七",
                       "target": "hdfs",
                       "value": "负责配置"
                   }
               ]
           };
   ```

3. 创建 基础的力导向图

   ```javascript
   const Graph = ForceGraph()(document.getElementById('graph')).graphData(gData);
   ```

   ![](1.png)

4. 节点之间添加小球滚动特效

   ```
   const Graph = ForceGraph()(document.getElementById('graph'))
   .graphData(gData)
   .linkDirectionalParticles(1) // .linkDirectionalParticles(int num) num为小球个数
   ```

   ![](2.png)

5. 显示节点标签

   ```JavaScript
   const Graph = ForceGraph()(document.getElementById('graph'))
         .graphData(data)
         .nodeLabel('id'); // id为节点的属性
   ```

   ![](4.png)

6. 显示节点间关系标签

   ```
   .linkLabel([str or fn]) //显示关系标签
   ```

7. 自动填充节点颜色

   （为了节省篇幅，以下省略 const Graph = ForceGraph()(document.getElementById('graph')) 部分代码，直接以属性方式添加）

   ```
   .nodeAutoColorBy('group') //根据节点属性自动填充不同的颜色
   ```

   ![](3.png)

8. 节点之间添加粒子滚动特效的粒子的颗粒直径（滚动粒子的节点直径）

   ```JavaScript
   .linkDirectionalParticleWidth(140) // 数字越大直径越大
   ```

   ![](5.png)

9. 弯曲节点之间的线条 .linkCurvature()

   ```
      const gData = {
         nodes: [...Array(9).keys()].map(i => ({ id: i })),
         links: [
           { source: 1, target: 4, curvature: 10 },
           { source: 1, target: 4, curvature: 0.5 },
           { source: 1, target: 4, curvature: -0.5 },
           { source: 5, target: 2, curvature: 0.3 },
           { source: 2, target: 5, curvature: 0.3 },
           { source: 0, target: 3, curvature: 0 },
           { source: 3, target: 3, curvature: 0.5 },
           { source: 0, target: 4, curvature: 0.2 },
           { source: 4, target: 5, curvature: 0.5 },
           { source: 5, target: 6, curvature: 0.7 },
           { source: 6, target: 7, curvature: 1 },
           { source: 7, target: 8, curvature: 2 },
           { source: 8, target: 0, curvature: 0.5 }
         ]
       };
    const Graph = ForceGraph()
         (document.getElementById('graph'))
           .linkDirectionalParticles(2)
           .linkCurvature('curvature') // 按照 curvature 对节点之间线条进行弯
           .graphData(gData); 
   ```

   ![](6.png)

10. 节点之间连线添加方向箭头  ->

   ```JavaScript
   .linkDirectionalArrowLength(6) 数字越大箭头越大
   ```

   ![](7.png)

### 力导向图-事件

1. 鼠标移动到节点上面时

   ```javascript
   .onNodeHover
   当鼠标移动到节点上面时候，将鼠标指针变成小手
   .onNodeHover(node => elem.style.cursor = node ? 'pointer' : null)
   
   ```

2. 点击节点时候触发

   ```JavaScript
   点击节点时候，将节点位置变为屏幕中央
   
   .onNodeClick(node => { // Center/zoom on node
                   Graph.centerAt(node.x, node.y, 1000);
                   Graph.zoom(8, 2000);
               });
   ```

   