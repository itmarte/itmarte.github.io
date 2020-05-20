---
title: scanner、buffer reader、jdk8 stream、apache common io 效率对比
date: 2020-05-20 14:48:27
tags:
    - java
    - io
    - nio
categories:
    - java
    - read line
---
  
## 文件行读效率测试

## scanner、buffer reader、jdk8 stream、apache common io

### 一、先看下代码

```java
public static void main(String[] args) throws IOException {
        File file = new File(FILE_PATH);

        // scanner
        System.out.println("------------************************-------------");
        System.out.println(">>>>>>scanner 开始 >>>>>>>");
        Long cur = System.currentTimeMillis();
        scanner(file);
        System.out.println("scanner 耗时[" + (System.currentTimeMillis() - cur) + "]ms");
        System.out.println("<<<<<<<scanner 结束 <<<<<<<");
        System.out.println("------------************************-------------");
        System.out.println();

        // buffer reader
        System.out.println("------------************************-------------");
        System.out.println(">>>>>>buffer reader 开始 >>>>>>>");
        cur = System.currentTimeMillis();
        bufferReader(file);
        System.out.println("buffer reader 耗时[" + (System.currentTimeMillis() - cur) + "]ms");
        System.out.println("<<<<<<<buffer reader 结束 <<<<<<<");
        System.out.println("------------************************-------------");
        System.out.println();

        // JDK8
        System.out.println("------------************************-------------");
        System.out.println(">>>>>>JDK8 stream开始 >>>>>>>");
        cur = System.currentTimeMillis();
        jdk8Reader(FILE_PATH);
        System.out.println("JDK8 stream 耗时[" + (System.currentTimeMillis() - cur) + "]ms");
        System.out.println("<<<<<<<JDK8 stream 结束 <<<<<<<");
        System.out.println("------------************************-------------");
        System.out.println();

        // apache common io
        System.out.println("------------************************-------------");
        System.out.println(">>>>>>apache common io >>>>>>>");
        cur = System.currentTimeMillis();
        commonIo(file);
        System.out.println("apache common io 耗时[" + (System.currentTimeMillis() - cur) + "]ms");
        System.out.println("<<<<<<<apache common io 结束 <<<<<<<");
        System.out.println("------------************************-------------");

    }

    private static void commonIo(File file) throws IOException {
        int total = 0;
        try (LineIterator lineIterator = FileUtils.lineIterator(file, "UTF-8")) {
            while (lineIterator.hasNext()) {
                lineIterator.next();
                total += 1;
            }
        } finally {
            System.out.println("[总行数]:" + total);
        }
    }

    /**
     * JDK8
     *
     * @param filePath
     * @throws IOException
     */
    private static void jdk8Reader(String filePath) throws IOException {
        // Files.readAllLine 内部使用的是buffer reader
//        Files.readAllLines(file.toPath());
        int total = 0;
        try (Stream<String> stream = Files.lines(Paths.get(filePath), StandardCharsets.UTF_8)) {
            total = stream.reduce(0, (cur, op) -> cur + 1, (a, b) -> a + b);
        } finally {
            System.out.println("[总行数]:" + total);
        }
    }

    /**
     * buffer reader
     *
     * @param file
     * @throws IOException
     */
    private static void bufferReader(File file) throws IOException {
        int total = 0;
        try (BufferedReader br = new BufferedReader(new FileReader(file))) {
            for (String line; (line = br.readLine()) != null; total++);
            // line is not visible here.
        } finally {
            System.out.println("[总行数]:" + total);
        }
    }

    private static void scanner(File file) throws FileNotFoundException {
        int total = 0;
        try(Scanner scanner = new Scanner(file)){
            while (scanner.hasNextLine()) {
                scanner.nextLine();
                total+=1;
            }
        } finally {
            System.out.println("[总行数]:" + total);
        }
    }
```

#### 100W条真实数据测试结果：

```java
------------************************-------------
>>>>>>scanner 开始 >>>>>>>
[总行数]:1056322
scanner 耗时[8379]ms
<<<<<<<scanner 结束 <<<<<<<
------------************************-------------
    
------------************************-------------
>>>>>>buffer reader 开始 >>>>>>>
[总行数]:1056322
buffer reader 耗时[901]ms
<<<<<<<buffer reader 结束 <<<<<<<
------------************************-------------
    
------------************************-------------
>>>>>>JDK8 stream开始 >>>>>>>
[总行数]:1056322
JDK8 stream 耗时[916]ms
<<<<<<<JDK8 stream 结束 <<<<<<<
------------************************-------------
    
------------************************-------------
>>>>>>apache common io >>>>>>>
[总行数]:1056322
apache common io 耗时[929]ms
<<<<<<<apache common io 结束 <<<<<<<
------------************************-------------
```

