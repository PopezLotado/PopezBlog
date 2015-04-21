---
layout: post
title:  "用lucene3.5搜索数据库和txt文件内容"
date:   2015-04-21 21:28:05
author: Popez Lotado
categories: Blogs
image: true
---


{% if page.image %}
<div class="post-img">
<img class="img-responsive img-post" src=" {{site.baseurl}}/img/post2.jpeg "/>
</div>
{% endif %}


我们以前经常碰到搜索数据库的内容；用like ％的sql语句；如果数据量大而且多表查询时；速度实在让人难以忍受。。。如果用lucene3.5那就可以把这个恼人的问题解决了。
 
lucene3.5搜索photo表的title，username，tagname，desr内容；
用一个例题来说明更直观；此例题能搜索中文分词；
（需要mysql5的jdbc包和lucene3.5的包）：
 
1、数据库我用mysql5；建一个photo表；数据库名是test。
--
-- 表的结构 `photo`
--
 ` CREATE TABLE IF NOT EXISTS 'photo' ( ` 
 ` 'photo_id' int(10) unsigned NOT NULL AUTO_INCREMENT,`
 ` 'title' varchar(11) DEFAULT NULL,`
 ` 'descr' text,`
 ` 'user_name' varchar(11) DEFAULT NULL,`
 ` 'tag_name' varchar(11) DEFAULT NULL,`
`  PRIMARY KEY ('photo_id')`
`) ENGINE=InnoDB  DEFAULT CHARSET=utf8 ROW_FORMAT=REDUNDANT `
`AUTO_INCREMENT=5 ;`
--
-- 导出表中的数据 `photo`
--
    INSERT INTO `photo` (`photo_id`, `title`, `descr`, `user_name`, `tag_name`) VALUES
    (1, '美女', '美女', '好人5', '美女'),
    (2, '美女', '美女', '美女', '美女'),
    (3, 'hagh', '说的就是我的是', '', NULL),
    (4, 'hagh', '说的就是我的是', ' ', NULL);
 

----------

    2、java文件有4个：
     
    文件Photo.java是数据库的photo表的操作文件；内容如下：
    package test;
    import java.sql.Connection;
    
    import java.util.ArrayList;
    import java.sql.PreparedStatement;
    import java.sql.ResultSet;
    import java.sql.SQLException;
    public class Photo {
     private long photoId;
     private String title;
     private String description;
     private String userName;
     private String tag;
     public String getDescription() {
      return description;
     }
     public void setDescription(String description) {
      this.description = description;
     }
     public long getPhotoId() {
      return photoId;
     }
     public void setPhotoId(long photoId) {
      this.photoId = photoId;
     }
     public String getTag() {
      return tag;
     }
     public void setTag(String tag) {
      this.tag = tag;
     }
     public String getTitle() {
      return title;
     }
     public void setTitle(String title) {
      this.title = title;
     }
     public String getUserName() {
      return userName;
     }
     public void setUserName(String userName) {
      this.userName = userName;
     }
     public static Photo[] loadPhotos(Connection con) throws Exception {
      ArrayList<Photo> list = new ArrayList<Photo>();
      PreparedStatement pstm = null;
      ResultSet rs = null;
      String sql = "select photo_id,title,descr,user_name,tag_name from photo";
      try {
       pstm = con.prepareStatement(sql);
       rs = pstm.executeQuery();
       while (rs.next()) {
    Photo photo = new Photo();
    photo.setPhotoId(rs.getLong(1));
    photo.setTitle(rs.getString(2));
    photo.setDescription(rs.getString(3));
    photo.setUserName(rs.getString(4));
    photo.setTag(rs.getString(5));
    list.add(photo);
       }
      } catch (SQLException e) {
       e.printStackTrace();
      } finally {
       if (rs != null) {
    rs.close();
       }
       if (pstm != null) {
    pstm.close();
       }
      }
      return (Photo[]) list.toArray(new Photo[list.size()]);
     }
    }

