# 🌎 OSM2PGSQL

## 说明

osm2pgsql 用于将osm文件导入到PostgreSQL/PostGIS数据库，以渲染成地图或者其他用途。通常它只是一个工具链的一部分，例如还需要其他软件来进行实际的渲染（即把数据变成地图），把地图提供给用户等等。

Osm2pgsql是一个相当复杂的软件，它以复杂的方式与数据库系统和工具链的其他部分互动。这需要一段时间，直到你对它有了一些经验。强烈建议你用少量的OSM数据，例如你所在城市的数据来尝试使用osm2pgsql。不要一开始就导入整个地球的数据，这很容易出错，而且在处理了几个小时或几天的数据后，它就会崩溃。你可以为自己省去很多麻烦，做一些试运行，从小的提取量开始，逐步增加，沿途观察内存和磁盘使用情况。

熟悉PostgreSQL数据库系统和PostGIS的扩展以及SQL数据库查询语言是有帮助的。对Lua语言的一些知识也是有用的。

本手册总是记录当前版本的osm2pgsql。如果相同的信息只在某些版本中有效，该部分将有类似这样的说明。版本 >= 1.4.0

建议你始终使用最新发布的osm2pgsql版本。早期的版本有时包含一些早已被修复的错误。

## 准备数据库

### 编码

OpenStreetMap的数据来自世界各地，它总是使用UTF-8编码。osm2pgsql将把数据原封不动地写入数据库，所以它也必须是UTF-8编码。

在任何现代系统中，新数据库的默认编码应该是UTF-8，但为了确保，你可以在用createdb为osm2pgsql创建数据库时使用-E UTF8或--encoding=UTF8选项

### 调试PostgreSQL服务器

通常安装的PostgreSQL服务器带有一个默认的配置，这个配置对于大型数据库来说不是很好的调整。在运行osm2pgsql之前，你应该改变postgresql.conf中的这些设置，并重新启动PostgreSQL，否则你的系统将比必要的慢得多。

下面的设置是针对拥有64GB内存和快速固态硬盘的系统。第二列中的数值是建议，为典型的设置提供一个良好的起点，你可能需要根据你的使用情况来调整它们。第三列中的值是PostgreSQL 13的默认设置。

...

## 几何处理

osm2pgsql工作的一个重要部分是从OSM数据中创建几何图形。在OSM文件中，只有node节点有位置属性，way线路图形来自于子节点，关系图形来自于子线路和子节点。Osm2pgsql将所有来自相关对象的数据集合成有效的几何图形。

### 几何类型

PostGIS支持的几何体类型来自OpenGIS联盟（OGC）定义的简单特征。Osm2pgsql从OSM数据中创建这些类型的几何体。

1. Point 点，来自于nodes
2. LineString，来自于ways线路
3. Polygon，来自于闭合线路或者一些relation关系
4. MultiPoint，不会创建
5. MultiLineString，来自分段的way线路和relation关系
6. MultiPolygon，来自闭合线路和一些关系

### 单几何 vs 多几何

一般来说，osm2pgsql会创建它能创建的最简单的几何图形。nodes 转成 point、way转成LineString 或者Polygons。多边形关系可以变成多边形或多边形，这取决于它是否有一个或多个外环。同样地，如果路线从起点到终点是连接的，路线关系可以变成LineString，如果是不连接的，或者有一些地方，路线分裂了，可以变成MultiLineString。

在某些情况下，osm2pgsql会将Multi\*几何图形分割成简单的几何图形，并将每个图形添加到自己的数据库行中。这可以使渲染速度更快，因为渲染器可以处理几个较小的几何体，而不是必须处理一个大的几何体。但是，根据你对数据的处理情况，也可能导致问题。这篇博文对这个问题有一些更深入的讨论。关于如何配置的细节，请看flex和pgsql的输出章节。这也意味着你的id列不是唯一的，因为现在有多条记录从同一个OSM对象中创建。关于如何配置的细节，请参见flex和pgsql输出章节。这也意味着你的id列不是唯一的，因为现在有多个行从同一个OSM对象创建。请参阅主键和唯一ID部分，了解如何解决这个问题的选项。

### 几何的有效性

点的几何形状总是有效的（只要坐标在正确的范围内）。LineString几何图形也总是有效的，线条可能会与自己交叉，但这也没关系。这对于Polygon和MultiPolygon几何体来说更为复杂。有多种方法可以使这种几何体无效。例如，如果一个多边形的边界被画成八字形，那么结果将不是一个有效的多边形。数据库会很高兴地存储这种无效的多边形，但这可能会导致以后的问题，当你试图绘制它们或基于无效的几何图形进行计算（如面积）时。这就是为什么osm2pgsql从不将无效的几何图形加载到你的数据库中。相反，该对象被简单地忽略，没有任何消息。

你可以使用OpenStreetMap检查器的区域视图来帮助诊断multipolygons的问题。

### 节点的处理

Node点的几何总被转换为Point的几何

### 线路的处理

