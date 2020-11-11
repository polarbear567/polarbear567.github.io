# Springboot启动方式一览及其原理

## java命令行运行jar文件

------

在实际的项目中，**java -jar**应该是使用最多的一种运行springboot项目的方式，下面分为几步来介绍下这种方式背后涉及的各种原理。

### jar包中的结构

随便找个springboot的项目，在项目根目录执行mvn package命令，就可以在target目录得到一个打包好的jar包，解压后可以看到里面主要分为3个部分：**BOOT-INF文件夹**，**META-INF文件夹**，**org文件夹**。

- BOOT-INF文件夹----该文件夹下面包含2个文件夹，classes文件夹下都是项目中编译好的class文件，静态文件，属性文件。而lib下面的文件是项目所依赖的jar包。
- META-INF文件夹----存放pom相关信息，最重要的是里面的MANIFEST.MF文件。
- org文件夹----是springboot启动程序所需要的编译文件

### 为什么java -jar可以运行springboot项目

#### 相关概念

##### MANIFEST.MF

这是jar(war)包中的清单文件，采用kv的形式展现，里面包含的是当前包中的一些信息，当该文件中包含了Main-class属性时，那么这个jar包便成为了可以被java命令行执行的jar包。

##### jar in jar

一般通过maven或其他插件生成的jar包中，都是已经编译好的class文件，但是我们可以发现在springboot项目生成的jar包中依然还存在其他jar包，关于怎么加载这些jar包，我们将在下面进行说明。

##### Archive

这是springboot抽象出来的概念，它有两个实现：JarFileArchive和ExplodedArchive，也就是jar文件可以是一个archive，一个文件夹也可以是一个archive。比如我们springboot项目通过mvn命令生成的jar包本身就是一个archive，BOOT-INF文件夹也是一个archive。

Archive的getUrl()返回的是java的URL，URL的协议有很多，比如ftp，http，file等，jar也是其中的一种协议，$但原生的协议只支持一个分隔符`!/`,这样支持不了springboot中jar in jar的特性，因此springboot扩展了该协议，使其能够支持多个分隔符。

#### 流程

了解了这些基本概念后，我们开始具体的分析运行背后的原理。

##### 从jar中的MANIFEST.MF开始

首先贴出一个springboot项目jar包中的MANIFEST.MF文件

```
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: springboot-start-demo
Implementation-Version: 0.0.1-SNAPSHOT
Start-Class: com.leoli.springbootstartdemo.SpringbootStartDemoApplicat
 ion
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.3.4.RELEASE
Created-By: Maven Jar Plugin 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher


```

通过上面的介绍，我们看到该文件中有Main-Class属性，因此这个jar是一个可行性的jar。但是我们看到还有个属性是Start-Class，所指向的类正是我们springboot项目中不可缺少的*主类*（包含了一个main方法的启动类），因此可能很多同学容易误解java命令行启动的时候，所启动的是这个类。实际上并不是，java会从Main-Class中寻找main方法，因此java命令行启动的时候，对于任何springboot项目打成的jar包来说，启动类都是**org.springframework.boot.loader.JarLauncher**。

##### JarLauncher

贴出代码：

```java
public class JarLauncher extends ExecutableArchiveLauncher {

	private static final String DEFAULT_CLASSPATH_INDEX_LOCATION = "BOOT-INF/classpath.idx";

	static final EntryFilter NESTED_ARCHIVE_ENTRY_FILTER = (entry) -> {
		if (entry.isDirectory()) {
			return entry.getName().equals("BOOT-INF/classes/");
		}
		return entry.getName().startsWith("BOOT-INF/lib/");
	};

	public JarLauncher() {
	}

	protected JarLauncher(Archive archive) {
		super(archive);
	}

	@Override
	protected ClassPathIndexFile getClassPathIndex(Archive archive) throws IOException {
		// Only needed for exploded archives, regular ones already have a defined order
		if (archive instanceof ExplodedArchive) {
			String location = getClassPathIndexFileLocation(archive);
			return ClassPathIndexFile.loadIfPossible(archive.getUrl(), location);
		}
		return super.getClassPathIndex(archive);
	}

	private String getClassPathIndexFileLocation(Archive archive) throws IOException {
		Manifest manifest = archive.getManifest();
		Attributes attributes = (manifest != null) ? manifest.getMainAttributes() : null;
		String location = (attributes != null) ? attributes.getValue(BOOT_CLASSPATH_INDEX_ATTRIBUTE) : null;
		return (location != null) ? location : DEFAULT_CLASSPATH_INDEX_LOCATION;
	}

	@Override
	protected boolean isPostProcessingClassPathArchives() {
		return false;
	}

	@Override
	protected boolean isSearchCandidate(Archive.Entry entry) {
		return entry.getName().startsWith("BOOT-INF/");
	}

	@Override
	protected boolean isNestedArchive(Archive.Entry entry) {
		return NESTED_ARCHIVE_ENTRY_FILTER.matches(entry);
	}

	public static void main(String[] args) throws Exception {
		new JarLauncher().launch(args);
	}

}
```