----------

    文件IndexerFile.java是把数据库的内容备份成索引文件到磁盘中去；
    内容如下：
    package test;
    import java.io.File;
    import java.io.IOException;
    import org.apache.lucene.analysis.standard.StandardAnalyzer;
    import org.apache.lucene.document.Document;
    import org.apache.lucene.index.IndexWriter;
    import org.apache.lucene.index.IndexWriterConfig;
    import org.apache.lucene.index.IndexWriterConfig.OpenMode;
    import org.apache.lucene.store.FSDirectory;
    import org.apache.lucene.util.Version;
    import org.apache.lucene.document.Field;
    public class IndexerFile {
     public static int indexFile(String indexDir,Photo[] list) throws IOException{
      IndexWriterConfig conf = new IndexWriterConfig(Version.LUCENE_35, new StandardAnalyzer(Version.LUCENE_35));
     conf.setOpenMode(OpenMode.CREATE);
     IndexWriter writer = new IndexWriter(FSDirectory.open(new File(indexDir)), conf);
     
      for(int i=0;i<list.length;i++){
       Document doc=new Document();
       doc.add(new Field("photoId", String.valueOf(list[i].getPhotoId()), Field.Store.YES, Field.Index.NO));
       if(list[i].getTitle()!=null && list[i].getTitle().length()>0)
    doc.add(new Field("title", list[i].getTitle(), Field.Store.YES, Field.Index.ANALYZED));
       if(list[i].getDescription()!=null && list[i].getDescription().length()>0)
    doc.add(new Field("description", list[i].getDescription(), Field.Store.YES, Field.Index.ANALYZED));
       if(list[i].getUserName()!= null && list[i].getUserName().length()>0)
       doc.add(new Field("userName", list[i].getUserName(), Field.Store.YES, Field.Index.ANALYZED));
       if(list[i].getTag()!= null && list[i].getTag().length()>0)
    doc.add(new Field("tag", list[i].getTag(), Field.Store.YES, Field.Index.ANALYZED));
       writer.addDocument(doc);
      }
     
      int numIndexed = writer.maxDoc();
      writer.forceMerge(1);
      writer.close();
      return numIndexed;
     }
    }
 

----------

    文件SearcherFile.java是搜索磁盘索引文件内容的；
    内容如下：
    package test;
    import java.io.IOException;
    import org.apache.lucene.analysis.Analyzer;
    import org.apache.lucene.analysis.standard.StandardAnalyzer;
    import org.apache.lucene.document.Document;
    import org.apache.lucene.queryParser.MultiFieldQueryParser;
    import org.apache.lucene.queryParser.ParseException;
    import org.apache.lucene.search.IndexSearcher;
    import org.apache.lucene.search.Query;
    import org.apache.lucene.search.ScoreDoc;
    import org.apache.lucene.search.TopDocs;
    import org.apache.lucene.util.Version;
    public class SearcherFile {
     public static void search(IndexSearcher searcher, String[] q) throws IOException, ParseException {
      Analyzer analyzer = new StandardAnalyzer(Version.LUCENE_35);
      String[] fields = {"title","description","tag","userName"};
    Query query = MultiFieldQueryParser.parse(Version.LUCENE_35, q, fields, analyzer);
    TopDocs topDocs = searcher.search(query, 100);//100是显示队列的Size
     ScoreDoc[] hits = topDocs.scoreDocs;
     System.out.println("共有" + searcher.maxDoc() + "条索引，命中" + hits.length + "条");
     for (int i = 0; i < hits.length; i++) {
     int DocId = hits[i].doc;
     Document document = searcher.doc(DocId);
     System.out.println("photoId==="+document.get("photoId"));
     }
     }
    }
 

----------

    文件TestDb.java是操作的主文件；
    内容如下：
        package test;
    import java.io.File;
    import java.io.IOException;
    import java.sql.Connection;
    import java.sql.SQLException;
    import java.util.Date;
    import org.apache.lucene.index.IndexReader;
    import org.apache.lucene.queryParser.ParseException;
    import org.apache.lucene.search.IndexSearcher;
    import org.apache.lucene.store.FSDirectory;
    public class TestDb {
     public final static String indexDir ="E:\\TestLucene";
     private static Connection getConnection() {
      Connection conn = null;
      String url = "jdbc:mysql://localhost:3306/test";
      String userName = "root";
      String password = "root";
      try {
       Class.forName("com.mysql.jdbc.Driver");
       conn = java.sql.DriverManager
     .getConnection(url, userName, password);
      } catch (Exception e) {
       e.printStackTrace();
       System.out.println("Error Trace in getConnection() : "+ e.getMess());
      }
      return conn;
     }
     public static void main(String[] args) throws IOException, ParseException, SQLException {
      index();//做索引
      IndexSearcher searcher=null;
      try{
       IndexReader reader = IndexReader.open(FSDirectory.open(new File(indexDir)),false); 
      searcher = new IndexSearcher(reader);
       search(searcher);//搜索
      }catch(Exception e){
       e.printStackTrace();
      }finally{
       if(searcher!=null)
       searcher.close();
      }
     }
     public static void search(IndexSearcher searcher) throws IOException, ParseException{
      //以下是搜索的关键词
      String[] q = {"美女1","美女2","好人3","好人5"};
      long start=new Date().getTime();
      SearcherFile.search(searcher,q);
      long end=new Date().getTime();
      System.out.println("花费时间："+(double)(end-start)/1000+"秒");
     }
     public static void index() throws SQLException{
      Connection conn = null;
      try {
       conn = getConnection();
       Photo[] list = Photo.loadPhotos(conn);
       IndexerFile.indexFile(indexDir,list);
      } catch (Exception e) {
       e.printStackTrace();
      } finally {
       if (conn != null) {
    conn.close();
       }
      }
     }
    }
 
