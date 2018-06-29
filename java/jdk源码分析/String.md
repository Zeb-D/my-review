[TOC]





### String

#### 数据结构

```java
/** The value is used for character storage. */
private final char value[];
```

```java
/** Cache the hash code for the string */
private int hash; // Default to 0
```

`private static final long serialVersionUID`

```java
/**
 * Class String is special cased within the Serialization Stream Protocol.
 *
 * A String instance is written into an ObjectOutputStream according to
 * <a href="{@docRoot}/../platform/serialization/spec/output.html">
 * Object Serialization Specification, Section 6.2, "Stream Elements"</a>
 */
private static final ObjectStreamField[] serialPersistentFields =
    new ObjectStreamField[0];
```

```java
/**
 * A Comparator that orders {@code String} objects as by
 * {@code compareToIgnoreCase}. This comparator is serializable.
 * <p>
 * Note that this Comparator does <em>not</em> take locale into account,
 * and will result in an unsatisfactory ordering for certain locales.
 * The java.text package provides <em>Collators</em> to allow
 * locale-sensitive ordering.
 *
 * @see     java.text.Collator#compare(String, String)
 * @since   1.2
 */
public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                     = new CaseInsensitiveComparator();
```



### 实现方式

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence
```



### 主要方法

主要传入 ASCII值（如 'a'为65）

```java
/**
 * Allocates a new {@code String} that contains characters from a subarray
 * of the <a href="Character.html#unicode">Unicode code point</a> array
 * argument.  The {@code offset} argument is the index of the first code
 * point of the subarray and the {@code count} argument specifies the
 * length of the subarray.  The contents of the subarray are converted to
 * {@code char}s; subsequent modification of the {@code int} array does not
 * affect the newly created string.
 *
 * @param  codePoints
 *         Array that is the source of Unicode code points
 *
 * @param  offset
 *         The initial offset
 *
 * @param  count
 *         The length
 *
 * @throws  IllegalArgumentException
 *          If any invalid Unicode code point is found in {@code
 *          codePoints}
 *
 * @throws  IndexOutOfBoundsException
 *          If the {@code offset} and {@code count} arguments index
 *          characters outside the bounds of the {@code codePoints} array
 *
 * @since  1.5
 */
public String(int[] codePoints, int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count < 0) {
        throw new StringIndexOutOfBoundsException(count);
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > codePoints.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }

    final int end = offset + count;

    // Pass 1: Compute precise size of char[]
    int n = count;
    for (int i = offset; i < end; i++) {
        int c = codePoints[i];
        if (Character.isBmpCodePoint(c))
            continue;
        else if (Character.isValidCodePoint(c))
            n++;
        else throw new IllegalArgumentException(Integer.toString(c));
    }

    // Pass 2: Allocate and fill in char[]
    final char[] v = new char[n];

    for (int i = offset, j = 0; i < end; i++, j++) {
        int c = codePoints[i];
        if (Character.isBmpCodePoint(c))
            v[j] = (char)c;
        else
            Character.toSurrogates(c, v, j++);
    }

    this.value = v;
}
```



```java
/**
 * Allocates a new {@code String} constructed from a subarray of an array
 * of 8-bit integer values.
 *
 * <p> The {@code offset} argument is the index of the first byte of the
 * subarray, and the {@code count} argument specifies the length of the
 * subarray.
 *
 * <p> Each {@code byte} in the subarray is converted to a {@code char} as
 * specified in the method above.
 *
 * @deprecated This method does not properly convert bytes into characters.
 * As of JDK&nbsp;1.1, the preferred way to do this is via the
 * {@code String} constructors that take a {@link
 * java.nio.charset.Charset}, charset name, or that use the platform's
 * default charset.
 *
 * @param  ascii
 *         The bytes to be converted to characters
 *
 * @param  hibyte
 *         The top 8 bits of each 16-bit Unicode code unit
 *
 * @param  offset
 *         The initial offset
 * @param  count
 *         The length
 *
 * @throws  IndexOutOfBoundsException
 *          If the {@code offset} or {@code count} argument is invalid
 *
 * @see  #String(byte[], int)
 * @see  #String(byte[], int, int, java.lang.String)
 * @see  #String(byte[], int, int, java.nio.charset.Charset)
 * @see  #String(byte[], int, int)
 * @see  #String(byte[], java.lang.String)
 * @see  #String(byte[], java.nio.charset.Charset)
 * @see  #String(byte[])
 */
