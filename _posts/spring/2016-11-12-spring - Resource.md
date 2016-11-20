---
layout: article
title: spring Resource
excerpt_separator: <!--more-->
---
简单介绍 Resource接口
<!--more-->
# 简述
Interface for a resource descriptor that abstracts from the actual type of underlying resource,
 such as a file or class path resource.

上面是Resource接口的doc描述，很明确的定义了接口的职责。它是资源的抽象，为应用提供了统一的访问api。
![](https://github.com/whyDK37/note/blob/master/_posts/spring/uml/Resource.png?raw=true)

## resource 接口方法

Resource 接口继承自 InputStreamSource ，接口中定了了很多方法，通过方法名可以很直观的知道方法的功能。

``
public interface Resource extends InputStreamSource {

	/**
	 * Return whether this resource actually exists in physical form.
	 * <p>This method performs a definitive existence check, whereas the
	 * existence of a {@code Resource} handle only guarantees a
	 * valid descriptor handle.
	 */
	boolean exists();

	/**
	 * Return whether the contents of this resource can be read,
	 * e.g. via {@link #getInputStream()} or {@link #getFile()}.
	 * <p>Will be {@code true} for typical resource descriptors;
	 * note that actual content reading may still fail when attempted.
	 * However, a value of {@code false} is a definitive indication
	 * that the resource content cannot be read.
	 * @see #getInputStream()
	 */
	boolean isReadable();

	/**
	 * Return whether this resource represents a handle with an open
	 * stream. If true, the InputStream cannot be read multiple times,
	 * and must be read and closed to avoid resource leaks.
	 * <p>Will be {@code false} for typical resource descriptors.
	 */
	boolean isOpen();

	/**
	 * Return a URL handle for this resource.
	 * @throws IOException if the resource cannot be resolved as URL,
	 * i.e. if the resource is not available as descriptor
	 */
	URL getURL() throws IOException;

	/**
	 * Return a URI handle for this resource.
	 * @throws IOException if the resource cannot be resolved as URI,
	 * i.e. if the resource is not available as descriptor
	 */
	URI getURI() throws IOException;

	/**
	 * Return a File handle for this resource.
	 * @throws IOException if the resource cannot be resolved as absolute
	 * file path, i.e. if the resource is not available in a file system
	 */
	File getFile() throws IOException;

	/**
	 * Determine the content length for this resource.
	 * @throws IOException if the resource cannot be resolved
	 * (in the file system or as some other known physical resource type)
	 */
	long contentLength() throws IOException;

	/**
	 * Determine the last-modified timestamp for this resource.
	 * @throws IOException if the resource cannot be resolved
	 * (in the file system or as some other known physical resource type)
	 */
	long lastModified() throws IOException;

	/**
	 * Create a resource relative to this resource.
	 * @param relativePath the relative path (relative to this resource)
	 * @return the resource handle for the relative resource
	 * @throws IOException if the relative resource cannot be determined
	 */
	Resource createRelative(String relativePath) throws IOException;

	/**
	 * Determine a filename for this resource, i.e. typically the last
	 * part of the path: for example, "myfile.txt".
	 * <p>Returns {@code null} if this type of resource does not
	 * have a filename.
	 */
	String getFilename();

	/**
	 * Return a description for this resource,
	 * to be used for error output when working with the resource.
	 * <p>Implementations are also encouraged to return this value
	 * from their {@code toString} method.
	 * @see Object#toString()
	 */
	String getDescription();
}
```

InputStreamSource 中只有一个方法，返回输入流，这个方法不是所有的子类都实现了，后面我会举例说明。

```
public interface InputStreamSource {

	/**
	 * Return an {@link InputStream}.
	 * <p>It is expected that each call creates a <i>fresh</i> stream.
	 * <p>This requirement is particularly important when you consider an API such
	 * as JavaMail, which needs to be able to read the stream multiple times when
	 * creating mail attachments. For such a use case, it is <i>required</i>
	 * that each {@code getInputStream()} call returns a fresh stream.
	 * @return the input stream for the underlying resource (must not be {@code null})
	 * @throws IOException if the stream could not be opened
	 * @see org.springframework.mail.javamail.MimeMessageHelper#addAttachment(String, InputStreamSource)
	 */
	InputStream getInputStream() throws IOException;
}
```

## 举例

spring Resource 有很多实现，下面描述几个有代表性的实现。

### AbstractResource

是一个模板模式的抽象类，实现了 exists , isReadable ， isOpen ， getURI ，contentLength， 等方法，
有些有些方法是空实现，如getFilename，createRelative，getFile，这些方法需要子类按需实现。

### ClassPathResource

用来加载classpath中的资源，spring在初始化的时候回使用这个类来加载xml配置文件。

```
public ClassPathXmlApplicationContext(String[] paths, Class<?> clazz, ApplicationContext parent)
			throws BeansException {
    super(parent);
    Assert.notNull(paths, "Path array must not be null");
    Assert.notNull(clazz, "Class argument must not be null");
    this.configResources = new Resource[paths.length];
    for (int i = 0; i < paths.length; i++) {
        this.configResources[i] = new ClassPathResource(paths[i], clazz);
    }
    refresh();
}
```

### FileSystemResource

实现了 WritableResource 接口，该实现由写的能力。支持系统文件资源

```
FileSystemResource resource = new FileSystemResource(System.getProperty("java.io.tmpdir") + "/tmp.ftl");
```

### UrlResource

可以表示URl和file资源。

```
new UrlResource("http://localhost:8080")
```

# 小结

我们可以在项目中使用Resource接口，灵活的使用各种资源，这样屏蔽了实现，系统的兼容性更好。