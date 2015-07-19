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
索引组件主要负责索引的创建，即通过对文档内容的分析，建立一定格式的索引文件，以供搜索组件使用。  
>索引：为了快速搜索大量的文本，首先建立针对文本的索引，将文本内容按照一定的索引规则转换成能够快速搜索的格式，这个过程称之为索引操作，而它的输出被称之为“索引”。  
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

结合此构造方法解读词：词就是一个字串，更具体地说是经过分词处理后的字串。比如在上面的构造方法中value作为“源词”，当它经过索引组件处理后便可称之为“词”。词具有两个基本信息：一是它所属的域，二是它的具体内容。从这个意义上讲即使两个词的内容完全相同，但是只要所属的域不同，它们则属于两个不同的词。  
文档的创建：文档对象由域对象组成，创建时只需把字段域对象添加进文档即可。因而文档对象实质就是多个域的组合。


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
        String docsPath = "d:/lucene/docs";//被索引的文档目录：该目录下有多个原始文档
        try {
            Long startTime = System.currentTimeMillis();
            final Path docDir = Paths.get(docsPath);
            //构建索引文件目录
            Directory indexDir = FSDirectory.open(Paths.get(indexPath));
            Analyzer analyzer = new StandardAnalyzer();
            //IndexWriterConfig 为IndexWriter保存所有配置信息
            IndexWriterConfig iwc = new IndexWriterConfig(analyzer);
            iwc.setOpenMode(IndexWriterConfig.OpenMode.CREATE.CREATE);
            //iwc.setInfoStream(System.out); //设置信息输出源
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

>本段代码的核心是构建IndexWriter对象，构建IndexWriter对象必须的两个参数   
>1.IndexWriterConfig对象：负责保存IndexWriter相关的配置信息  
>2.Directory对象：负责指定索引文件的存储位置

###2.搜索索引
**IndexFiles.java 代码片段：**  
````java
  public static void main(String[] args) {
        try {
 			//构建Query对象
            String field = "contents";
            Analyzer analyzer = new StandardAnalyzer();
            QueryParser parser = new QueryParser(field, analyzer);
            Query query = parser.parse("server");      
            //创建IndexReader对象
            String indexPath = "d:/lucene/index";
            IndexReader reader = DirectoryReader.open(FSDirectory.open(Paths.get(indexPath)));
			//创建IndexSearch对象
            IndexSearcher searcher = new IndexSearcher(reader);
            TopDocs topDocs = searcher.search(query, 5);
			//打印查询结果
            System.out.println("查询结果：" + topDocs.totalHits + "条");
            ScoreDoc[] hits = topDocs.scoreDocs;
            for(ScoreDoc scoreDoc:hits){
                Document doc = searcher.doc(scoreDoc.doc);
                System.out.println("文档路径："+doc.get("path"));
                System.out.println("文档内容："+doc.get("contents"));
            }
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ParseException e) {
            e.printStackTrace();
        }
    }

```
>查询结果：2条  
>文档路径：d:\lucene\docs\server.xml  
>文档内容：null  
>文档路径：d:\lucene\docs\catalina.properties  
>文档内容：null  

分析：从查询结果来看，发现文档内容并没有获取到，这时因为我们在创建索引时并没有对文档内容进行存储(在IndexFiles中，我们通过TextField来创建文档内容的文本域，但默认的文本域是不进行存储的)，如果要想在这里显示，在创建索引时将内容转换为字串并进行存储即可。  



##四、索引过程核心类
在创建索引过程中主要用到下图所示的五个类
![索引文件列表](https://raw.githubusercontent.com/hutea/qsms/master/%E6%88%91%E7%9A%84%E7%AC%94%E8%AE%B0/Lucene%E7%AC%94%E8%AE%B0/images/lucene%E5%88%9B%E5%BB%BA%E7%B4%A2%E5%BC%95%E4%BD%BF%E7%94%A8%E5%88%B0%E7%9A%84%E7%B1%BB.png)  

###1.IndexWriter
IndexWriter是索引过程的核心组件，主要负责创建新的索引或者打开已有索引，以及向索引中添加、删除或更新被索引的文档信息。

###2.Directory
Directory类指定了索引文件的存放位置，比如我们在实例中使用其子类FSDirectory.open方法来获取真实的存储路径。

###3.Analyzer
IndexWriter并不能直接索引文本，需由经Analyzer将文本进行分词后才能进行文本索引。Analyzer主要负责从被索引的文本文件中提取语汇单元，并去掉无用信息。事实上对一个应用程序选择一个合适的Analyzer是非常重要的，因为一个合适的Analyzer对于索引搜索的准确性起着非常重要的作用。

###4.Document
Document是搜索和搜索的原子单位，包含一个或多个Field对象的容器，而IndexWriter最终要进行直接处理的对象也正是此Doucment对象。从本质上来说，Document决定了文档怎样被索引，当然最本质上是由Field决定。

###5.Field
域通常由域名和对应的域值，再加上存储选项组成(当然有些域是没有存储选项的，如TextField不进行存储，则没有存储选项)。

##五、搜索过程核心类
###1.IndexSearcher
IndexSearcher负责搜索由IndexWriter类创建的索引，具体来讲主要通过search方法来进行搜索。IndexSearch可看作是一个以只读方式打开索引的类，在前面的实例中，我们使用了最简单的搜索方法，即通过Query对象和int topN来实现搜索。

###2.Query
要实现查询功能，通常我们会将查询内容构建成一个Query对象，然后再将此Query对象传递给IndexSearcher的search方法。除了在前面的实例中我们通过QuerParser对象来解析出一个Query对象外，我们还可以构建TermQuery、BooleanQuery、PhraserQuery等Query子类来进行查询操作。
###3.TopDocs
简单来说TopDocs是对查询结果集的封装，通过TopDocs对象的scoreDocs属性我们可以得到具体的查询结果文档信息(ScoreDocs数组)。然后可以对此ScoreDocs数组进行遍历操作得到我们期望的内容。

