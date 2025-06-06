---
layout: post
title:  "Netty Self-taught Diary (2)"
date:   2024-02-05 16:53:55 +0800
categories: Demo
tags: Tech-Share
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### ByteBuffer(Java)/ByteBuf(Netty)
先來介紹數據類型ByteBuf，這個ByteBuf類是Netty根據Java的ByteBuffer所演化而來的，兩者都是爲了處理數據而產生。因爲在ChannelHandler中所接收到的數據只能保證順序而沒辦法保證接收到的段落。例如Client中按順序發送: "Hello", "World"；但是到了Server端的ChannelHandler可能會被分成"He","lloW","orld"，因此需要設計一種數據類型來處理這類情況。前面所述的情況是因爲傳輸協議本身沒辦法保證，而并非是受到Netty框架所影響。

在介紹這個數據類之前還要先知道在Java NIO中是如何設計的，這樣方便做直觀的比較而且兩者也有相似之處。`java.nio.bytebuffer` 是 `java.nio.buffer` 的子類，而ByteBuffer中分爲兩種Buffer: `Direct Buffer` 和 `None-Direct Buffer`。就如同名字所述，direct buffer被分配在heap之外，能直接被程序訪問，所以速度較none-direct快；而none-direct buffer被存放在heap中，也就是大部分程序訪問數據的流程，而通常Java會為這個buffer再額外分配一個array用來備份數據，這個array被稱爲 `Backing Array`。因此，可以在程序中透過 `hasArray()` 來檢查這個buffer是否擁有backing array，若是有就代表這個buffer屬於none-direct(heap buffer)，反之為direct buffer: 

| Direct Buffer | None-Direct buffer |
| :---- | :---- |
| 位於heap外，可以直接被stack中的java程序調用 | 位於heap内 |
| 訪問速度快 | 訪問速度慢 |
| 要釋放較麻煩 | 釋放容易 |
| 沒有backing array | 有backing array |
| 使用 `allowcateDirect()` 創建 | 先 `allocate()` 創建buffer，然後在手動創建backing array並把它與buffer綁定（調用 `wrap()`）|

讓我們把視角回到Netty的ByteBuf類型，它包含了三種格式，除了前面提到的HeapBuffer和DirectBuffer，Netty還設計了CompositeBuffer，它表示多個ByteBuf的組合類，方便同時處理和傳遞多個ByteBuf；同時若要讀取CompositeByteBuf，可以把它視爲DirectBytebuf，因爲在CompositeByteBuf中并沒有Backing Array。下面就是使用CompositeByteBuf組合兩個ByteBuf和讀取其中數據的範例，非常直觀的描述前面提到的觀念:

```
// 創建CompositeByteBuf
CompositeByteBuf messageBuf = Unpooled.compositeBuffer();
ByteBuf buf_1 = ...; // can be backing or direct
ByteBuf buf_2 = ...; // can be backing or direct
messageBuf.addComponents(buf_1, buf_2);

// 讀取CompositeByteBuf后針對array處理即可
int length = compBuf.readableBytes();
byte[] array = new byte[length];
compBuf.getBytes(compBuf.readerIndex(), array);
```

介紹完不同種類的Buffer后，讓我們把是視角拉回Buffer内部。若是要訪問Buffer的數據，Netty又與原生Java做出區別，這邊爲了簡化流程故而用表格做對比，相信也會更加直觀的看出差異，由於ByteBuffer設計不同，故而使用方法也差距甚大，這邊先不琢磨細節，等到後續使用時再詳細説明：

| Java NIO | Netty |
| :---- | :---- |
| ByteBuffer擁有四個基礎屬性: `capacity/limit/position/mark`。`capacity` 表示這個buffer定義大小，一但聲明了便不可更改；`limit` 表示不應該被讀取/寫入資料的位置，例如 [0,limit) 可以被讀取，而 [limit, capacity] 則棄用；`position` 表示目前讀取/寫入的指針；`mark`被用來保存當前的`position`屬性用於後續的重置| 擁有兩個主要屬性: `writerIndex` 和 `readerIndex` 用於分別表示在讀寫中的指針|
| 由於只有一個`position`記錄當下位置，因此需要使用`flip()/clear()`切換讀寫模式，還需要注意`limit`和`position`之間的關係，非常複雜| 兩個指針分別記錄讀寫模式下的位置，避免許多繁雜的操作以及不必要的問題|

### How to read/write ByteBuf
```
+-------------------+------------------+------------------+
| discardable bytes |  readable bytes  |  writable bytes  |
+-------------------+------------------+------------------+
|                   |                  |                  |
0    <=      readerIndex   <=   writerIndex    <=    capacity
```

前面提到在ByteBuf中有兩個基礎屬性`readerIndex`和`writerIndex`，分別指向當前讀指針和寫指針。先來介紹讀取部分，Netty提供兩種主要讀取的方法: `read` 和 `get`；前者當你讀取時指針會跟著變動，後者則不會，這樣做的目的是當你只是用於判斷式且不想讓指針移動時便可以使用`get`來防止下次讀取時指針跑掉。另外，Netty提供很多類型的read和get方法，下面就以`ByteBuf.getByte`做介紹：
```
getByte(int i)
getBytes(int i, ByteBuf dst)
getBytes(int i, ByteBuf dst, int len)
getBytes(int i, ByteBuf dst, int dstIndex, int len)
```

- 我們假設調用方法的為 **ByteBuf target**
- `getByte(int i)`: 最直接將數據從target讀出, readerIndex不變
- `getBytes(int i, ByteBuf dst)`: 將數據從target讀出並存入到dst中，target的readerIndex不變但dst的writerIndex會變
- `getBytes(int i, ByteBuf dst, int len)`: 將長度為len的數據從target讀出並存入到dst中，target的readerIndex不變但dst的writerIndex會變
- `getBytes(int i, ByteBuf dst, int dstIndex, int len)`: 將長度為len的數據從target讀出並從dst的dstIndex位置開始存放，target的readerIndex不變且dst的writerIndex也不變

相反的，若是使用 `read` 方法，則`readerIndex` 會改變並影響到下次讀取的位置，這邊便不再贅述，可以參考 [Netty 4.x API](https://netty.io/4.1/api/index.html)。同理，寫入部分Netty提供的是 `set` 和 `write` 兩種類型，前者寫入時 `writerIndex` 不變，後者則會隨著寫入長度改變

## 參考資料
1. [Netty in Action](https://github.com/zuzeep/book/blob/master/Netty%20in%20Action.pdf)
2. [Java NIO - Buffer Basic Concepts](https://medium.com/@clu1022/java-nio-buffer-c98b52fd93ca)
3. [Netty 4.x API](https://netty.io/4.1/api/index.html)

</div>
</body>
</html>