@Deprecated
public String(byte ascii[], int hibyte, int offset, int count) {
    checkBounds(ascii, offset, count);
    char value[] = new char[count];

    if (hibyte == 0) {
        for (int i = count; i-- > 0;) {
            value[i] = (char)(ascii[i + offset] & 0xff);
        }
    } else {
        hibyte <<= 8;
        for (int i = count; i-- > 0;) {
            value[i] = (char)(hibyte | (ascii[i + offset] & 0xff));
        }
    }
    this.value = value;
}
```



```java
/**
 * Returns the character (Unicode code point) at the specified
 * index. The index refers to {@code char} values
 * (Unicode code units) and ranges from {@code 0} to
 * {@link #length()}{@code  - 1}.
 *
 * <p> If the {@code char} value specified at the given index
 * is in the high-surrogate range, the following index is less
 * than the length of this {@code String}, and the
 * {@code char} value at the following index is in the
 * low-surrogate range, then the supplementary code point
 * corresponding to this surrogate pair is returned. Otherwise,
 * the {@code char} value at the given index is returned.
 *
 * @param      index the index to the {@code char} values
 * @return     the code point value of the character at the
 *             {@code index}
 * @exception  IndexOutOfBoundsException  if the {@code index}
 *             argument is negative or not less than the length of this
 *             string.
 * @since      1.5
 */
public int codePointAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointAtImpl(value, index, value.length);
}
```

```java
/**
 * Copy characters from this string into dst starting at dstBegin.
 * This method doesn't perform any range checking.
 */
void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
```

```java
/**
 * Compares this string to the specified object.  The result is {@code
 * true} if and only if the argument is not {@code null} and is a {@code
 * String} object that represents the same sequence of characters as this
 * object.
 *
 * @param  anObject
 *         The object to compare this {@code String} against
 *
 * @return  {@code true} if the given object represents a {@code String}
 *          equivalent to this string, {@code false} otherwise
 *
 * @see  #compareTo(String)
 * @see  #equalsIgnoreCase(String)
 */
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

```java
/**
 * Compares this string to the specified {@code CharSequence}.  The
 * result is {@code true} if and only if this {@code String} represents the
 * same sequence of char values as the specified sequence. Note that if the
 * {@code CharSequence} is a {@code StringBuffer} then the method
 * synchronizes on it.
 *
 * @param  cs
 *         The sequence to compare this {@code String} against
 *
 * @return  {@code true} if this {@code String} represents the same
 *          sequence of char values as the specified sequence, {@code
 *          false} otherwise
 *
 * @since  1.5
 */
public boolean contentEquals(CharSequence cs) {
    // Argument is a StringBuffer, StringBuilder
    if (cs instanceof AbstractStringBuilder) {
        if (cs instanceof StringBuffer) {
            synchronized(cs) {
               return nonSyncContentEquals((AbstractStringBuilder)cs);
            }
        } else {
            return nonSyncContentEquals((AbstractStringBuilder)cs);
        }
    }
    // Argument is a String
    if (cs.equals(this))
        return true;
    // Argument is a generic CharSequence
    char v1[] = value;
    int n = v1.length;
    if (n != cs.length()) {
        return false;
    }
    for (int i = 0; i < n; i++) {
        if (v1[i] != cs.charAt(i)) {
            return false;
        }
    }
    return true;
}
```

```java
private boolean nonSyncContentEquals(AbstractStringBuilder sb) {
    char v1[] = value;
    char v2[] = sb.getValue();
    int n = v1.length;
    if (n != sb.length()) {
        return false;
    }
    for (int i = 0; i < n; i++) {
        if (v1[i] != v2[i]) {
            return false;
        }
    }
    return true;
}
```

