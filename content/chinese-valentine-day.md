画个心吧！ 之前应该有看到别人画过，但是是什么时间，具体第一次在哪看过，来源已不可考。既然七夕快到了，自己写着玩一下吧。

![](static/love0.png)

原理是这样的，我们可以把输出当作比如 `50 * 100` 的点素点阵，这就是我们的 canvas 了。

    for y := 0; y < height; y++ {
        for x := 0; x < width; x++ {
            fmt.Printf("%c", '*')
        }
        fmt.Println()
    }

这是一个 x 轴往右， y 轴往下的坐标系。怎么样画一个函数呢？ `y = f(x)`，如果把 x 和 y 代入到函数里面，就 print 一个星号，否则 print 一个空格。我们可以写一个函数来判断：

    func point(x, y int) byte {
        if x == y {
            return '*'
        } 
        return ' '
    }
    
函数可以换成别的，这样就可以画出不同的曲线了。接下来是一个神奇的公式：

`(x^2 + y^2 - 1)^3 = x^2 * y^3`

这个函数画出来的图案，就是一个心。它的值域范围 x 和 y 轴大概都在 -1.5 到 1.5 左右，所以我们要把 `50 * 100` 的 canvas 缩放到这个范围。另外，坐标系需要调整到 canvas 的中心，而不左上角。y 轴反向之后：

    x => [0 ~ 100]
    -y => [-50 ~ 0]
    
调整坐标原点，以 canvas 为中心而不是左上角：

    x - 50 => [-50 ~ 50]
    25 - y => [-25 ~ 25]
    
再把范围缩放到 `-1.5 ~ 1.5` 之间：

    (x - 50) / 100 * 3 => [-1.5 ~ 1.5]
    (25 -y) / 50 * 3 => [-1.5 ~ 1.5]
    
于是我们就可以把整段代码写出来了：

    package main
    import "fmt"
    func main() {
        for y := 0; y < 50; y++ {
            for x := 0; x < 100; x++ {
                y1 := (25.0 - float32(y)) / float32(50) * 3
                x1 := (float32(x) - 50.0) / 100.0 * 3
                tmp := (x1*x1 + y1*y1 - 1)
                if tmp*tmp*tmp-x1*x1*y1*y1*y1 < 0 {
                    fmt.Printf("%c", '*')
                } else {
                    fmt.Printf("%c", ' ')
                }
            }
            fmt.Println()
        }
    }
    
效果是这样子的：

![](static/love1.png)

我们可以把星号换掉，随机生成一些字符:

![](static/love2.png)

可以用自己喜欢的妹子（或者汉子也行）的名字去随机，注意需要是 ascii 字符，中文字体宽度有问题，可以用 base64 编码一下。既然是内容，可以在里面写一些句子。唔，似乎用摩尔斯码也不错，只用 . 和 - 去写摩尔斯码。

继续说画心，我们可以画好几层的心，大小不一样，越靠内层的，"颜色" 越重，反正就是 canvas 输出当像素点阵用的：

![](static/love3.png)

做这一步需要注意两个点，一个是不同大小的心，缩放要调整一下，不能占满整个 canvas，而中心还是要对齐的。另一个是由于点的颜色有深度了，需要用数组先存起来，之后再画。颜色深度由浅入深可以依次用 `. ! + * #`

![](static/love4.png)

把代码融合进去也可以，像 C 语言代码混乱大赛那样玩。

![](static/love5.png)

甚至可以生成一段心形程序，发个密码给 TA 去解。或者藏一个 brain fuck 代码。对于一个 geek 来说，这都不算个事儿。

祝大家玩得开心！今天画个心，明天可以画点别的，后天再 new 一个女朋友，完美！(手动滑稽
