
**大纲**


**1\.为什么不能直接更新磁盘上的数据**


**2\.为什么要引入数据页的概念**


**3\.一行数据在磁盘上是如何存储的**


**4\.一行数据中的NULL值是如何处理的**


**5\.一行数据的数据头存储的是什么**


**6\.一行数据的真实数据如何存储**


**7\.数据在物理存储时的行溢出和溢出页**


**8\.数据页的物理存储结构**


**9\.表空间的物理存储结构**


**10\.InnoDB存储模型及读写机制总结**


 


**前面介绍了MySQL的数据缓存机制和内存数据更新机制，接下来介绍MySQL的表空间、数据区、数据页等磁盘上的物理文件**


 


**1\.为什么不能直接更新磁盘上的数据**


为何MySQL要设计一套复杂的数据存取机制，即基于内存、日志、磁盘上的数据文件来完成数据读写？对于增改请求为何不直接更新磁盘文件？因为来一个请求就直接对磁盘文件进行随机读写，然后更新磁盘文件里的数据，这会导致执行请求的性能极差。


 


我们知道磁盘的随机读写性能是很差的，所以直接更新磁盘文件必然导致数据库完全无法抗下稍微高并发场景。于是MySQL才设计了一套数据缓存机制和内存数据更新机制。首先通过内存更新数据，然后写redo日志以及提交事务，最后通过后台线程不定时刷新内存里的数据到磁盘文件。通过这种方式可保证每个更新请求都更新内存，然后顺序写日志文件。


 


由于更新内存的性能是极高的，而且往磁盘文件顺序写日志的性能也比较高(顺序写的性能远高于随机写)，所以这套机制可让MySQL在16核32G的机器上抗下每秒几千的读写请求。


 


**2\.为什么要引入数据页的概念**


当MySQL要更新数据时，并不是直接去更新磁盘文件的。而是首先把磁盘上的一些数据加载到内存里，然后对内存的数据进行更新，同时写redo日志。


 


但MySQL并非每次都把磁盘里的一条数据加载到内存里去进行更新。然后下次要更新别的数据时，再从磁盘里加载另外一条数据到内存里。因为这样每次一条一条的数据加载到内存进行更新，效率就太低了。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4365e990fd7442fcab19a3b3e3a81e66~tplv-obj.image?lk3s=ef143cfe&traceid=202411241939060C3C4F138384E73CBF95&x-expires=2147483647&x-signature=gFiWw1J5hJp1tbCjCkq55MFdxo4%3D)
所以InnoDB存储引擎引入了数据页的概念。具体就是把数据组织成一页一页的结构，然后每一页有16KB。这样每次加载磁盘的数据到内存时，至少加载一页数据，甚至会通过预读机制加载多页数据。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/520acb1f314843d597e3b6345dfa70f8~tplv-obj.image?lk3s=ef143cfe&traceid=202411241939060C3C4F138384E73CBF95&x-expires=2147483647&x-signature=eA25dpCOQfoqQJsXjhGD9s9dmwM%3D)
假设要更新一条id\=1的数据：



```
update xxx set xxx=xxx where id = 1;
```

那么此时MySQL会把id\=1这条数据所在的一页数据都加载到内存里，这一页数据可能还包含了id\=2等其他数据。然后当MySQL更新完id\=1的数据后，如果接着需要更新id\=2的数据时，就不用再次读取磁盘里的数据了。


 


这就是数据页的意义。磁盘和内存之间的数据交换通过数据页来执行，包括内存里更新后的脏数据被刷回磁盘时，也是以数据页为单位进行。


 


**3\.一行数据在磁盘上是如何存储的**


**(1\)行格式**


**(2\)变长字段在磁盘中是怎么存储的**


**(3\)引入变长字段列表后，如何读取变长字段**


**(4\)如果有多个变长字段，如何存放它们的长度**


 


**(1\)行格式**


我们创建表的时候可以指定表的行使用什么样的存储格式。比如下面的语句就指定使用compact格式进行存储。可以在建表的时候指定一个行存储的格式，也可以后续修改行存储的格式。



```
create table table_name (columns) row_format=compact
alter table table_nale row_format=compact
```

在compact行存储格式下，每一行实际存储时，格式如下。除了每个字段的值以外，还包含一些额外的信息。这些额外的信息就是用来描述这一行数据的。



