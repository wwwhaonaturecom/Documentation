# Dynamic loading

Users who switch from PHP to C++ often ask whether it is more difficult to manage a system if you use C++ code instead of PHP code. We must be honest here: working with PHP is easier than working with C++. To activate a PHP script, you for example don't need root access, and you can simply copy the script to the web server. Deploying a native C++ extension requires more work than that: you need to stop the webserver first, compile the extension, install it, and then restart the web server.

On top of that, when an extension is deployed it is immediately active for all websites that are hosted on the webserver. A deployed PHP script only changes the behavior of a single website, but a deployed C++ extension affects all sites. It is not really possible to activate an extension for only specific sites, or to test a new version of an extension for just a single website, as the extensions are shared by all PHP processes. If you really want to use different extensions for different sites, you need multiple servers that all have their own configuration.

Or you can use dynamic loading.

PHP has a builtin [dl()](http://php.net/dl) function that you can use to load extensions on the fly. This allows you to call the 'dl("myextension.so")' function from a PHP script to load an extension, so that an extension only works for a specific site. This builtin 'dl()' function has some restrictions because of security considerations (it would otherwise allow users to run arbitrary native code), but if you're the only one who is in charge of a system, or when a server is not shared by multiple organizations, you can use PHP-CPP to create a function similar to 'dl()' that does not have this restriction.

## Why is dl() restricted?

The dl() function is restricted because of security issues. When you use dl(), you can only load extensions that are stored in the system wide extensions directory, and not for loading extensions that users have placed in other locations. A call to dl("/home/user/myextension.so") will therefore fail, because "/home/user" is not the official extensions directory. Why this restriction?

To understand this, one must first realize that in a normal PHP installation PHP scripts are edited by users who do not have root access. On shared hosting environments different users all run their own website on the same system. In such a setup, it is an absolute no-go if one user can write a script or program that has access to the data of others. With an unrestricted dl() function however, exactly this would be possible. An unrestricted dl() call would allow PHP programmers to write a native extension, store that in their home directory or in the /tmp directory, and have it loaded by the webserver process. They could then execute arbitrary code, and possibly install loggers or other malicious code inside other people's websites. By only allowing extensions from the system wide extensions directory to be loaded, PHP ensures every dynamically loaded extension must at least have been installed by the system administrator.

When you write your own extensions however - either directly on top of the Zend API, or by using the PHP-CPP library - you already are in a position to write and execute arbitrary code anyway. No need for security checks here. From you C/C++ code you can do whatever you want. Would it not be cool if you could dynamically load an extension based on the requirements of a site? One website needs to be stable, and loads the well tested version 1.0 of your extension, while a second website is more experimental, and loads version 2.0\. You could be running two versions of your extension on the same machine.

## Thin loader extension

Imagine you're writing your own extension "MyExtension" that has many different classes and functions, and for which you plan to bring out new releases all the time. You do not want to deploy a new release in a "big bang" style, but you want to roll out new versions slowly, one customer or one website at a time. How would you do this?

You start by developing a thin loader extension: the ExtensionLoader extension. This extension has only one function: `enable_my_extension()` - which takes the version number of your actual extension, and dynamically loads that version of your extension. This is a simple extension (it only has one function) that you install globally, and that you will probably never have to update.

```cpp
    /**
     *  Function to load an extension by its version number
     *
     *  It takes one argument: the version number of your extension,
     *  and returns a boolean to indicate whether the extension was 
     *  correctly loaded.
     *
     *  @param  params      Vector of parameters
     *  @return boolean
     */
    Php::Value enable_my_extension(Php::Parameters &params)
    {
        // get version number
        int version = params[0];

        // construct pathname to your extension (this is for example
        // /path/to/MyExtension.so.1 or /path/to/MyExtension.so.2)
        std::string path = "/path/to/MyExtension.so." + std::to_string(version);

        // load the extension
        return Php::dl(path);
    }

    /**
     *  Switch to C context to ensure that the get_module() function
     *  is callable by C programs (which the Zend engine is)
     */
    extern "C" {
        /**
         *  Startup function that is called by the Zend engine 
         *  to retrieve all information about the extension
         *  @return void*
         */
        PHPCPP_EXPORT void *get_module() {
            // create static instance of the extension object
            static Php::Extension myExtension("ExtensionLoader", "1.0");

            // the extension has one method
            myExtension.add("enable_my_extension", enable_my_extension, {
                Php::ByVal("version", Php::Type::Numeric)
            });

            // return the extension
            return myExtension;
        }
    }
```

The above code holds the full source code of the ExtensionLoader extension. You can install this extension on your system, by copy'ing it to the global php extensions directory and updating the php.ini files.

After you've installed this thin loader extension, you can write your actual big extension full with classes and functions, and compile this extension into *.so files: the first version you compile into MyExtension.so.1, and later versions into MyExtension.so.2, MyExtension.so.3, and so on. For every new release you introduce a new version number, and you copy these shared objects to the /path/to directory (the same path hardcoded in the 'loader' extension showed above). And although this is not the official PHP extensions directory, such extensions can nevertheless be loaded by the `enable_my_extension()` function.

You do not have to copy the extensions into the PHP extension directory, nor do you have to update the php.ini configuration. To activate an extension, you simply need to call the `enable_my_extension()` function that was introduced but he thin loader:

```php
    <?php
    // enable version 2 of the extension (this will load MyExtension.so.2)
    if (!enable_my_extension(2)) die("Version 2 of extension is missing");

    // from now on we can use classes and functions from version 2 of the extension
    $object = new ClassFromMyExtension();
    $object->methodFromMyExtension();

    // you get the idea...

    ?>
```

The thin loader that we showed above is still pretty secure. It is not possible to run arbitrary code, or to open arbitrary `*.so` files. The worst thing that can happen is that someone opens an extension with a wrong version number - which is not disastrous at all.

But if you really trust the users on your system, you can easily adjust the thin loader extension to allow other types of parameters too. In the most open scenario, you could even write a function that allows users to literally open every possible shared object file:

```cpp
    /**
     *  Function to load every possible extension by pathname
     *
     *  It takes one argument: the filename of the PHP extension, and returns a 
     *  boolean to indicate whether the extension was correctly loaded.
     *
     *  This function goes further than the original PHP dl() fuction, because
     *  it does not check whether the passed in extension object is stored in the
     *  right directory. Literally every possible extension, also local ones 
     *  created by end users, can be loaded.
     *
     *  @param  params      Vector of parameters
     *  @return boolean
     */
    Php::Value dl_unrestricted(Php::Parameters &params)
    {
        // get extension name
        std::string pathname = params[0];

        // load the extension
        return Php::dl(pathname);
    }

    /**
     *  Switch to C context to ensure that the get_module() function
     *  is callable by C programs (which the Zend engine is)
     */
    extern "C" {
        /**
         *  Startup function that is called by the Zend engine 
         *  to retrieve all information about the extension
         *  @return void*
         */
        PHPCPP_EXPORT void *get_module() {
            // create static instance of the extension object
            static Php::Extension myExtension("load_extension", "1.0");

            // the extension has one method
            myExtension.add("dl_unrestricted", dl_unrestricted, {
                Php::ByVal("pathname", Php::Type::String)
            });

            // return the extension
            return myExtension;
        }
    }
```

The above code would allow PHP scripts to dynamically load PHP extensions, no matter where they are stored on the system:

```php
    <?php
        // load the C++ extension stored in the same directory as this file
        if (!dl_unrestricted(__DIR__.'/MyExtension.so')) die("Extension could not be loaded");

        // from now on we can use classes and functions from the extension
        $object = new ClassFromMyExtension();
        $object->methodFromMyExtension();

    ?>
```
The `dl_unrestricted()` function is an awesome function, but be careful here: if you're the administrator of a shared hosting platform, you definitely do not want to install it!

## Persistent extensions

A dynamically loaded extension is automatically unloaded at the end of the page view. If a subsequent pageview also dynamically loads the same extension, it will start with a completely new and fresh environment. If you want to write an extension that uses static data or static resources (think of a persistent database connection, or a worker thread that processes tasks) this may not always be the desired behavior. You want to keep the database connection active, or the thread running, also after the extension is unloaded.

To overcome this, the `Php::dl()` function comes with a second boolean parameter that you can use to specify whether you want to load the extension persistently, or only for that specific pageview.

Notice that if you set this parameter to true, the only thing that persists is the data _in_ an extension. In subsequent pageviews you will still have to load the extension to activate the functions and classes in it, even if you had already loaded the extension persistently in an earlier pageview. But because the extension was already loaded before, the static data in it (like the database connection or the thread) is preserved.

The `dl_unrestricted()` function that we demonstrated above can be modified to include this persistent parameter:

```cpp
    /**
     *  Function to load every possible extension by pathname
     *
     *  It takes two arguments: the filename of the PHP extension, and a boolean to
     *  specify whether the extension data should be kept in memory. It returns a 
     *  boolean to indicate whether the extension was correctly loaded.
     *
     *  This function goes further than the original PHP dl() fuction, because
     *  it does not check whether the passed in extension object is stored in the
     *  right directory, and because it allows persistent loading of extensions. 
     *  Literally every possible extension, also local ones created by end users, 
     *  can be loaded.
     *
     *  @param  params      Vector of parameters
     *  @return boolean
     */
    Php::Value dl_unrestricted(Php::Parameters &params)
    {
        // get extension name
        std::string pathname = params[0];

        // persistent setting
        bool persistent = params.size() > 1 ? params[1].boolValue() : false;

        // load the extension
        return Php::dl(pathname, persistent);
    }

    /**
     *  Switch to C context to ensure that the get_module() function
     *  is callable by C programs (which the Zend engine is)
     */
    extern "C" {
        /**
         *  Startup function that is called by the Zend engine 
         *  to retrieve all information about the extension
         *  @return void*
         */
        PHPCPP_EXPORT void *get_module() {
            // create static instance of the extension object
            static Php::Extension myExtension("load_extension", "1.0");

            // the extension has one method
            myExtension.add("dl_unrestricted", dl_unrestricted, {
                Php::ByVal("pathname", Php::Type::String),
                Php::ByVal("persistent", Php::Type::Bool, false)
            });

            // return the extension
            return myExtension;
        }
    }
```

And this extension allows us to dynamically load extensions, while preserving persistent data _inside_ the extension:

```php
    <?php
    // load the C++ extension stored in the same directory as this file, the
    // extension is persistently loaded, so it may use persistent data like
    // database connections and so on.
    if (!dl_unrestricted(__DIR__.'/MyExtension.so', true)) die("Extension could not be loaded");

    // from now on we can use classes and functions from the extension
    $object = new ClassFromMyExtension();
    $object->methodFromMyExtension();

    ?>
```