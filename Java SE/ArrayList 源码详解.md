## 域


```
    private static final long serialVersionUID = 8683452581122892189L;

    //默认初始容量。
    private static final int DEFAULT_CAPACITY = 10;

    //用于空实例的共享空数组实例。
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     *用于默认大小空实例的共享空数组实例。我们
     *将其与emptyelementdata区分开来，以了解在什么时候膨胀
     *第一个元素被添加。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 数组缓冲区中存储ArrayList元素的数组缓冲区。
     * ArrayList的容量是这个数组缓冲区的长度。任何
     * 带有elementData==default电容性元素数据的空ArrayList
     * 当第一个元素被添加时，将扩展到default容量。
     */
    transient Object[] elementData; // 非私有来简化嵌套类访问

    // 数组大小
    private int size;
    
    /**
     * 要分配的数组的最大大小。
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     * Integer.MAX_VALUE = 2147483647
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    
```
[transient](http://www.cnblogs.com/lanxuezaipiao/p/3369962.html)防止被序列化

数组最大大小为Integer 对象的最大大小-8的缘故是，数组对象需要8个byte 的大小来存储自己的元数据：
> 数组对象的形状和结构（如int值数组）与标准Java对象类似。主要区别在于数组对象有一个额外的元数据，用于表示数组的大小。然后，数组对象的元数据由以下部分组成：

> Class：指向描述对象类型的类信息的指针。在int数组的情况下，这是一个指向int []类的指针。

> 标志：描述对象状态的标志集合，包括该对象的散列码（如果有）以及对象的形状（即对象是否为数组）。

> 锁定：对象的同步信息 - 即对象是否当前同步。

> 大小：数组的大小。
## 构造函数
###### （1） 带有初始容量的构造函授

```
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```
- 初始容量 > 0 时，new 一个指定容量大小的Object 数组
- 初始容量 == 0 时，把数组指向数组大小为0 的实例

```
private static final Object[] EMPTY_ELEMENTDATA = {};
```

- 其他，抛出非法异常IllegalArgumentException

###### （2） 默认构造函数

```
    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
默认构造函数将数组指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA

```
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```
###### （3） 传入一个集合的构造函数

```
public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
1. elementData 指向了传入数组c 的toArray()
2. 如果数组大小为0，则赋值给私有的空数组
3. 不为0，则强制保证数组为Object 数组，因为经过toArray 的数组不一定为Object（见toArray 函数）

```
public Object[] toArray() {
        // Estimate size of array; be prepared to see more or fewer elements
        Object[] r = new Object[size()];
        Iterator<E> it = iterator();
        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext()) // fewer elements than expected
                return Arrays.copyOf(r, i);
            r[i] = it.next();
        }
        return it.hasNext() ? finishToArray(r, it) : r;
    }
```
toArray 并不是并发安全的函数，但是对并发修改进行了预防处理
- 先新建一个数组对象，大小为本身的大小
- 用一个循环迭代赋值，如果我们改一下代码就很清楚

```
for (int i = 0; i < r.length; i++) {
            r[i] = it.next();
        }
```
这里不好理解的是为什么需要这个判断

```
if (! it.hasNext()) // fewer elements than expected
    return Arrays.copyOf(r, i);
```
根据注释（少于预期的元素）可见这是预防并发修改导致数组变少的处理，如果数组在赋值新数组的时候并发减少，则直接返回一个capy,看一下capy 函数的代码：

```
// original 新建的数组
// newLength 复制到的数组的位置
public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }
    
    
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        //如果传入的数组是Object 数组，则新建Object 数组，否则新增对应Type 数组
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        //复制数据
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

- toArray（）函数如果没有并发减少，在返回的时候还需判断并发增加的情况

```
return it.hasNext() ? finishToArray(r, it) : r;

private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
        int i = r.length;
        while (it.hasNext()) {
            int cap = r.length;
            if (i == cap) {
                int newCap = cap + (cap >> 1) + 1;
                // overflow-conscious code
                if (newCap - MAX_ARRAY_SIZE > 0)
                    newCap = hugeCapacity(cap + 1);
                r = Arrays.copyOf(r, newCap);
            }
            r[i++] = (T)it.next();
        }
        // trim if overallocated
        return (i == r.length) ? r : Arrays.copyOf(r, i);
    }   
    
private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError
                ("Required array size too large");
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
finishToArray 函数的逻辑很简单，主要是先判断需不需要数组进行扩容，扩容过程中对最大扩容数就行判断，防止错误情况发生。（这里我不太明白为什么是Integer.MAX_VALUE ，按照前面的说法，MAX_ARRAY_SIZE 就已经是最大的了。超过就已经溢出，这里感觉应该是

 (minCapacity > MAX_ARRAY_SIZE) ?
            MAX_ARRAY_SIZE :
            minCapacity;）才比较合理）

toArray 函数保证了elementData 数组在当时是最新的数据。

Arrays.copyOf 函数通俗点的作用就是把原本的数组变成指定大小的数组

# 添加函数
ArrayList 的添加函数主要涉及到边界判定和扩容，所以我们先了解了解对应的函数。
###### 扩容函数——ensureCapacityInternal(int minCapacity)