二、下面是lucene3.5搜索txt文本文件
 
建一个E:\\TestLucene\\fileS的文件夹,放需要搜索的文件。
在该文件夹里面随便建三个txt文件，"1.txt","2.txt"和"3.txt"
 

其中1.txt的内容如下：  

老周
北京人民 
2009年

2.txt和3.txt也随便写些。

 

再建一个E:\\TestLucene\\fileIndex的文件夹；放索引文件。

 

 

java文件TestQueryFile：内容如下

 

    package test;
    
    import java.io.BufferedReader;
    import java.io.File;
    import java.io.FileInputStream;
    import java.io.IOException;
    import java.io.InputStreamReader;
    import java.util.Date;
    
    import org.apache.lucene.analysis.standard.StandardAnalyzer;
    import org.apache.lucene.document.Document;
    import org.apache.lucene.index.IndexReader;
    import org.apache.lucene.index.IndexWriter;
    import org.apache.lucene.index.IndexWriterConfig;
    import org.apache.lucene.index.IndexWriterConfig.OpenMode;
    import org.apache.lucene.queryParser.ParseException;
    import org.apache.lucene.queryParser.QueryParser;
    import org.apache.lucene.search.IndexSearcher;
    import org.apache.lucene.search.Query;
    import org.apache.lucene.search.ScoreDoc;
    import org.apache.lucene.search.TopDocs;
    import org.apache.lucene.store.FSDirectory;
    import org.apache.lucene.util.Version;
    import org.apache.lucene.document.Field;
    public class TestQueryFile {
     
      public static void main(String[] args) throws Exception {
    indexF(); 
    String queryString = "北京";
       Query query = null;
    IndexReader reader = IndexReader.open(FSDirectory.open(new File("E:\\TestLucene\\fileIndex")),true);//read-only
       IndexSearcher searcher = new IndexSearcher(reader);
       String fields = "body";
       try {
    QueryParser qp = new QueryParser(Version.LUCENE_35, fields, new StandardAnalyzer(Version.LUCENE_35));//有变化的地方
       query = qp.parse(queryString);
       } catch (ParseException e) {
       }
       if (searcher != null) {
       TopDocs topDocs = searcher.search(query, 100);//100是显示队列的Size
       ScoreDoc[] hits = topDocs.scoreDocs;
       System.out.println("共有" + searcher.maxDoc() + "条索引，命中" + hits.length + "条");
       }
       }
     
      private static void indexF() throws Exception {
      
       File fileDir = new File("E:\\TestLucene\\fileS");
      
      
       File indexDir = new File("E:\\TestLucene\\fileIndex");
     
    IndexWriterConfig conf = new IndexWriterConfig(Version.LUCENE_35, new StandardAnalyzer(Version.LUCENE_35));
     conf.setOpenMode(OpenMode.CREATE);
     IndexWriter indexWriter = new IndexWriter(FSDirectory.open(indexDir), conf);
    
       File[] textFiles = fileDir.listFiles();
       long startTime = new Date().getTime();
      
       //增加document到索引去
       for (int i = 0; i < textFiles.length; i++) {
       if (textFiles[i].isFile()
       && textFiles[i].getName().endsWith(".txt")) {
       System.out.println("File " + textFiles[i].getCanonicalPath()
       + "正在被索引....");
       String temp = FileReaderAll(textFiles[i].getCanonicalPath(),
       "GBK");
       System.out.println(temp);
       Document document = new Document();
       Field FieldPath = new Field("path", textFiles[i].getPath(),
       Field.Store.YES, Field.Index.NO);
       Field FieldBody = new Field("body", temp, Field.Store.YES,
       Field.Index.ANALYZED,
       Field.TermVector.WITH_POSITIONS_OFFSETS);
       document.add(FieldPath);
       document.add(FieldBody);
       indexWriter.addDocument(document);
     }
       }
       //optimize()方法是对索引进行优化
       indexWriter.forceMerge(1);
       indexWriter.close();
      
       //测试一下索引的时间
       long endTime = new Date().getTime();
       System.out
       .println("这花费了"
       + (endTime - startTime)
       + " 毫秒来把文档增加到索引里面去!"
       + fileDir.getPath());
       }
      
       private static String FileReaderAll(String FileName, String charset)
       throws IOException {
       BufferedReader reader = new BufferedReader(new InputStreamReader(
       new FileInputStream(FileName), charset));
       String line = null;
     StringBuffer temp = new StringBuffer("");
      
       while ((line = reader.readLine()) != null) {
       temp.append(line);
       }
       reader.close();
       return temp.toString();
       }
    }
    
 

 

一执行就知道结果了