---
title: EntityFileService - 基于实体的简单文件管理设计
date: 2020-08-04 22:37:43
categories:
- '笔记'
- '技术'
tags:
- Java
- Spring
- Web
---
# 它从何而来

在我的项目里，经常会出现一个叫做 FileServiceAdapter 的东西，从第一个项目开始我就设计了这个东西，到现在它的设计还没有完善。根据命名规则可知，它并不是一个实体类。的确它并不能实际使用，必须继承他实现一个子类才能使用。

它的设计基于业务实体，例如订单、票据、用户等。订单可能有订单相关的附件，票据也会有票据相应的附件。实体有不同的种类，实体都有其对应的实体 ID （Entity ID），而每个实体都可能会有其对应的文件，所以检索实体相关的文件，只需要找到实体的类型和实体的 ID 即可。

EntityFileService 的设计就是这么来的，它通过实体类型和实体 ID 来管理归属于不同实体的附件文件。

# 结构

它的构造函数长这样：

```java
public FileServiceAdapter(File rootFolder, String domainName) {
  this.rootFolder = rootFolder;
  this.domainName = domainName;

  File currentContextFolder = new File(rootFolder, domainName);
  if (!currentContextFolder.exists() && !currentContextFolder.mkdir())
    throw new RuntimeException("Could not create domain folder " + currentContextFolder.getAbsolutePath());
}
```

继承他的子类需要给予一个 rootFolder 和一个 domainName。rootFolder 一般是指根数据目录，是固定的，在 Spring 的上下文内指定一个就好。 domainName 就是所谓的“实体类型”——但叫他“域名”可能会比较好。在我的设计里，它服务于“业务域”，代表了一个域下面的所有文件。而每个域都应该有其唯一的主要业务域实体（例如订单、商品），也就是 Entity 。

Entity 在实现上一般就是一条数据库记录。EntityFileService 的设计里，通过 Entity 的 ID ，通过负责对应业务域的 FileService 就可以拿到这个 Entity 专用的文件夹，这样就可以开始存放文件了。

## 保存上下文的 FileContext.class

在我的设计里，获取实体的附件文件夹的方法并没有直接返回一个 File 类型，而是返回一个包装类 `FileContext.class`。

```java
public FileContext contextOf(I id) {
  if (Objects.isNull(id)) throw new NullPointerException();
  
  FileContext fileContext = new FileContext();
  fileContext.setRootPath(getRootFolder().toPath());
  fileContext.setDomainPath(String.format("%s/%s", getDomainName(), String.valueOf(id)));
  fileContext.setFile(new File(getRootFolder(), fileContext.getDomainPath()));
  
  return fileContext;
}
```



`FileContext.class` 存储了对应的领域位置上下文，包括实际文件、根目录的绝对路径以及领域内路径。

```java
public class FileContext {
  // The file entity, maybe null
  private File file;

  // Absolute path of the root folder
  private Path rootPath;

  // Relative path of the current domain, related to the rootPath
  private String domainPath;
  
  //....
}
```



`file` 是指向领域路径所对应的真实文件系统的文件对象，上下文初始化的时候它不一定被初始化。触发初始化的是 `getFile()` ，而 `exists()` 也会间接触发初始化。

`getFile()` 作用是直接获取原始的文件对象，如果没有就直接创建一个。因为 EntityFileService 的设计很原始，所以选择尽量不限制原来的文件处理逻辑。`exists()` 的作用就是判断实际文件是否存在。

```java
/**
* Check if target context file is exists,
* rootPath and domainPath is necessary
*
* @return true or false
* @throws NullPointerException if rootPath or domainPath is null
*/
public boolean exists() {
  return getFile().exists();
}

//
// 获取上下文的文件本身，有可能是文件夹也有可能是文件。
// 要求 rootPath 和 domainPath 都不能为 null
//
// rootPath 指向根存储文件夹，domainPath 指向域内位置。
// 实际上来说，domainPath 只是一个基于 rootPath 的相对路径。
//
public File getFile() {
  if (Objects.isNull(file)) {
    assertCanGetContextFolder();
    this.file = new File(rootPath.toFile(), domainPath);
  }

  return this.file;
}
```



当需要把文件名存储在数据库里时，需要先取出文件在域内的位置

```java
//
// 获取领域内路径，这个路径的结构是 "域名/实体 ID/{一级或多级文件夹或文件}"
// 总体跟文件系统很类似
//
public String getDomainPath() {
  return domainPath;
}
```



当前上下是文件夹的话，就会有需要取出文件的需求，靠这两个方法：

```java
//
// 通过文件名，获取下一级文件夹的文件的上下文对象。
// 原则上当前级上下文必须是文件夹，但这里并没有做限制。
//
public FileContext contextOf(String filename) {
  FileContext fileContext = new FileContext();
  fileContext.setDomainPath(String.format("%s/%s", domainPath, filename));
  fileContext.setRootPath(this.rootPath);
  fileContext.setFile(new File(rootPath.toFile(), fileContext.getDomainPath()));
  return fileContext;
}

//
// 通过文件名，直接获取上下文下的文件。
// 原则上当前级上下文必须是文件夹，但这里并没有做限制。
//
public File fileOf(String filename) {
  if(!this.exists())
    throw new IllegalStateException("Context folder not exists");

  if (!this.getFile().isDirectory())
    throw new IllegalStateException("Not a directory");

  return new File(this.file, filename);
}
```



`FileContext.class` 扮演了另外一个 `File.class` 的角色，它负责存储的是领域存储内的上下文信息，在需要的时候转化为真实的文件对象。

其他的常用的文件操作，例如列出文件夹内容，就直接通过 `File.class` 本身的功能去做了。



## 存储文件

作为负责处理文件的 Service ，除了查出文件，当然还得写入文件。写入文件的操作靠 `save()` 方法处理