非常简单，可以看到main方法里面先是实例化了一个JarLauncher，这会调用的父类ExecutableArchiveLauncher的构造方法：

```java
public ExecutableArchiveLauncher() {
   try {
      this.archive = createArchive();
      this.classPathIndex = getClassPathIndex(this.archive);
   }
   catch (Exception ex) {
      throw new IllegalStateException(ex);
   }
}
```

```java
protected final Archive createArchive() throws Exception {
   ProtectionDomain protectionDomain = getClass().getProtectionDomain();
   CodeSource codeSource = protectionDomain.getCodeSource();
   URI location = (codeSource != null) ? codeSource.getLocation().toURI() : null;
   String path = (location != null) ? location.getSchemeSpecificPart() : null;
   if (path == null) {
      throw new IllegalStateException("Unable to determine code source archive");
   }
   File root = new File(path);
   if (!root.exists()) {
      throw new IllegalStateException("Unable to determine code source archive from " + root);
   }
   return (root.isDirectory() ? new ExplodedArchive(root) : new JarFileArchive(root));
}
```

createArchive()会根据当前类(org.springframework.boot.loader.JarLauncher)来找到当前的jar，并创建出一个archive。回到launch方法：

```java
protected void launch(String[] args) throws Exception {
   if (!isExploded()) {
      JarFile.registerUrlProtocolHandler();
   }
   ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
   String jarMode = System.getProperty("jarmode");
   String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
   launch(args, launchClass, classLoader);
}

protected void launch(String[] args, String launchClass, ClassLoader classLoader) throws Exception {
		Thread.currentThread().setContextClassLoader(classLoader);
		createMainMethodRunner(launchClass, args, classLoader).run();
}


```

首先会注册自定义的URL处理器org.springframework.boot.loader.jar.Handler，getClassPathArchivesIterator方法则是将BOOT-INF/classes/和BOOT-INF/lib/下面的资源都作为archives，然后根据这些archives的urls创建一个自定义的LaunchedURLClassLoader来加载类，接着从MANIFEST.MF的Start-Class属性中找到程序的主类，利用这些通过反射来真正地run程序：

```java
public void run() throws Exception {
   Class<?> mainClass = Class.forName(this.mainClassName, false, Thread.currentThread().getContextClassLoader());
   Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
   mainMethod.setAccessible(true);
   mainMethod.invoke(null, new Object[] { this.args });
}
```

这就是通过java命令行运行一个springboot项目jar包的基本流程，下面解析下其中一些背后的细节。

##### org.springframework.boot.loader.jar.JarFile & org.springframework.boot.loader.jar.Handler & org.springframework.boot.loader.jar.JarURLConnection

这三个类都是springboot为了实现jar in jar而做的扩展。

- JarFile----组装的时候会把`!/`拼接起来
- Handler和JarURLConnection----Handler可以处理自定义jar协议的url，并返回JarURLConnection

其中Launcher类的createClassLoader方法会为每个archives的url创建一个Handler，在classloader加载资源时，JarURLConnection类的get方法会循环处理，最终返回正确的JarURLConnection。

###### 注册Handler

在launch方法中，`JarFile.registerUrlProtocolHandler()`就是用来注册自定义的Handler的，

那么这个Handler为什么需要这样注册呢？先看下这段注册的代码：

```java
private static final String PROTOCOL_HANDLER = "java.protocol.handler.pkgs";
private static final String HANDLERS_PACKAGE = "org.springframework.boot.loader";
public static void registerUrlProtocolHandler() {
   String handlers = System.getProperty(PROTOCOL_HANDLER, "");
   System.setProperty(PROTOCOL_HANDLER,
         ("".equals(handlers) ? HANDLERS_PACKAGE : handlers + "|" + HANDLERS_PACKAGE));
   resetCachedUrlHandlers();
}

private static void resetCachedUrlHandlers() {
		try {
			URL.setURLStreamHandlerFactory(null);
		}
		catch (Error ex) {
			// Ignore
		}
}
```

可以看到注册就是在系统属性中将java.protocol.handler.pkgs加上org.springframework.boot.loader。

