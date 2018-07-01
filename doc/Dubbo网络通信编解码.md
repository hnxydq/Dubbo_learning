## Dubbo网络通信-编解码

### Dubbo远程通讯协议头定义

![dubbo_protocol_header](img/dubbo_protocol_header.jpg)



**1.为什么Dubbo要自己增加协议扩展？**

----为了解决TCP/IP粘包和拆包的问题。

所谓粘包和拆包，就是一个完整的业务数据可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送。 

**2.TCP为什么会出现粘包与拆包？**

TCP是以流动的方式传输数据，传输的最小单位为一个报文段（segment）。Tcp Header中有个Options标识位，常见的标识为mss(Maximum Segment Size)指的是，连接层每次传输的数据有个最大限制MTU(Maximum Transmission Unit)，一般是1500比特，超过这个量要分成多个报文段，mss则是这个最大限制减去TCP的header，光是要传输的数据的大小，一般为1460比特。换算成字节，也就是180多字节。

TCP为提高性能，发送端会将需要发送的数据发送到缓冲区，等待缓冲区满了之后，再将缓冲中的数据发送到接收方。同理，接收方也有缓冲区这样的机制，来接收数据。

发生TCP粘包、拆包主要是由于下面一些原因：

①应用程序写入的数据大于套接字缓冲区大小，这将会发生拆包。

②应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生粘包。

③进行mss（最大报文长度）大小的TCP分段，当TCP报文长度-TCP头部长度>mss的时候将发生拆包。

④接收方法不及时读取套接字缓冲区数据，这将发生粘包。

**3.如何解决拆包粘包**

既然知道了tcp是无界的数据流，且协议本身无法避免粘包，拆包的发生，那我们只能在应用层数据协议上，加以控制。通常在制定传输数据时，可以使用如下方法：

- 使用带消息头的协议、消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容。
- 设置定长消息，服务端每次读取既定长度的内容作为一条完整消息。
- 设置消息边界，服务端从网络流中按消息编辑分离出消息内容，如在包尾增加回车或者空格等特殊字符作为分割，典型如FTP。


### Consumer请求编码

通过断点跟踪Consumer请求编码过程：

运行dubbo-demo-consumer工程，在dubbo-remoting-netty工程NettyCodecAdapter.java内部类InternalEncoder#encode()断点codec.encode(channel, buffer, msg);：

```java
@Sharable
    private class InternalEncoder extends OneToOneEncoder {

        @Override
        protected Object encode(ChannelHandlerContext ctx, Channel ch, Object msg) throws Exception {
            com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer =
                    com.alibaba.dubbo.remoting.buffer.ChannelBuffers.dynamicBuffer(1024);
            NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
            try {
                codec.encode(channel, buffer, msg);
            } finally {
                NettyChannel.removeChannelIfDisconnected(ch);
            }
            return ChannelBuffers.wrappedBuffer(buffer.toByteBuffer());
        }
    }
```

接下来跟踪调用层次：

```java
-->NettyCodecAdapter.InternalEncoder.encode
  -->DubboCountCodec.encode
    -->ExchangeCodec.encode
      -->ExchangeCodec.encodeRequest
        -->ExchangeCodec.encodeRequestData
          -->DubboCodec.encodeRequestData
            通过默认的Hessian2ObjectOutput将RpcInvocation数据写入序列化buffer，最终由netty发送。
```

需要关注请求端编码过程：

```java
protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        Serialization serialization = getSerialization(channel);
        // header.
        byte[] header = new byte[HEADER_LENGTH];
        // set magic number.
        Bytes.short2bytes(MAGIC, header);

        // set request and serialization flag.
        header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());

        if (req.isTwoWay()) header[2] |= FLAG_TWOWAY;
        if (req.isEvent()) header[2] |= FLAG_EVENT;

        // set request id.
        Bytes.long2bytes(req.getId(), header, 4);

        // encode request data.
        int savedWriteIndex = buffer.writerIndex();
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        if (req.isEvent()) {
            encodeEventData(channel, out, req.getData());
        } else {
            encodeRequestData(channel, out, req.getData());
        }
        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }
        bos.flush();
        bos.close();
        int len = bos.writtenBytes();
        checkPayload(channel, len);
        Bytes.int2bytes(len, header, 12);

        // write
        buffer.writerIndex(savedWriteIndex);
        buffer.writeBytes(header); // write header.
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
```

这段代码对应的是前面那张图，Dubbo自己扩展了16字节头部。根据上述代码，可以看到，魔数占用2字节，第三个字节，存储FLAG_REQUEST与序列化器id（如Hessian2Serialization类定义的ID=2，DubboSerialization的id为1，源码可查）以及是FLAG_TWOWAY（双向）还是FLAG_EVENT（单向）。在请求端编码时，未使用第四字节。第5-12共8个字节，用于存储异步变同步的全局唯一ID。第13-16个字节为消息体总长度（消息头+请求数据）。

```
dubbo的消息头是一个定长的 16个字节数据。
第1-2个字节：魔数, 一个固定的数字 
第3个字节：是双向(有去有回) 或单向（有去无回）的标记 
第四个字节：？？？ （request 没有第四个字节）
第5-12个字节：请求id：long型8个字节。异步变同步的全局唯一ID，用来做consumer和provider的来回通信标记。
第13-16个字节：消息体的长度，也就是消息头+请求数据的长度。
```

