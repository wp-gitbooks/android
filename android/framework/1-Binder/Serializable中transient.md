用transient声明一个实例变量，当对象存储时，它的值不需要维持。

# 作用

Java的serialization提供了一种持久化对象实例的机制。当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。当一个对象被序列化的时候，transient型变量的值不包括在序列化的表示中，然而非transient型的变量是被包括进去的。

# 验证

```
package com.qjk;
import java.io.*;
public class TestTransient {

    static class User implements Serializable {
        String name;
        int age;
        transient int transient_age;
        String sex;
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        User user = new User();
        user.age = 12;
        user.transient_age = 12;
        user.name = "小丸子";
        user.sex = "nv";

        System.out.println("age:" + user.age);
        System.out.println("transient_age:" + user.transient_age);
        System.out.println("name:" + user.name);
        System.out.println("sex:" + user.sex);

        System.out.println("----------------序列化+反序列化----------------");

        // 序列化后写入缓存
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(user);
        oos.flush();
        oos.close();

        byte[] buf = baos.toByteArray();
        // 从缓存中读取并执行反序列化
        ByteArrayInputStream bais = new ByteArrayInputStream(buf);
        ObjectInputStream ois = new ObjectInputStream(bais);

        User u = (User) ois.readObject();

        System.out.println("age:" + u.age);
        System.out.println("transient_age:" + u.transient_age);
        System.out.println("name:" + u.name);
        System.out.println("sex:" + u.sex);

    }
}
```

结果输出

```
age:12
transient_age:12
name:小丸子
sex:nv
----------------序列化+反序列化----------------
age:12
transient_age:0
name:小丸子
sex:nv
```

# 扩展

## Exteranlizable

Java序列化提供两种方式。一种是实现Serializable接口。另一种是实现Exteranlizable接口。Externalizable接口是继承于Serializable ，需要重写writeExternal和readExternal方法，它的效率比Serializable高一些，并且可以决定哪些属性需要序列化（即使是transient修饰的），但是对大量对象，或者重复对象，则效率低。

## ArrayList中的transient

下面是ArrayList中的一段代码

```
    /**
     * ArrayList 的元素存储在其中的数组缓冲区。ArrayList 的容量就是这个数组缓冲
     * 区的长度。添加第一个元素时，任何带有 elementData == 
     * DEFAULTCAPACITY_EMPTY_ELEMENTDATA 的空 ArrayList 都将扩展为 
     * DEFAULT_CAPACITY。
     */
    transient Object[] elementData; // non-private to simplify nested class access
```

ArrayList提供了自定义的wirteObject和readObject方法：

```
    /**
     * Save the state of the <tt>ArrayList</tt> instance to a stream (that
     * is, serialize it).
     *
     * @serialData The length of the array backing the <tt>ArrayList</tt>
     *             instance is emitted (int), followed by all of its elements
     *             (each an <tt>Object</tt>) in the proper order.
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            int capacity = calculateCapacity(elementData, size);
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

我理解这里定义elementData为transient是为了防止数据重复被写入磁盘，节省空间。
如果对象声明了readObject和writeObject方法，对象在被序列化的时候会执行对象的readObject和writeObject方法。

## 静态属性

静态属性，无论是否被transient修饰，都不会被序列化。