我们知道Handler继承了URLStreamHandler，在java中我们可以通过以下方式获取：

1. 实现URLStreamHandlerFactory，然后调用URL.setURLStreamHandlerFactory(URLStreamHandlerFactory fac)。
2. 继承URLStreamHandler，springboot采用的就是这种方式，这种方式类全名必须是{path}.{protocol}.Handler，并且在系统属性java.protocol.handler.pkgs中设置包名前缀，如果有多个包名前缀，那么就用`|`分隔。

```java
static URLStreamHandler getURLStreamHandler(String protocol) {

    URLStreamHandler handler = handlers.get(protocol);
    if (handler == null) {

        boolean checkedWithFactory = false;

        // Use the factory (if any)
        if (factory != null) {
            handler = factory.createURLStreamHandler(protocol);
            checkedWithFactory = true;
        }

        // Try java protocol handler
        if (handler == null) {
            String packagePrefixList = null;

            packagePrefixList
                = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction(
                    protocolPathProp,""));
            if (packagePrefixList != "") {
                packagePrefixList += "|";
            }

            // REMIND: decide whether to allow the "null" class prefix
            // or not.
            packagePrefixList += "sun.net.www.protocol";

            StringTokenizer packagePrefixIter =
                new StringTokenizer(packagePrefixList, "|");

            while (handler == null &&
                   packagePrefixIter.hasMoreTokens()) {

                String packagePrefix =
                  packagePrefixIter.nextToken().trim();
                try {
                    String clsName = packagePrefix + "." + protocol +
                      ".Handler";
                    Class<?> cls = null;
                    try {
                        cls = Class.forName(clsName);
                    } catch (ClassNotFoundException e) {
                        ClassLoader cl = ClassLoader.getSystemClassLoader();
                        if (cl != null) {
                            cls = cl.loadClass(clsName);
                        }
                    }
                    if (cls != null) {
                        handler  =
                          (URLStreamHandler)cls.newInstance();
                    }
                } catch (Exception e) {
                    // any number of exceptions can get thrown here
                }
            }
        }

        synchronized (streamHandlerLock) {

            URLStreamHandler handler2 = null;

            // Check again with hashtable just in case another
            // thread created a handler since we last checked
            handler2 = handlers.get(protocol);

            if (handler2 != null) {
                return handler2;
            }

            // Check with factory if another thread set a
            // factory since our last check
            if (!checkedWithFactory && factory != null) {
                handler2 = factory.createURLStreamHandler(protocol);
            }

            if (handler2 != null) {
                // The handler from the factory must be given more
                // importance. Discard the default handler that
                // this thread created.
                handler = handler2;
            }

            // Insert this handler into the hashtable
            if (handler != null) {
                handlers.put(protocol, handler);
            }

        }
    }

    return handler;

}
```

由代码可知，获取URLStreamHandler会先从一个叫做handlers的Hashtable缓存中获取，如果没有，就会先看有没有`1`中实现的URLStreamHandlerFactory，如果还是没有，那么就会看java.protocol.handler.pkgs属性里面的Handler类，如果最终还是没有，就会用默认的sun.net.www.protocol包下面的Handler类，最后将handler放入handlers缓存中。这也是为什么springboot会在注册完以后调用来resetCachedUrlHandlers()，这个方法就是将URLStreamHandlerFactory清空，好让直接使用设置好的属性中的Handler。

##### LaunchedURLClassLoader