### Provider接收数据解码

provider端运行dubbo-demo-provider工程，在dubbo-remoting-netty工程NettyCodecAdapter.java内部类InternalDecoder#messageReceived()断点msg = codec.decode(channel, message)：

```java
--NettyCodecAdapter.InternalDecoder.messageReceived
  -->DubboCountCodec.decode
    -->ExchangeCodec.decode
      -->ExchangeCodec.decodeBody
```

### Provider发送响应结果编码

Provider接收到Consumer请求调用后，将返回的数据进行编码发送给consumer.

```java
-->NettyCodecAdapter.InternalEncoder.encode
  -->DubboCountCodec.encode
    -->ExchangeCodec.encode
      -->ExchangeCodec.encodeResponse
        -->DubboCodec.encodeResponseData//先写入一个字节 这个字节可能是RESPONSE_NULL_VALUE:2 RESPONSE_VALUE:1  RESPONSE_WITH_EXCEPTION: 0
        正常返回：out.writeByte(RESPONSE_VALUE);  out.writeObject(ret);
```

编码的过程大体都是一样的，但是响应端的编码稍有差别：

```java
protected void encodeResponse(Channel channel, ChannelBuffer buffer, Response res) throws IOException {
        int savedWriteIndex = buffer.writerIndex();
        try {
            Serialization serialization = getSerialization(channel);
            // header.
            byte[] header = new byte[HEADER_LENGTH];
            // set magic number.
            Bytes.short2bytes(MAGIC, header);
            // set request and serialization flag.
            header[2] = serialization.getContentTypeId();
            if (res.isHeartbeat()) header[2] |= FLAG_EVENT;
            // set response status.
            byte status = res.getStatus();
            header[3] = status;
            // set request id.
            Bytes.long2bytes(res.getId(), header, 4);

            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
            ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
            ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
            // encode response data or error message.
            if (status == Response.OK) {
                if (res.isHeartbeat()) {
                    encodeHeartbeatData(channel, out, res.getResult());
                } else {
                    encodeResponseData(channel, out, res.getResult());
                }
            } else out.writeUTF(res.getErrorMessage());
            out.flushBuffer();
            if (out instanceof Cleanable) {
                ((Cleanable) out).cleanup();
            }
            bos.flush();
            bos.close();

            int len = bos.writtenBytes();
            checkPayload(channel, len);
            Bytes.int2bytes(len, header, 12);
            // write
            buffer.writerIndex(savedWriteIndex);
            buffer.writeBytes(header); // write header.
            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
        } catch (Throwable t) {
            // clear buffer
            buffer.writerIndex(savedWriteIndex);
            // send error message to Consumer, otherwise, Consumer will wait till timeout.
            if (!res.isEvent() && res.getStatus() != Response.BAD_RESPONSE) {
                Response r = new Response(res.getId(), res.getVersion());
                r.setStatus(Response.BAD_RESPONSE);

                if (t instanceof ExceedPayloadLimitException) {
                    logger.warn(t.getMessage(), t);
                    try {
                        r.setErrorMessage(t.getMessage());
                        channel.send(r);
                        return;
                    } catch (RemotingException e) {
                        logger.warn("Failed to send bad_response info back: " + t.getMessage() + ", cause: " + e.getMessage(), e);
                    }
                } else {
                    // FIXME log error message in Codec and handle in caught() of IoHanndler?
                    logger.warn("Fail to encode response: " + res + ", send bad_response info instead, cause: " + t.getMessage(), t);
                    try {
                        r.setErrorMessage("Failed to send response: " + res + ", cause: " + StringUtils.toString(t));
                        channel.send(r);
                        return;
                    } catch (RemotingException e) {
                        logger.warn("Failed to send bad_response info back: " + res + ", cause: " + e.getMessage(), e);
                    }
                }
            }

            // Rethrow exception
            if (t instanceof IOException) {
                throw (IOException) t;
            } else if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else if (t instanceof Error) {
                throw (Error) t;
            } else {
                throw new RuntimeException(t.getMessage(), t);
            }
        }
    }
```

dubbo的消息头是一个定长的 16个字节数据。
第1-2个字节：魔数, 一个固定的数字 
第3个字节：序列化组件类型，它用于和客户端约定的序列化器ID
第四个字节：它是response的结果响应码 OK=20
第5-12个字节：请求id：long型8个字节。异步变同步的全局唯一ID，用来做consumer和provider的来回通信标记。
第13-16个字节：消息体的长度，也就是消息头+请求数据的长度。

### Consumer接收响应结果解码

Consumer接收到Provider的响应结果后，将数据进行反序列化，结束调用。

```java
--NettyCodecAdapter.InternalDecoder.messageReceived
  -->DubboCountCodec.decode
    -->ExchangeCodec.decode
      -->DubboCodec.decodeBody
        -->DecodeableRpcResult.decode//根据RESPONSE_NULL_VALUE  RESPONSE_VALUE  RESPONSE_WITH_EXCEPTION进行响应的处理
```