根据tag标签的不同，OSM的way模型不是LineString就是Polygon，或者两个都有。比如一个way标记为highway=primary，通常这是线条的特征，再比如，一个tag标记为landuse=farmland，通常是一个多边形的特征。如果一个带有多边形类型标签的方式没有被关闭，那么几何图形是无效的，这是一个错误，该对象被忽略。对于某些标签，如man\_made=pier，非封闭的方式是线性特征，封闭的方式是多边形特征。

如果地图想换一种方式解析，它可以使用area标签，area=yes会将通常是线性特征解析成多边形特征。举一个例子，将一个步行街(highway=pedestrian)变成步行区。同理，area=no，将多边形特征变成线性特征。

没有明确的清单，哪些标签表示线性或多边形特征。Osm2pgsql让用户决定。这取决于你选择的输出（见下一章）如何配置。示例配置文件中的列表应涵盖大多数常用的标签，但如果你使用更多不常用的标签，你可能需要扩展这些列表。

Osm2pgsql 可以切割长的LineString(创建于way)，使其变成更小的片段。这样可以使瓦片渲染速度更快，因为在渲染一个特定的瓦片时，只需要从数据库中检索出较小的几何图形。pgsql输出总是将长的LineStrings分割开来，在latlong投影中，线条不会超过1°，在Web Mercator中，线条不会超过100,000单位（在赤道上约为100,000米）。只有在使用split\_at转换参数的情况下，flex输出才会分割LineStrings，详见flex输出章节中的Geometry转换部分。也请看上面的单一几何体与多重几何体部分。

### 关系的处理

关系有很多变化，它们可以用于各种几何形状。通常，它取决于关系的类型标签，它应该有什么样的几何形状。

* multipolygon， (Multi)Polygon
* boundary，(Multi)LineString或(Multi)Polygon，取决于你对边界本身或它所包围的区域感兴趣。
* route， (Multi)LineString

osm2pgsql flex输出可以从任何关系中创建一个(Multi)Polygon或(Multi)LineString几何体，目前不支持其他几何体。详见柔性输出章节中的几何体转换部分。

如果你使用旧的pgsql输出的 "C转换"，多多边形、边界和路线类型的关系的几何类型是硬编码的。如果你使用的是 "Lua转换"，你可以对它们进行配置。

请注意，osm2pgsql在组装多聚体时将忽略多聚体和边界关系上的角色（内部和外部），因为这些角色经常是错误的或丢失的。

### 处理不完整的OSM数据

有时你会将不完整的OSM数据送入osm2pgsql。最常见的情况是，当你使用地理摘录时，无论是你从某个地方下载的数据还是自己创建的数据，都会发生这种情况。不完整的数据，在这种情况下，意味着一些方式的成员节点或关系的成员被遗漏。通常情况下，在创建提取物时，这一点是无法避免的，你只需要把这个切口放在某个地方。

当osm2pgsql遇到不完整的OSM数据时，它仍然会尽力去使用它。缺少一些节点的路径将被缩短到可用的部分。多边形可能会缺少一些部分。在大多数情况下，这将工作得很好（如果你的提取物已经创建了一个足够大的缓冲区），但有时这将导致错误的结果。在最坏的情况下，如果一个多角形的完整外环缺失，这个多角形就会出现 "倒置"，外环和内环的角色互换。

不幸的是，osm2pgsql（或任何人）都无法改善这一点，这只是OSM数据的性质。

### 投影

Osm2pgsql可以在许多投影中创建几何体。如果你使用的是pgsql输出，投影可以通过命令行选项来选择。当使用flex输出时，投影被指定在Lua风格文件中。默认的是 "Web Mercator"。

* **Latlong (WGS84)** 来自OpenStreetMap的WGS84坐标参考系统的原始经纬度坐标。如果你想对数据进行某种分析，或在以后对其进行再投影，通常会选择这个。
*   **Web Mercator** 这是最常用于平铺网络地图的投影。它是osm2pgsql中的默认投影。

    超过南北纬85°的数据将被切断，因为它不能用这个投影来表示。
*   其他，如果 osm2pgsql 是在支持 PROJ 库的情况下编译的，那么它支持该库所支持的所有投影。

    版本 >= 1.4.0 调用osm2pgsql --版本以查看你的二进制文件是否被编译为PROJ，以及哪个版本。

请注意，影射风格通常取决于所使用的投影。大多数地图风格配置会根据地图比例或缩放级别来启用或禁用某些渲染风格。但有意义的尺度将取决于投影。你遇到的大多数样式可能是为Web Mercator制作的，要想在其他投影中获得漂亮的地图，需要进行修改。

## 运行osm2pgsql命令

### 基本命令行操作

Osm2pgsql可以以两种方式中的一种工作。仅导入或导入且更新。

如果你是osm2pgsql的新手，我们建议你先尝试只导入的方法，并使用一小部分OSM数据的提取。

#### 仅导入

OSM数据被导入数据库一次，之后就不会再改变。如果你想更新数据，你必须删除它，然后做一个完整的重新导入。当不需要更新或只是很少发生时，就会使用这种方法。有时这也是唯一的选择，例如，当你使用的是没有更改文件的摘录时。如果磁盘空间紧张，这也是一种可能的方法，因为你不需要存储只在更新时需要的数据。