```
变长字段长度列表,null值列表,数据头,column01的值,column02的值...column0n的值...
```

**(2\)变长字段在磁盘中是怎么存储的**


假设有两行数据，它的几个字段类型为varchar(10\),char(1\),char(1\)。第一个字段是varchar(10\)，其长度是可变的。第一行数据可能类似"hello a a"，第一个字段是hello，后两个字段是a。第二行数据可能类似"hi a a"，第一个字段是hi，后两个字段也是a。


 


这时如果要把这两行数据写入磁盘文件，且要求这两行数据要挨在一起。那么磁盘文件里可能就有类似这样的数据："hello a a hi a a"。其实我们平时看到表里的多行数据，落地到磁盘时，都是按上面的方式紧挨着存储的。


 


现在问题来了，假设要从磁盘文件读取上面形式的数据。也就是把"hello a a"这一行数据读取出来，那就比较困难了。因为没有标记也没有分隔符。所以MySQL引入了变长字段的长度列表，来解决标记和分隔符的问题。


 


也就是说，在存储"hello a a"这行数据时，要带上一些额外的附加信息。比如第一个就是这一行数据里的变长字段的长度列表。由于只有这个"hello"是varchar(10\)类型的变长字段的值，而且它的长度是5，对应的十六进制是0x05，所以这个0x05就是这一行数据对应的变长字段的长度列表。


 


因此，需要在"hello a a"前面补充如下的额外信息："0x05 null值列表 数据头 hello a a"。于是上面挨着的两行数据放在一起存储在磁盘文件里的格式就类似为："0x05 null值列表 数据头 hello a a 0x02 null值列表 数据头 hi a a"。


 


**(3\)引入变长字段列表后，如何读取变长字段**


现在假设要读取"hello a a"这行数据。


 


**一.首先要确定表的字段类型**


MySQL可知表里的三个字段的类型是varchar(10\) char(1\) char(1\)。


 


**二.然后读取第一个字段的值**


由于第一个字段是变长的，所以从磁盘读出数据时，会从开头的变长字段的长度列表读到一个0x05的十六进制的数字。由此可知第一个变长字段的长度是5，于是便可以按照长度5去读取出第一个字段的值"hello"。


 


**三.接着读取后面两个字段的值**


由于后续两个字段都是char(1\)，长度都是固定的1个字符。于是就依次按照长度为1读取出后续两个字段的值，分别是"a"和"a"。这样就把该行数据"hello a a"读取出来了。


 


如果要读取第二行数据，也是先看一下第二行数据的变长字段列表。发现第一个变长字段的长度是0x02，于是读取长度为2的字段值，即"hi"。然后再读取两个长度固定为1的字符值，都是"a"。这样也把该行数据"hi a a"读出来了。


 


**(4\)如果有多个变长字段，如何存放它们的长度**


比如一行数据有5个字段：varchar(10\) varchar(5\) varchar(20\) char(1\) char(1\)，其中3个是变长字段。假设这一行数据是这样的："hello hi hao a a"。那么这一行数据会如何在磁盘中存储？


 


首先会在数据开头的变长字段长度列表中存储几个变长字段的长度。需要注意的是，变长字段的长度是逆序存储的。也就是先存放varchar(20\)字段的长度，然后再存放varchar(5\)字段的长度，最后才存放varchar(10\)字段的长度。所以这一行数据实际存储在磁盘文件是长这样的："0x03 0x02 0x03 null值列表 头字段 hello hi hao a a"。


 


**4\.一行数据中的NULL值如何处理**


**(1\)为什么一行数据里的NULL值不能直接存储**


**(2\)NULL值是以二进制bit位来存储的**


**(3\)磁盘上的一行数据会怎么读出来**


 


**(1\)为什么一行数据里的NULL值不能直接存储**


磁盘上存储的每一行数据里除了有变长字段的长度列表外，还有另外一块特殊的数据区域，就是NULL值列表。


 


一行数据中有些字段值是NULL，NULL表示的是什么值都没有。如果在磁盘上存储时按"NULL"字符串存储，这样就太浪费空间了。


 


**(2\)NULL值是以二进制bit位来存储的**


NULL值在磁盘上并不是通过字符串来存储的，而是通过bit位来存储的。具体来说，假设一行数据里有多个值是NULL。那么这多个NULL值，会以bit位的形式存放在NULL值列表中。


 