java的双亲委派机制只能加载在classpath下的jar包，所以只能自己定义一个classloader，而LaunchedURLClassLoader将会加载META-INF/classes和META-INF/lib下面的class的加载。LaunchedURLClassLoader主要是重写了loadClass方法：

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
   if (name.startsWith("org.springframework.boot.loader.jarmode.")) {
      try {
         Class<?> result = loadClassInLaunchedClassLoader(name);
         if (resolve) {
            resolveClass(result);
         }
         return result;
      }
      catch (ClassNotFoundException ex) {
      }
   }
   if (this.exploded) {
      return super.loadClass(name, resolve);
   }
   Handler.setUseFastConnectionExceptions(true);
   try {
      try {
         definePackageIfNecessary(name);
      }
      catch (IllegalArgumentException ex) {
         // Tolerate race condition due to being parallel capable
         if (getPackage(name) == null) {
            // This should never happen as the IllegalArgumentException indicates
            // that the package has already been defined and, therefore,
            // getPackage(name) should not return null.
            throw new AssertionError("Package " + name + " has already been defined but it could not be found");
         }
      }
      return super.loadClass(name, resolve);
   }
   finally {
      Handler.setUseFastConnectionExceptions(false);
   }
}
```

当走到MainMethodRunner类中的Class.forName方法里面后，加载流程是这样的（忽略部分步骤）：

1. LaunchedURLClassLoader.loadClass(String name, boolean resolve)
2. URLClassloader.findClass(String name)
3. URLClassPath.getResource(String var1, boolean var2)
4. URL.openConnection()
5. Handler.openConnection(URL url)
6. JarURLConnection.getInputStream()

### jar包是怎么生成的

#### maven插件

maven插件主要的作用是以maven组织的形式来运行maven阶段，即使是一个简单的clean命令，背后也是以插件的形式支持的。

##### mojo

全称是**Maven plain Old Java Object**，一个mojo对应maven里面要执行的一个goal，插件则是统一这些mojos，每个mojo都对应一个java类。其中有注解`@Mojo`可以标注这个类。

#### spring-boot-maven-plugin插件

这是springboot官方的maven插件，功能非常强大，主要是给springboot项目提供maven操作。下面会选择repackage操作来介绍。

##### repackage的打包流程

在使用中，我们使用`mvn clean package`就可以在target目录下打出一个可以使用java命令行的jar包，那么这背后的原理是什么呢？

在springboot插件源码中可以找到plugin.xml文件，再从中找到repackage的部分，这里部分展示出来：

```xml
<mojo>
      <goal>repackage</goal>
      <description>Repackage existing JAR and WAR archives so that they can be executed from the command
line using {@literal java -jar}. With &lt;code&gt;layout=NONE&lt;/code&gt; can also be used simply
to package a JAR with nested dependencies (and no main class, so not executable).</description>
      <requiresDependencyResolution>compile+runtime</requiresDependencyResolution>
      <requiresDirectInvocation>false</requiresDirectInvocation>
      <requiresProject>true</requiresProject>
      <requiresReports>false</requiresReports>
      <aggregator>false</aggregator>
      <requiresOnline>false</requiresOnline>
      <inheritedByDefault>true</inheritedByDefault>
      <phase>package</phase>
      <implementation>org.springframework.boot.maven.RepackageMojo</implementation>
      <language>java</language>
      <instantiationStrategy>per-lookup</instantiationStrategy>
      <executionStrategy>once-per-session</executionStrategy>
      <since>1.0.0</since>
      <requiresDependencyCollection>compile+runtime</requiresDependencyCollection>
      <threadSafe>true</threadSafe>
      ...
    </mojo>
```

可以看到，对于repackage这个goal，它是绑定在maven的package phase，也就是说只要mvn命令执行到package这个phase，就会执行其绑定的goals，当然包括了现在的repackage，其对应的mojo可以在`implementation`这个标签中看到：`org.springframework.boot.maven.RepackageMojo`：

```java
@Override
public void execute() throws MojoExecutionException, MojoFailureException {
   if (this.project.getPackaging().equals("pom")) {
      getLog().debug("repackage goal could not be applied to pom project.");
      return;
   }
   if (this.skip) {
      getLog().debug("skipping repackaging as per configuration.");
      return;
   }
   repackage();
}

private void repackage() throws MojoExecutionException {
   Artifact source = getSourceArtifact();
   File target = getTargetFile();
   Repackager repackager = getRepackager(source.getFile());
   Libraries libraries = getLibraries(this.requiresUnpack);
   try {
      LaunchScript launchScript = getLaunchScript();
      repackager.repackage(target, libraries, launchScript, parseOutputTimestamp());
   }
   catch (IOException ex) {
      throw new MojoExecutionException(ex.getMessage(), ex);
   }
   updateArtifact(source, target, repackager.getBackupFile());
}
```

核心的代码在21行，咱们接着看：

```java
public void repackage(File destination, Libraries libraries, LaunchScript launchScript, FileTime lastModifiedTime)
      throws IOException {
   Assert.isTrue(destination != null && !destination.isDirectory(), "Invalid destination");
   if (lastModifiedTime != null && getLayout() instanceof War) {
      throw new IllegalStateException("Reproducible repackaging is not supported with war packaging");
   }
   destination = destination.getAbsoluteFile();
   File source = getSource();
   if (isAlreadyPackaged() && source.equals(destination)) {
      return;
   }
   File workingSource = source;
   if (source.equals(destination)) {
      workingSource = getBackupFile();
      workingSource.delete();
      renameFile(source, workingSource);
   }
   destination.delete();
   try {
      try (JarFile sourceJar = new JarFile(workingSource)) {
         repackage(sourceJar, destination, libraries, launchScript, lastModifiedTime);
      }
   }
   finally {
      if (!this.backupSource && !source.equals(workingSource)) {
         deleteFile(workingSource);
      }
   }
}

