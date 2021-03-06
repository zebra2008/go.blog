相比于写 C，写 Go 的时候一个很方便的地方在于，panic 里面很容易弄到栈信息。比如 defer 然后 recover 一下。出现空指针之类的问题在 Go 里面就比较好查。
在 C 里面就不太方便拿到当时的栈信息，只有挂了之后 gdb 去看看尸体，远不如 Go 对用户友好。

那么问题来了，捕获 panic 然后打印栈，在 Go 里面是如何实现的呢？

看了一下其实并不复杂，在 `runtime/signal_unix.go` 这个文件里面，runtime 注册了 unix 信号处理，比如发生内存错误的时候，进程会收到 sigsegv 信号，这类信号都是可以捕获的，只有像 sigkill 没法捕获。
捕获信号之后，就可以继续抛出异常，又或者是有 defer recover 的处理。

至于栈的信息，这类在 C 语言里面可能通过 ebp 一级一级的往上追，就可以获取到，不过要映射到具体的函数需要 debug 信息，所以像 gdb 之类的 debug 工具容易处理一些。
Go 里面编译就没有干掉 ELF 里面的调试信息的，生成的 binary 巨大，不过在追踪栈信息会比较方便。

