#T:lucene-01从HelloWorld开始

##一、基本概念
###1．Lucene的定义
Lucene是一个由java实现的开源的软件项目，是一个高性能、可伸缩的信息检索（IR）工具库。

###2．Lucene具有的特点
(1).索引文件格式独立于应用平台，Lucene自定义了一套以8位字节为基础的索引文件格式，使得它能在不同的应用平台共享所建立
的索引文件。  
(2).在传统全文检索引擎的倒排序基础上，实现了分块索引，对新文档（Document）建立小文件索引，提升索引速度。但同时支持与原索引文件进行合并优化。  
(3).设计了可以供用户设计的分词器接口，解决了不同应用分词需求不同的问题。  
(4).具有高性能的创建索引与搜索实现。


##二、Lucene核心组件

通常要实现类似于Google/Baidu这样的搜索功能，大致需要以下三步：
>1.获取内容：通过网络爬虫或蜘蛛程序来获取需要索引的内容，当然内容也可是来自本地文件或者是数据库中的数据。  
>2.建立索引：通过对获取的内容进行数据格式化，形成能被搜索组件快速探索的格式。  
>3.搜索索引：将用户的查询条件转换成符合搜索组件的查询格式，然后通过查询组件完成查询操作并将结果返回给用户。

Lucene的核心功能主要完成上面的两步：即“建立索引”和“搜索索引”，而这种先建立索引再对索引进行搜索的过程称之为“全文检索”。
全文检索过程：计算机索引程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置等信息，当户查询时，根据索引信息查出具体的数据。从某种意义上讲全文检索类似于字典的索引查字过程。
###1.索引组件
为了快速搜索大量的文本，首先建立针对文本的索引，将文本内容按照一定的索引规则转换成能够快速搜索的格式，这个过程称之为索引操作，而它的输出被称之为“索引”。
###2.搜索组件
搜索组件主要负责对用户的查询语句进行转换，并从索引中查找出相应的结果并能对结果进行加权排序。

##三、实例
###1.建立索引
**IndexFiles.java 代码片段一：**
```java
	static void indexDoc(IndexWriter writer, Path file, long lastModified) throws IOException {
	    //创建Document对象
	    Document doc = new Document();
	    //向Document添加“path”域：
	    Field pathField = new StringField("path", file.toString(), Field.Store.YES);
	    doc.add(pathField);
	    //向Document添加“modified”域：
	    doc.add(new LongField("modified", lastModified, Field.Store.NO));
	    //向Document添加“contents”域：
	    InputStream stream = Files.newInputStream(file);
	    doc.add(new TextField("contents", new BufferedReader(new InputStreamReader(stream, StandardCharsets.UTF_8))));
	    //创建索引：在write关闭时，完成索引创建
	    writer.addDocument(doc);
	}
```
>说明：以上代码主要完成对单个文档的索引创建。 对文档创建索引主要步骤如下：  
>>步骤一：创建Document对象  
>>步骤二：构建域对象(Field)并添加到Document对象中  
>>步骤三：使用IndexWriter对象的add方法对文档进行索引  

>域对象(Field)创建，构造方法主要参数说明：
>>name：域的名称  
>>value:域对应的值  
>>store：域对应的值是否存储：Field.Store.YES（域对应的值完全被存储，可以进行文本还原）；Field.Store.NO（域中的内容不进行存储，但可以被索引）  
>>通常文章的内容由于过多不被存储，但文章内容会参与索引。也可以通过对文件生成摘要并存储摘要以供使用。
>>


字段域的创建：字段域就是把一个源文件的所有信息进行分类处理，比如在上面实例中，我们把文件信息分为四类字段域：一是文件名、二是文件大小、三是文件内容、四是文件的存储路径。例子中用到的字段域Field构造方法参数解析：
Field(String name, String value,Field.Store store,Field.Index index)
 
