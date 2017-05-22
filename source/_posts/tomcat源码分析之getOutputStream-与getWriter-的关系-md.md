---
title: tomcat源码分析之getOutputStream()与getWriter()的关系.md
date: 2017-05-11 19:20:18
tags:
---
在使用tomcat的时候,如果同时使用了getOutputStream()与getWriter()就会抛
java.lang.IllegalStateException: getOutputStream() has already been called for this response异常
    
    这是因为,如果同时使用了getOutputStream是直接发送给客户端,而getWriter则是放到了一个缓存里(stringbuf)里,   
    如果同时存在两种操作,务必灰引起紊乱,所以,tomcat会避开这种操作.

### 先来看下getOutputStream()方法
    
####    如果是nio模式,该方法返回的是NioServletOutputStream类,

``` code
private int doWriteInternal (boolean block, byte[] b, int off, int len)
            throws IOException {
        channel.getBufHandler().getWriteBuffer().clear();
        channel.getBufHandler().getWriteBuffer().put(b, off, len);
        channel.getBufHandler().getWriteBuffer().flip();

        int written = 0;
        NioEndpoint.KeyAttachment att =
                (NioEndpoint.KeyAttachment) channel.getAttachment();
        if (att == null) {
            throw new IOException("Key must be cancelled");
        }
        long writeTimeout = att.getWriteTimeout();
        Selector selector = null;
        try {
            selector = pool.get();
        } catch ( IOException x ) {
            //ignore
        }
        try {
            written = pool.write(channel.getBufHandler().getWriteBuffer(),
                    channel, selector, writeTimeout, block);
        } finally {
            if (selector != null) {
                pool.put(selector);
            }
        }
        if (written < len) {
            channel.getPoller().add(channel, SelectionKey.OP_WRITE);
        }
        return written;
    }
``` 
直接与网络层接触,直接返回给客户端
### 再看下与getWriter
    实际上返回的是CoyoteWriter类,查看其源码write方法
``` code

protected OutputBuffer ob;
@Override
    public void write(char buf[], int off, int len) {

        if (error) {
            return;
        }

        try {
            ob.write(buf, off, len);
        } catch (IOException e) {
            error = true;
        }

    }
```
其实是写到一个buffer里面,tomcat再从buffer里面读取,这部分逻辑我还没查清楚,有可能读跟写是并行的.

#### 所以tomcat要杜绝你即使用先来看下getOutputStream,又使用getWriter.防止出现数据的不连续性.


