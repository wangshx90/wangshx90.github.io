---
title:  golang中的io.Copy 和 ioutil.ReadAll
date: 2021-02-04 17:00:00 +0800
categories: golang
tags: golang io.Copy ioutil.ReadAll
mermaid: true
---

golang中当我们需要拷贝数据时，常常用到 io.Copy 和 ioutil.ReadAll。那么这两个函数有什么区别，以及在何时应该用哪个呢？

## ioutil.ReadAll

之所以把`ReadAll`单独拿出来讲，一来是因为我们经常需要把数据从某个 `io.Reader`对象读出来，二来也是因为它的性能问题常常被人吐槽。

先来看下它的使用场景。比如说，我们使用`http.Client`发送`GET`请求：

```go
func main() {
	res, err := http.Get("http://www.google.com/robots.txt")
	if err != nil {
		log.Fatal(err)
	}
	robots, err := io.ReadAll(res.Body)
	res.Body.Close()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%s", robots)
}
```

`http.Get()`返回的数据，存储在`res.Body`中，我们通过`ioutil.ReadAll`将其读取出来。

下面看看`ioutil.ReadAll`的源码实现：

```go
func ReadAll(r io.Reader) ([]byte, error) {
	return io.ReadAll(r)
}
```

从1.16版本开始，`ioutil.ReadAll()`直接调用`io.ReadAll()`。我们接着跟踪：

```go
// ReadAll reads from r until an error or EOF and returns the data it read.
// A successful call returns err == nil, not err == EOF. Because ReadAll is
// defined to read from src until EOF, it does not treat an EOF from Read
// as an error to be reported.
func ReadAll(r Reader) ([]byte, error) {
	b := make([]byte, 0, 512)
	for {
		if len(b) == cap(b) {
			// Add more capacity (let append pick how much).
			b = append(b, 0)[:len(b)]
		}
		n, err := r.Read(b[len(b):cap(b)])
		b = b[:len(b)+n]
		if err != nil {
			if err == EOF {
				err = nil
			}
			return b, err
		}
	}
}
```
从功能上来讲，`ReadAll`会从`r`中持续读取数据，直到返回 `EOF`或者出错；但是在返回给上层时，`EOF`并不会被当成`error`。

接下来分析其实现。

- 第6行，创建一个512字节的buffer；
- 第7~13行，反复读取数据到buffer，如果buffer满，调用`append()`追加1个字节，迫使其重新分配内存
- 第14~18行，如果调用`r.Read()`出错，终止循环，并在返回之前将`EOF`过滤掉

这里的关键就是拷贝数据的单位了：512Bytes。如果待拷贝的数据量小于512B，那么没啥关系；如果待拷贝数据超过512B，就会发生频繁的realloc和数据拷贝了，且数据量越大就越严重。

这里还涉及一个知识点，就是slice的扩容策略。

- 如果现有容量小于1024，那么新slice容量将扩大为原来的2倍，防止频繁扩容；
- 如果现有容量超过1024，那么新slice容量是现有的1.25倍，防止空间浪费。

那有没有替代品呢？

## io.Copy

话不多说，直接上代码：
```go
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
```
功能：将数据从 `src`读出来，写入到`dst`；返回成功复制的字节数。

可以看到，从接口的语义来看，`ReadAll`获取的是数据缓冲区，相当于只完成读出数据的功能；而`Copy`则实现了数据处理的完整流程：读出数据，然后再写入（使用）数据。

也正是因为语义上受限，使用`ReadAll`来处理数据，必须先将数据全部读出来，才能使用数据；而`Copy`则可以将两者结合起来，一边读一边写，非常适合处理数据量大的场景。

下面接着看`copyBuffer()`的实现。

```go
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw < 0 || nr < nw {
				nw = 0
				if ew == nil {
					ew = errInvalidWrite
				}
			}
			written += int64(nw)
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```

- 第2~6行，如果`src`底层对象也实现了`WriterTo`接口，那么直接执行`src.WriteTo(dst)`；
- 第7~10行，如果`dst`底层对象也实现了`ReaderFrom`接口，那么直接执行`dst.ReadFrom(src)`；
- 第11~21行，如果传入的`buf`为空，那么新创建一个32KB的buffer。如果`src`底层同时是一个`*LimitedReader`对象（意味着能从它读取出的数据量有限制），且剩余可读的数据量小于32KB，那么就把buffer大小限制为跟它一样长；
- 第22~41行，反复将数据从`src`读取到`buf`，再将数据写入到`dst`；
- 第42~46行，如果出错直接终止循环，并在返回之前将`EOF`过滤掉。


可以看到，相比于`io.ReadAll`，`io.Copy`有以下优势：

- 如果`src`和`dst`分别是`WriterTo`或`ReaderFrom`，那么就省去了中间buf缓存的环节，数据直接从`src`到`dst`；
- 使用固定长度的buffer作为临时缓冲区，不会出现slice的频繁扩容。


## 总结
综上所述，如果是小数据量的拷贝，使用`ioutil.ReadAll`无伤大雅；数据量较大时，`ReadAll`就是性能炸弹了，最好使用`io.Copy`。

此外，`Copy`提供更完整的语义，所以针对使用`ReadAll()`的场景，建议将数据处理流程也考虑进来，将其抽象为一个`Writer`对象，然后使用`Copy`完成数据的读取和处理流程。

特别的，如果读取出来的数据，是要用json再解码，可以连`io.Copy`都不用：
```go
type Result struct {
    Msg string `json:"msg"`
    Rescode string `json:"rescode"`
}
func parseBody(body io.Reader) {
	var v Result
	err := json.NewDecoder(body).Decode(&v)
	if err != nil {
		return nil, fmt.Errorf("DecodeJsonFailed:%s", err.Error())
	}
}
```
