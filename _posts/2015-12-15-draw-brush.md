---
layout: post
date: 2015-12-15 12:49
title: iOS 画板，刷子效果的实现
disqus: y
---

效果图：

![](http://obb77efas.bkt.clouddn.com/IMG_0491.jpg?imageView2/2/w/300)

发现了一个精美的 iOS 绘图 App `memopad`，提供了各种笔刷效果， 画出来的线条十分柔和， 铅笔效果、毛刷效果十分形象， 自己也想做一个类似的 demo， 效果图如上， 误笑
实现这种效果最简单的方法就是在用户触摸的地方把刷子效果的图片绘制上去， 那么问题来了， 我并没有一张这样的图片，于是我就解压了 `memopad` 看到了一个文件夹里有很多宝贝 ~逃

![](http://obb77efas.bkt.clouddn.com/textures.jpg?imageView2/2/w/300)

哈哈， 就是他们了，我就挑选了其中一个。 然后开始撸代码了:

```objectivec
@interface OCTDrawingView : UIView
@end
@interface OCTDrawingView ()
@property (strong, nonatomic) NSMutableArray *frames;
@property (strong, nonatomic) UIImage *texture;
@end

static CGFloat kBrushWidth = 50;
@implementation OCTDrawingView

- (void)drawRect:(CGRect)rect {
        for (NSValue *value in self.frames) {
        CGRect frame = [value CGRectValue];
        [self.texture drawInRect: frame];
    }
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesBegan:touches withEvent:event];
    UITouch *touch = [touches anyObject];
    CGPoint p = [touch locationInView: self];
    CGRect frame = CGRectMake(p.x - kBrushWidth / 2, p.y - kBrushWidth / 2, kBrushWidth, kBrushWidth);
    [self.frames addObject: [NSValue valueWithCGRect: frame]];
    [self setNeedsDisplay];
    }

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesMoved:touches withEvent:event];
    UITouch *touch = [touches anyObject];
    CGPoint p = [touch locationInView: self];
    CGRect frame = CGRectMake(p.x - kBrushWidth / 2, p.y - kBrushWidth / 2, kBrushWidth, kBrushWidth);
    [self.frames addObject: [NSValue valueWithCGRect: frame]];
    [self setNeedsDisplay];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesEnded:touches withEvent:event];
    UITouch *touch = [touches anyObject];
    CGPoint p = [touch locationInView: self];
    CGRect frame = CGRectMake(p.x - kBrushWidth / 2, p.y - kBrushWidth / 2, kBrushWidth, kBrushWidth);
    
    [self.frames addObject: [NSValue valueWithCGRect: frame]];
    [self setNeedsDisplay];
}

- (NSMutableArray *)frames {
    if (!_frames) {    
        _frames = [NSMutableArray array];
    }
    return _frames;
}

- (UIImage *)texture {    
    if (!_texture) {
        _texture = [UIImage imageNamed:@"texture"];
    }
    return _texture;
}
@end
```

运行一下

![](http://obb77efas.bkt.clouddn.com/fuck.jpg?imageView2/2/w/300)

可以绘制了， 显然这并不是我们想要的效果， 我尝试了使用 `BlendMode` 发现都不能满足我的条件（好吧， 我并没有尝试 ~）, 我们要把图片处理成我们想要的样子，恩 撸几行代码试试

```objectivec
@interface UIImage (TintColor)
- (UIImage *)textureWithTintColor:(UIColor *)color;
@end

typedef NS_ENUM(NSInteger, Pixel) {
    Alpha = 0,
    Blue = 1,
    Green = 2,
    Red = 3,
};

@implementation UIImage (TintColor)

- (UIImage *)textureWithTintColor:(UIColor *)color {
    
    CGFloat red, green, blue, alpha;
    
    [color getRed:&red green:&green blue:&alpha alpha:&alpha];
    // 1    
    uint8_t blendRed = ( (1 - alpha) + alpha * red) * 255;
    uint8_t blendGreen = ( (1 - alpha) + alpha * green ) * 255;
    uint8_t blendBlue = ( (1 - alpha) + alpha * blue ) * 255;
    
    CGSize size = self.size;
    int width = size.width;
    int height = size.height;
    
    uint32_t *pixels = (uint32_t *)malloc(width * height * sizeof(uint32_t));
    
    memset(pixels, 0, width * height * sizeof(uint32_t));
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    CGContextRef ctx = CGBitmapContextCreate(pixels, width, height, 8, width * sizeof(uint32_t), colorSpace, kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedLast);
    
    CGContextDrawImage(ctx, CGRectMake(0, 0, width, height), self.CGImage);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            uint8_t *rgbaPixel = (uint8_t *) &pixels[y * width + x];
            uint8_t pixelRed = rgbaPixel[Red];
            uint8_t dgreen = rgbaPixel[Green] / 255.0 * blendGreen;
            uint8_t dblue = rgbaPixel[Blue] / 255.0 * blendBlue;
            uint8_t dred = rgbaPixel[Red] / 255.0 * blendRed;            
            rgbaPixel[Red] = dred;
            rgbaPixel[Green] = dgreen;
            rgbaPixel[Blue] = dblue;
            rgbaPixel[Alpha] = pixelRed;
        }
    }    
    CGImageRef cgImage = CGBitmapContextCreateImage(ctx);
    CGContextRelease(ctx);
    CGColorSpaceRelease(colorSpace);    
    UIImage *texture = [UIImage imageWithCGImage: cgImage];    
    CGImageRelease(cgImage);    
    return texture;
}
@end
```

首先我们获取了图片每个像素的 RGB 颜色， 并把图片的每个像素都替换成了我们想要的颜色。 我们偷的那张图片是灰度图，所以我用原图片红色的值作为 alpha 值（当然你也可以用绿色或者蓝色， 效果就是白色的部分不透明度高， 纯黑色就是透明了）
说明: 1. 此处我们假装背景色是白色， 然后根据`tintColor`计算出图片应该显示的颜色

```bash
 R = S + D * ( 1 – Sa )
```

 `S` 表示顶部的颜色（乘以`alpha`以后的`RGB`颜色）, `Sa`表示顶部的 `alpha`， `D` 表示底部的颜色（也是乘以 `alpha` 之后的 `RGB` 颜色），`R` 就是混合后的颜色， 也就是`tintColor`在白色背景下应该显示的颜色。

![](http://obb77efas.bkt.clouddn.com/texture_ture.jpg?imageView2/2/w/300)

运行后的效果如上图， 恩终于是我们想要的样子了。 当我跃跃欲试画一副恢宏巨作的时候， 发现画得过多时会出现明显的卡顿， 因为我们每次触摸都会重绘所有的图片，但是界面允许最多同时出现几百个图层， 别说巨作， 随便画几下就达到上限了。
是时候祭出我们的`脏矩形`大法了
我们只需要绘制触摸的那一小块区域就够了， 可以调用这个方法`- setNeedsDisplayInRect:` 然后 `touchBegan...`、`touchMoved`、 `touchEnded` 里就可以这样写:

```objectivec
 - (void)drawRect:(CGRect)rect {
         for (NSValue *value in self.frames) {        
         CGRect frame = [value CGRectValue];
         if (CGRectIntersectsRect(rect, frame)) {            
             [self.texture drawInRect: frame];
         }
     }
 }
```

现在 GPU 的负担就减轻了， 没有再出现卡顿的现象。