name
字段域名称
value
字段域的值
store
是否把此字段域存储进索引
index
字段域的值是否进行分词处理
结合此构造方法解读词：词就是一个字串，更具体地说是经过分词处理后的字串。比如在上面的构造方法中value作为“源词”，当它经过index处理后便可称之为“词”。词具有两个基本信息：一是它所属的字段域，二是它的具体内容。从这个意义上讲即使两个词的内容完全相同，但是只要所属的域不同，它们则属于两个不同的词。
文档的创建：文档对象由字段域对象组成，创建时只需把字段域对象添加进文档即可。因而文档对象实质就是字段域的合并。
索引段的创建：索引段由文档对象组成，创建索引段也同样只需对文档对象进行添加操作即可。索引段的创建依赖于IndexWriter对象，此对象的主要作用就是把文档添加到索引中建立索引段，实现索引的创建。例子中用到的IndexWriter构造方法参数解析：

**IndexFiles.java 代码片段二：**
```java
    static void indexDocs(final IndexWriter writer, Path path) throws IOException {
        if (Files.isDirectory(path)) {
            Files.walkFileTree(path, new SimpleFileVisitor<Path>() {
                @Override
                public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
                    indexDoc(writer, file, attrs.lastModifiedTime().toMillis());
                    return FileVisitResult.CONTINUE;
                }
            });
        } else {
            indexDoc(writer, path, Files.getLastModifiedTime(path).toMillis());
        }
    }
```
>说明：以上代码主通过NIO的方法对目录下的所有文件进行索引操作。  

**IndexFiles.java 代码片段三：**
```java
    public static void main(String[] args) {
        String indexPath = "d:/lucene/index"; //索引文件存放目录
        String docsPath = "d:/lucene/docs";//被索引的文档目录
        try {
            Long startTime = System.currentTimeMillis();
            final Path docDir = Paths.get(docsPath);
            //构建索引文件目录
            Directory indexDir = FSDirectory.open(Paths.get(indexPath));
            System.out.print(Paths.get(indexPath).toAbsolutePath().toString());
            Analyzer analyzer = new StandardAnalyzer();
            //IndexWriterConfig 为IndexWriter保存所有配置信息
            IndexWriterConfig iwc = new IndexWriterConfig(analyzer);
            iwc.setOpenMode(IndexWriterConfig.OpenMode.CREATE.CREATE);
            //iwc.setInfoStream(System.out); //
            IndexWriter writer = new IndexWriter(indexDir, iwc);
            indexDocs(writer, docDir);
            writer.close();
            System.out.print("创建索引用时："+(System.currentTimeMillis() - startTime));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
说明：通过main方法实现索引创建，创建好索引后的效果图如下：
![索引文件列表](https://raw.githubusercontent.com/hutea/qsms/master/%E6%88%91%E7%9A%84%E7%AC%94%E8%AE%B0/Lucene%E7%AC%94%E8%AE%B0/images/%E4%BA%A7%E7%94%9F%E7%9A%84%E7%B4%A2%E5%BC%95%E6%96%87%E4%BB%B6.png)  

>代码分析  
>1.域的创建：


步骤四、执行完成后可以发现在索引目录下创建了三类文件：一是cfs文件、二是gen文件、三是segments_N文件。
详细分析细节过程：在分析前我们应明确lucene索引文件的基本层次结构：索引段、文档、字段域、词。下面我们结合上面的实例来具体分析：

IndexWriter(Directory d,Analyzer a,boolean create, IndexWriter.MaxFieldLength mfl) 
d
索引的保存目录
a
使用的分词器
create
是否自动创建索引目录
mfl
索引中域的长度（词的总数）
实例总结：（1）索引创建过程：基于词创建字段域，把字段域添加文档，把文档添加进索引段。（2）索引实质就是由不同的索引段构造，或者说在索引目录中保存了许许多多的索引段。（3）cfs文件就是一种复合后的索引文件，它持有所有索引文件的句柄（引用），以便进行频繁的索引操作，而gen和segment_N文件记录了索引段的基本元信息。

