# RunningTimeButton
![你的小可爱已上线](http://upload-images.jianshu.io/upload_images/7882691-886183f3a8e30c93.GIF?imageMogr2/auto-orient/strip)
>这个文章主要是针对 **短信验证码** 倒计时

思路如下：
说到倒计时无非想到的是NSTimer倒计时，在Button触发点击方法时候进行倒计时调用，在每一秒钟更改button样式，倒计时结束时候还原Button样式！（*其实GCD也可以实现倒计时，向这样简单的倒计时 我们可以使用GCD的方式实现，若需求中要求倒计时的时候停止倒计时的操作 GCD是不提供停止的功能 只有到达指定的时间才能停止，而NSTimer可以提供stop功能，我过几天再写一个文章吧！简单介绍下具体的*）

###首先介绍一下buuon代码吧！
- 提供创建button的frame以及倒计时的时间参数**（以下方法根据自己是否用到调用的）**
```
//设置倒是按钮需要到时的时间
- (instancetype)initWithFrame:(CGRect)frame andTimer:(int)time
```
- 用于button点击触发的方法分为两种情况1.当按钮处于没有进行倒计时时候返回`NO`时候点击，button将要进行倒计时，我们可以在这里进行接口访问获取到验证码进行比对 2.当按钮正处于倒计时的时候返回为`YES`，button被点击的时候 我们可以提示用户 **“请耐心等待验证码”** 之类的hub提示框。
```
//点击时候触发的块 和 是否在倒计时 如果yes标示在倒计时  如果no标示没有开始
@property (nonatomic, copy) void (^startTimeButtonAction)(TimeButton * btn,BOOL isRunning);
```
- 可以更改正在倒计时时候按钮的样式，我已经写了一个默认样式，可以选择重写 **（意义不是很大，其实可以直接改源码的 但是如果遇到一个项目两个倒计时按钮样式不一样，孩子傻了吧！）**
```
//进行倒数计时
@property (nonatomic, copy) void (^runningTimeButtonAction)(TimeButton * btn);
```
- 完成倒计时时候进行的操作，需要写上还原时候Button样式的 （我也不知道你还原什么样式，demo只是还原到之前的，万一你需求跟之前不一样呢！）
```
//完成倒计时
@property (nonatomic, copy) void (^endTimeButtonAction)(TimeButton * btn);
```
- 这个还用解释么？在页面释放之前先释放下NSTimer，要么下次进入就出了意想不到的效果
```
//释放NSTimer
- (void)releaseTimer;
```
![自定义button的.h.png](https://upload-images.jianshu.io/upload_images/7882691-8397c0891a163187.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
#import <UIKit/UIKit.h>

@interface TimeButton : UIButton

//设置倒是按钮需要到时的时间
- (instancetype)initWithFrame:(CGRect)frame andTimer:(int)time;

//点击时候触发的块 和 是否在倒计时 如果yes标示在倒计时  如果no标示没有开始
@property (nonatomic, copy) void (^startTimeButtonAction)(TimeButton * btn,BOOL isRunning);
//进行倒数计时
@property (nonatomic, copy) void (^runningTimeButtonAction)(TimeButton * btn);
//完成倒计时
@property (nonatomic, copy) void (^endTimeButtonAction)(TimeButton * btn);

//释放NSTimer
- (void)releaseTimer;

@end
```

#####下面是.m
```
#import "TimeButton.h"

@interface TimeButton()

@property (nonatomic, strong) NSTimer *timer;//创建

@property (nonatomic, assign) int timeNum;//获取到倒计时的时间
@property (nonatomic, assign) int runningTimeNum;//用于倒计时用处

@end

@implementation TimeButton

- (instancetype)initWithFrame:(CGRect)frame andTimer:(int)time {
    if (self = [super initWithFrame:frame]) {
        //设置按钮的时间进行获取
        _timeNum = time;
        _runningTimeNum = 0;
        
        
        //默认样式
        [self setTitle:@"获取倒计时" forState:UIControlStateNormal];
        [self setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
        self.titleLabel.font = [UIFont systemFontOfSize:14];
        self.backgroundColor = [UIColor blueColor];
        
        [self addTarget:self action:@selector(onClick:) forControlEvents:UIControlEventTouchUpInside];
        
    }
    return self;
}

- (void)onClick:(UIButton *)sender{
    //倒数计时开始 的block
    if (_startTimeButtonAction) {
        _startTimeButtonAction(self,_runningTimeNum != 0);
    }
    /*判断如果 _runningTimeNum == 0 的时候表示button没有进入倒计时状态的  如果不是0就是正在进行倒计时（不需要创建NSTimer）
      没有进入倒计时计划要重新创建NSTimer
     */
    if (_runningTimeNum == 0) {
        if (self.timer) {
            [self releaseTimer];
        }
        _runningTimeNum = _timeNum;
        self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(refreshButton) userInfo:nil repeats:YES];
        [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
    }
}


// 刷新按钮
- (void)refreshButton {
    _runningTimeNum --;
    //倒计时更新文字
    [self setTitle:[NSString stringWithFormat:@"%d秒后重新获取",_runningTimeNum] forState:UIControlStateNormal];
    self.backgroundColor = [UIColor greenColor];
    //正在进行倒计时 的block
    if (_runningTimeButtonAction) {
        _runningTimeButtonAction(self);
    }
    //_runningTimeNum == 0 已经倒计时完成 更改回原来button的样式
    if (_runningTimeNum == 0) {
        [self releaseTimer];
        [self setTitle:@"获取倒计时" forState:UIControlStateNormal];
        //倒计时完成 的block
        if (_endTimeButtonAction) {
            _endTimeButtonAction(self);
        }
    }
}

//释放方法
- (void)releaseTimer {
    if (self.timer) {
        [self.timer invalidate];
        self.timer = nil;
    }
}

- (void)dealloc {
    NSLog(@"倒计时按钮已释放");
}

@end
```

###再次介绍如何使用的问题
```
#import "ViewController.h"
#import "TimeButton.h"

@interface ViewController ()

@property (nonatomic,weak) TimeButton * btn;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    TimeButton * btn = [[TimeButton alloc] initWithFrame:CGRectMake(100, 100, 200, 40) andTimer:60];
    btn.backgroundColor = [UIColor redColor];
    btn.startTimeButtonAction = ^(TimeButton * btn,BOOL isRunning){
        if (isRunning) {
            //点击按钮可以进行的接口访问
            NSLog(@"开始倒计时呢");
        }else{
            //可以提示hub，告诉用户正在倒计时等待短信验证
            NSLog(@"正在倒计时 等待一下好么？");
        }
        
    };
    btn.runningTimeButtonAction = ^(TimeButton *btn) {
        //可以设置倒计时的样式
        btn.backgroundColor = [UIColor lightGrayColor];
        NSLog(@"正在进行倒计时");
    };
    btn.endTimeButtonAction = ^(TimeButton *btn) {
        //完成时候
        btn.backgroundColor = [UIColor redColor];
        NSLog(@"倒计时完成");
    };
    _btn = btn;
    [self.view addSubview:_btn];
}

- (void)dealloc {
    [self.btn releaseTimer];
    NSLog(@"页面二已释放");
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}
@end
```
###demo地址💕git 地址：https://github.com/sunrose11/RunningTimeButton
>***需要的人可以直接copy走吧！记得帮我点点❤ 爱你哟!***

![](http://upload-images.jianshu.io/upload_images/7882691-ca45d1830b9b562a.gif?imageMogr2/auto-orient/strip)