```java
/**
 * Compares this {@code String} to another {@code String}, ignoring case
 * considerations.  Two strings are considered equal ignoring case if they
 * are of the same length and corresponding characters in the two strings
 * are equal ignoring case.
 *
 * <p> Two characters {@code c1} and {@code c2} are considered the same
 * ignoring case if at least one of the following is true:
 * <ul>
 *   <li> The two characters are the same (as compared by the
 *        {@code ==} operator)
 *   <li> Applying the method {@link
 *        java.lang.Character#toUpperCase(char)} to each character
 *        produces the same result
 *   <li> Applying the method {@link
 *        java.lang.Character#toLowerCase(char)} to each character
 *        produces the same result
 * </ul>
 *
 * @param  anotherString
 *         The {@code String} to compare this {@code String} against
 *
 * @return  {@code true} if the argument is not {@code null} and it
 *          represents an equivalent {@code String} ignoring case; {@code
 *          false} otherwise
 *
 * @see  #equals(Object)
 */
public boolean equalsIgnoreCase(String anotherString) {
    return (this == anotherString) ? true
            : (anotherString != null)
            && (anotherString.value.length == value.length)
            && regionMatches(true, 0, anotherString, 0, value.length);
}
```

```java
public boolean regionMatches(boolean ignoreCase, int toffset,
        String other, int ooffset, int len) {
    char ta[] = value;
    int to = toffset;
    char pa[] = other.value;
    int po = ooffset;
    // Note: toffset, ooffset, or len might be near -1>>>1.
    if ((ooffset < 0) || (toffset < 0)
            || (toffset > (long)value.length - len)
            || (ooffset > (long)other.value.length - len)) {
        return false;
    }
    while (len-- > 0) {
        char c1 = ta[to++];
        char c2 = pa[po++];
        if (c1 == c2) {
            continue;
        }
        if (ignoreCase) {
            // If characters don't match but case may be ignored,
            // try converting both characters to uppercase.
            // If the results match, then the comparison scan should
            // continue.
            char u1 = Character.toUpperCase(c1);
            char u2 = Character.toUpperCase(c2);
            if (u1 == u2) {
                continue;
            }
            // Unfortunately, conversion to uppercase does not work properly
            // for the Georgian alphabet, which has strange rules about case
            // conversion.  So we need to make one last check before
            // exiting.
            if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                continue;
            }
        }
        return false;
    }
    return true;
}
```

```java
public int compareTo(String anotherString) {
    int len1 = value.length;
    int len2 = anotherString.value.length;
    int lim = Math.min(len1, len2);
    char v1[] = value;
    char v2[] = anotherString.value;

    int k = 0;
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {
            return c1 - c2;
        }
        k++;
    }
    return len1 - len2;
}
```

```java
public int compareToIgnoreCase(String str) {
    return CASE_INSENSITIVE_ORDER.compare(this, str);
}
```

```java
public int compare(String s1, String s2) {
    int n1 = s1.length();
    int n2 = s2.length();
    int min = Math.min(n1, n2);
    for (int i = 0; i < min; i++) {
        char c1 = s1.charAt(i);
        char c2 = s2.charAt(i);
        if (c1 != c2) {
            c1 = Character.toUpperCase(c1);
            c2 = Character.toUpperCase(c2);
            if (c1 != c2) {
                c1 = Character.toLowerCase(c1);
                c2 = Character.toLowerCase(c2);
                if (c1 != c2) {
                    // No overflow because of numeric promotion
                    return c1 - c2;
                }
            }
        }
    }
    return n1 - n2;
}
```

```java
public boolean startsWith(String prefix, int toffset) {
    char ta[] = value;
    int to = toffset;
    char pa[] = prefix.value;
    int po = 0;
    int pc = prefix.value.length;
    // Note: toffset might be near -1>>>1.
    if ((toffset < 0) || (toffset > value.length - pc)) {
        return false;
    }
    while (--pc >= 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}
```

```java
/**
     * Returns a hash code for this string. The hash code for a
     * {@code String} object is computed as
     * <blockquote><pre>
     * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
     * </pre></blockquote>
     * using {@code int} arithmetic, where {@code s[i]} is the
     * <i>i</i>th character of the string, {@code n} is the length of
     * the string, and {@code ^} indicates exponentiation.
     * (The hash value of the empty string is zero.)
     *
     * @return  a hash code value for this object.
     */
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

```java
public int indexOf(int ch, int fromIndex) {
    final int max = value.length;
    if (fromIndex < 0) {
        fromIndex = 0;
    } else if (fromIndex >= max) {
        // Note: fromIndex might be near -1>>>1.
        return -1;
    }

    if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
        // handle most cases here (ch is a BMP code point or a
        // negative value (invalid code point))
        final char[] value = this.value;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == ch) {
                return i;
            }
        }
        return -1;
    } else {
        return indexOfSupplementary(ch, fromIndex);
    }
}
```

```java
/**
 * Handles (rare) calls of indexOf with a supplementary character.
 */
