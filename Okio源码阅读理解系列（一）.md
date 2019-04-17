@[TOC](Okio源码阅读理解系列（一）)
# Okio
> Okio是一个java.io和java.nio的补充库，使访问、存储和处理数据变得更加容易。它最初是OkHttp的一个组件，OkHttp是Android中包含的功能强大的HTTP客户机。它训练有素，随时准备解决新问题。

## 几个概念
在Okio中定义了几个类，我们先来了解一下它们的概念：
- ByteString

> 字节串是一个不可变的字节序列。对于字符数据，字符串是基本的。ByteString是String丢失已久的兄弟，因此很容易将二进制数据视为值。这个类是符合人体工程学的:它知道如何将自己编码和解码为hex、base64和UTF-8。

- Buffer

> Buffer是一个可变的字节序列。像ArrayList一样，您不需要预先调整缓冲区的大小。您将缓冲区的读写作为一个队列:将数据写到末尾并从前端读取。没有管理职位、限制或能力的义务。

> 在内部，ByteString和Buffer做了一些聪明的事情来节省CPU和内存。如果您将UTF-8字符串编码为ByteString，它将缓存对该字符串的引用，这样，如果您稍后对其进行解码，就不需要做任何工作。
缓冲区被实现为Segment的链表。当您将数据从一个缓冲区移动到另一个缓冲区时，它将重新分配段的所有权，而不是跨段复制数据。这种方法对于多线程程序特别有用:与网络通信的线程可以与工作线程交换数据，而不需要任何复制或仪式。

- Source 输入流，对应java原生io中的InputStream。
- Sink 输出流，对应java原生io中的OutputStream。

![关键类和接口](https://img-blog.csdnimg.cn/20190417162919836.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NTMyMTQw,size_16,color_FFFFFF,t_70)

## 基本使用

```java
public void readLines(File file) throws IOException {
  try (Source fileSource = Okio.source(file);
      BufferedSource bufferedSource = Okio.buffer(fileSource)) {
      String s = bufferedSource.readUtf8();
      if (s != null) System.out.println(line);
  }
}
```
## 源码分析
这里又是使用kotlin改造过的工程，没有了解过的小伙伴们还是去了解下，以便更好地阅读源码。
这里Okio的source方法已经被替换了，换成了输入源的扩展方法，我们这里是File，也就是换成了File.source()：

```kotlin
@Throws(FileNotFoundException::class)
fun File.source(): Source = inputStream().source()
```
这里的inputStream也是File的一个扩展方法，不过它不是Okio中提供的，它是kotlin的标准库中提供的：

```kotlin
@kotlin.internal.InlineOnly
public inline fun File.inputStream(): FileInputStream {
    return FileInputStream(this)
}
```
再来看source()方法，是InputStream的一个扩展函数：

```kotlin
fun InputStream.source(): Source = InputStreamSource(this, Timeout())
```
把InputStream包装成了InputStreamSource，它是Source接口的一个实现。
接着通过Okio的buffer方法将这个source包装成BufferedSource：

```kotlin
actual fun Source.buffer(): BufferedSource = RealBufferedSource(this)
```
我们看到RealBufferedSource的readUtf8()方法：

```kotlin
override fun readUtf8(): String {
  buffer.writeAll(source)
  return buffer.readUtf8()
}
```
我们可以看到首先将source中的数据全部写入缓冲区，接着再从缓冲区中读取，我们进入write方法：

```kotlin
@Throws(IOException::class)
override fun writeAll(source: Source): Long {
  var totalBytesRead = 0L
  while (true) {
    val readCount = source.read(this, Segment.SIZE.toLong())
    if (readCount == -1L) break
    totalBytesRead += readCount
  }
  return totalBytesRead
}
```
我们看到它是从source中读取数据，每次读取一个Segment的大小：

```kotlin
override fun read(sink: Buffer, byteCount: Long): Long {
  if (byteCount == 0L) return 0
  require(byteCount >= 0) { "byteCount < 0: $byteCount" }
  try {
    timeout.throwIfReached()
    val tail = sink.writableSegment(1)
    val maxToCopy = minOf(byteCount, Segment.SIZE - tail.limit).toInt()
    val bytesRead = input.read(tail.data, tail.limit, maxToCopy)
    if (bytesRead == -1) return -1
    tail.limit += bytesRead
    sink.size += bytesRead
    return bytesRead.toLong()
  } catch (e: AssertionError) {
    if (e.isAndroidGetsocknameError) throw IOException(e)
    throw e
  }
}
```
这里的工作就是先从缓冲区获取可以写入的Segment然后从inputstream中读取数据到Segment的data数组中。

```kotlin
internal actual fun writableSegment(minimumCapacity: Int): Segment =
  commonWritableSegment(minimumCapacity)

internal fun Buffer.commonWritableSegment(minimumCapacity: Int): Segment {
  require(minimumCapacity >= 1 && minimumCapacity <= Segment.SIZE) { "unexpected capacity" }

  if (head == null) {
    val result = SegmentPool.take() // Acquire a first segment.
    head = result
    result.prev = result
    result.next = result
    return result
  }

  var tail = head!!.prev
  if (tail!!.limit + minimumCapacity > Segment.SIZE || !tail.owner) {
    tail = tail.push(SegmentPool.take()) // Append a new empty segment to fill up.
  }
  return tail
}
```
这样就把inputStream中读取到的数据写入了Buffer中以head为头的Segment链表中。
接着我们看buffer的readUtf8方法：

```kotlin
@Throws(EOFException::class)
override fun readString(byteCount: Long, charset: Charset): String {
  require(byteCount >= 0 && byteCount <= Integer.MAX_VALUE) { "byteCount: $byteCount" }
  if (size < byteCount) throw EOFException()
  if (byteCount == 0L) return ""

  val s = head!!
  if (s.pos + byteCount > s.limit) {
    // If the string spans multiple segments, delegate to readBytes().
    return String(readByteArray(byteCount), charset)
  }

  val result = String(s.data, s.pos, byteCount.toInt(), charset)
  s.pos += byteCount.toInt()
  size -= byteCount

  if (s.pos == s.limit) {
    head = s.pop()
    SegmentPool.recycle(s)
  }

  return result
}
```
readUtf8方法会调用readString，然后通过kotlin的String标准库解码读出内容。
**Okio的源码差不多，我们这里就分析这么多，其实只做了整体框架的分析，其实源码的细节涉及非常多的链表数据结构的操作，还是比较复杂的。有兴趣的同学可以自行研究。这里使用缓存减少io操作的次数也是对文件系统一个非常好的优化，因为我们知道，内存的速度是要比磁盘快很多的，因此Okio对于性能的提升还是非常明显的～**