举个例子，有一张表如下：有5个字段，其中4个变长字段、1个定长字段，name字段不能为NULL。



```
create table customer (
    name varchar(10) not null,
    address varchar(20),
    gender char(1),
    job varchar(30),
    school varchar(50)
) row_format=compact;
```

在customer表里有一行数据："jack NULL m NULL xx\_school"。现在已经知道一行数据在磁盘上的compact存储格式为：



```
变长字段的长度列表 null值列表 数据头 column01的值 column02的值 ... column0n的值
```

**一.先看变长字段长度列表应该存放的内容**


由于多个变长字段按照逆序存放，所以会先放school字段的长度，再放job字段、address字段、name字段的长度。


 


**二.如果某变长字段的值是NULL**


那么就不用在变长字段长度列表里存放这个变长字段的值长度。所以只有name和school两个变长字段有值。即需要把name和school的长度按照逆序放在变长字段长度列表中：



```
0x09 0x04 NULL值列表 数据头 column1=value1 column2=value2 ... columnN=valueN
```

**三.对于所有允许值为NULL的字段，每个字段都有一个二进制bit位的值**


如果bit值是1，则说明该字段是NULL；如果bit值是0，则说明该字段不是NULL。


 


**四.上面的表4个字段都允许为NULL，所以每个字段都有一个bit位**


对于数据"jack NULL m NULL xx\_school"来说，4个bit位应该就是1010。


 


**五.存放NULL值列表和存放变长字段的长度列表一样，也按逆序存放**


所以4个bit位是0101，最后这一行数据是：



```
0x09 0x04 0101 数据头 column1=value1 column2=value2 ... columnN=valueN
```

**六.实际存放NULL值列表时按照8个bit位的整数倍来存放**


不会按4个bit位来存放，如果不足8个bit位则进行高位补0。所以最后的数据是：



```
0x09 0x04 00000101 数据头 column1=value1 column2=value2 ... columnN=valueN
```

**(3\)磁盘上的一行数据会怎么读出来**


对于读取磁盘上存储的数据：



```
0x09 0x04 00000101 数据头 column1=value1 column2=value2 ... columnN=valueN
```

首先要把变长字段长度列表和NULL值列表读出来。通过分析可知有多少个变长字段，哪些字段是NULL。如果NULL值列表的某个bit位是1，则说明其对应的字段值是NULL。对于变长字段的值，就按变长字段长度列表来获得长度再根据长度去读取。对于固定长度字段的值，则直接按固定长度进行读取。


 


**5\.一行数据的数据头存储的是什么**


在磁盘上存储数据时：


**一.每一行数据都会有变长字段的长度列表**


变长字段的长度列表，会通过逆序存放这一行数据里的变长字段的长度。


**二.每一行数据都可能会有NULL值列表**


允许为NULL的字段会有一个bit位标识该字段是否为NULL且逆序存放。


**三.每一行数据还会有40个bit位的数据头**


这个数据头是用来描述这行数据的


 


这40个bit位里面：


第一个和第二个bit位，都是预留位，没有含义；


第三个bit位是delete\_mask：标识这行数据是否被删除。删除一行数据时不会马上把数据从磁盘上清理，而会在数据头进行标记；


第四个bit位是min\_rec\_mask：标记B\+树里每一层的非叶子节点里的最小值；


接着有4个bit位的n\_owned：这是一个记录数；


接着有13个bit位的heap\_no：代表当前这行数据在记录堆里的记录；


接着有3个bit位的record\_type：指的是这行数据的类型。0代表普通类型，1代表B\+树非叶子节点，2代表最小值，3代表最大值；


最后有16个bit位的next\_record：指向该行数据下一条数据的指针；


 


**6\.一行数据的真实数据如何存储**


现已知一行数据在磁盘文件中存储时：首先会包含自己的变长字段的长度列表，然后是NULL值列表，接着是数据头，最后才是真实数据。


 


那么存储真实数据时，是怎么处理的呢？比如一行数据是"jack NULL m NULL xx\_school"。那么它的真实存储大概如下：



```
0x09 0x04 00000101 00000000000000000000100000000000000011001 jack m xx_school
```

一开始是变长字段的长度，使用了十六进制来存储。然后是NULL值列表，指出了允许NULL的字段谁是NULL。接着是40个bit位的数据头，最后是真实数据值放在后面。


 


