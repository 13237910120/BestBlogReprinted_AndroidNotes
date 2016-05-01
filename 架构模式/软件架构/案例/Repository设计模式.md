# Repository设计模式

来源:[Repository设计模式](http://blog.chengdazhi.com/index.php/153#rd?sukey=fa67fe3435f5c4be025aa0c46de5482aac4b01176fe3a21a7581e432d5e3c9d5fc283e1ccc59d0747d8d1d73f02e570d)

原文链接：[https://medium.com/@krzychukosobudzki/repository-design-pattern-bc490b256006#.r0my8xrj6](https://medium.com/@krzychukosobudzki/repository-design-pattern-bc490b256006#.r0my8xrj6)

在Android中我们需要将数据存储起来以持久化，大部分情况下我们使用SQLite存储数据。但当你想通过一种简单而只读的方式存储，那该怎么做呢？没错，你要找一个好的ORM。你可以开始写制表的类并设计所有需要的方法。

```
public interface NewsesDao {
    void add(News news);

    void update(News news);

    void remove(News news);

    News getById();

    List<News> getNewest();

    List<News> getAll();

    List<News> getByCategory(Category category);

    List<News> getNewerThan(long date);
}
```

看起来熟悉吗？这个有错吗？当然没有，但很有提高的空间。

与其写一个（或两个）超级类做所有的事情，还不如遵循单一职责原则并应用[Repository模式](http://martinfowler.com/eaaCatalog/repository.html)。

Repository是domain层与数据层的中介，如同一个内存中的domain模型集合。使用端会以声明的方式构造查询条件并将其发送给Repository以获取数据。Repository可以增删对象，就如一个对象集合，而其内部封装的数据映射代码会在后台进行逻辑操作。

我们根据Repository的基础定义开始写代码:

```
public interface Repository<T> {
    void add(T item);

    void update(T item);

    void remove(T item);

    List<T> query(Specification specification);
}
```

在这里我用Specification替代了Criteria，二者区别除了名字以外，还有Specification不关注数据会存在哪里。在后续使用过程中，我发现有两个方法特别有用，极建议添加。

```
public interface Repository<T> {
    void add(T item);

    void add(Iterable<T> items);

    void update(T item);

    void remove(T item);

    void remove(Specification specification);

    List<T> query(Specification specification);
}
```

当然还是要根据具体需求来写。在几乎所有的应用中你都需要同时存储于删除多个条目。Repository是不是简化了你的工作？

回到Specification，这只是一个普通的名字接口，一旦定义了Repository是什么，就可以开始实现Specification了：

```
public interface SqlSpecification extends Specification {
    String toSqlQuery();
}
```

基于数据库的基本Repository实现如下：

```
public class NewsSqlRepository implements Repository<News> {
    private final SQLiteOpenHelper openHelper;

    private final Mapper<News, ContentValues> toContentValuesMapper;
    private final Mapper<Cursor, News> toNewsMapper;

    public NewsSqlRepository(SQLiteOpenHelper openHelper) {
        this.openHelper = openHelper;

        this.toContentValuesMapper = new NewsToContentValuesMapper();
        this.toNewsMapper = new CursorToNewsMapper();
    }

    @Override
    public void add(News item) {
        add(Collections.singletonList(item));
    }

    @Override
    public void add(Iterable<News> items) {
        final SQLiteDatabase database = openHelper.getWritableDatabase();
        database.beginTransaction();

        try {
            for (News item : items) {
                final ContentValues contentValues = toContentValuesMapper.map(item);

                database.insert(NewsTable.TABLE_NAME, null, contentValues);
            }

            database.setTransactionSuccessful();
        } finally {
            database.endTransaction();
            database.close();
        }
    }

    @Override
    public void update(News item) {
        // TODO to be implemented
    }

    @Override
    public void remove(News item) {
        // TODO to be implemented
    }

    @Override
    public void remove(Specification specification) {
        // TODO to be implemented
    }

    @Override
    public List<News> query(Specification specification) {
        final SqlSpecification sqlSpecification = (SqlSpecification) specification;

        final SQLiteDatabase database = openHelper.getReadableDatabase();
        final List<News> newses = new ArrayList<>();

        try {
            final Cursor cursor = database.rawQuery(sqlSpecification.toSqlQuery(), new String[]{});

            for (int i = 0, size = cursor.getCount(); i < size; i++) {
                cursor.moveToPosition(i);

                newses.add(toNewsMapper.map(cursor));
            }

            cursor.close();

            return newses;
        } finally {
            database.close();
        }
    }
}
```

当使用Lambda和Kotlin时代码会显得简短一些。在这个例子中我故意只实现了add和query方法因为其他的方法非常类似。相信你已经注意到了Mapper接口，其职责是将一个对象映射到另一个。在这个例子中Mapper起到了很大的作用。在使用Repository模式时，Mapper类让你可以只关注重要的事情。

```
public interface Mapper<From, To> {
    To map(From from);
}
```

为了进行数据库或其他存储的查询，我们需要实现Specification：

NewestNewsesSpecification.class

```
public class NewestNewsesSpecification implements SqlSpecification {

    @Override
    public String toSqlQuery() {
        return String.format(
                "SELECT * FROM %1$s ORDER BY `%2$s` DESC;", 
                NewsTable.TABLE_NAME, 
                NewsTable.Fields.DATE
        );
    }
}
```

NewsByIdSpecification.class

```
public class NewsByIdSpecification implements SqlSpecification {
    private final int id;

    public NewsByIdSpecification(final int id) {
        this.id = id;
    }

    @Override
    public String toSqlQuery() {
        return String.format(
                "SELECT * FROM %1$s WHERE `%2$s` = %3$d';",
                NewsTable.TABLE_NAME,
                NewsTable.Fields.ID,
                id
        );
    }
}
```

NewsesByCategorySpecification.class

```
public class NewsesByCategorySpecification implements SqlSpecification {
    private final Category category;

    public NewsesByCategorySpecification(final Category category) {
        this.category = category;
    }

    @Override
    public String toSqlQuery() {
        return String.format(
                "SELECT * FROM %1$s WHERE `%2$s` = '%3$d'",
                NewsTable.TABLE_NAME,
                NewsTable.Fields.CATEGORY_ID,
                category.getId()
        );
    }
}
```

Specification很简单，一般只有一个方法。你想写多少Specification都行，而不会影响DAO类。而且这样测试就变得更加简单，通过接口，Specification可以被随意模拟、替换，甚至可以用假数据测试。你能想象与又臭又长的DAO相比Specification类有多么简单易懂吗？

Repository可以与MVP很好地融合，也能在第三方开源框架中使用。如果你用Dagger2你需要指定在某处要注入哪种Repository实现。

```
public class LatestNewsesPresenter implements Presenter<LatestNewsesView> {
    private final LatestNewsesView view;
    private final Repository<News> repository;

    public LatestNewsesPresenter(LatestNewsesView view, Repository<News> repository) {
        this.view = view;
        this.repository = repository;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        final List<News> newses = repository.query(new NewestNewsesSpecification());

        view.setNewses(newses);
    }

    @Override
    public void onDestroy() {

    }
}
```

下面开始复杂一些

现在需求变了，你不能用SQLite了，现在要用Realm。

//那边的程序员我听到了你们的声音！！！

如果你已经用DAO实现了应用，难道需要重写一堆一堆的类，包括那些非数据存储层的吗？因为你在使用Repository模式，这都是不必的，你只需要实现Repository和Specification来适应新需求即可（下面就以Realm为例）。你不需要修改其他的类，现在让我们看一下新的实现：

```
public class NewsRealmRepository implements Repository<News> {
    private final RealmConfiguration realmConfiguration;

    private final Mapper<NewsRealm, News> toNewsMapper;

    public NewsRealmRepository(final RealmConfiguration realmConfiguration) {
        this.realmConfiguration = realmConfiguration;

        this.toNewsMapper = new NewsRealmToNewsMapper();
    }

    @Override
    public void add(final News item) {
        final Realm realm = Realm.getInstance(realmConfiguration);

        realm.executeTransaction(new Realm.Transaction() {
            @Override
            public void execute(Realm realm) {
                final NewsRealm newsRealm = realm.createObject(NewsRealm.class);
                newsRealm.setCategory(item.getCategory());
                newsRealm.setDate(item.getDate());
                newsRealm.setTitle(item.getTitle());
                newsRealm.setText(item.getText());
            }
        });

        realm.close();
    }

    @Override
    public void add(Iterable<News> items) {
        // TODO to be implemented
    }

    @Override
    public void update(News item) {
        // TODO to be implemented
    }

    @Override
    public void remove(News item) {
        // TODO to be implemented
    }

    @Override
    public void remove(Specification specification) {
        // TODO to be implemented
    }

    @Override
    public List<News> query(Specification specification) {
        final RealmSpecification realmSpecification = (RealmSpecification) specification;

        final Realm realm = Realm.getInstance(realmConfiguration);
        final RealmResults<NewsRealm> realmResults = realmSpecification.toRealmResults(realm);

        final List<News> newses = new ArrayList<>();

        for (NewsRealm news : realmResults) {
            newses.add(toNewsMapper.map(news));
        }

        realm.close();

        return newses;
    }
}
```

Specification接口也要重写一下：

```
public interface RealmSpecification extends Specification {
    RealmResults<NewsRealm> toRealmResults(Realm realm);
}
```

以及一个RealmSpecification的实现：

```
public class NewsByIdSpecification implements RealmSpecification {
    private final int id;

    public NewsByIdSpecification(final int id) {
        this.id = id;
    }

    @Override
    public RealmResults<NewsRealm> toRealmResults(Realm realm) {
        return realm.where(NewsRealm.class)
                .equalTo(NewsRealm.Fields.ID, id)
                .findAll();
    }
}
```

现在你就可以拿Realm替代SQLite了。（还记得Presenter中依赖是如何被定义的吗？）

**奖**

这里还有一个用例，你可以用Repository模式提供缓存。就像前面的例子一样，写一个CacheSpecification。

```
public interface CacheSpecification<T> extends Specification {
    boolean accept(T item);
}
```

与其他接口一起使用：

```
public class NewsByIdSpecification implements RealmSpecification, CacheSpecification<News> {
    private final int id;

    public NewsByIdSpecification(final int id) {
        this.id = id;
    }

    @Override
    public RealmResults<NewsRealm> toRealmResults(Realm realm) {
        return realm.where(NewsRealm.class)
                .equalTo(NewsRealm.Fields.ID, id)
                .findAll();
    }

    @Override
    public boolean accept(News item) {
        return id == item.getId();
    }
}
```

最重要的是，不是所有的Specification都需要实现所有的接口，如果在缓存中只用到NewsByIdSpecification，那就不需要让NewsByCatetorySpecification也实现CacheSpecification。

现在你的代码更加整洁了，你的队友更加爱你了，产品经理不用死了，就算你跳槽了，接替你的人也不会砍你了。

欢迎关注我的公众号，将零碎时间都用在刷干货上！

![](http://7xpg2f.com1.z0.glb.clouddn.com/qrcode_for_gh_f9319618373d_430-2.jpg)