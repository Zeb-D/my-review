[TOC]

## AbstractStringBuilder

### 数据结构

`char[] value;`

`int count;`

### 主要方法

```
/**
 * Returns the character (Unicode code point) before the specified
 * index. The index refers to {@code char} values
 * (Unicode code units) and ranges from {@code 1} to {@link
 * #length()}.
 *
 * <p> If the {@code char} value at {@code (index - 1)}
 * is in the low-surrogate range, {@code (index - 2)} is not
 * negative, and the {@code char} value at {@code (index -
 * 2)} is in the high-surrogate range, then the
 * supplementary code point value of the surrogate pair is
 * returned. If the {@code char} value at {@code index -
 * 1} is an unpaired low-surrogate or a high-surrogate, the
 * surrogate value is returned.
 *
 * @param     index the index following the code point that should be returned
 * @return    the Unicode code point value before the given index.
 * @exception IndexOutOfBoundsException if the {@code index}
 *            argument is less than 1 or greater than the length
 *            of this sequence.
 */
public int codePointBefore(int index) {
    int i = index - 1;
    if ((i < 0) || (i >= count)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointBeforeImpl(value, index, 0);
}
```

```
private AbstractStringBuilder appendNull() {
    int c = count;
    ensureCapacityInternal(c + 4);
    final char[] value = this.value;
    value[c++] = 'n';
    value[c++] = 'u';
    value[c++] = 'l';
    value[c++] = 'l';
    count = c;
    return this;
}`
```

```
public void getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)
{
    if (srcBegin < 0)
        throw new StringIndexOutOfBoundsException(srcBegin);
    if ((srcEnd < 0) || (srcEnd > count))
        throw new StringIndexOutOfBoundsException(srcEnd);
    if (srcBegin > srcEnd)
        throw new StringIndexOutOfBoundsException("srcBegin > srcEnd");
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```

```
private void reverseAllValidSurrogatePairs() {
    for (int i = 0; i < count - 1; i++) {
        char c2 = value[i];
        if (Character.isLowSurrogate(c2)) {
            char c1 = value[i + 1];
            if (Character.isHighSurrogate(c1)) {
                value[i++] = c1;
                value[i] = c2;
            }
        }
    }
}`
```

```
public AbstractStringBuilder reverse() {
    boolean hasSurrogates = false;
    int n = count - 1;
    for (int j = (n-1) >> 1; j >= 0; j--) {
        int k = n - j;
        char cj = value[j];
        char ck = value[k];
        value[j] = ck;
        value[k] = cj;
        if (Character.isSurrogate(cj) ||
            Character.isSurrogate(ck)) {
            hasSurrogates = true;
        }
    }
    if (hasSurrogates) {
        reverseAllValidSurrogatePairs();
    }
    return this;
}
```

### 实现类

`abstract class AbstractStringBuilder implements Appendable, CharSequence`



## StringBuffer

### 数据结构

```
/**
 * A cache of the last value returned by toString. Cleared
 * whenever the StringBuffer is modified.
 */
private transient char[] toStringCache;
```

```
/** use serialVersionUID from JDK 1.0.2 for interoperability */
static final long serialVersionUID = 3388685877147921107L;
```



### 主要方法

默认value的长度为  `seq.length() + 16`

```
public StringBuffer(CharSequence seq) {
    this(seq.length() + 16);
    append(seq);
}
```



好多方法都重写了父类`AbstractStringBuilder`  并且加上了synchronized 关键字

在进行对value 操作时，会 先清除，toString会 `toStringCache=value`

```
toStringCache = null;
super.delete(start, end);
```

```
public synchronized String toString() {
    if (toStringCache == null) {
        toStringCache = Arrays.copyOfRange(value, 0, count);
    }
    return new String(toStringCache, true);
}
```

```
/**
 * readObject is called to restore the state of the StringBuffer from
 * a stream.
 */
private synchronized void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    java.io.ObjectOutputStream.PutField fields = s.putFields();
    fields.put("value", value);
    fields.put("count", count);
    fields.put("shared", false);
    s.writeFields();
}
```



```
/**
 * readObject is called to restore the state of the StringBuffer from
 * a stream.
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    java.io.ObjectInputStream.GetField fields = s.readFields();
    value = (char[])fields.get("value", null);
    count = fields.get("count", 0);
}
```



### 实现类

```
public final class StringBuffer
   extends AbstractStringBuilder
   implements java.io.Serializable, CharSequence
```





## StringBuilder

### 数据结构

```java
/** use serialVersionUID for interoperability */
static final long serialVersionUID = 4383685877147921099L;
```

### 主要方法

构造器默认16，实际参数、str.length() + 16,与StringBuffer 构造器一样

```java
public StringBuilder(int capacity) {
    super(capacity);
}
```

但是toString()、序列化方式不一样

```java
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

```java
/**
 * Save the state of the {@code StringBuilder} instance to a stream
 * (that is, serialize it).
 *
 * @serialData the number of characters currently stored in the string
 *             builder ({@code int}), followed by the characters in the
 *             string builder ({@code char[]}).   The length of the
 *             {@code char} array may be greater than the number of
 *             characters currently stored in the string builder, in which
 *             case extra characters are ignored.
 */
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    s.defaultWriteObject();
    s.writeInt(count);
    s.writeObject(value);
}

/**
 * readObject is called to restore the state of the StringBuffer from
 * a stream.
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    count = s.readInt();
    value = (char[]) s.readObject();
}
```

### 实现类

```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```