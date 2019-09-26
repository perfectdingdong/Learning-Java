# Learning-Java
Java学习

动态编译：
编译是将源代码转换成机器码的过程，在java中是将java源代码转换成class字节码的过程，而不是真正的机器码，因为java使用虚拟机JVM运行java代码。

编译过程需要的工具：
编译工具（编译器）、待编译的源代码文件、源代码字节码文件的管理（靠文件系统，包括文件的创建和管理）、编译过程中的选项（代码版本，目标，源代码位置，classpath，编码）、编译中编译器输出的诊断信息。

上述工具的对应可编程接口对象：
编译器：JavaCompiler、JavaCompiler.CompilationTask、ToolProvider
在上面的接口和类中，ToolProvider类似是一个工具箱，它可以提供JavaCompiler类的实例并返回，然后该实例可以获取JavaCompiler.CompilationTask实例，然后由JavaCompiler.CompilationTask实例来执行对应的编译任务，其实这个执行过程是一个并发的过程。

源代码文件：FileObject、ForwardingFileObject、JavaFileObject、JavaFileObject.Kind、ForwardingJavaFileObject、SimpleJavaFileObject.
上述后面的4个接口和类都是FileObject子接口或者实现类，FIleObject接口代表了对文件的一种抽象，可以包括普通的文件，也可以包括数据库中的数据库等，其中规定了一些操作，包括读写操作，读取信息，删除文件等操作。我们要用的其实是JavaFileObject接口，其中还增加了一些操作Java源文件和字节码文件特有的API，而SimpleJavaFileObject是JavaFileObject接口的实现类，但是其中你可以发现很多的接口其实就是直接返回一个值，或者抛出一个异常，并且该类的构造器由protected修饰的，所以要实现复杂的功能，需要我们必须扩展这个类。ForwardingFileObject、ForwardingJavaFileObject类似，其中都是包含了对应的FileObject和JavaFileObject，并将方法的执行委托给这些对象，它的目的其实就是为了提高扩展性。

文件的创建和管理：JavaFileManager、JavaFileManager.Location、StandardJavaFileManager、ForwardingJavaFileManager、StandardLocation。
JavaFileManager用来创建JavaFileObject，包括从特定位置输出和输入一个JavaFileObject，ForwardingJavaFileManager也是出于委托的目的。而StandardJavaFileManager是JavaFileManager直接实现类，JavaFileManager.Location和StandardLocation描述的是JavaFileObject对象的位置，由JavaFileManager使用来决定在哪创建或者搜索文件。由于在javax.tools包下没有JavaFileManager对象的实现类，如果我们想要使用，可以自己实现该接口，也可以通过JavaCompiler类中的getStandardFileManager完成。

编辑选项的管理：OptionChecker。

诊断信息的收集：Diagnostic、DiagnosticListener、Diagnostic.Kind、DiagnosticCollector。
Diagnostic会输出编译过程中产生的问题，包括问题的信息和出现问题的定位信息，问题的类别则在Diagnostic.Kind中定义。DiagnosticListener则是从编译器中获取诊断信息，当出现诊断信息时则会调用其中的report方法，DiagnosticCollector则是进一步实现了DiagnosticListener，并将诊断信息收集到一个list中以便处理。

动态编译流程：
1.准备编译对象
JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
// ......
// 在其他实例都已经准备完毕后, 构建编译任务, 其他实例的构建见如下
Boolean result = compiler.getTask(null, manager, collector, options,null,Arrays.asList(javaFileObject));

2.诊断信息收集
// 初始化诊断收集器
DiagnosticCollector<JavaFileObject> collector = new DiagnosticCollector<>();
// ......
// 编译完成之后，获取编译过程中的诊断信息
collector.getDiagnostics().forEach(item -> System.out.println(item.toString()))

3.源代码文件对象的构建
public static class MyJavaFileObject extends SimpleJavaFileObject {
    private String source;
    private ByteArrayOutputStream outPutStream;
    // 该构造器用来输入源代码
    public MyJavaFileObject(String name, String source) {
        // 1、先初始化父类，由于该URI是通过类名来完成的，必须以.java结尾。
        // 2、如果是一个真实的路径，比如是file:///test/demo/Hello.java则不需要特别加.java
        // 3、这里加的String:///并不是一个真正的URL的schema, 只是为了区分来源
        super(URI.create("String:///" + name + Kind.SOURCE.extension), Kind.SOURCE);
        this.source = source;
    }
    // 该构造器用来输出字节码
    public MyJavaFileObject(String name, Kind kind){
        super(URI.create("String:///" + name + kind.extension), kind);
        source = null;
    }
 
    @Override
    public CharSequence getCharContent(boolean ignoreEncodingErrors){
        if(source == null){
            throw new IllegalArgumentException("source == null");
        }
        return source;
    }
 
    @Override
    public OutputStream openOutputStream() throws IOException {
        outPutStream = new ByteArrayOutputStream();
        return outPutStream;
    }
 
    // 获取编译成功的字节码byte[]
    public byte[] getCompiledBytes(){
        return outPutStream.toByteArray();
    }
}

4.文件管理器对象创建
JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
DiagnosticCollector<JavaFileObject> collector = new DiagnosticCollector<>();
// 该JavaFileManager实例是com.sun.tools.javac.file.JavacFileManager
JavaFileManager manager= compiler.getStandardFileManager(collector, null, null);

private static Map<String, JavaFileObject> fileObjects = new ConcurrentHashMap<>();
// 这里继承类，不实现接口是为了避免实现过多的方法
public static class MyJavaFileManager extends ForwardingJavaFileManager<JavaFileManager> {
        protected MyJavaFileManager(JavaFileManager fileManager) {
            super(fileManager);
        }
 
        @Override
        public JavaFileObject getJavaFileForInput(Location location, String className, JavaFileObject.Kind kind) throws IOException {
            JavaFileObject javaFileObject = fileObjects.get(className);
            if(javaFileObject == null){
                super.getJavaFileForInput(location, className, kind);
            }
            return javaFileObject;
        }
 
        @Override
        public JavaFileObject getJavaFileForOutput(Location location, String qualifiedClassName, JavaFileObject.Kind kind, FileObject sibling) throws IOException {
            JavaFileObject javaFileObject = new MyJavaFileObject(qualifiedClassName, kind);
            fileObjects.put(qualifiedClassName, javaFileObject);
            return javaFileObject;
        }
}

5.编译选项的选择
List<String> options = new ArrayList<>();
options.add("-target");
options.add("1.8");
options.add("-d");
options.add("/");
// 省略......
compiler.getTask(null, javaFileManager, collector, options, null, Arrays.asList(javaFileObject));

6.其他问题
想将编译完成的字节码输出为文件，也不需要上面自定义JavaFileManager，直接使用JavaCompiler提供的即可，而且在自定义的JavaFileObject中也不需要实现OpenOutStream这种方法，代替要提供options.add(“-d”)，options.add(“/”)等编译选项；如果不输出为文件按照上述的例子即可；
StandardLocation中的元素可以代替真实的路径位置，但是不会输出为文件，可以为一个内存中的文件；
在编译完成之后要将字节码文件加载进来，因此就要涉及到类加载机制，由于这也是一个很大的话题，所以后面会专门总结一篇，但是在这里还是要说明一下，由于上面编译时没有额外的依赖包，所以不用考虑加载依赖文件的问题，但是当如果有这样的需求时，我们可以利用类加载的委托机制，将依赖文件的加载全部交给父加载器去做即可。