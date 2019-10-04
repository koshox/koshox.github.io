---
title: 使用OSM开源地图库计算多点间驾车距离
date: 2019-10-04 23:09:45
tags: 
- OpenStreetMap    
- 驾车距离计算
categories:
- 其他

---

公司有个项目要计算多点间的驾车距离，开始想用Google的Distance Matrix API，看了下免费Key每天只能计算2500个元素（元素 = 起点数量 * 终点数量），收费标准每1000个元素0.5刀，6000个点就是1.8w刀。且限制颇多，数据只允许缓存一个月，QPS不超过100，每天计算元素上限10w，基本不可用。然后就想到了开（mian）源（fei）的OpenStreetMap地图库。
<!-- more -->

###  地图下载
OSM可以在http://download.geofabrik.de/ 下载各国家地图包，地图数据还是比较全的。

### 寻路框架
有了地图数据，还需要一个寻路计算框架，我找到了一个免费的Java库[osm2po](http://osm2po.de/)

### 地图数据生成
下载osm2po以后修改demo.sh或demo.bat的路径为你下载的pbf地图文件路径：
![生成地图数据](http://pypc1ne42.bkt.clouddn.com/blog/20191004/WclYQKdEYbIY.jpg)

执行后会在目录下生成gph地图数据文件，并启动一个Http服务器http://localhost:8888/Osm2poService ，打开就可以看见地图界面了：
![中国地图](http://pypc1ne42.bkt.clouddn.com/blog/20191004/gabifDrX01Lk.jpg?imageslim)



### 寻路效果

随便寻个路，效果还可以，国内路线看起来和高德地图差不多。
#### osm2po：
![osm2po寻路](http://pypc1ne42.bkt.clouddn.com/blog/20191004/4K0NMXoIU0w7.jpg?imageslim)

#### 高德地图：
![高德地图寻路](http://pypc1ne42.bkt.clouddn.com/blog/20191004/rms2VCSWsPaS.jpg?imageslim)



### 计算多点间路网距离（Java）

Http访问方式只提供了这么些参数可以使用，不是很完善，没有distance的选项，http接口效率也不高，最好还是通过Java API来计算
![API](http://pypc1ne42.bkt.clouddn.com/blog/20191004/sNodupwxHXcd.jpg?imageslim)

计算两点间距离可以直接用官网示例的DefaultRouter，很简单。

多点距离在http://gis.stackexchange.com 发现作者说有提供Distance Matrix API，emmm不错，看了下jar包的确是有一个TspDefaultMatrix的类，直接上代码吧：

```java
public static void main(String[] args) throws Exception {
    File graphFile = new File(args[0]);
    Graph graph = new Graph(graphFile);

    // Somewhere in Graph
    LatLon source = new LatLon(32.0452460989,118.8318873038);
    LatLon target = new LatLon(31.8870800000,118.8300200000);

    // additional params for TspDefaultMatrix
    Properties params = new Properties();
    params.setProperty("findShortestPath", "true");
    params.setProperty("ignoreRestrictions", "false");
    params.setProperty("ignoreOneWays", "false");
    // 0.0 Dijkstra, 1.0 good A*
    params.setProperty("heuristicFactor", "0.0");

    int[] vertexIds = findClosestVertexIds(graph, source, target);
    Log log = new Log(Log.LEVEL_DEBUG).addLogWriter(new LogConsoleWriter());
    TspDefaultMatrix matrix = new TspDefaultMatrix(graph, vertexIds, Float.MAX_VALUE, log, params);

    float[][] distances = matrix.getCosts();
    for (int i = 0; i < distances.length; i++) {
        for (int j = 0; j < distances.length; j++) {
            System.out.println(distances[i][j]);
        }
    }

    graph.close();
}

public static int[] findClosestVertexIds(Graph graph, LatLon... latLons) {
    int[] vertexIds = new int[latLons.length];
    for (int i = 0; i < latLons.length; i++) {
        // if failed, return -1
        vertexIds[i] = graph.findClosestVertexId(
                (float) latLons[i].getLat(), (float) latLons[i].getLon());
    }
    return vertexIds;
}
```
算出来的结果21.01734公里，和高德地图完全吻合。

公司项目第一个使用的地方是一个鸟不拉屎的非洲小国家，除了主干道基本都是人踩出来的小路，固定地点寻路成功率达到了83%以上，失败的场景谷歌地图一般也找不到路。国内道路情况好多了，估计95%成功率是可以的。

实际测试1700个点，地图大小30M，生成有效数据240w条，堆内存使用6.5g，只用了60秒。

完美代替谷歌地图的Distance Matrix API。

### ps：

1. 距离相近的点，得到的地图块id可能相同，直接传入TspDefaultMatrix会导致error，要做一个去重
2. License只是说免费软件，具体不是很完善，如果和我一样只用来生成数据，应该还是ok的。