读取时先读取第一个字段。根据变长字段的长度列表知道长度是4，先读取出jack值。然后发现第二个字段是NULL，不用读取。接着第三个字段是定长字段，直接读取1个字符即可，也就是m值。接着第四个字段是NULL，也不用读取。第五个字段是变长字段。根据变长字段的长度列表知道长度是9，读取出xx\_school。


 


但在磁盘上存储时，真实数据并不是以字符串的形式存储在磁盘上的。而是根据数据库指定的字符集编码，对字符串进行编码之后再存储。所以一行数据最终会如下所示进行存储：



```
0x09 0x04 00000101 00000000000000000000100000000000000011001 616161 636320 62626262
```

此外在实际存储一行数据时，还会在真实数据部分，加入一些隐藏字段。


**一.首先有一个DB\_ROW\_ID字段**


这是该行的唯一标识，是数据库针对该行设定的一个标识，不是主键ID。当没有给表指定主键和唯一索引时，该表会自动加一个ROW\_ID作为主键。


**二.接着有一个DB\_TRX\_ID字段**


这是事务ID，代表着这是哪个事务更新的数据。


**三.最后有个DB\_ROLL\_PTR字段**


这是回滚指针，用来进行事务回滚。


 


所以加上隐藏字段后，一行数据可能看起来如下：



```
0x09 0x04 00000101 00000000000000000000100000000000000011001 00000000094C (DB_ROW_ID) 000000000032D (DB_TRX_ID) EA0000010078E (DB_ROL_PTR) 616161 636320 62626262
```

 


**7\.数据在物理存储时的行溢出和溢出页**


每一行的数据都是放在一个数据页里的，这个数据页默认的大小是16KB。但如果一行数据的大小超过了数据页的大小时怎么办？


 


比如有一个表的字段类型是varchar(65532\)，意思就是最大可以包含65532个字符(65532字节)，远大于16KB。


 


这时MySQL是这样处理的：首先会在一个数据页里尽量存储这一行所有字段数据，然后在特别长的字段中会仅仅包含一部分数据，同时包含一个20字节的指针指向其他数据页，那些数据页会用链表串联起来，存放这个特别长的字段的数据。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/cc3a38cf5f65421da76321fe4a2f3d18~tplv-obj.image?lk3s=ef143cfe&traceid=202411241939060C3C4F138384E73CBF95&x-expires=2147483647&x-signature=ASUhwhoBwcSBOLwNVEDsYxzPYcY%3D)
行溢出指的是：一行数据存储的内容太多，一个数据页放不下时，只能溢出这个数据页。把数据存放到其他数据页里，那些其他数据页就叫做溢出页。除了varchar(65532\)这种字段，其他如text、blob字段，都可能出现溢出，然后一行数据会存储在多个数据页里。


 


总结：当往数据库插入一行数据时，实际上是在内存里插入一个有复杂存储结构的一行数据。然后随着一些条件的发生，这行数据会被刷到磁盘文件里去。在磁盘文件里存储时，这行数据会按照这个复杂的存储结构去存放。每一行的数据都放在数据页里。如果一行数据太大，则会产生行溢出，导致一行数据溢出到多个数据页里。那么这行数据在Buffer Pool可能就是存在于多个缓存页里的，刷入磁盘时也会用磁盘的多个数据页来存放这行数据。


 


**8\.数据页的物理存储结构**


在MySQL中进行数据操作的最小单位是数据页，默认大小是16KB。当一行的数据量比16KB大时，会发生行溢出使用其他数据页存放。存放该行数据的数据页会使用一个20字节大小的指针指向其他溢出页。


 


存放一行的数据页会被拆分成多个部分：包括文件头、数据页头、最小记录和最大记录、多个数据行、空闲空间、数据页目录、文件尾部。下图所示包含了一个数据页的各个部分：


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/73ab25fb705144a8b63db6cfe13977b6~tplv-obj.image?lk3s=ef143cfe&traceid=202411241939060C3C4F138384E73CBF95&x-expires=2147483647&x-signature=o%2FhTQgYHZQpg92%2BP%2FCt6%2BlDkhw0%3D)
文件头会占38个字节；


数据页头会占56个字节；


最大记录和最小记录会占26个字节；


数据行区域大小是不固定的；


空闲区域的大小也是不固定的；


数据页目录的大小也是不固定的；


文件尾部会占8个字节；


 