private int indexOfSupplementary(int ch, int fromIndex) {
    if (Character.isValidCodePoint(ch)) {
        final char[] value = this.value;
        final char hi = Character.highSurrogate(ch);
        final char lo = Character.lowSurrogate(ch);
        final int max = value.length - 1;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == hi && value[i + 1] == lo) {
                return i;
            }
        }
    }
    return -1;
}
```

```java
public String substring(int beginIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    int subLen = value.length - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
}
```

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        while (++i < len) {
            if (val[i] == oldChar) {
                break;
            }
        }
        if (i < len) {
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            return new String(buf, true);
        }
    }
    return this;
}
```



```java
static int indexOf(char[] source, int sourceOffset, int sourceCount,
        char[] target, int targetOffset, int targetCount,
        int fromIndex) {
    if (fromIndex >= sourceCount) {
        return (targetCount == 0 ? sourceCount : -1);
    }
    if (fromIndex < 0) {
        fromIndex = 0;
    }
    if (targetCount == 0) {
        return fromIndex;
    }

    char first = target[targetOffset];
    int max = sourceOffset + (sourceCount - targetCount);

    for (int i = sourceOffset + fromIndex; i <= max; i++) {
        /* Look for first character. */
        if (source[i] != first) {
            while (++i <= max && source[i] != first);
        }

        /* Found first character, now look at the rest of v2 */
        if (i <= max) {
            int j = i + 1;
            int end = j + targetCount - 1;
            for (int k = targetOffset + 1; j < end && source[j]
                    == target[k]; j++, k++);

            if (j == end) {
                /* Found whole string. */
                return i - sourceOffset;
            }
        }
    }
    return -1;
}
```



**使用正则表达式来查找是否存在**