```java
public FileContext save(I i, String filename, InputStream inputStream) throws IOException {
  // 根据 Entity Id 获取 Entity 的存储上下文
  FileContext fileContext = contextOf(i);
  // 如果对应的文件夹实际不存在则新建一个
  if (!fileContext.exists() && !fileContext.getFile().mkdir())
    throw new IllegalStateException("Could not create entity folder " + fileContext.getDomainPath());
	
  // 通过 FilenameGenerator 更改文件名
  String newFilename = filenameGenerator.get(filename);
  FileContext newFile = fileContext.contextOf(newFilename);
  // 获取实际的文件对象，写入文件内容
  File targetFile = newFile.getFile();
  IOUtils.copyLarge(inputStream, new FileOutputStream(targetFile));
  // 返回新文件的 FileContext
  return newFile;
}
```

### FilenameGenerator.class

`FilenameGenerator.class` 是个接口，提供文件改名的抽象。

```java
public interface FilenameGenerator {
  /**
  * Generate a new filename
  *
  * @param origin origin filename, might be empty("")
  * @return new filename
  */
  String get(String origin);
}
```



有时候为了防止上传上来的文件因为重名而被覆盖，需要为上传上来的每个文件改一个唯一的名字。在 `FileServiceAdapter.class` 初始化的时候，会默认设定一个文件名生成器，而它的行为是按照原来的文件名来存储文件。

```java
public class FileServiceAdapter<I extends Serializable> implements FileService<I> {
  public final static FilenameGenerator DEFAULT_GENERATOR = s -> s;
  
  //...
  
  // 默认的文件名生成器
  private FilenameGenerator filenameGenerator = DEFAULT_GENERATOR;
  
  //
  // Getter & Setter
  //
  
  public FilenameGenerator getFilenameGenerator() {
    return filenameGenerator;
  }

  public void setFilenameGenerator(FilenameGenerator filenameGenerator) {
    this.filenameGenerator = filenameGenerator;
  }
}
```



要修改的话，在实现子类的时候，于构造函数内修改它就可以了，例如：

```java
// 一个专门存储用户信息文件的存储服务
public class UserFileService extends FileServiceAdapter<String> {

  public UserFileService(FileServiceConfiguration configuration) {
    // 塞预先设定好的配置类内的 rootFile ，给一个 domainName 叫 "user"
    super(configuration.dataDirectoryFile(), "user");
    // 修改文件名生成器为 UUID 生成器
    setFilenameGenerator(FilenameGenerators.RANDOM_UUID);
  }
}
```



我写了两个文件名生成器

```java
public final class FilenameGenerators {

  // 基于 UUID 的生成器，保留后缀
  public final static FilenameGenerator RANDOM_UUID = (origin) -> {
    String ext = origin.substring(origin.lastIndexOf("."));
    return String.format("%s%s", UUID.randomUUID().toString(), ext);
  };

  // 基于时间戳和随机数字的生成器，保留后缀
  public final static FilenameGenerator TIMESTAMP_WITH_RANDOM = (origin) -> {
    String ext = origin.substring(origin.lastIndexOf("."));
    return String.format("%d_%s%s", System.currentTimeMillis(), Randoms.randomString(6), ext);
  };

}
```



## 取文件

在我的设计里，一个 Entity 一般只有一级的存储，当只是要取里面的文件的时候，可以直接使用 `get()` 来获取

```java
public FileContext get(I i, String filename) {
  return contextOf(i).contextOf(filename);
}
```



这里会造成一点麻烦，因为返回的是 FileContext 而不是 File ，所以需要再用 `getFile()` 来把路径转换成真实文件。

其实一般来说，取文件的时候都是直接拿到存储在业务实体内的 domainPath 到 FileService 取文件，所以我做了另外一个更为常用的方法。

```java
public File fileOfContextPath(String contextPath) {
  File file = new File(rootFolder, contextPath);
  return file.exists() ? file : null;
}
```

这个方法直接把 EntityFileService 这个东西的实际操作暴露无遗（-w -||| ）

## 删除

删除文件直接通过取到文件之后直接删除实现，也可以通过 `deleteByContextPath()` 直接输入 domainPath 删除。而删除实体文件夹的方法我还没并入到现在的实现里，但在其他的 FileService 里是有实现的。

```java
public void deleteContextFolder(Long id) throws IOException {
  FileUtils.deleteDirectory(contextOf(id).getFile());
}
```

嗯很干脆直接。。。

# 反思

其实它本身没有名字。最后觉得它应该叫 EntityFileService，因为它是附随实体存在的。

我在不同的项目里使用它的同时，已经迭代了三次。我依旧觉得它对我来说有点怪异。它更多地是一个 Helper 去辅助“根据实体 ID 和实体类型来归组文件”的想法。

在文件的存储和定位 url 的拼接上，首先是：

1. 实体文件夹（/domainName/id/)
2. 存储文件（/domainName/id/filename）

然后得到这个完整的地址之后，存储在需要的地方（例如实体信息本身）。需要用的时候，通过 service 找到文件本身，然后对文件进行读取删除操作。

因为它本身也反映了真实文件位置，在没有权限控制要求的情况下，我直接使用 nginx 暴露这个文件夹，就能直接通过普通 http 请求获取到文件本身，service 的读功能基本没有什么太大的作用。

在内部代码操作上，只要获取到 context ，基本就直接 getFile 来获取到实际的文件并开始操作，FileContext 本身承担的功能很少。

我有点想让它变成类似 RedisTemplate 的存在，每个 Service 持有一个 Template 而不是继承一个 FileServiceAdapter 然后再持有 。

或者让它变成一些更加高级的文件管理服务允许 Auditing 之类的，但不确定这是否是个好想法。