# 由Android-65K方法数限制引发的思考

来源:[由Android-65K方法数限制引发的思考](http://jayfeng.com/2016/03/10/由Android-65K方法数限制引发的思考/)

## 前言

没想到，65536真的很小。

> Unable to execute dex: method ID not in [0, 0xffff]: 65536

**PS:**本文只是纯探索一下这个65K的来源，仅此而已。

## 到底是65k还是64k?

都没错，同一个问题，不同的说法而已。<br/>
65536按1000算的话，是65k ~ 65 1000;<br/>
65536按1024算的话，是64k = 64 1024。<br/>
重点是65536=2^16，请大家记住这个数字。

## 时间点

从大家的经历和这篇文章：[http://developer.android.com/tools/building/multidex.html](http://developer.android.com/tools/building/multidex.html)来看，**这个错误是发生在构建时期**。

## 65536是怎么算出来的？

65536网上众说纷纭，有对的，有不全对的，也有错的。

下面将跟踪最新的AOSP源码来顺藤摸瓜，但是探索问题必然迂回冗余，仅作记录，读者可直接跳过看结果。

### 1. 首先，查找Dex的结构定义。

```
/*
 * Direct-mapped "header_item" struct.
 */
struct DexHeader {
    u1  magic[8];
    u4  checksum;
    u1  signature[kSHA1DigestLen];
    u4  fileSize;
    u4  headerSize;
    u4  endianTag;
    u4  linkSize;
    u4  linkOff;
    u4  mapOff;
    u4  stringIdsSize;
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize; // 这里存放了方法字段索引的大小，methodIdsSize的类型为u4
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};
```

u4的类型定义如下：

```
/*
 * These match the definitions in the VM specification.
 */
typedef uint8_t             u1;
typedef uint16_t            u2;
typedef uint32_t            u4;
typedef uint64_t            u8;
typedef int8_t              s1;
typedef int16_t             s2;
typedef int32_t             s4;
typedef int64_t             s8;
```

进一步推出，methodIdsSize的类型是uint32_t，但它的限制为2^32 = 65536 * 65536，比65536大的多。

所以，**65k不是dex文件结构本身限制造成的**。

**PS：**Dex文件中存储方法ID用的并不是short类型，无论最新的DexFile.h新定义的u4是uint32_t，还是老版本DexFile引用的vm/Common.h里定义的u4是uint32或者unsigned int，都不是short类型，特此说明。

### 2. DexOpt优化造成？

这个说法源自：

> 当Android系统启动一个应用的时候，有一步是对Dex进行优化，这个过程有一个专门的工具来处理，叫DexOpt。DexOpt的执行过程是在第一次加载Dex文件的时候执行的。这个过程会生成一个ODEX文件，即Optimised Dex。执行ODex的效率会比直接执行Dex文件的效率要高很多。但是在早期的Android系统中，DexOpt有一个问题，也就是这篇文章想要说明并解决的问题。DexOpt会把每一个类的方法id检索起来，存在一个链表结构里面。但是这个链表的长度是用一个short类型来保存的，导致了方法id的数目不能够超过65536个。当一个项目足够大的时候，显然这个方法数的上限是不够的。尽管在新版本的Android系统中，DexOpt修复了这个问题，但是我们仍然需要对老系统做兼容。

鉴于我能力有限，没有找到这块逻辑对应的代码。

但我有个疑问，这个限制是在Android启动一个应用的时候发生的，但从前面的“时间点”章节，65k问题是在构建的时候就发生了，还没到启动或者运行这一步。

我不敢否定这种说法，但说明65k至少还有其他地方限制。

### 3. DexMerger的检测

只能在dalvik目录下搜索关键字”methid ID not in”，在DexMergger里找到了抛出异常的地方：

```
/**
 * Combine two dex files into one.
  */
public final class DexMerger {
    private void mergeMethodIds() {
        new IdMerger<MethodId>(idsDefsOut) {
            @Override 
            TableOfContents.Section getSection(
            							TableOfContents tableOfContents) {
                return tableOfContents.methodIds;
            }
            @Override 
            MethodId read(Dex.Section in, IndexMap indexMap, int index) {
                return indexMap.adjust(in.readMethodId());
            }
            @Override 
            void updateIndex(int offset, IndexMap indexMap, 
            							int oldIndex, int newIndex) {
                if (newIndex < 0 || newIndex > 0xffff) {
                    throw new DexIndexOverflowException(
                            "method ID not in [0, 0xffff]: " + newIndex);
                }
                indexMap.methodIds[oldIndex] = (short) newIndex;
            }
            @Override 
            void write(MethodId methodId) {
                methodId.writeTo(idsDefsOut);
            }
        }.mergeSorted();
    }
}
```

这里定义了indexMap的methodIds的单项值要强转short，所以在存放之前check一下范围是不是0 ~ 0xffff。

我们看看IndexMap的定义：

```
/**
 * Maps the index offsets from one dex file to those in another. For example, if
 * you have string #5 in the old dex file, its position in the new dex file is
 * {@code strings[5]}.
 */
public final class IndexMap {
    private final Dex target;
    public final int[] stringIds;
    public final short[] typeIds;
    public final short[] protoIds;
    public final short[] fieldIds;
    public final short[] methodIds;
    // ... ...
}
```

看上去是对了，可是这个DexMerger是合并两个dex的，默认情况下我们只有一个dex的，那么这个65k是哪里限制的呢？再查！

### 4. 回归DexFile

基本上前面基本是一个摸着石头过河、反复验证网络说法的一个过程，虽然回想起来傻傻的，但是这种记录还是有必要的。

前面看到DexFile的存放方法数大小的类型是uint32，但是根据后面的判断，我们确定是打包的过程中产生了65k问题，所以我们得回过头老老实实研究一下dx的打包流程。

… 此处省略分析流程5000字 …

OK，我把dx打包涉及到流程记录下来：

> // 源码目录：dalvik/dx<br/>
> // Main.java<br/>
> -> main() -> run() -> runMonoDex()(或者runMultiDex()) -> writeDex()<br/>
> // DexFile<br/>
> -> toDex() -> toDex0()<br/>
> // MethodIdsSection extends MemberIdsSection extends UniformItemSection extends  Section<br/>
> -> prepare() -> prepare0() -> orderItems() -> getTooManyMembersMessage()<br/>
> // Main.java<br/>
> -> getTooManyIdsErrorMessage()<br/>

最终狐狸的尾巴是在MemberIdsSection漏出来了：

```
package com.android.dx.dex.file;
import com.android.dex.DexException;
import com.android.dex.DexFormat;
import com.android.dex.DexIndexOverflowException;
import com.android.dx.command.dexer.Main;
import java.util.Formatter;
import java.util.Map;
import java.util.TreeMap;
import java.util.concurrent.atomic.AtomicInteger;
/**
 * Member (field or method) refs list section of a {@code .dex} file.
 */
public abstract class MemberIdsSection extends UniformItemSection {
    /**
     * Constructs an instance. The file offset is initially unknown.
     *
     * @param name {@code null-ok;} the name of this instance, for annotation
     * purposes
     * @param file {@code non-null;} file that this instance is part of
     */
    public MemberIdsSection(String name, DexFile file) {
        super(name, file, 4);
    }
    /** {@inheritDoc} */
    @Override
        protected void orderItems() {
            int idx = 0;
            if (items().size() > DexFormat.MAX_MEMBER_IDX + 1) {
                throw new DexIndexOverflowException(getTooManyMembersMessage());
            }
            for (Object i : items()) {
                ((MemberIdItem) i).setIndex(idx);
                idx++;
            }
        }
    private String getTooManyMembersMessage() {
        Map<String, AtomicInteger> membersByPackage = 
        				new TreeMap<String, AtomicInteger>();
        for (Object member : items()) {
            String packageName = ((MemberIdItem) member)
            						.getDefiningClass().getPackageName();
            AtomicInteger count = membersByPackage.get(packageName);
            if (count == null) {
                count = new AtomicInteger();
                membersByPackage.put(packageName, count);
            }
            count.incrementAndGet();
        }
        Formatter formatter = new Formatter();
        try {
            String memberType = this instanceof MethodIdsSection 
            					? "method" : "field";
            formatter.format("Too many %s references: %d; max is %d.%n" +
                    Main.getTooManyIdsErrorMessage() + "%n" +
                    "References by package:",
                    memberType, items().size(), DexFormat.MAX_MEMBER_IDX + 1);
            for (Map.Entry<String, AtomicInteger> entry : 
            							membersByPackage.entrySet()) {
                formatter.format("%n%6d %s", 
                						entry.getValue().get(), entry.getKey());
            }
            return formatter.toString();
        } finally {
            formatter.close();
        }
    }
}
```

里面有一段：

```
// 如果方法数大于0xffff就提示65k错误
if (items().size() > DexFormat.MAX_MEMBER_IDX + 1) {
    throw new DexIndexOverflowException(getTooManyMembersMessage());
}
// 这个DexFormat.MAX_MEMBER_IDX就是0xFFFF
/**
 * Maximum addressable field or method index.
 * The largest addressable member is 0xffff, 
 * 			in the "instruction formats" spec as field@CCCC or
 * meth@CCCC.
 */
public static final int MAX_MEMBER_IDX = 0xFFFF;
```

至此，真相大白！

### 5. 根本原因

为什么定义DexFormat.MAX_MEMBER_IDX为0xFFFF?

虽然我们找到了65k报错的地方，但是为什么程序中方法数超过0xFFFF就要报错呢？

通过搜索”instruction formats”, 我最终查到了Dalvik VM Bytecode，找到最新的官方说明：

[https://source.android.com/devices/tech/dalvik/dalvik-bytecode.html](https://source.android.com/devices/tech/dalvik/dalvik-bytecode.html)

里面说明了上面的@CCCC的范围必须在0～65535之间，这是dalvik bytecode的限制。

所以，65536是bytecode的16位限制算出来的：2^16。

**PS：**以上分析得到群里很多朋友的讨论和帮忙。

### 6. 回顾

我好像明白了什么：

> 1. 65k问题是dx打包单个Dex时报的错，所以只要用dx打包单个dex就可能有这个问题。
> 2. 不仅方法数，字段数也有65k问题。
> 3. 目前来说，65k问题和系统无关。
> 4. 目前来说，65k问题和art无关。
> 5. 即使分包MultiDex，当主Dex的方法数超过65k依然会报错。
> 6. MultiDex方案不是从根本上解决了65k问题，但是大大缓解甚至说基本解决了65k问题。

新的Jack能否解决65k问题？

> Jack (Java Android Compiler Kit) is a new Android toolchain that comprises a compiler from Java programming language source to the Android dex file format.

![](4/1.gif)

新的编译器目前还不了解实现的细节，网上的资料是说解决了65k问题，但看了最新的图，我觉得并不能终结65k问题，暂无法评论。

[Experimental New Android Tool Chain - Jack and Jill](http://tools.android.com/tech-docs/jackandjill)

## 小结

了解65k问题不会对工作有什么帮助，工作偶遇，略做梳理，做一总结！


## 评论

* 赞。关于DexOpt这个说法，我可以补充说明下：DexOpt的失败的原因是LinearAlloc的内存太小，即使方法数没有到65536，也可能会导致DexOpt失败。而且LinearAlloc太小不仅仅是影响到DexOpt，还会影响到运行期即使做了分dex。具体说明见[https://github.com/simpleton/dalvik_patch](https://github.com/simpleton/dalvik_patch)

* 手机不方便贴链接.. 关于整个jack and jill 的编译流程 [http://tools.android.com/tech-docs/jackandjill](http://tools.android.com/tech-docs/jackandjill) 2014年推出的....这次N的高调宣布..应该是差不多能用了..