```java
/**
 * Tells whether or not this string matches the given <a
 * href="../util/regex/Pattern.html#sum">regular expression</a>.
 * <p> An invocation of this method of the form
 * <i>str</i>{@code .matches(}<i>regex</i>{@code )} yields exactly the
 * same result as the expression
 * <blockquote>
 * {@link java.util.regex.Pattern}.{@link java.util.regex.Pattern#matches(String,CharSequence)
 * matches(<i>regex</i>, <i>str</i>)}
 * </blockquote>
 * @param   regex
 *          the regular expression to which this string is to be matched
 * @return  {@code true} if, and only if, this string matches the
 *          given regular expression
 * @throws  PatternSyntaxException
 *          if the regular expression's syntax is invalid
 * @see java.util.regex.Pattern
 * @since 1.4
 * @spec JSR-51
 */
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

```java
public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);
    Matcher m = p.matcher(input);
    return m.matches();
}
```

```java
public String replaceFirst(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
}
```

```java
public String replaceFirst(String replacement) {
    if (replacement == null)
        throw new NullPointerException("replacement");
    reset();
    if (!find())
        return text.toString();
    StringBuffer sb = new StringBuffer();
    appendReplacement(sb, replacement);
    appendTail(sb);
    return sb.toString();
}
```

```java
public String replaceAll(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceAll(replacement);
}
```

```java
public String replace(CharSequence target, CharSequence replacement) {
    return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(
            this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
}
```



```java
/**
 * Splits this string around matches of the given
 * <a href="../util/regex/Pattern.html#sum">regular expression</a>.
 *
 * <p> The array returned by this method contains each substring of this
 * string that is terminated by another substring that matches the given
 * expression or is terminated by the end of the string.  The substrings in
 * the array are in the order in which they occur in this string.  If the
 * expression does not match any part of the input then the resulting array
 * has just one element, namely this string.
 *
 * <p> When there is a positive-width match at the beginning of this
 * string then an empty leading substring is included at the beginning
 * of the resulting array. A zero-width match at the beginning however
 * never produces such empty leading substring.
 *
 * <p> The {@code limit} parameter controls the number of times the
 * pattern is applied and therefore affects the length of the resulting
 * array.  If the limit <i>n</i> is greater than zero then the pattern
 * will be applied at most <i>n</i>&nbsp;-&nbsp;1 times, the array's
 * length will be no greater than <i>n</i>, and the array's last entry
 * will contain all input beyond the last matched delimiter.  If <i>n</i>
 * is non-positive then the pattern will be applied as many times as
 * possible and the array can have any length.  If <i>n</i> is zero then
 * the pattern will be applied as many times as possible, the array can
 * have any length, and trailing empty strings will be discarded.
 *
 * <p> The string {@code "boo:and:foo"}, for example, yields the
 * following results with these parameters:
 *
 * <blockquote><table cellpadding=1 cellspacing=0 summary="Split example showing regex, limit, and result">
 * <tr>
 *     <th>Regex</th>
 *     <th>Limit</th>
 *     <th>Result</th>
 * </tr>
 * <tr><td align=center>:</td>
 *     <td align=center>2</td>
 *     <td>{@code { "boo", "and:foo" }}</td></tr>
 * <tr><td align=center>:</td>
 *     <td align=center>5</td>
 *     <td>{@code { "boo", "and", "foo" }}</td></tr>
 * <tr><td align=center>:</td>
 *     <td align=center>-2</td>
 *     <td>{@code { "boo", "and", "foo" }}</td></tr>
 * <tr><td align=center>o</td>
 *     <td align=center>5</td>
 *     <td>{@code { "b", "", ":and:f", "", "" }}</td></tr>
 * <tr><td align=center>o</td>
 *     <td align=center>-2</td>
 *     <td>{@code { "b", "", ":and:f", "", "" }}</td></tr>
 * <tr><td align=center>o</td>
 *     <td align=center>0</td>
 *     <td>{@code { "b", "", ":and:f" }}</td></tr>
 * </table></blockquote>
 *
 * <p> An invocation of this method of the form
 * <i>str.</i>{@code split(}<i>regex</i>{@code ,}&nbsp;<i>n</i>{@code )}
 * yields the same result as the expression
 *
 * <blockquote>
 * <code>
 * {@link java.util.regex.Pattern}.{@link
 * java.util.regex.Pattern#compile compile}(<i>regex</i>).{@link
 * java.util.regex.Pattern#split(java.lang.CharSequence,int) split}(<i>str</i>,&nbsp;<i>n</i>)
 * </code>
 * </blockquote>
 *
 *
 * @param  regex
 *         the delimiting regular expression
 *
 * @param  limit
 *         the result threshold, as described above
 *
 * @return  the array of strings computed by splitting this string
 *          around matches of the given regular expression
 *
 * @throws  PatternSyntaxException
 *          if the regular expression's syntax is invalid
 *
 * @see java.util.regex.Pattern
 *
 * @since 1.4
 * @spec JSR-51
 */
public String[] split(String regex, int limit) {
    /* fastpath if the regex is a
     (1)one-char String and this character is not one of the
        RegEx's meta characters ".$|()[{^?*+\\", or
     (2)two-char String and the first char is the backslash and
        the second is not the ascii digit or ascii letter.
     */
    char ch = 0;
    if (((regex.value.length == 1 &&
         ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
         (regex.length() == 2 &&
          regex.charAt(0) == '\\' &&
          (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
          ((ch-'a')|('z'-ch)) < 0 &&
          ((ch-'A')|('Z'-ch)) < 0)) &&
        (ch < Character.MIN_HIGH_SURROGATE ||
         ch > Character.MAX_LOW_SURROGATE))
    {
        int off = 0;
        int next = 0;
        boolean limited = limit > 0;
        ArrayList<String> list = new ArrayList<>();
        while ((next = indexOf(ch, off)) != -1) {
            if (!limited || list.size() < limit - 1) {
                list.add(substring(off, next));
                off = next + 1;
            } else {    // last one
                //assert (list.size() == limit - 1);
                list.add(substring(off, value.length));
                off = value.length;
                break;
            }
        }
        // If no match was found, return this
        if (off == 0)
            return new String[]{this};

        // Add remaining segment
        if (!limited || list.size() < limit)
            list.add(substring(off, value.length));

        // Construct result
        int resultSize = list.size();
        if (limit == 0) {
            while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
                resultSize--;
            }
        }
        String[] result = new String[resultSize];
        return list.subList(0, resultSize).toArray(result);
    }
    return Pattern.compile(regex).split(this, limit);
}
```





- JDK 6和JDK 7中substring的原理及区别、

- replaceFirst、replaceAll、replace区别、

- String对“+”的重载、

- String.valueOf和Integer.toString的区别、

- 字符串的不可变性

  ​