在这种模式下，osm2pgsql与-c, --create命令行选项一起使用。这也是默认模式。下面的命令行是等价的。

```bash
osm2pgsql -c OSMFILE
osm2pgsql --create OSMFILE
osm2pgsql OSMFILE
```

如果系统没有太多的主内存，你可以添加-s、-slim和-drop选项。在这种情况下，使用的主内存较少，而更多的数据被存储在数据库中。不过，这使得导入的速度大大降低。有关这些选项的详情，请参见中间一章。

#### 导入和更新

在这种方法中，OSM数据被导入一次，之后或多或少地从OSM变化文件中定期更新。OSM提供每分钟、每小时和每天的变化文件。当需要定期更新并且有变化文件时，可以使用这种模式。

在这种模式下，首次使用osm2pgsql时，需要使用-c, --create命令行选项（这也是默认模式），另外还需要-s, --slim选项。下面的命令行是等价的。

```bash
osm2pgsql -c -s OSMFILE
osm2pgsql --create --slim OSMFILE
osm2pgsql --slim OSMFILE
```

对于更新运行，必须使用-a，-append和-s，-slim选项。下面的命令行是等价的。

```bash
osm2pgsql -a -s OSMFILE
osm2pgsql --append --slim OSMFILE
```

在这种情况下，OSMFILE通常是一个OSM变更文件（后缀为.osc或.osc.gz）。

这种方法比 "仅导入 "方法需要更多的数据库磁盘空间，因为更新所需的所有信息都必须储存在某个地方。

通常，在做导入和更新时，你至少需要使用-C、-cache和-flat-nodes命令行选项。详见中间一章。

### 日志

Osm2pgsql 将把它正在做的事情的信息写到控制台（到 STDERR）。如果你想在一个文件中得到这些信息，请在你的 shell 中使用输出重定向（osm2pgsql ... 2>osm2pgsql.log）。

一些命令行选项允许你改变将被记录的内容和方式。

* \--log-level=LEVEL 设置日志等级 debug、info、warn、error
* \--log-progress=VALUE，启用（真）或禁用（假）进度记录。将此设置为自动将启用控制台的进度记录，如果输出被重定向到一个文件，则禁用它。默认值：true。
* \--log-sql，启用SQL命令的日志以进行调试。
* \--log-sql-data， 启用记录所有添加到数据库的数据。这将写出大量的数据! 用于调试。
* \-v, --verbose，类似`--log-level=debug`.

### 数据库连接

在创建和追加模式下，你必须告诉 osm2pgsql 要访问哪个数据库。如果不设置，它将尝试使用 unix 域套接字连接到默认的数据库（通常为用户名）。大多数用法只需要设置 -d, --数据库。

* \-d, --database=DB， 指定数据库名
* \-U, --user=USERNAME 指定数据库用户名
* \-W, --password 指定数据库密码
* \-H, --host=HOST，数据库地址或者socket位置
* \-P, --port=PORT，数据库端口

你也可以使用libpq环境变量来设置连接参数。关于可用参数的完整列表，请参考PostgreSQL文档。

版本 >= 1.4.0 除了用-d, --数据库选项指定一个数据库名称之外，你还可以用关键字/值连接字符串的形式指定一个连接字符串（类似host=localhost port=5432 dbname=mydb）或者URI（postgresql://\[user\[:password]@]\[netloc]\[:port]\[,...]\[/dbname]\[?param1=value1&...] ）。详见PostgreSQL文档。

### 处理OSM数据

Osm2pgsql处理可以分为多个步骤。

1. 输入法从OSM文件中读取OSM数据。
2. 中间层存储所有对象，并跟踪对象之间的关系。
3. 输出端对数据进行转换并将其加载到数据库中。

当然，这是一个有点简化的观点，但足以理解程序的运作。下面的章节将更详细地描述每一个步骤。

#### 数据输入

根据操作模式的不同，你的输入文件要么是

* OSM数据文件（在创建模式下）
* OSM变化文件（在附加模式下）

通常情况下，osm2pgsql会自动检测文件格式，但请看下面的-r, --input-reader选项。Osm2pgsql不能处理OSM历史文件。

OSM数据文件几乎总是被排序的，首先是节点按其id的顺序，然后是途径按其id的顺序，然后是关系按其id的顺序。行星文件、变化文件和通常的摘录都遵循这个惯例。

Osm2pgsql只能读取以这种方式排序的OSM文件。这允许在代码中进行一些优化，从而加快正常的处理速度。

关于如何获取和准备OSM数据以用于osm2pgsql的更多信息，见附录A。

* \-r, --input-reader=FORMAT，选择输入文件的格式。可用的选择有自动（默认）用于自动检测格式，xml用于OSM XML格式的文件，o5m用于o5m格式的文件，pbf用于OSM PBF二进制格式。
* \-b, --bbox=BBOX，在导入的数据上应用格式为MINLON,MINLAT,MAXLON,MAXLAT的包围盒过滤器。例如： --bbox -0.5,51.25,0.5,51.75

不建议使用-b、-bbox选项。这是一种限制加载到数据库中的数据量的粗略方法，它并不总是能做到你所期望的那样，特别是在边界箱的边界。如果你使用它，请选择一个比你实际感兴趣的区域更大的边界框。一个更好的选择是在使用osm2pgsql之前创建一个数据的提取物。见附录A的选项。