一开始数据库初始化完后，数据页是空的，没有一行数据，对应于"多个数据行"的区域是空的。接着如果要插入一行数据，由于数据页和缓存页是对应的，所以会往一个空闲缓存页的多个数据行进行数据写入，并更新空闲区域。


 


**9\.表空间的物理存储结构**


**(1\)什么是表空间**


简单来说，我们平时创建的那些表，其实都对应一个表空间的。表空间在磁盘上都会对应着"表名.ibd"这样的一个磁盘数据文件。


 


在物理层面，表空间就是对应一些磁盘上的数据文件。系统的表空间可能会对应多个磁盘文件。我们创建的表对应的表空间，通常对应一个"表名.ibd"的数据文件。


 


在表空间的磁盘文件里，会有很多很多的数据页。一个数据页只有16KB，为了便于管理这么多数据页，表空间引入了数据区(extent)。一个数据区对应着连续的64个数据页。每个数据页16KB，所以一个数据区是1MB。一组数据区也就是256个数据区会被划分为一个组，所以一组数据区有256MB。


 


表空间的第一组数据区的第一个数据区的前3个数据页，都是固定的，里面存放了一些描述性的数据。比如FSP\_HDR数据页，存放了表空间和这一组数据区的一些属性；比如IBUF\_BITMAP数据页，存放了这一组数据页的所有insert buffer信息；比如INODE数据页，里面存放了一些特殊信息。


 


然后表空间的其他各组数据区的第一个数据区的头两个数据页，也都是固定的，里面存放了一些特殊信息：比如XDES数据页就是用来存放这一组数据区的一些相关属性的。


 


**(2\)表空间总结**


平时创建的表都是有对应的表空间，每个表空间对应磁盘上的数据文件。表空间里有很多数据区组，256个数据区分成一组。每个数据区包又有64个数据页，所以每个数据区的大小是1MB。


 


表空间的第一组数据区的第一个数据区的头三个数据页，存放特殊信息。表空间的其他组数据区的第一个数据区的头两个数据页，也存放特殊信息。


 


当InnoDB执行增删改查操作时，会从磁盘上的表空间的数据文件里，加载数据页到Buffer Pool的缓存页里。


 


下图展示了一个表空间内部的存储结构：一个表空间内部会包含一组一组的数据区，每一组数据区包含256个数据区，每个数据区又包含64个数据页。


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/72f3837359ae42d9926f160a5d5c672a~tplv-obj.image?lk3s=ef143cfe&traceid=202411241939060C3C4F138384E73CBF95&x-expires=2147483647&x-signature=tyBFo%2FxgDqMwIjH5prWCIgnvZU0%3D)
 


**10\.InnoDB存储模型及读写机制总结**


在逻辑层面上，InnoDB的数据是插入一个一个的表中。在物理层面上，InnoDB的数据是插入一个一个的表空间。


 


表空间对应着磁盘文件，磁盘文件里存放的就是数据。在磁盘文件存放数据时，会被拆分为一个一个的数据区组。每个数据区组包含256个数据区，每个数据区包含64个数据页。每个数据页包含一行一行的数据。


 


数据页又包含了：文件头、数据页头、最小记录和最大记录、多个数据行、空闲空间、数据页目录、文件尾部。


 


每个数据行又附加了真实数据外的很多信息：变长字段的长度列表、null值列表、数据头、真实数据和隐藏字段。


 


通过数据页、数据区、数据行附加的特殊信息，可以让InnoDB在磁盘文件里实现B\+索引、事务等复杂的机制。


 


当数据库执行增删改查时：必须把磁盘文件里的一个数据页加载到内存Buffer Pool中的缓存页里，然后增删改查都针对缓存页里的数据进行。


 


所以要读写数据时：会根据表找到一个表空间，通过表空间就可以找到对应的磁盘文件。通过磁盘文件就可以从里面找一个数据区组中的一个数据区。然后从该数据区中找一个数据页出来。最后就可以把这个数据页从磁盘加载到Buffer Pool缓存页里。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2eec588bb122465b95391c6334d067ab~tplv-obj.image?lk3s=ef143cfe&traceid=202411241939060C3C4F138384E73CBF95&x-expires=2147483647&x-signature=XzYefMw4cQIupvSkpK1KMorCS3k%3D)
 


 本博客参考[西部世界官网](https://tianchuang88.com)。转载请注明出处！
