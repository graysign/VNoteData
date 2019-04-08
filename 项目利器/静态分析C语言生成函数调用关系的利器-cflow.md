&emsp;&emsp;**cflow的说明和安装**  
&emsp;&emsp;cflow是一款静态分析C语言代码的工具，通过它可以生成函数的调用关系。和calltree不一样，cflow有独立的网页介绍它（https://www.gnu.org/software/cflow/#TOCdocumentation）。而且在Ubuntu系统上，我们可以不用去编译cflow的源码，而直接使用下面命令获取  
```bash
apt-get install cflow  
```

&emsp;&emsp;**cflow的使用**  
&emsp;&emsp;安装完毕，我们可以使用下面指令看到cflow的参数说明  
```bash
cflow --help  
```
&emsp;&emsp;我们可以得到如下提示  
```bash
Usage: cflow [OPTION...] [FILE]...  
generate a program flowgraph  
  
 General options:  
  -d, --depth=NUMBER         Set the depth at which the flowgraph is cut off  
  -f, --format=NAME          Use given output format NAME. Valid names are  
                             `gnu' (default) and `posix'  
  -i, --include=CLASSES      Include specified classes of symbols (see below).  
                             Prepend CLASSES with ^ or - to exclude them from  
                             the output  
  -o, --output=FILE          Set output file name (default -, meaning stdout)  
  -r, --reverse              * Print reverse call tree  
  -x, --xref                 Produce cross-reference listing only  
  
 Symbols classes for --include argument  
  
    _                        symbols whose names begin with an underscore  
    s                        static symbols  
    t                        typedefs (for cross-references only)  
    x                        all data symbols, both external and static  
  
 Parser control:  
  
  -a, --ansi                 * Accept only sources in ANSI C  
  -D, --define=NAME[=DEFN]   Predefine NAME as a macro  
  -I, --include-dir=DIR      Add the directory DIR to the list of directories  
                             to be searched for header files.  
  -m, --main=NAME            Assume main function to be called NAME  
  -p, --pushdown=NUMBER      Set initial token stack size to NUMBER  
      --preprocess[=COMMAND], --cpp[=COMMAND]  
                             * Run the specified preprocessor command  
  -s, --symbol=SYMBOL:[=]TYPE   Register SYMBOL with given TYPE, or define an  
                             alias (if := is used). Valid types are: keyword  
                             (or kw), modifier, qualifier, identifier, type,  
                             wrapper. Any unambiguous abbreviation of the above  
                             is also accepted  
  -S, --use-indentation      * Rely on indentation  
  -U, --undefine=NAME        Cancel any previous definition of NAME  
  
 Output control:  
  
  -b, --brief                * Brief output  
      --emacs                * Additionally format output for use with GNU  
                             Emacs  
  -l, --print-level          * Print nesting level along with the call tree  
      --level-indent=ELEMENT Control graph appearance  
  -n, --number               * Print line numbers  
      --omit-arguments       * Do not print argument lists in function  
                             declarations  
      --omit-symbol-names    * Do not print symbol names in declaration strings  
                              
  -T, --tree                 * Draw ASCII art tree  
  
 Informational options:  
  
      --debug[=NUMBER]       Set debugging level  
  -v, --verbose              * Verbose error diagnostics  
  
  -?, --help                 give this help list  
      --usage                give a short usage message  
  -V, --version              print program version  
  
Mandatory or optional arguments to long options are also mandatory or optional  
for any corresponding short options.  
  
* The effect of each option marked with an asterisk is reversed if the option's  
long name is prefixed with `no-'. For example, --no-cpp cancels --cpp.  
  
Report bugs to <bug-cflow@gnu.org>.  
```

&emsp;&emsp;我只列出我觉得有意思的几个参数:  
&emsp;&emsp;-T输出函数调用树状图  
&emsp;&emsp;-m指定需要分析的函数名  
&emsp;&emsp;-n输出函数所在行号  
&emsp;&emsp;-r输出调用的反向关系图  
&emsp;&emsp;--cpp预处理，这个还是很重要的  

&emsp;&emsp;**文本输出**  
&emsp;&emsp;最简单的使用方法是以ASCII文本的方式输出结果，比如  
```bash
cflow -T -m main -n timer.c  
```
&emsp;&emsp;其结果是一个包含文件名和函数所在代码行号的调用关系图
```bash
+-main() <int main (void) at timer.c:13>  
  +-ev_timer_init()  
  +-timeout_cb() <void timeout_cb (EV_P_ ev_timer *w, int revents) at timer.c:7>  
  | +-puts()  
  | \-ev_break()  
  +-ev_timer_start()  
  \-ev_run()  
```
&emsp;&emsp;然而，对于有一定代码量的项目，我们不会使用ASCII文本的方式去查看函数调用关系，因为调用是相当复杂的，而文本图并不适合人去理解。于是我们希望能cflow能产出一个可供其他软件转换成图片的格式的文件。可惜cflow并不支持，好在网上有开发者做了一个工具，可将其结果转换成dot格式。  

&emsp;&emsp;**转成dot文件**  
&emsp;&emsp;我们可以使用下面方法获取转换工具  
```bash
wget -c https://github.com/tinyclub/linux-0.11-lab/raw/master/tools/tree2dotx  
```
&emsp;&emsp;下载完tree2dotx后，可对其做个软链便于使用  
```bash
cd /usr/bin  
ln -s 【Your Path】/tree2dotx tree2dotx  
```
&emsp;&emsp;具体的转换方法是  
```bash
cflow -T -m main -n timer.c > main.txt  
cat main.txt | tree2dotx > main.dot  
```
&emsp;&emsp;**dot文件生成图片**  
&emsp;&emsp;我们需要借助graphviz（没有安装的可以使用apt-get install graphviz先安装）生成图片，指令是  
```bash
dot -Tgif main.dot -o main.gif    
```
![](_v_images/_1521458756_11997.png)