#### 与多个输入文件一起

通常情况下，你在使用osm2pgsql时只有一个输入文件。

版本>=1.4.0 Osm2pgsql可以一次读取多个输入文件，合并输入文件的数据，忽略任何重复的数据。为了使其发挥作用，输入文件的数据必须来自同一时间点。你可以用这个方法将两个或更多的地理摘录导入同一个数据库。如果这些摘录来自不同的时间点，并且包含同一对象的不同版本，这将会失败!

在追加模式下，不要使用多个变更文件作为输入，首先要合并和简化它们。

### 中间层

中间记录了由osm2pgsql读取的所有OSM对象以及这些对象之间的关系。例如，它知道哪些节点使用了哪些方式，或者一个关系有哪些成员。它还记录了所有节点的位置。这些信息对于从方式节点建立方式几何和从成员建立关系几何是必要的，而且在更新数据时也是必要的，因为OSM变化文件只包含变化对象本身，而不是创建对象几何所需的所有相关对象。

更多的细节在它自己的章节中。

### 输出

Osm2pgsql将OSM数据导入一个PostgreSQL数据库。它如何做到这一点是由输出（有时称为后端）所决定的。对于不同的使用情况，有几个输出是可用的。

#### 灵活的输出

这是最现代和最灵活的输出选项。如果你正在开始一个新的项目，请使用这个输出。osm2pgsql未来的许多改进将只在此输出中可用。

与其他所有的输出选项不同，OSM数据导入数据库的方式几乎没有限制。你可以决定哪些OSM对象应该被写入哪些数据库表中的哪些列。而且，你可以对你需要的数据定义任何转换，例如，将复杂的标签模式统一为一个更简单的模式，如果这足以满足你的使用情况。

该输出在其自己的章节中详细描述。

#### pgsql的输出

pgsql输出是原始输出，也是大多数使用过osm2pgsql的人所知道的。你在互联网上找到的许多教程只描述了这种输出。它在如何将数据写入数据库方面有相当大的局限性，但许多设置仍然使用它。

这种输出有两种 "口味"。原始的 "C转换 "和较新的 "Lua转换"，后者允许在导入数据之前对其进行一些修改。

#### 地名词典的输出

地名录输出是仅用于Nominatim的专门输出。本手册中没有关于其使用的信息，详见Nominatim文档。

#### 多输出

版本 >= 1.5.0 多重输出被删除。

#### 空输出

空输出并不把数据写在任何地方。它用于测试和基准测试，不用于正常操作。如果--slim与null输出一起使用，仍然会生成中间表。

下面是与输出有关的命令行选项（关于只用于特定输出的命令行选项，见后面的章节）。

* \-O, --output=OUTPUT， 选择输出。可用的输出是：Flex、pgsql（默认）、**gazetteer**和null。
* \-S, --style=STYLE，样式文件。这指定了数据如何被导入数据库，其格式取决于输出。(对于pgsql输出，默认是/usr/share/osm2pgsql/default.style，对于其他输出，没有默认。)

### 灵活输出

1.3.0至1.4.2版 灵活输出首次出现在1.3.0版的osm2pgsql中。在1.4.2版之前，它被标记为实验性的。

灵活输出，正如其名，允许一个灵活的配置，告诉osm2pgsql要在你的数据库中存储什么OSM数据，以及确切的位置和方式。它是通过一个Lua文件配置的。

* 定义了输出表的结构
* 定义了将OSM数据映射到数据库数据格式的函数

使用-S，-style=FILE选项来指定Lua文件的名称，同时使用-O flex或--output=flex选项来指定使用flex输出。

与pgsql输出不同，flex输出不使用命令行选项进行配置，而只使用Lua配置文件。

flex风格文件是一个Lua脚本。你可以使用Lua语言的所有功能。这段描述假设你对Lua语言有一定的了解，但要掌握基础知识是非常容易的，你可以使用flex-config目录下的例子配置文件，其中包含大量的注释让你入门。

所有的配置都是通过Lua中的osm2pgsql全局对象完成的。它有以下字段和功能。

* version， osm2pgsql的版本是一个字符串。
* config\_dir，版本 >=1.5.1 你的 Lua 配置文件所在的目录。当你想从Lua中包含更多的文件时很有用。
* mode，根据命令行选项（-c, --create 或 -a, --append），可以是 "创建 "或 "追加"。
* stage，1或2（数据的第一/第二阶段处理）。见下文。
* define\_node\_table(NAME, COLUMNS\[, OPTIONS])，定义一个节点表。
* define\_way\_table(NAME, COLUMNS\[, OPTIONS])，定义一个途径表。
* define\_relation\_table(NAME, COLUMNS\[, OPTIONS])，定义一个关系表。
* define\_area\_table(NAME, COLUMNS\[, OPTIONS])，定义一个区域表。
* define\_table(OPTIONS)，定义一个表。这是所有其他define\_\*\_table()函数背后更灵活的函数。它比其他更方便的函数给你更多的控制。

Osm2pgsql还在附录B中描述的Lua辅助库中提供了一些额外的功能。