private void repackage(JarFile sourceJar, File destination, Libraries libraries, LaunchScript launchScript,
      FileTime lastModifiedTime) throws IOException {
   try (JarWriter writer = new JarWriter(destination, launchScript, lastModifiedTime)) {
      write(sourceJar, libraries, writer);
   }
   if (lastModifiedTime != null) {
      destination.setLastModified(lastModifiedTime.toMillis());
   }
}

private void repackage(JarFile sourceJar, File destination, Libraries libraries, LaunchScript launchScript,
			FileTime lastModifiedTime) throws IOException {
		try (JarWriter writer = new JarWriter(destination, launchScript, lastModifiedTime)) {
			write(sourceJar, libraries, writer);
		}
		if (lastModifiedTime != null) {
			destination.setLastModified(lastModifiedTime.toMillis());
		}
}

protected final void write(JarFile sourceJar, Libraries libraries, AbstractJarWriter writer) throws IOException {
		Assert.notNull(libraries, "Libraries must not be null");
		WritableLibraries writeableLibraries = new WritableLibraries(libraries);
		if (this.layers != null) {
			writer = new LayerTrackingEntryWriter(writer);
		}
		writer.writeManifest(buildManifest(sourceJar));
		writeLoaderClasses(writer);
		writer.writeEntries(sourceJar, getEntityTransformer(), writeableLibraries);
		writeableLibraries.write(writer);
		if (this.layers != null) {
			writeLayerIndex(writer);
		}
}
```

很明显，57行-60行的代码分别就是向最终的jar包里面写入三个部分：

- MANIFEST.MF
- spring-boot-loader
- 项目的代码及其依赖的jar包

到这里其实还没有结束，执行过springboot打包命令的同学应该都可以注意到最终还多出了一个以.original结尾的文件，这实际上是mvn自己的package打出来的原始的jar包，springboot的插件把它重新命名了而已。我们解压出来可以看到，这个jar包里面只有项目的静态文件及class文件，以及只有基本属性的MANIFEST.MF文件等。

这个工作是在RepackageMojo类repackage()中的`updateArtifact(source, target, repackager.getBackupFile())`进行的，大家有兴趣可以自己看看，同样，关于springboot这个插件其他的goals，也可以按照这样的方式去看看。

#### 一些补充

##### 为什么我的项目中没有对插件进行配置，就可以直接运行repackage？

这是因为项目中我们pom文件中指定了parent：

```xml
<parent>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-parent</artifactId>
   <version>2.3.4.RELEASE</version>
   <relativePath/> <!-- lookup parent from repository -->
</parent>
```

跳转到spring-boot-starter-parent的pom文件中我们就可以看到里面也包含了spring-boot-maven-plugin插件的声明：

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution>
      <id>repackage</id>
      <goals>
        <goal>repackage</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <mainClass>${start-class}</mainClass>
  </configuration>
</plugin>
```

这就很清晰了，如果我们引用了springboot官方的parent，那么默认的goal就是repackage，也不用我们自己指定mainClass，如果不使用父pom，那么我们就需要自己配置。

##### 两种运行maven命令的方式

- 以phase为目标运行
- 以goal为目标运行

上面我们讨论的一直都是mvn package方式打包，这种方式就是以phase的运行，package是maven生命周期的一个阶段，因此运行我们可以直接运行这个阶段就行。

第二种就是直接精确到目标运行，你可以直接指定运行到目标，比如我们可以直接运行mvn spring-boot:repackage😜，然后你就会看到：

`Execution default-cli of goal org.springframework.boot:spring-boot-maven-plugin:2.3.4.RELEASE:repackage failed: Source file must not be null`

这个错误来自于这里：

```java
protected Packager(File source, LayoutFactory layoutFactory) {
   Assert.notNull(source, "Source file must not be null");
   Assert.isTrue(source.exists() && source.isFile(),
         "Source must refer to an existing file, got " + source.getAbsolutePath());
   this.source = source.getAbsoluteFile();
   this.layoutFactory = layoutFactory;
}
```

显然，在repackage之前我们没有maven默认打出的原始jar包，那么怎么办呢？只需要中间加一个package就好了：mvn package spring-boot:repackage。

因此我们得知，直接以goal为目标运行，会直接执行当前的goal，除了我们命令行前面指明了其他阶段，或者mojo类上有`@Execute`表明了具体的阶段。

