### Tomcat的初始化--init

#### Bootstrap入口

```java
//整个Tomcat的入口，在这之前会先调用静态代码块
public static void main(String args[]) {
    synchronized (daemonLock) {
        if (daemon == null) {
            // Don't set daemon until init() has completed
            Bootstrap bootstrap = new Bootstrap();
            try {
                //调用初始化方法
                bootstrap.init();
            } catch (Throwable t) {
                handleThrowable(t);
                t.printStackTrace();
                return;
            }
            daemon = bootstrap;
        } else {
            // When running as a service the call to stop will be on a new
            // thread so make sure the correct class loader is used to
            // prevent a range of class not found exceptions.
            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
        }
    }

    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }

        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            //先load再start
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            //调用停止方法
            daemon.stop();
        } else if (command.equals("start")) {
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
            if (null == daemon.getServer()) {
                System.exit(1);
            }
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null == daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        // Unwrap the Exception for clearer error reporting
        if (t instanceof InvocationTargetException &&
            t.getCause() != null) {
            t = t.getCause();
        }
        handleThrowable(t);
        t.printStackTrace();
        System.exit(1);
    }
}
```

#### Bootstrap的静态代码块

```java
//静态代码块的主要功能是设置工作目录catalinaBaseFile和安装目录catalinaHomeFile
static {
    // Will always be non-null
    String userDir = System.getProperty("user.dir");

    // Home first
    String home = System.getProperty(Globals.CATALINA_HOME_PROP);
    File homeFile = null;

    if (home != null) {
        File f = new File(home);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    if (homeFile == null) {
        // First fall-back. See if current directory is a bin directory
        // in a normal Tomcat install
        File bootstrapJar = new File(userDir, "bootstrap.jar");

        if (bootstrapJar.exists()) {
            File f = new File(userDir, "..");
            try {
                homeFile = f.getCanonicalFile();
            } catch (IOException ioe) {
                homeFile = f.getAbsoluteFile();
            }
        }
    }

    if (homeFile == null) {
        // Second fall-back. Use current directory
        File f = new File(userDir);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    catalinaHomeFile = homeFile;
    System.setProperty(
        Globals.CATALINA_HOME_PROP, catalinaHomeFile.getPath());

    // Then base
    String base = System.getProperty(Globals.CATALINA_BASE_PROP);
    if (base == null) {
        catalinaBaseFile = catalinaHomeFile;
    } else {
        File baseFile = new File(base);
        try {
            baseFile = baseFile.getCanonicalFile();
        } catch (IOException ioe) {
            baseFile = baseFile.getAbsoluteFile();
        }
        catalinaBaseFile = baseFile;
    }
    System.setProperty(
        Globals.CATALINA_BASE_PROP, catalinaBaseFile.getPath());
}
```

#### Bootstrap中的init方法

```java
//这里的主要功能是初始化了三个类加载器和创建了Catalina对象
    //ClassLoader commonLoader = null;
    //ClassLoader catalinaLoader = null;
    //ClassLoader sharedLoader = null;
public void init() throws Exception {
	//初始化三个类加载器
    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    //通过反射创建Catalina对象
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();

    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    //为Catalina对象设置类加载器sharedLoader
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);

    catalinaDaemon = startupInstance;
}
```

* 回到main入口方法继续看

#### Bootstrap中的load方法

````java
//这里是通过反射转调Catalina对象的load方法
private void load(String[] arguments) throws Exception {

    // Call the load() method
    String methodName = "load";
    Object param[];
    Class<?> paramTypes[];
    if (arguments==null || arguments.length==0) {
        paramTypes = null;
        param = null;
    } else {
        paramTypes = new Class[1];
        paramTypes[0] = arguments.getClass();
        param = new Object[1];
        param[0] = arguments;
    }
    Method method =
        catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    if (log.isDebugEnabled()) {
        log.debug("Calling startup class " + method);
    }
    method.invoke(catalinaDaemon, param);
}
````

#### Catalina中的load方法

```java
public void load() {

    if (loaded) {
        return;
    }
    loaded = true;

    long t1 = System.nanoTime();
	//初始化temp目录
    initDirs();

    // Before digester - it may be needed
    initNaming();

    // Create and execute our Digester
    Digester digester = createStartDigester();

    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        try {
            file = configFile();
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail", file), e);
            }
        }
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource(getConfigFile()).toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                                           getConfigFile()), e);
                }
            }
        }

        // This should be included in catalina.jar
        // Alternative: don't bother with xml, just create it manually.
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource("server-embed.xml").toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                                           "server-embed.xml"), e);
                }
            }
        }


        if (inputStream == null || inputSource == null) {
            if  (file == null) {
                log.warn(sm.getString("catalina.configFail",
                                      getConfigFile() + "] or [server-embed.xml]"));
            } else {
                log.warn(sm.getString("catalina.configFail",
                                      file.getAbsolutePath()));
                if (file.exists() && !file.canRead()) {
                    log.warn("Permissions incorrect, read permission is not allowed on the file.");
                }
            }
            return;
        }

        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        } catch (SAXParseException spe) {
            log.warn("Catalina.start using " + getConfigFile() + ": " +
                     spe.getMessage());
            return;
        } catch (Exception e) {
            log.warn("Catalina.start using " + getConfigFile() + ": " , e);
            return;
        }
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }

    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    // Stream redirection
    initStreams();

    // Start the new server
    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error("Catalina.start", e);
        }
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {
        log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
    }
}
```