#### 定义一个表

你必须定义一个或多个数据库表，让你的数据最终放那里。这是通过osm2pgsql.define\_table()函数完成的，或者，更常见的是，通过一个稍微方便的函数osm2pgsql.define\_(node|way|relation|area)\_table()完成的。在创建模式下，osm2pgsql将为你在数据库中创建这些表。

#### 基本表的定义

定义一个表格的简单方法是这样的：

```lua
osm2pgsql.define_(node|way|relation|area)_table(NAME, COLUMNS[, OPTIONS])
```

这里NAME是表的名称, COLUMNS是一个描述列的Lua表的列表，如下文所述, OPTIONS是一个Lua表，包含整个表的选项, 举个例子可能更有帮助:

```lua
local restaurants = osm2pgsql.define_node_table('restaurants', {
    { column = 'name', type = 'text' },
    { column = 'tags', type = 'jsonb' },
    { column = 'geom', type = 'point' }
})
```

在这种情况下，我们正在创建一个表，打算用OSM节点来填充。将在数据库中创建一个名为 "restaurants "的表，其中有四列:一个文本类型的name列（估计以后会填上餐厅的名称）,一个名为tags的列，类型为jsonb（以后可能会被填入一个节点的所有标签）, 一个名为geom的列，它将包含节点的Point geometry还有一个名为node\_id的Id列。

node表、way表、relation表、area表，分别表示来自node、way、relation或者area的数据。Osm2pgsql确保OSM对象的ID将被存储在表中，这样以后对这些OSM对象的更新（或删除）将正确反映在表中。区域表很特别，它们可以包含从方式和（多多边形）关系中得到的数据。

#### 高级表的定义

有时，define\_(node|way|relation|area)\_table()函数有点过于限制性，例如，如果你想对Id列的类型和命名有更多的控制。在这种情况下，你可以使用函数osm2pgsql.define\_table()。

下面是osm2pgsql.define\_table(OPTIONS)函数的可用OPTIONS。你可以在define\_(node|way|relation|area)\_table()函数上使用同样的选项，除了name和columns选项。

* name，表名，不包含schema名称
* ids，一个Lua表，定义这个表应该如何处理ID（详见Id处理部分）。注意，你可以定义没有Ids的表，但它们不能被osm2pgsql更新。
* schema，设置用于此表的PostgreSQL schema。默认为 public
* columns，一个列的数组（详见定义列的部分）。
* data\_tablespace，用于此表中的数据的PostgreSQL表空间。
* index\_tablespace，用于此表的所有索引的PostgreSQL表空间。
* cluster，版本 >= 1.5.0 设置聚类策略。使用 "auto"（默认）来启用几何聚类，osm2pgsql将选择最佳方法。使用 "no "来禁用聚类。

所有的osm2pgsql.define\*table()函数都返回一个数据库表Lua对象。你可以对它调用以下函数：

* name()， 定义函数中指定的表的名称。
* schema()，定义函数中指定的表的schema。
* columns()，定义函数中指定的表的列。
* add\_row()，向数据库表添加一行。详见下文。

#### ID处理

Id列允许osm2pgsql跟踪哪些数据库表项是由哪些对象生成的。当对象改变或被删除时，osm2pgsql使用这些ID来更新或删除数据库中所有具有这些ID的现有行。

如果你使用的是osm2pgsql.define\_(node|way|relation|area)\_table()便利函数，osm2pgsql将自动创建一个id列，分别名为(node|way|relation|area)\_id。

如果你想对id列有更多的控制，可以使用osm2pgsql.define\_table()函数。然后你可以使用 id 选项来定义 id 列应该是什么样子的。为了生成上面描述的相同的 "餐馆 "表，define\_table()调用将看起来像这样。

```lua
local restaurants = osm2pgsql.define_table({
    name = 'restaurants',
    ids = { type = 'node', id_column = 'node_id' },
    columns = {
        { column = 'name', type = 'text' },
        { column = 'tags', type = 'jsonb' },
        { column = 'geom', type = 'point' }
}})
```

如果你没有id字段，生成的表就没有id。没有Id的表不能被更新，所以你可以在创建模式下填充它们，但在追加模式下它们将无法工作。

ids字段的类型允许有以下值。

* node，一个节点的Id将被 "原封不动 "地保存在id\_column命名的列中。
* way，一种way的Id将被 "按原样 "存储在id\_column命名的列中。
* relation，一个relation Id将被 "原封不动 "地存储在id\_column命名的列中。
* area，一种way的Id将被 "按原样 "存储在id\_column命名的列中，对于relation，Id将被存储为负数。
* any，任何类型的对象都可以被存储。见下文。

any类型是针对可以存储任何类型OSM对象（节点、方式或关系）的表。有两种方式可以存储id：