## mvn spring-boot:run (ignore Gradle Plugin)

------

基于前文关于maven插件以及repackage的介绍，大家应该怎么这个启动方式是怎么回事了，这里直接就从RunMojo入手了。

### 流程

RunMojo并没有重写execute方法，所以我们找到了它的父类AbstractRunMojo：

```java
@Override
public void execute() throws MojoExecutionException, MojoFailureException {
   if (this.skip) {
      getLog().debug("skipping run as per configuration.");
      return;
   }
   run(getStartClass());
}

private void run(String startClassName) throws MojoExecutionException, MojoFailureException {
		boolean fork = isFork();
		this.project.getProperties().setProperty("_spring.boot.fork.enabled", Boolean.toString(fork));
		if (fork) {
			doRunWithForkedJvm(startClassName);
		}
		else {
			logDisabledFork();
			runWithMavenJvm(startClassName, resolveApplicationArguments().asArray());
		}
}

private void doRunWithForkedJvm(String startClassName) throws MojoExecutionException, MojoFailureException {
		List<String> args = new ArrayList<>();
		addAgents(args);
		addJvmArgs(args);
		addClasspath(args);
		args.add(startClassName);
		addArgs(args);
		runWithForkedJvm((this.workingDirectory != null) ? this.workingDirectory : this.project.getBasedir(), args,
				determineEnvironmentVariables());
}
```

非常清晰，启动参数添加agents，添加jvmArgs，添加classpath，然后加上startClassName，这个className如果goal没有指定，就会对应目录下去找有`org.springframework.boot.autoconfigure.SpringBootApplication`注解的类。接着到了runWithForkedJvm方法：

```java
protected void runWithForkedJvm(File workingDirectory, List<String> args, Map<String, String> environmentVariables)
      throws MojoExecutionException {
   int exitCode = forkJvm(workingDirectory, args, environmentVariables);
   if (exitCode == 0 || exitCode == EXIT_CODE_SIGINT) {
      return;
   }
   throw new MojoExecutionException("Application finished with exit code: " + exitCode);
}

private int forkJvm(File workingDirectory, List<String> args, Map<String, String> environmentVariables)
      throws MojoExecutionException {
   try {
      RunProcess runProcess = new RunProcess(workingDirectory, getJavaExecutable());
      Runtime.getRuntime().addShutdownHook(new Thread(new RunProcessKiller(runProcess)));
      return runProcess.run(true, args, environmentVariables);
   }
   catch (Exception ex) {
      throw new MojoExecutionException("Could not exec java", ex);
   }
}
```

getJavaExecutable()则是根据`java.home`到环境变量去找到java可执行文件，最终通过jdk自带的Process执行java命令，最终的命令大致如下：

`/Home/jre/bin/java -Xverify:none -XX:TieredStopAtLevel=1 -cp /project/springboot-start-demo/target/classes:/org/springframework/boot/spring-boot-starter-thymeleaf/2.3.4.RELEASE/spring-boot-starter-thymeleaf-2.3.4.RELEASE.jar:/org/springframework/boot/spring-boot-starter/2.3.4.RELEASE/spring-boot-starter-2.3.4.RELEASE.jar...... {your application start class} `

这个命令大家应该也是非常熟悉的，于是springboot项目就通过这种方式启动了。

### 一些补充

#### -Xverify:none -XX:TieredStopAtLevel=1

这两个参数主要用于加快编译速度，verify:none指不需要验证，TieredStopAtLevel=1则是禁用C2编译器。

#### 为什么mvn spring-boot:run不像mvn spring-boot:repackage报错

这个其实前文已经说过了，我们可以看到在RunMojo类上有这样的注解

```java
@Execute(phase = LifecyclePhase.TEST_COMPILE)
```

因此，在run之前，mvn会帮助我们执行完test-compile阶段，一切工作都准备好了。

#### mvn spring-boot:run与mvn spring-boot:start

关于这两个goal的不同之处，springboot官方已经给出了说明：https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/html/#goals，这里不多讲述，大家可以再根据前面介绍的相关知识分析下它们在命令执行的时候到底有哪些不同。

## shell方式运行真正可执行的jar包

------

使用shell方式启动可以说是非常冷门的启动springboot项目的方式，但是里面依然有不少值得探究的地方，因此我们还是来看下它背后的原理是怎么样的。

### 启动方式和流程

首先，通过repackage方式打出的jar包依然是需要的，然后我们可以通过`./{your application}.jar {command}`或者`sh {your application}.jar {command}`，这里的command可以是run/start/stop等，不是必须的。

