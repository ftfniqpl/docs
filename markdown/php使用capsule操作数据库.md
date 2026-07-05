# 使用illuminate database Capsule查询构造器进行数据库操作


Illuminate database是一个非常强大非常优秀的ORM类库，也是一个非常实用的数据库操作组件。使用它可以轻松对数据库进行查询、插入、更新、删除等操作，支持MySQL，Postgres，SQL Server，SQLlite等。它还是Laravel框架的数据库组件。

本文单独将illuminate database拿出来，脱离框架，主要讲讲使用illuminate database查询构造器进行数据库操作。

#### 安装

使用 composer 安装，直接在项目根目录的命令行里，执行
```
composer require illuminate/database
```
建议PHP版本使用PHP^7.2。

#### 建立配置

创建一个Capsule管理实例来配置数据库连接参数。
```
<?php 
$capsule = new \Illuminate\Database\Capsule\Manager;
// 创建链接
$capsule->addConnection([
    'driver' => 'mysql',
    'host' => 'localhost',
    'database' => 'demo',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8mb4',
    'port' => 3306,
    'collation' => 'utf8mb4_general_ci',
    'prefix' => 'web_',
]);
// 设置全局静态可访问DB
$capsule->setAsGlobal();
// 启动Eloquent （如果只使用查询构造器，这个可以注释）
$capsule->bootEloquent();
```

将配置文件保存为database.php。再新建文件index.php，代码如下：
```
<?php 
date_default_timezone_set("PRC");

require 'vendor/autoload.php';
require 'database.php';

use Illuminate\Database\Capsule\Manager as DB;
```

自行准备好数据库表，现在，可以直接在index.php后面写数据库操作语句了。

#### 获取结果集
###### 从一张表中获取一行/一列

如果我们只是想要从数据表中获取一行数据，可以使用**first** 方法，该方法将会返回单个StdClass对象：
```
$article = DB::table('article')->where('author', '月光光')->first();
echo $article->title;
```

如果我们不需要完整的一行，可以使用value 方法从结果中获取单个值，该方法会直接返回指定列的值：
```
$title = DB::table('article')->where('author', '月光光')->value('title');
```

###### 获取数据列值列表
如果只要查询表中的某一列值，可使用pluck方法。
```
$titles = DB::table('article')->where('author', '月光光')->pluck('title');
foreach ($titles as $title) {
     echo $title;
}
```

如果要获取一个表中的其中几个字段列的结果，可以使用select和get方法。
```
$list = DB::table('article')->select('id', 'title')->get();
$list = DB::table('article')->get(['id', 'title']);
```
两条语句返回的结果是一样的。要获取结果，需要遍历$list:
```
foreach ($list as $key => $val) {
    echo $val->id;
    echo $val->title;
}
```
###### 强制不重复
distinct方法允许你强制查询返回不重复的结果集
```
$list = DB::table('article')->distinct()->get();
```
#### Where查询
###### 简单 Where 子句
使用查询构建器上的where方法可以添加where子句到查询中，调用where最基本的方式需要传递三个参数，第一个参数是列名，第二个参数是任意一个数据库系统支持的操作符，第三个参数是该列要比较的值。

如，查询id值为100的记录。
```
$row = DB::table('article')->where('id', '=', '100')->get();
```
当要查询的列值和给定数组相等时，可以将等号省略。上面的语句可以这样写：
```
DB::table('article')->where('id', '100')->get();
```
除了等号，还有>=，<=，<>，like
```
DB::table('article')->where('title', 'like', 'php%')->get();
```
###### Where数组
还可以传递条件数组到where函数：

```
$list = DB::table('article')->where([
     ['id', '>', '100'],
     ['source', '=', 'helloweba.com']
 ])->get();
```
###### or 语句

我们可以通过方法链将多个where约束链接到一起，也可以添加 or 子句到查询，orWhere方法和 where 方法接收参数一样：

```
$list = DB::table('article')
         ->where('source', 'helloweba.com')
         ->orWhere('hits', '<', '1000')
         ->get(['id', 'title', 'hits']);
```

###### whereIn语句
whereIn方法验证给定列的值是否在给定数组中。whereNotIn方法验证给定列的值不在给定数组中。
```
$list = DB::table('article')->whereIn('id', [10,100,200])->get(['id', 'title']);
```
###### whereBeteen语句
whereBetween方法验证列值是否在给定值之间，whereNotBetween方法验证列值不在给定值之间。
```
$list = DB::table('article')
         ->whereBetween('hits', [1, 1000])->get(['id', 'title', 'hits']);
```
###### whereNull语句
whereNull方法验证给定列的值为NULL，whereNotNull方法验证给定列的值不是 NULL。
```
$list = DB::table('article')
        ->whereNull('updated_at')
        ->get();
```
###### whereDate语句
如果我们要查询创建日期在2019-08-29的文章记录，可以使用whereDate。
```
$list = DB::table('article')->whereDate('created_at', '2019-08-29')->get(['id', 'title', 'created_at']);
```
###### whereMonth语句
如果我们要查询创建月份在10月份的所有文章记录，可以使用whereMonth。
```
$list = DB::table('article')->whereMonth('created_at', '10')->get(['id', 'title', 'created_at']);
```