```
private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
    
 private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
- 先判断数组是不是默认构造函数（即未声明容量）的数组，如果是去传入容量值和默认容量值（10）的大者，传入ensureExplicitCapacity(int minCapacity)
- ensureExplicitCapacity(int minCapacity) 函数的主要功能是判断数组的大小有没有超过数组的长度，超过了就扩容
- 数组进行扩容的时候先增大数组长度的一半，在比较和指定容量和最大数组值（前面遗留问题，为什么hugeCapacity会返回最大Integer 值）后确定newCapacity 的最终值，最后复制数组。

###### 边界检查——rangeCheckForAdd(int index)

```
private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```
很简单自己看吧

###### add函数集
在知道扩容和检查函数后再看函数就很简单了


```
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
    
public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```

无论是添加元素还是添加元素集合套路都一样
- 获取添加过后的元素大小
- 判断需不需要扩容
- 最后复制数组

# 删除函数

删除主要了解三个函数就可以了
1. public E remove(int index)
2. private void fastRemove(int index)
3. private boolean batchRemove(Collection<?> c, boolean complement)

其中 1 ,2 都是删除指定位置的元素，不过2 没有验证边界

```
private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            //复制数组
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // 便于垃圾清理

        return oldValue;
    }
    
private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
remove 函数很简单，主要是batchRemove 函数
```
private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```
batchRemove 函数传入了一个集合和布尔值，这里的布尔值代表了你最后的数组是保留指定集合还是删除：
- false ：删除
- true ： 保留

循环代码的主要作用就是筛选出指定的元素
```
 for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
```
举个例子
> 从数组[1,2,3,4,5]中删除[1,3]（即complement = false）

经过这个循环后会变成(你可以自己去写一写)

> [2,4,5,4,5]

finally 中的代码是善后处理
```
finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
```
根据注释我们知道在for 循环中可能会抛出异常导致循环提前退出

我们先看正常的情况，继续上面的例子循环过后数组变成[2,4,5,4,5]，但我们的正确结果应该是[2,4,5]，在finally 的关于w 的判断体下就是做的这个事
> 将w 值的index 之后的数组元素变为null 

异常的情况下，假设数组在判断2 时抛出，数组就会变成[2,2,3,4,5]
> r = 1, w=0;

关于r 的判断体代码会把r 之后元素复制到w 之后
> 数组变为[2,3,4,5,5]

> w = 4

重复正常情况-->[2,3,4,5,null]

简单点讲，这个函数就是会删除/保留传入的集合，但是在抛出异常时，它只会返回判断到一半的情况（例子中1删除了，但是3没删除，是因为在3之前抛出）

然后删除函数就自己参悟了哈

```
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

//保留函数
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
```

# 迭代函数


```
public Iterator<E> iterator() {
        return new Itr();
    }
```
在ArrayList 中实现的iterator() 方法返回了一个Itr 类。

Itr 类是ArrayList 的一个内部私有类。

```
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;
        。。。
}
```
Itr 类继承了Iterator 接口，他自定义了3个变量

- cursor : 返回的下一个元素的索引
- lastRet : 返回的最后元素的索引;如果没有默认-1
- expectedModCount ： 预期modCount

Iterator 接口定义了一下方法：

```

boolean hasNext()               如果迭代具有更多元素，则返回 true 。  
E next()                        返回迭代中的下一个元素。  
default void remove()           从底层集合中删除此迭代器返回的最后一个元素（可选操作）。  

```
我们来看看在Itr 中的实现：
###### hasNext()

```
public boolean hasNext() {
            return cursor != size;
        }
```
hasNext 函授的实现很简单，判断cursor 是否等于 size

###### next()

```
public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
        
final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
```
**checkForComodification 函数的作用是判断是否存在并发修改**。如果实际的modCount 不等于预期，就抛出并发修改异常。
modCount 在add，remove 等操作ArrayList 集合中会的数据modCount++。

- 在next 方法中用一个局部变量存储cursor ，如果i 超过size 抛出NoSuchElementException 。
- 用一个数组获取 ArrayList.this.elementData 的引用，避免多次指向
- 再次判断，防止删除的并发修改
- cursor lastRet 变量赋值

###### remove()

面试题中有一道经典的基础问题：怎样在集合删除元素

如果回答，for 循环遍历，匹配，remove 方法删除，那么恭喜你，game over

直接在循环里面删除会抛出并发修改/下标越界的异常。
```
public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```
remove 函数必须配合着next 函授使用，因为lastRet 只有在next 函数中才有赋值。通过迭代器删除元素可以避免并发修改问题

###### 迭代器增强——ListItr

```
private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }

        public boolean hasPrevious() {
            return cursor != 0;
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor - 1;
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            checkForComodification();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }

        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.set(lastRet, e);
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        public void add(E e) {
            checkForComodification();

            try {
                int i = cursor;
                ArrayList.this.add(i, e);
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }

```

# 修改容量

```
public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            //size=0指向EMPTY_ELEMENTDATA，否则缩减到size 大小
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```