现在可以试试😜，我们就会发现`cannot execute binary file`，这是怎么回事呢？

原来我们这时候的jar包仅仅是可供java命令行运行的文件，并不是真正的可执行文件，怎么变成真正的可执行文件？在前文repackage代码中，有一段我没有提及到，RepackageMojo中repackage()的`LaunchScript launchScript = getLaunchScript();`

```java
private LaunchScript getLaunchScript() throws IOException {
   if (this.executable || this.embeddedLaunchScript != null) {
      return new DefaultLaunchScript(this.embeddedLaunchScript, buildLaunchScriptProperties());
   }
   return null;
}
```

这里有两个参数：executable和embeddedLaunchScript，其中executable的默认值是false，embeddedLaunchScript默认文件跟踪进去发现是launch.script。因此如果不设置executable为true，这段代码就会return null。如果我们改成true，那么会返回一个DefaultLaunchScript，让我们接着看这个有什么用。

一路追踪到这里：

```java
//Repackager.java
private void repackage(JarFile sourceJar, File destination, Libraries libraries, LaunchScript launchScript,
      FileTime lastModifiedTime) throws IOException {
   try (JarWriter writer = new JarWriter(destination, launchScript, lastModifiedTime)) {
      write(sourceJar, libraries, writer);
   }
   if (lastModifiedTime != null) {
      destination.setLastModified(lastModifiedTime.toMillis());
   }
}
```

在实例化JarWriter的时候我们传了这个launchScript进去：

```java
public JarWriter(File file, LaunchScript launchScript, FileTime lastModifiedTime)
      throws FileNotFoundException, IOException {
   FileOutputStream fileOutputStream = new FileOutputStream(file);
   if (launchScript != null) {
      fileOutputStream.write(launchScript.toByteArray());
      setExecutableFilePermission(file);
   }
   this.jarOutputStream = new JarArchiveOutputStream(fileOutputStream);
   this.jarOutputStream.setEncoding("UTF-8");
   this.lastModifiedTime = lastModifiedTime;
}
```

关键地方来了，代码第四行开始中，如果launchScript不为null，就会将launchScript写入到jar文件中，然后给予这个jar真正的可执行权限。有了这两步，我们就可以用shell的方式运行jar文件了。

### 怎么让launchScript不为null？

大家还记得前文plugin与Mojo的对应关系吧，在plugin.xml中repackage这个goal有一个可以配置的参数，就是`executable`，因此，我们只需要在我们的pom文件中配置上即可：

```xml
<plugin>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-maven-plugin</artifactId>
   <configuration>
      <executable>true</executable>
   </configuration>
</plugin>
```

然后我们执行到repackage的时候，自然就可以打出可以shell执行的jar包了。

### 为什么可以这么做

我们先不急了解背后的原理，先看看这个默认的launchScript是什么。

#### launch.script

因为这里仅考虑启动，所以选取了脚步文件中关于启动方面的部分(其他诸如start, stop等命令可以用相同的方法分析)：

```shell
#!/bin/bash
...
# Determine the script mode
action="run"
...
# Find Java
if [[ -n "$JAVA_HOME" ]] && [[ -x "$JAVA_HOME/bin/java" ]]; then
    javaexe="$JAVA_HOME/bin/java"
elif type -p java > /dev/null 2>&1; then
    javaexe=$(type -p java)
elif [[ -x "/usr/bin/java" ]];  then
    javaexe="/usr/bin/java"
else
    echo "Unable to find Java"
    exit 1
fi

arguments=(-Dsun.misc.URLClassPath.disableJarChecking=true $JAVA_OPTS -jar "$jarfile" $RUN_ARGS "$@")

# Action functions
...
run() {
  pushd "$(dirname "$jarfile")" > /dev/null
  "$javaexe" "${arguments[@]}"
  result=$?
  popd > /dev/null
  return "$result"
}

# Call the appropriate action function
case "$action" in
start)
  start "$@"; exit $?;;
stop)
  stop "$@"; exit $?;;
force-stop)
  force_stop "$@"; exit $?;;
restart)
  restart "$@"; exit $?;;
force-reload)
  force_reload "$@"; exit $?;;
status)
  status "$@"; exit $?;;
run)
  run "$@"; exit $?;;
*)
  echo "Usage: $0 {start|stop|force-stop|restart|force-reload|status|run}"; exit 1;
esac
...
```

我们来分析下这个脚本是怎么运行的：