###### whereDay语句
如果要查询创建日期在1号的所有文章，可以使用whereDay。
```
$list = DB::table('article')->whereDay('created_at', '1')->get(['id', 'title', 'created_at']);
```
###### whereYear语句
如果要查询创建年份是2016年的所有文章，可以使用whereYear。
```
$list = DB::table('article')->whereYear('created_at', '2016')->get(['id', 'title', 'created_at']);
```
###### whereTime语句
如果要查询创建时间在10:20的所有文章，可以使用whereTime。
```
$list = DB::table('article')->whereTime('created_at', '10:20')->get(['id', 'title', 'created_at']);
```
###### whereColumn语句
如果要查询文章表中创建时间和更新时间相等的所有文章，可以使用whereColumn。
```
$list = DB::table('article')->whereColumn('created_at', '=', 'updated_at')->get(['id', 'title', 'created_at']);
```
#### 聚合查询
查询构建器还提供了多个聚合方法，如总记录数: count, 最大值: max, 最小值:min,平均数：avg 和总和: sum，我们可以在构造查询之后调用这些方法。
```
$count = DB::table('article')->count(); //总记录数
$max = DB::table('article')->max('hits'); //点击量最大
```
###### 判断记录是否存在
除了通过 count 方法来判断匹配查询条件的结果是否存在外，还可以使用exists 或doesntExist 方法：

```
$exists = DB::table('article')->where('author', '月光光')->exists();
```
返回的是true和false。

#### 排序、分组与限定
###### orderBy
我们要对查询的记录进行顺序asc和倒序desc排序，可以使用orderBy。
```
$list = DB::table('article')->orderBy('id', 'desc')->get(['id', 'title']);
```
###### latest / oldest
我们可以使用latest和oldest对日期字段created_at。
```
$list = DB::table('article')->latest()->first();
```
###### inRandomOrder
如果我们要从文章表中随机排序，查询一条随机记录，可以使用inRandomOrder。
```
$list = DB::table('article')->inRandomOrder()->first();
```
###### groupBy / having
如果要对结果集进行分组，可以使用groupBy方法和having方法。
```
DB::table('article')->groupBy('cate')-having('id', '>', 100)->get();
```
###### skip / take
如果要对结果集进行跳过给定的数目结果，可以使用skip和take方法，该方法常用于数据分页。
```
$list = DB::table('article')->skip(10)->take(5)->get(['id', 'title']);
```
以上语句等价于：
```
$list = DB::table('article')->offset(10)->limit(5)->get(['id', 'title']);
```

#### 连接Join
查询构建器还可以用于编写join连接语句，常见的几种连接类型有：join、leftJoin、rightJoin等。
```
$list = DB::table('mark')
            ->join('user', 'mark.user_id', '=', 'user.id')
            ->join('article', 'mark.article_id', '=', 'article.id')
            ->get(['article.id','article.title','user.username','user.nickname']);
```
#### 插入数据
查询构建器还提供了insert方法用于插入记录到数据表。insert方法接收数组形式的字段名和字段值进行插入操作
```
DB::table('article')->insert(
    ['title' => 'PDO操作数据库', 'author' => '月光光']
);
```
###### 支持批量插入：
```
DB::table('article')->insert(
    ['title' => 'PDO操作数据库', 'author' => '月光光'],
    ['title' => 'PDO那些事', 'author' => '想吃鱼的猫'],
);
```
#### 自增ID
使用insertGetId方法来插入记录并返回ID值，如果表中的id为自增长ID，则返回的即为自增ID。
```
$id = DB::table('article')->insertGetId(
    ['title' => 'PDO操作数据库', 'author' => '月光光'],
);
```
#### 更新数据
使用update方法可以更新对应的字段。
```
DB::table('article')->where('id', 1)->update(['author' => '月光光']);
```
#### 增减数字
我们可以使用increment和decrement方法增减某个列值，比如增加点击量。
```
DB::table('article')->increment('hits'); //点击量+1
DB::table('article')->increment('hits', 5); //点击量+5
DB::table('article')->decrement('hits'); //点击量-1
DB::table('article')->decrement('hits', 5); //点击量-5
```
在操作过程中你还可以指定额外的列进行更新：
```
DB::table('article')->increment('hits', 1, ['name' => 'John']); //点击量+1,并且name
更新为John
```
#### 删除数据
使用delete方法可以从表中删除记录。
```
DB::table('article')->where('id', 10)->delete();
```
如果我们要清空一张表，将自增长id归0，可以使用truncate方法。
```
DB::table('article')->truncate();
```
#### 打印sql日志
有时我们需要调试sql语句，查看最后一次执行的原生的sql语句，可以使用以下方法：
```
DB::connection()->enableQueryLog();
$list = DB::table('article')->skip(10)->take(5)->get(['id', 'title']);
print_r(DB::getQueryLog());
```


#### 事务
想要在一个数据库事务中运行一连串操作，可以使用 DB 门面的transaction 方法，使用transaction方法时不需要手动回滚或提交：如果事务闭包中抛出异常，事务将会自动回滚；如果闭包执行成功，事务将会自动提交：
```
DB::transaction(function () {
    DB::table('users')->update(['votes' => 1]);
    DB::table('posts')->delete();
});
```
当然我们也可以使用手动控制事务，从而对回滚和提交有更好的控制，可以使用 DB 门面的 beginTransaction方法：
```
DB::beginTransaction();
```
可以通过rollBack方法回滚事务：
```
DB::rollBack();
```
最后，我们可以通过commit方法提交事务：
```
DB::commit();
```
小结
Illuminate database提供的查询构造器可以轻松实现对数据库的操作，能满足我们日常开发需求，当然，在框架中使用的时候更多的使用ORM进行数据操作，后面我们会有文章介绍Illuminate database的ORM功能，彻底摆脱sql语句的束缚。