#### 1000W条真实数据测试结果：

```java
------------************************-------------
>>>>>>scanner 开始 >>>>>>>
[总行数]:10563211
scanner 耗时[79109]ms
<<<<<<<scanner 结束 <<<<<<<
------------************************-------------

------------************************-------------
>>>>>>buffer reader 开始 >>>>>>>
[总行数]:10563211
buffer reader 耗时[8477]ms
<<<<<<<buffer reader 结束 <<<<<<<
------------************************-------------

------------************************-------------
>>>>>>JDK8 stream开始 >>>>>>>
[总行数]:10563211
JDK8 stream 耗时[8623]ms
<<<<<<<JDK8 stream 结束 <<<<<<<
------------************************-------------

------------************************-------------
>>>>>>apache common io >>>>>>>
[总行数]:10563211
apache common io 耗时[8573]ms
<<<<<<<apache common io 结束 <<<<<<<
------------************************-------------
```

总结：

scanner：

```java
// Tries to read more input. May block.
private void readInput() {
    if (buf.limit() == buf.capacity())
        makeSpace();

    // Prepare to receive data
    int p = buf.position();
    buf.position(buf.limit());
    buf.limit(buf.capacity());

    int n = 0;
    try {
        n = source.read(buf);
    } catch (IOException ioe) {
        lastException = ioe;
        n = -1;
    }

    if (n == -1) {
        sourceClosed = true;
        needInput = false;
    }

    if (n > 0)
        needInput = false;

    // Restore current position and limit for reading
    buf.limit(buf.position());
    buf.position(p);
}
```

buffer reader

```java
// 初始大小为8192字节
private static int defaultCharBufferSize = 8192;
private static int defaultExpectedLineLength = 80;

String readLine(boolean ignoreLF) throws IOException {
    StringBuffer s = null;
    int startChar;

    synchronized (lock) {
        ensureOpen();
        boolean omitLF = ignoreLF || skipLF;

    bufferLoop:
        for (;;) {

            if (nextChar >= nChars)
                fill();
            if (nextChar >= nChars) { /* EOF */
                if (s != null && s.length() > 0)
                    return s.toString();
                else
                    return null;
            }
            boolean eol = false;
            char c = 0;
            int i;

            /* Skip a leftover '\n', if necessary */
            if (omitLF && (cb[nextChar] == '\n'))
                nextChar++;
            skipLF = false;
            omitLF = false;

        charLoop:
            for (i = nextChar; i < nChars; i++) {
                c = cb[i];
                if ((c == '\n') || (c == '\r')) {
                    eol = true;
                    break charLoop;
                }
            }

            startChar = nextChar;
            nextChar = i;

            if (eol) {
                String str;
                if (s == null) {
                    str = new String(cb, startChar, i - startChar);
                } else {
                    s.append(cb, startChar, i - startChar);
                    str = s.toString();
                }
                nextChar++;
                if (c == '\r') {
                    skipLF = true;
                }
                return str;
            }

            if (s == null)
                s = new StringBuffer(defaultExpectedLineLength);
            s.append(cb, startChar, i - startChar);
        }
    }
}
```

JDK stream

```java
public static Stream<String> lines(Path path, Charset cs) throws IOException {
    // 实际也用的buffer reader
    BufferedReader br = Files.newBufferedReader(path, cs);
    try {
        return br.lines().onClose(asUncheckedRunnable(br));
    } catch (Error|RuntimeException e) {
        try {
            br.close();
        } catch (IOException ex) {
            try {
                e.addSuppressed(ex);
            } catch (Throwable ignore) {}
        }
        throw e;
    }
}
```

common io

```java
public boolean hasNext() {
    if (this.cachedLine != null) {
        return true;
    } else if (this.finished) {
        return false;
    } else {
        try {
            String line;
            do {
                line = this.bufferedReader.readLine();
                if (line == null) {
                    this.finished = true;
                    return false;
                }
            } while(!this.isValidLine(line));

            this.cachedLine = line;
            return true;
        } catch (IOException var4) {
            try {
                this.close();
            } catch (IOException var3) {
                var4.addSuppressed(var3);
            }

            throw new IllegalStateException(var4);
        }
    }
}
```

所以出了Scanner用的自己的缓存机制，其他的都用的buffer reader的缓存机制，所以后面的三种方法效果差不多，不过笔者还是喜欢jdk8的stream机制，所以选择了jdk8的