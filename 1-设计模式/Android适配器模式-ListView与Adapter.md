# 适配器ListView与Adapter

## 例子
### 出水口出水量
出水量有大有小，于是先定义两个接口
```java
public interface BigOutlet {
    public void bigOutlet();
}

public interface SmallOutlet {
    public void smallOutlet();
}
```

### 出水口water tap
```java
public class BigWaterTap implements BigOutlet {
    private static final String TAG = WaterTap.class.getSimpleName();

    @Override
    public void bigOutlet() {
        Log.d(TAG,"bigOutlet");
    }
}
```

### 定义适配器
#### 类适配器模式
```java
public class ClassWaterTapAdapter extends BigWaterTap implements SmallOutlet {
    private static final String TAG = ClassWaterTapAdapter.class.getSimpleName();

    @Override
    public void smallOutlet() {
        Log.d(TAG,"smallOutlet");
    }
}
```

调用
```java
ClassWaterTapAdapter classWaterTapAdapter = new ClassWaterTapAdapter();
classWaterTapAdapter.bigOutlet();
classWaterTapAdapter.smallOutlet();
```

#### 对象适配器模式
```java
public class ProxyWaterTapAdapter implements SmallOutlet {
    private static final String TAG = ProxyWaterTapAdapter.class.getSimpleName();
    private BigWaterTap bigWaterTap;

    public ProxyWaterTapAdapter(BigWaterTap bigWaterTap) {
        this.bigWaterTap = bigWaterTap;
    }

    public void adapterBigOutlet() {
        bigWaterTap.bigOutlet();
    }

    @Override
    public void smallOutlet() {
        Log.d(TAG,"smallOutlet");
    }
}
```

调用
```java
        ProxyWaterTapAdapter proxyWaterTapAdapter = new ProxyWaterTapAdapter(new BigWaterTap());
        proxyWaterTapAdapter.adapterBigOutlet();
        proxyWaterTapAdapter.smallOutlet();

```

## ListView与适配器模式
### ListWaterTapAdapter等价于ListAdapter
```java
public abstract class ListWaterTapAdapter implements SmallOutlet{
    public abstract void middleOutlet();
}
```

ListWaterTapAdapter抽象类，需要我们客户使用的时候具体去实现。然后重新写一下BigWaterTap,因为它本身有的功能就是bigOutlet()。等价于listview
```java
public class ListViewWaterTap implements BigOutlet{
    private static final String TAG = ListViewWaterTap.class.getSimpleName();
    private ListWaterTapAdapter listWaterTapAdapter;

    @Override
    public void bigOutlet() {
        Log.d(TAG,"bigOutlet");
    }

    public void setAdapter(ListWaterTapAdapter listWaterTapAdapter) {
        this.listWaterTapAdapter = listWaterTapAdapter;
    }

    public void smallToBigOutlet() {
        listWaterTapAdapter.smallOutlet();
        int i = 100;
        while (i-- > 0);
        listWaterTapAdapter.middleOutlet();
    }
}
```

这里的ListViewWaterTap并非适配器。我们的适配器就是ListWaterTapAdapter。setAdapter来拥有适配器抽象类，调用抽象方法，让客户自己去实现smallOutlet和middleOutlet这两个方法。接着我们看下调用

```java
        ListViewWaterTap listViewWaterTap = new ListViewWaterTap();
        ListWaterTapAdapter listWaterTapAdapter = new ListWaterTapAdapter() {
            @Override
            public void middleOutlet() {
                Log.d(TAG,"middleOutlet");
            }

            @Override
            public void smallOutlet() {
                Log.d(TAG,"smallOutlet");
            }
        };
        listViewWaterTap.setAdapter(listWaterTapAdapter);

        listViewWaterTap.bigOutlet();
        listViewWaterTap.smallToBigOutlet();
```

这里就是和listview与adapter一样的用法了。并不是标准的适配器模式写法。但确实最经典的适配器模式应用。下面我们来分析一下listview和adapter的关系。

### 普通示例
```java
    listView = (ListView) findViewById(R.id.listView);
    MyAdapter myAdapter = new MyAdapter(this,mListTile);

public class MyAdapter extends BaseAdapter {
    private LayoutInflater layoutInflater;
    List<String> litTile;

    public MyAdapter(Context context, List<String> listTile) {
        layoutInflater = LayoutInflater.from(context);
        this.litTile = listTile;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder holder;
        if (convertView == null) {
            holder = new ViewHolder();
            convertView = layoutInflater.inflate(R.layout.layout_item,null);
            holder.title = (TextView) convertView.findViewById(R.id.title);
            convertView.setTag(holder);
        } else {
            holder = (ViewHolder)convertView.getTag();
        }
        holder.title.setText(litTile.get(position));
        return convertView;
    }

    class ViewHolder{
        public TextView title;
    }
// 篇幅原因，省略getCount、getItem和getItemId
```

这是我们最普通的运用listview的写法了。然后我们从适配器讲起，先看adapter是个啥  
listview.java的setAdapter方法
```java
@Override
    public void setAdapter(ListAdapter adapter) {
    ...
}

```

