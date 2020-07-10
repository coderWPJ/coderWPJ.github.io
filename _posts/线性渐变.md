## <center>iOS开发 — 线性渐变
>轴向颜色梯度（有时也称为线性颜色梯度）由两个点指定，并且在每个点处指定颜色。 沿着通过那些点的线的颜色使用线性插值计算，然后垂直于该线延伸。
><p align="right">--- 维基百科</p> 

<div style="text-indent: 2em;">这篇文章缘起于一次和设计组同事的关于UI验收的讨论，我们项目中有好几处用到了非水平或竖直方向的渐变色设计，在此之前对于渐变色的需求，都是用 <font color=#FF0000>CAGradientLayer</font> 来处理，不过这次由于是一个比较长的控件，展示的效果和设计出的效果出入较大，用红色到蓝色渐变示例如图：</div>
<div align=center>
    <img src="https://upload-images.jianshu.io/upload_images/14229668-2278c59e8e77c29e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/500">
    <p align='center'><font color=' #A9A9A9'>图1</font></p>
</div>

### <font color=#FF0000>CAGradientLayer</font> 具体代码如下

```
let gradientLayer = CAGradientLayer()
gradientLayer.frame = CGRect(x: 0, y: 0, width: viewWidth, height: viewHeight)
gradientLayer.startPoint = CGPoint.zero
gradientLayer.endPoint = CGPoint.init(x: 1.0, y: 1.0)
gradientLayer.locations = [0.0, 1.0]
gradientLayer.colors = [color_start.cgColor, color_end.cgColor]
gradientBgView.layer.insertSublayer(gradientLayer, at: 0)
```
经过和设计师反复比对最终发现问题，<u>**渐变角度不对**</u>。那么我们又该如何处理呢，是修改渐变色角度么还是，这些让我们先了解下渐变色之后再思考。
我们先看下 Sketch 中如何制作一个上图中的红色到蓝色的渐变色矩形，添加一个矩形后修改填充样式，设置渐变样式为线性渐变，设置渐变起始点及渐变颜色即可。在图中由A点到D点的渐变过程中，做垂直于AD的辅助线，则同一条线上的点颜色相同

 <div align=center>
    <img src="https://upload-images.jianshu.io/upload_images/14229668-dd691f31290abc07.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/520">
    <p align='center'><font color=' #A9A9A9'>图2</font></p>
</div>
和 CAGradientLayer 的使用方式不是一模一样么，都是设置起始点与起始点颜色，为什么效果会不一样，先说结论：使用CAGradientLayer实现的渐变颜色梯度线角度和Sketch中不一致。

我们再结合 [CSS3](https://www.runoob.com/css3/css3-gradients.html) 来理解下渐变，里面有详细解释常用的两种渐变（线性渐变和径向渐变），在这里我们仅讨论下线性渐变。其中线性渐变可以通过修改渐变方向或角度来修改梯度线（其实方向和角度是一一对应的）。
>角度是指水平线和渐变线之间的角度，逆时针方向计算。换句话说，0deg 将创建一个从下到上的渐变，90deg 将创建一个从左到右的渐变。

<div align=center>
    <img src="https://user-gold-cdn.xitu.io/2018/3/13/1621e27b8a2df26a?imageslim">
    <p align='center'><font color=' #A9A9A9'>图3</font></p>
</div>
回到我们图中的控件来找渐变角度，过矩形中心点做一条竖直向上的轴线，与该轴线重合的方向为0度，过o点指向BD方向为90度，如图4:

在了解了渐变色角度后回到iOS开发中，幸运的是，我们常用到的是方向而非角度，即类似于Sketch中指定颜色起始点的方式来决定渐变角度，CAGradientLayer 的确已经做好了这些，但不同的是其渐变梯度线并非是我们指定的起点到终点，虽然看似都是从红色点至蓝色点的渐变且起始点相同，经过颜色对比可得知，CAGradientLayer梯度线并不是图中的AD，而是和AD垂直的MN，此时渐变角度也不再是期望的α，而是β。

<div align=center>
    <img src="https://upload-images.jianshu.io/upload_images/14229668-8138516e89704d0c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700">
    <p align='center'><font color=' #A9A9A9'>图4</font></p>
</div>

显然我们不能直接用CAGradientLayer，那么又如何做出和Sketch一样效果的渐变色呢？
## 解决方法：
####  使用CoreGraphics，而不是CAGradientLayer
对于layout更新频率不是特别高的控件，可以使用自定义渐变色View的方式来实现，直接使用CoreGraphics框架，利用drawRect方法来实现。示例代码如下：

```
class MockGradientView: UIView {

    open var colors: [UIColor]?
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        self.backgroundColor = .clear
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }
    
    override func draw(_ rect: CGRect) {
        guard let colors = colors, colors.count > 0 else {
            return
        }
        let context = UIGraphicsGetCurrentContext()
        guard context != nil else {
            return
        }
        context!.saveGState()
        context!.clip(to: rect)
        
        let numOfComponents = 4
        let colorSpace: CGColorSpace = CGColorSpaceCreateDeviceRGB()
        var components: [CGFloat] = []
        for row in 0..<colors.count {
            let comp = colors[row].cgColor.components
            for colum in 0..<numOfComponents {
                let value = comp?[colum] ?? 1
                components.append(value)
            }
        }
        let locations: [CGFloat] = [0, 1]
        let gradient = CGGradient.init(colorSpace: colorSpace, colorComponents: components, locations: locations, count: colors.count)!
        let start_point = CGPoint.zero
        let end_point = CGPoint.init(x: rect.maxX, y: rect.maxY)
        context!.drawLinearGradient(gradient, start: start_point, end: end_point, options: CGGradientDrawingOptions.drawsAfterEndLocation)
        context!.restoreGState()
    }
}

```
对于layout更新频率比较高的，可以将生成的渐变信息保存为UIImage来使用。
### CAGradientLayer使用场景推荐：
在了解了渐变色原理后可以清楚了解到，对于水平方向、竖直方向、正方形对角线方向的渐变我们依然可以正常使用，因为这些情况下设置起始点后渐变角度未改变。

## 拓展：
### 如果需求是固定渐变角度要如何处理？
对于固定渐变角度，我们可以过矩形中心点画对应渐变角度的线，然后过A、D点做垂直于该线的辅助线，两个交点即为新的起始点。


### Demo 地址
[https://github.com/coderWPJ/GradientDemo](https://github.com/coderWPJ/GradientDemo)

## 参考
1. [https://www.runoob.com/css3/css3-gradients.html](https://www.runoob.com/css3/css3-gradients.html)
2. [https://juejin.im/post/5aa75da9f265da23994e3084](https://juejin.im/post/5aa75da9f265da23994e3084)
