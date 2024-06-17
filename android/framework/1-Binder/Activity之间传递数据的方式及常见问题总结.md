https://blog.csdn.net/rflyee/article/details/47431633

Activity之间传递数据一般通过以下几种方式实现：
1. 通过intent传递数据
2. 通过Application
3. 使用单例
4. 静态成员变量。（可以考虑   WeakReferences ）
5. 持久化（sqlite、share preference、file等）

# 一、通过intent传递数据
（1）直接传递， intent . putExtra(key, value)
（2）通过bundle， intent . putExtras ( bundle );
PS：
（1）这两种都要求传递的对象必须可序列化（Parcelable、Serializable）
（2）Parcelable实现相对复杂
（3）关于Parcelable和Serializable，官方说法：
         Serializable : it's error prone and horribly slow. So in general:  stay away from Serializable  if possible.
     也就是说和Parcelable相比，Seriaizable容易出错并且速度相当慢。是否这样，可参见下一篇博客说明。
（4）通过intent传递数据是有大小限制滴，超过限制，要么抛异常，要么新的Activity启动失败，所以还是很严重的啊，可参见下一篇博客说明。

# 二、Application
   这个应该也都接触过，将数据保存早全局Application中，随整个应用的存在而存在，这样很多地方都能访问。具体使用就不多说了。
但是需要 注意的是：
  当由于某些原因（比如系统内存不足），我们的app会被系统强制杀死，此时再次点击进入应用时，系统会直接进入被杀死前的那个界面，制造一种从来没有被杀死的假象。那么问题来了，系统强制停止了应用，进程死了，那么再次启动时Application自然新的，那里边的数据自然木有啦，如果直接使用很可能报空指针或者其他错误。
  因此还是要考虑好这种情况的：
  （1）使用时一定要做好非空判断
  （2）如果数据为空，可以考虑逻辑上让应用直接返回到最初的activity，比如用   FLAG_ACTIVITY_CLEAR_TASK  或者  BroadcastReceiver 杀掉其他的activity。

# 三、使用单例
比如一种常见的写法：
public class DataHolder {
  private String data;
  public String getData() {return data;}
  public void setData(String data) {this.data = data;}
  private static final DataHolder holder = new DataHolder();
  public static DataHolder getInstance() {return holder;}
}
这样在启动activity之前：
DataHolder.getInstance().setData(data);
新的activity中获取数据：
String data = DataHolder.getInstance().getData();

# 四、静态Static
这个可以直接在activity中也可以单独一个数据结构体，就和单例差不多了。
比如：
public class DataHolder {
  private static String data;
  public static String getData() {return data;}
  public static String setData(String data) {this.data = data;}
}
启动之前设置数据，新的activity获取数据。

注意：这些情况如果数据很大很多，比如bitmap，处理不当是很容易导致内存泄露或者内存溢出的。
所以可以考虑使用 WeakReferences 将数据包装起来。
比如：
public class DataHolder {
  Map<String, WeakReference<Object>> data = new HashMap<String, WeakReference<Object>>();
  void save(String id, Object object) {
    data.put(id, new WeakReference<Object>(object));
  }
 
  Object retrieve(String id) {
    WeakReference<Object> objectWeakReference = data.get(id);
    return objectWeakReference.get();
  }
}
启动之前：
DataHolder.getInstance().save(someId, someObject);
新activity中：
DataHolder.getInstance().retrieve(someId);
这里可能需要通过intent传递id，如果数据唯一，id都可以不传递的。save() retrieve()中id都固定即可。

# 五、持久化数据
那就是sqlite、share preference、file等了。
优点：
（1）应用中所有地方都可以访问
（2）即使应用被强杀也不是问题了
缺点：
（1）操作麻烦
（2）效率低下
（3）io读写嘛，其实还是比较容易出错的

