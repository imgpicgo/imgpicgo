# AndroidQ-DEX文件加载大致流程

## 前言

之前就一直想去看看这个东西，咕咕咕了好久，这次补上吧。

这篇文章可能写着写着就流水账起来了。。

## 怎么动态加载DEX

```
    private void try_load_dex() {
        String dexPath = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separator + "example.dex";
        File dexFile = new File(dexPath);
        if(dexFile.exists()){
            DexClassLoader dexClassLoader = new DexClassLoader(dexPath,null,null,getClassLoader());
            try {
                Class cls = dexClassLoader.loadClass("com.a5k1l.dexloader.example.example");
                Method method  = cls.getMethod("add",int.class,int.class,Context.class);
                int result = (int) method.invoke(null ,1,2,this);//invoke static method
                Toast.makeText(getApplicationContext(),"com from app "+result,Toast.LENGTH_SHORT).show();
            } catch (Exception e) {
                Toast.makeText(getApplicationContext(),"load Class failed!",Toast.LENGTH_SHORT).show();
                e.printStackTrace();
            }
        }else{
            Toast.makeText(getApplicationContext(),"dexFile don't exists!",Toast.LENGTH_SHORT).show();
        }

    }
```

## 源码

![image](https://raw.githubusercontent.com/imgpicgo/imgpicgo/master/img/clipboard.png)

这里可以看到这个类只是一个包装，具体则调用了`BaseDexClassLoader`

![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/f83f07b4c8dd4df1ab4a373aa86b43aa/clipboard.png)

之后又到了`DexPathList`

```
   DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
        ·······
        ·······
        
        this.definingContext = definingContext;

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);

        // Native libraries may exist in both the system and
        // application library paths, and we use this search order:
        //
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        this.nativeLibraryPathElements = makePathElements(getAllNativeLibraryDirectories());

        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
    }
```

接着主要是`makeDexElements`方法（代码比较多，之后只粘贴关键部分）

这个方法根据是否文件夹，文件后缀是否dex等进行分别处理调用到了`loadDexFile`,之后来到了这

![image](C:%5CUsers%5C5k1l%5CAppData%5CLocal%5CYNote%5Cdata%5Cm18364011937@163.com%5C4d38debad73940328e71c24626360216%5Cclipboard.png)

再之后就调用到了native函数 ![image](C:%5CUsers%5C5k1l%5CAppData%5CLocal%5CYNote%5Cdata%5Cm18364011937@163.com%5C556ec713865f401687d12310c4e91fea%5Cclipboard.png)![image](C:%5CUsers%5C5k1l%5CAppData%5CLocal%5CYNote%5Cdata%5Cm18364011937@163.com%5Ce66d59bee33d4b9399d55a2dbf5e5f0c%5Cclipboard.png)

跳到了这 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/c187358c3ab243e1b7820ffa2d74921b/clipboard.png)

下边的`CreateCookieFromOatFileManagerResult`没有对dex加载进行操作

来看`OpenDexFilesFromOat` 首先说一下大致流程,由于代码比较长，我们采用动态调试的方法进行分析

![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/8938d168ceb24e8488f4e0766e547b6d/clipboard.png)

首先一堆检查之后调用了`CreateContextForClassLoader`

之后初始化了一个`oat_file_assistant`

![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/e55edb46cc20452a8952523b20ddf48a/clipboard.png)

他做了什么呢，通过动态调试，我们发现他通过传入的dexlocation生成了一个路径 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/c6eb60d2416042359cf2b3f2d7556da8/clipboard.png)并且会去检测父路径是否可写 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/628e7307b05c46418a65760b15ab0f52/clipboard.png)`GetBestOatFile`的操作 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/440fc0981fcc486a8eb032238ed7d098/clipboard.png)

之后对于我们的这种加载dex方式跳转到了这里，正式开始加载dex文件 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/84aadb27c52e4e67a4967b641f44e313/clipboard.png)这个Open做了什么呢 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/d9adedbf56144672b146f5ba2ede3453/clipboard.png)他根据文件magic判断是哪种文件 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/0ec0206b62304489a6554e1bb1712c52/clipboard.png)这里的Openfile将dex文件映射到了内存当中![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/093095598a784872b75f8fbd3116eeb5/clipboard.png)并将其转成DexFile对象，如果加载成功，就将加载的dexfile放到dex_files这个vector中 之后做了一些check将dexfile返回了![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/a0efc2e37ba14e34b282d02947b305d3/clipboard.png)再之后就是转成cookie返回到java层了 ![image](C:/Users/5k1l/AppData/Local/YNote/data/m18364011937@163.com/c187358c3ab243e1b7820ffa2d74921b/clipboard.png)至此dex算是加载完了。。。可是我还没有看懂。。。只是大略的看了一遍，很多细节都直接略过了

## 参考

[Android Q Source](https://cs.android.com/android/platform/superproject/+/android-10.0.0_r30:)

[老罗的Android之旅（总结）](https://www.kancloud.cn/alex_wsc/androids)

[入门ART虚拟机(1)——加载DEX文件](https://www.jianshu.com/p/20dcfcf27004)

[入门ART虚拟机(2)——加载DEX文件续](https://www.jianshu.com/p/b89d0b03e82c)

[入门ART虚拟机(3)——加载类和方法](https://www.jianshu.com/p/61ff20b53e40)

[入门ART虚拟机(4)——加载类和方法续](https://www.jianshu.com/p/fa4bf6c210b8)

[通过一款早期代码抽取壳入门学习 so 层分析](https://bbs.pediy.com/thread-260251.htm)