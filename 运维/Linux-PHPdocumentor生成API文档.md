PHPDocumentor是一个用PHP写的工具，对于有规范注释的php程序，它能够快速生成具有相互参照，索引等功能的API文档。老的版本是[phpdoc](http://baike.baidu.com/view/1608868.htm)，从1.3.0开始，更名为phpDocumentor，新的版本加上了对[php5](http://baike.baidu.com/subview/16710/16710.htm)语法的支持，同时，可以通过在客户端浏览器上操作生成文档，文档可以转换为[PDF](http://baike.baidu.com/subview/15963/15963.htm),HTML,CHM几种形式，非常的方便。

phpDocumentor是一个非常强大的文档自动生成工具，利用它可以帮助我们编写规范的注释，生成易于理解，结构清晰的文档，对我们的代码升级，维护,移交等都有非常大的帮助。

## 软件安装编辑

phpDocumentor和其他pear下的模块一样，phpDocumentor的安装也分为自动安装和手动安装两种方式，两种方式都非常方便：

a． 通过pear 自动安装

在命令行下输入

pear install PhpDocumentor

b． 手动安装

在http://manual.phpdoc.org/下载最新版本的PhpDocumentor（现在是1.4.0），把内容解压即可。

## 生成文档编辑

命令行方式：

在phpDocumentor所在目录下，输入

[phpdoc](http://baike.baidu.com/view/1608868.htm) –h

会得到一个详细的参数表，其中几个重要的参数如下：

-f 要进行分析的文件名，多个文件用逗号隔开

-d 要分析的目录，多个目录用逗号分割

-t 生成的文档的存放路径

-o 输出的文档格式，结构为输出格式：转换器名：模板目录。

例如：phpdoc -o HTML:frames:earthli -f test.php -t docs

Web界面生成

在新的[phpdoc](http://baike.baidu.com/view/1608868.htm)中，除了在命令行下生成文档外，还可以在客户端[浏览器](http://baike.baidu.com/view/7718.htm)上操作生成文档，具体方法是先把PhpDocumentor的内容放在[apache](http://baike.baidu.com/subview/28283/5418753.htm)目录下使得通过浏览器可以访问到，访问后显示如下的界面：

点击files按钮，选择要处理的php文件或文件夹，还可以通过该指定该界面下的Files to ignore来忽略对某些文件的处理。

然后点击output按钮来选择生成文档的存放[路径](http://baike.baidu.com/view/59642.htm)和格式.

最后点击create，phpdocumentor就会自动开始生成文档了，最下方会显示生成的进度及状态，如果成功，会显示

Total Documentation Time: 1 seconds

done

Operation Completed!!

然后，我们就可以通过查看生成的文档了，如果是pdf格式的，名字默认为documentation.pdf。