看看BaseAdapter的集成关系，然后开始讲适配器BaseAdapter
```java
public abstract class BaseAdapter implements ListAdapter, SpinnerAdapter { ... }

public interface ListAdapter extends Adapter { ... }

public interface Adapter { ... }

```

### 适配器
```java
public interface Adapter {
    void registerDataSetObserver(DataSetObserver observer);
    void unregisterDataSetObserver(DataSetObserver observer);
    int getCount();   
    Object getItem(int position);
    long getItemId(int position);
    boolean hasStableIds();
    View getView(int position, View convertView, ViewGroup parent);
    static final int IGNORE_ITEM_VIEW_TYPE = AdapterView.ITEM_VIEW_TYPE_IGNORE;
    int getItemViewType(int position);
    int getViewTypeCount();
    static final int NO_SELECTION = Integer.MIN_VALUE;
    boolean isEmpty();
    default @Nullable CharSequence[] getAutofillOptions() {
        return null;
    }
}

```
1>根据上面的继承关系，BaseAdapter跟开始举例一样，抽象类来定义适配器，要实现的接口是Adapter  
2> 可以看到Adapter就是接口，接口的这些方法是要提供给ListView内部使用的。我们自己实现Adapter，完成这些接口，或者抽象方法。

  
  
### listview如何使用adapter
首先看下listView的继承关系

```java
public class ListView extends AbsListView { ... }

public abstract class AbsListView extends AdapterView<ListAdapter> implements TextWatcher,
        ViewTreeObserver.OnGlobalLayoutListener, Filter.FilterListener,
        ViewTreeObserver.OnTouchModeChangeListener,
        RemoteViewsAdapter.RemoteAdapterConnectionCallback { ... }

public abstract class AdapterView<T extends Adapter> extends ViewGroup { ... }
```

ListView就是一个ViewGroup,然后通过主要的逻辑实现代码就在ListView.java和AbsListView.java了

**先讲一个getCount**  
在AbsListView.java的onAttachedToWindow方法

```java
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        ...
        if (mAdapter != null && mDataSetObserver == null) {
            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);

            // Data may have changed while we were detached. Refresh.
            mDataChanged = true;
            mOldItemCount = mItemCount;
            mItemCount = mAdapter.getCount();
        }
    }
```

把view关联到window的时候，mAdapter.getCount()，这个getConut是我们继承时写的，就确定了我们这个listView有多少个item了。

**再梳理一下getView是如何把item view加载出来的**  
大致流程如下，这里就只列出简化的代码了，本文主要理解适配器模式的思想  
1>AbsListView:onlayout

```java
 protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);
        ...
        layoutChildren();
        ....
    }
```

ViewGroup是[组合模式](https://www.jianshu.com/p/b512a047604d)，它在调用onlayout的时候调用layoutChildren来布局子控件,layoutChildren在AbsListView是一个空实现，实现代码在ListView  
2>ListView:layoutChildren

```cpp
protected void layoutChildren() {
    ...
   switch (mLayoutMode) {
            ...
            case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
```

调用到layoutChildren后来布局item view.  
3>ListView:fillDown

```java
private View fillDown(int pos, int nextTop) {
  ...
   while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }
```

每个子view都是调用makeAndAddView然后调用AbsListView的obtainView方法

```java
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
    ...
     final View child = obtainView(position, mIsScrap);
```

4>AbsListView:obtainView

```java
 View obtainView(int position, boolean[] outMetadata) {
     ...
       final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else if (child.isTemporarilyDetached()) {
                outMetadata[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }
    ...
}
```

这里用到了mAdapter.getView。从ListView布局每个item view的过程来看，最后布局使用view的时候就用到了我们去实现的getView方法返回的view。

**我们对listview优化的时候，为啥写法是判断if (convertView == null)**  
接着分析上面第四步的代码  
mRecycler.getScrapView是获得可复用的view，然后带入mAdapter.getView(position, scrapView, this);  
如果还没有被加入到缓存list则

```cpp
if (child != scrapView) {
    // Failed to re-bind the data, return scrap to the heap.
    mRecycler.addScrapView(scrapView, position);
} 
```

所以我们写代码的时候则这样来优化判断，当然获得缓存后数据也是原来的。所以我们要重新设置title

```csharp
     if (convertView == null) {
            holder = new ViewHolder();
            convertView = layoutInflater.inflate(R.layout.layout_item,null);
            holder.title = (TextView) convertView.findViewById(R.id.title);
            convertView.setTag(holder);
        } else {
            holder = (ViewHolder)convertView.getTag();
        }
        holder.title.setText(litTile.get(position));
```

这里ListView和ListAdapter的关系就梳理到这里了。有兴趣的同学还可以去看看GridView,RecyleView喔。比如RecycleView就是ListView的一个升级版，RecycleView定义了ViewHolder的机制。更加巧妙。


## 参考
https://www.jianshu.com/p/fb558642823e