* 如果你在ids字段中设置了type\_column，它将在一个额外的char(1)列中存储对象的类型，如N、W或R。
* 如果你没有type\_column，node的id保持不变（所以它们是正数），way id将被存储为负数，relation id被存储为负数并从中减去1e17。
* 这导致了所有ID的不同范围，所以它们可以被分开。(这与Imposm程序使用的转换相同）。

#### ID 唯一性

在数据库表上通常希望有一个唯一的PRIMARY KEY。许多程序都需要这个。

似乎有一个自然的唯一键，即数据来自的OSM node、way或relation ID。但这里面有一个问题。根据配置的不同，osm2pgsql有时会在输出表中添加几条具有相同OSM ID的记录。这通常发生在长线字符串被分割成较短的片段或多聚体被分割成其组成的多聚体时，但如果你的Lua配置在同一个表中添加了两行，也可能发生这种情况。

如果你需要在你的数据库表上有唯一键，有两个选择。使用这些自然键和确保你没有重复的条目。或者增加一个额外的ID列。后者更容易做到，而且在所有情况下都能工作，但会增加一些开销。

#### 使用自然键获得唯一的ID

要使用OSM ID作为主键，你必须确保：

1. 你只为每个OSM对象在输出表中添加一条记录，也就是说，不要为同一个OSM对象在同一个表中多次调用add\_row。
2. osm2pgsql不会将长线字符串分割成小线字符串，也不会将多面体分割成多边形。所以你不能在几何变换中使用split\_at选项。
3. 版本 == 1.3.0 osm2pgsql不会将multipolygons分割成polygons。所以你必须在所有区域的几何变换中设置multi = true。

你可能还想在ID列上建立一个索引。如果你是在瘦身模式下运行，osm2pgsql将为你创建该索引。但是在非瘦身模式下，你必须自己用CREATE UNIQUE INDEX来做这件事。你也可以使用ALTER TABLE来使该列成为 "官方 "主键列。

#### 使用一个额外的ID列

PostgreSQL有一个有点神奇的 "串行 "数据类型。如果你在一个列的定义中使用这种数据类型，PostgreSQL会在表中添加一个整数列，并自动用一个自动递增的值来填充它。

在flex配置中，你可以用这样的方式在你的表中添加这样一列：

```lua
...
{ column = 'id', sql_type = 'serial', create_only = true },
...
```

create\_only告诉osm2pgsql，它应该创建这个列，但在添加记录时不尝试填充它（因为PostgreSQL为我们做了这个）。

你可能还想在这一列上建立一个索引。在使用osm2pgsql第一次导入数据后，使用CREATE UNIQUE INDEX来创建一个索引。你也可以使用ALTER TABLE来使该列成为 "正式 "的主键列。

从PostgreSQL 10开始，你可以使用GENERATED ... AS IDENTITY子句来代替SERIAL类型，后者的作用非常类似，尽管在这里使用任何不符合PostgreSQL要求的类型都不被官方支持。

### 定义列

在表的定义中，列被指定为具有以下键的Lua表的列表。

| 键            | 说明                                                                                        |
| ------------ | ----------------------------------------------------------------------------------------- |
| column       | 列名，必填                                                                                     |
| type         | 数据类型，默认text                                                                               |
| sql\_type    | 版本 >= 1.5.0 列的SQL类型（可选，默认取决于类型，见下表）。在1.5.0之前的版本中，类型字段也被用于此。                               |
| not\_null    | 设置为 "true"，使其成为一个NOT NULL列。(可选，默认为false。)                                                 |
| create\_only | 设置为 "true "可以将该列添加到CREATE TABLE命令中，但在添加数据时不要尝试填充该列。这对SERIAL列或你想以后自己填入该列时很有用。(可选，默认为false） |
| projection   | 仅在几何列上。设置为EPSG的id或投影的名称。(可选的，默认为web mercator，3857）。                                       |

type字段从osm2pgsql的角度描述了该列的类型，sql\_type从PostgreSQL数据库的角度描述了该列的类型。通常情况下，它们要么是相同的，要么是根据下表的类型直接派生出来的SQL类型。但在特殊情况下，也可以对它们进行单独设置。

版本<1.5.0 在早期版本中，sql\_type字段不存在。使用类型字段来代替。如果你从<1.5.0的版本升级到1.5.0或以上的版本，通常只需将sql\_type设置为与类型字段之前的值相同，对于每个类型字段，你从osm2pgsql得到错误信息就足够了。

| type               | 默认sql类型                           | 说明                     |
| ------------------ | --------------------------------- | ---------------------- |
| text               | text                              | 默认                     |
| bool, boolean      | boolean                           |                        |
| int2, smallint     | int2                              |                        |
| int4, int, integer | int4                              |                        |
| int8, bigint       | int8                              |                        |
| real               | real                              |                        |
| hstore             | hstore                            | 必须加载PostgreSQL扩展hstore |
| json               | json                              |                        |
| jsonb              | jsonb                             |                        |
| direction          | int2                              |                        |
| geometry           | geometry(GEOMETRY, _SRID_)        | (\*)                   |
| point              | geometry(POINT, _SRID_)           | (\*)                   |
| linestring         | geometry(LINESTRING, _SRID_)      | (\*)                   |
| polygon            | geometry(POLYGON, _SRID_)         | (\*)                   |
| multipoint         | geometry(MULTIPOINT, _SRID_)      | (\*)                   |
| multilinestring    | geometry(MULTILINESTRING, _SRID_) | (\*)                   |
| multipolygon       | geometry(MULTIPOLYGON, _SRID_)    | (\*)                   |

(\*)几何图形SQL类型的SRID来自于投影参数。

对于特殊情况，你通常希望保持type字段不设置，只设置sql\_type字段。So if you want to have an array of integer column, for instance, set only the sql\_type to int\[].类型字段将默认为文本，这意味着你必须以PostgreSQL期望的形式创建数组的文本表示。

关于数据如何根据类型进行转换的细节，见类型转换一节。

sql\_type 字段的内容不被 osm2pgsql 检查，它被原封不动地传递给 PostgreSQL。除了一个有效的PostgreSQL类型，不要把它设置为其他任何东西。

#### 定义几何列

大多数表都会有一个几何列。(目前只支持0或1个几何列。）可能的几何列的类型取决于输入数据的类型。对于节点表，你几乎被限制在点的几何形状上，但对于关系表，则有多种选择。

支持的几何体类型有：

* point，点几何，通常由nodes创建。
* linestring，线条几何图形，通常由way创建。
* polygon，区域表的多边形几何形状，由way或relation创建。
* multipoint，目前没有使用。
* multilinestring，从（可能是分裂的）way或relation中产生。
* multipolygon，对于区域表，由way或relation创建。
* geometry，任何种类的几何体。也用于应该同时容纳多边形和多多边形几何体的面积表。

默认情况下，几何列将以web mercator（EPSG 3857）创建。要改变这一点，请将列的projection参数设置为你想要的EPSG代码（或latlon(g)、WGS84或merc(ator)中的一个字符串，大小写被忽略）。

有一种特殊的几何列类型叫做area。它可以在polygon或multipolygon列之外使用。

与普通的几何列类型不同，产生的数据库类型将不是几何类型，而是浮点数。它将被自动填充成几何体的面积。面积将以web mercator计算，或者你可以将列的投影参数设置为4326，以WGS84坐标来计算它。目前不支持其他投影方式。

#### 定义其他列

除了id和geometry列之外，每个表可以有任意数量的 "正常 "列，使用PostgreSQL支持的任何类型。有些类型是由osm2pgsql特别识别的：`text`, `boolean`, `int2` (`smallint`), `int4` (`int`, `integer`), `int8` (`bigint`), `real`, `direction`, `hstore`, `json` 、 `jsonb`.关于这个类型如何影响OSM数据与数据库类型的转换，请参见类型转换部分。

你可以使用任何你想要的SQL类型来代替上述类型。如果你这样做，你必须在向这类列添加数据时提供PostgreSQL的字符串表示（或者用Lua nil来设置该列为NULL）。

#### 处理回调

你能定义以下一个或多个功能：

* osm2pgsql.process\_node(object)，为每个新的或变化的节点调用。
* osm2pgsql.process\_way(object)，为每一种新的或改变了的way调用。
* osm2pgsql.process\_relation(object)，为每个新的或改变的relation调用。

它们都有一个表类型的单一参数（这里称为对象），没有返回值。如果你对所有的对象类型不感兴趣，你不必提供所有的函数。

这些函数对输入文件中的每个新的或修改的OSM对象都被调用。对于被删除的对象没有调用任何函数，osm2pgsql将自动删除你的数据库表中所有来自被删除对象的数据。修改被处理为删除，然后创建一个 "新 "对象，为其调用函数。

参数表（对象）有以下字段和功能：

* id， node, way, or relation的  ID.
* tags，该对象的所有标签
* version，OSM对象的版本(\*)
* timestamp，时间戳(\*)
* changeset，包含OSM对象的这个版本的变化集。(\*)
* uid，创建或最后改变这个OSM对象的用户的ID。(\*)
* user，创建或最后更改此OSM对象的用户的用户名。(\*)
* grab\_tag(KEY)，返回指定键的标签值，并从标签列表中删除该标签。local name = object:grab\_tag('name')，当你想把一些标签存储在特殊的列中，而把其余的标签存储在jsonb或hstore列中时，通常会用到这个方法。
* get\_bbox()，获取当前节点或方式的包围盒。这个函数返回四个结果值：界线盒左下角的lot/lat值，然后是右上角的lon/lat值。如果是节点，两个lon/lat值都是相同的。例子：lon, lat, dummy, dummy = object.get\_bbox() (这个函数目前对关系不起作用。)
* is\_closed，判断线路是否是闭合的，只支持线路
* nodes，获取线条的节点 ID，只支持线路
* members，只支持关系，返回一个数组，没个成员都有type类型、ref成员 ID、role角色属性

(\*)这些只有在使用了-x|--extra-attributes 选项并且OSM输入文件实际包含这些字段时才可用。

你可以在这些处理函数中做任何事情，以决定如何处理这些数据。如果你对该OSM对象不感兴趣，只需从该函数中返回。如果你想将OSM对象添加到某个表中，请在该表中调用add\_row()函数。

```lua
-- definition of the table:
table_pois = osm2pgsql.define_node_table('pois', {
    { column = 'tags', type = 'jsonb' },
    { column = 'name', type = 'text' },
    { column = 'geom', type = 'point' },
})
...
function osm2pgsql.process_node(object)
...
    table_pois:add_row({
        tags = object.tags,
        name = object.tags.name,
        geom = { create = 'point' }
    })
...
end
```

add\_row()函数需要一个单一的表参数，它描述了要填入所有数据库列的内容。任何没有提到的列将被设置为NULL。

几何列有些特殊。你必须定义一个几何变换，用来将OSM对象数据转换成适合几何列的几何体。详见下一节。

注意，你不能设置对象的ID，这将在幕后为你处理。

### 几何变换

目前支持这些几何形状的变换:

* { create = 'point' },只对节点有效，创建一个 "点 "的几何体。
* { create = 'line' }, 对于方式或关系。创建一个 "线型 "或 "多线型 "的几何体。
* { create = 'area' }, 对于方式或关系。创建一个 "多边形 "或 "多多边形 "的几何体。

其中一些转换可以有参数:

* 线条转换有一个可选的参数split\_at。如果这个参数被设置为0以外的任何值，长的线段将被分割成不长于这个值的部分。
* 版本 == 1.3.0 区域变换有一个可选的参数 multi。如果设置为false（默认），一个多多边形几何体将被分割成几个多边形。如果设置为 "true"，多多边形几何体将被保留为一个。
* 版本 >= 1.4.0 面积变换有一个可选的参数 split\_at。如果没有设置或设置为nil（默认），（多）多边形几何体将保持原样，如果设置为 "multi"，多多边形几何体将被分割成其多边形部分。

请注意，一般来说，你有责任确保几何列的列类型可以接受转换所创建的几何图形。一个点的几何列将不能存储一个区域变换的结果。

版本 >= 1.4.0 如果一个转换将导致一个多边形几何体，但该几何体的列类型是多多边形。多边形将被自动 "包裹 "在一个多边形中，以适应该列。当你希望你的所有多边形和多多边形都是相同的数据库类型时，这可能很有用。有些程序能更好地处理统一几何类型的列。

如果没有设置几何变换，在某些情况下，osm2pgsql会假定一个默认的变换。这些是默认的：

* 对于节点表，一个点列可以得到节点的位置。
* 对于way表，linestring列获得完整的way几何，polygon列获得作为面积的way几何（如果way是封闭的并且面积是有效的）。

### 阶段

在处理OSM数据时，osm2pgsql按顺序读取输入文件，先是nodes，然后是ways，再是relations。这意味着，当方式被读取和处理时，osm2pgsql还不能知道一个way是否在一个relation中（或在几个relation中）。但对于某些用例，我们需要知道一个ways在哪些relations中，这些relations的标签是什么，或者这些成员ways的作用。典型的情况是路线类型的关系（公共汽车路线等），我们可能想把路线关系中的名称或参考文献渲染到路径几何上。

osm2pgsql的柔性输出通过增加一个额外的 "再处理 "步骤来支持这一用例。Osm2pgsql将为每个添加、修改或删除的relation调用Lua函数osm2pgsql.select\_relation\_members（）。你的工作是找出该relation中的哪些成员可能需要关系中的信息来正确呈现，并在一个Lua表中返回这些ID，其中只有一个字段 "ways"。这通常是通过这样的一个函数来完成的：

```lua
function osm2pgsql.select_relation_members(relation)
    if relation.tags.type == 'route' then
        return { ways = osm2pgsql.way_member_ids(relation) }
    end
end
```

与其使用辅助函数osm2pgsql.way\_member\_ids()来返回所有方式成员的id，你可以编写自己的代码，例如，如果你想检查角色。

请注意，select\_relation\_members()对于被删除的关系和被修改的关系的旧版本，以及对于新的关系和被修改的关系的新版本都被调用。这是需要的，例如，正确标记已删除关系的成员方式，因为它们也需要被更新。决定一个方式是否被标记只能基于关系的标签和/或成员的角色。如果你考虑到其他信息，更新可能无法正常工作。

此外，你必须将你在process\_relation()函数中需要的关于关系的任何信息存储在一个全局变量中。

在所有关系被处理后，osm2pgsql将通过再次调用process\_way()函数来重新处理所有标记的方式。这一次，你在全局变量中拥有来自关系的信息，并可以使用它。

如果你不标记任何方式，在这个再处理阶段就不会做任何事情。

(目前不可能对节点或关系进行标记。这可能会或可能不会在未来的osm2pgsql版本中添加）。

你可以看一下osm2pgsql.stage，看看你处于哪个阶段。

你想在第1阶段做所有你能做的处理，因为它更快，而且内存开销更少。对于大多数使用情况，第1阶段已经足够了。

分两个阶段处理会增加相当多的开销。因为这个功能是新的，没有多少操作经验。因此，当你在试验时要有点小心，注意内存和磁盘空间的消耗以及你所使用的任何额外时间。请记住:

* 在Lua脚本中，所有存储在第1阶段供第2阶段使用的数据将使用主内存。
* 追踪第一阶段标记的方式ID需要一些内存。
* 为了在第二阶段进行额外的处理，需要时间将对象从对象存储中取出并重新处理。
* Osm2pgsql将在所有方式的表上创建一个id索引，以查找需要在第2阶段删除和重新创建的方式。