1. 首先脚本是用**/bin/sh**来解释执行；
2. `javaexe`存放来java的执行文件
3. `arguments`存放参数；
4. 有一个run方法，组装成java -jar形式的java命令行运行；
5. shell命令行后面的参数解析到对应的方法，代码中第四行告诉我们，如果不写具体的action，那么默认就是run。

脚本的内容比较容易理解，那么为什么将脚本内容写入一个可执行的jar里面就可以了呢？

#### 为什么写入脚本内容就可执行

这背后的原理在于linux允许添加一个**Binary Payload**到一个shell脚本中，当shell脚本运行时，将会提取这个**Binary Payload**，并用其执行任务。当我们使用shell执行jar包时，实际上会执行这个默认的脚步，然后jar包会被脚本提取到，用来执行。

#### 一些补充

上述只是最基本的分析，了解到可以使jar包可执行后，我们甚至可以将jar包作为服务安装到unix系统中。另外在使用默认脚本的时候，我们还可以替换其中的参数，做到可配置化。我们还可以使用自己的脚本去做一些更magic的事情，只需要配置<embeddedLaunchScript>为自己的脚本文件即可。

## 其他启动方式

------

在这一章中，我们将会介绍一些其他的启动方式，为什么不像前文那样一个一个详细的介绍呢，原因是这些启动方式背后包含的原理比较容易，这边捎带介绍下大家就能很容易理解。

### idea启动

这种方式大家最熟悉了，本地开发的时候必不可少，

`{your java path} -XX:TieredStopAtLevel=1 -noverify {some -D params} "-javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=58390:/Applications/IntelliJ IDEA.app/Contents/bin" -Dfile.encoding=UTF-8 -classpath {class and jar} com.leoli.springbootstartdemo.SpringbootStartDemoApplication`

针对这个命令在这里只提三点。

#### 如何加载到依赖jar包

很简单，idea已经将依赖的jar包放在了classpath下面。

#### idea_rt.jar这个javaagent做了什么

在真正的main方法启动之前，启动一个线程监听端口，如果接受到STOP消息，则退出。

#### 编译的class文件从哪里来的

在idea到configurations中，会有before launch的设置，这里的设置会是build，这个命令会在启动之前将项目的java代码编译成class文件。

### war包部署到servlet容器中

这个启动方式挺“老”的了，传统的部署方式，这里不做介绍。具体可参考官方文档。

### 热部署

这个平时本地开发的时候也会时有用到，具体请参考：https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#using-boot-hot-swapping

主要涉及的是字节码。

### Spring CLI

用的非常的少，需要安装Spring cli，具体操作参考：https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-cli.html#cli-using-the-cli

#### 原理及流程

org.springframework.boot.cli.SpringCli是总入口：

```java
public static void main(String... args) {
   System.setProperty("java.awt.headless", Boolean.toString(true));
   LogbackInitializer.initialize();

   CommandRunner runner = new CommandRunner("spring");
   ClassUtils.overrideThreadContextClassLoader(createExtendedClassLoader(runner));
   runner.addCommand(new HelpCommand(runner));
   addServiceLoaderCommands(runner);
   runner.addCommand(new ShellCommand());
   runner.addCommand(new HintCommand(runner));
   runner.setOptionCommands(HelpCommand.class, VersionCommand.class);
   runner.setHiddenCommands(HintCommand.class);

   int exitCode = runner.runAndHandleErrors(args);
   if (exitCode != 0) {
      // If successful, leave it to run in case it's a server app
      System.exit(exitCode);
   }
}
```

跟踪进runAndHandleErrors方法，我们以run为例，会一直跟踪到RunProcessCommand中的方法：

```java
@Override
public ExitStatus run(String... args) throws Exception {
   return run(Arrays.asList(args));
}

protected ExitStatus run(Collection<String> args) throws IOException {
   this.process = new RunProcess(this.command);
   int code = this.process.run(true, StringUtils.toStringArray(args));
   if (code == 0) {
      return ExitStatus.OK;
   }
   else {
      return new ExitStatus(code, "EXTERNAL_ERROR");
   }
}
```

`this.process.run()`点进去就是我们在**mvn spring-boot:run**章节提到过的方法，后面流程就是差不多的。

## 总结

------
本文的目的主要是介绍springboot中的一些启动方式以及背后的原理，希望自己与大家能够学到一些背后的知识并且重要的是能够在平时运用上，当然可能还有一些方式没有提到或者详细的介绍，比如前文提到的unix系统的服务启动，还有gradle启动等，这些大家可以自己去了解下，大抵上都逃不过本文已经介绍过的这几种所提到的根本方式。另本文形成仓促，如有不足，还请指正。
