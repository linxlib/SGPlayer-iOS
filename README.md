# SGPlayer-iOS
rtsp player framework for iOS with RTSP Authorization support based on ffmpeg

# 说明

- 来自 https://github.com/libobjc/SGPlayer ，未对代码进行修改， 此仓库放可用的framework， 以及如何从原仓库获得对应文件的方法 
- https://github.com/yegail/SGPlayer-iOS ， 好家伙， 直接把ffmpeg给弄没了
- 测试了ijkplayer， 不知道咋回事就是不能支持 rtsp://user:password@IP:Port/Streaming/Channels/102 这种海康的NVR录像机的URL链接
- 测试了MobileVLCKit，可以播放，不过每次播放需要等待接近10s（ip ping值20ms以下，用VLC播放器播放貌似也很慢， 又没有找到合适的优化选项），这库挺大，集成进去后估计ipa得大一圈， 不过问题不大也就多下载一会
- 测试了ffmpeg官网的ffplay.exe 发现是直接支持上面那种带用户名密码的链接的，而ijk的issue里找不到相关的解决方案，这就很奇怪了，ijk明明也是基于ffmpeg的
- 不太会OC，所以都是瞎搞


# Guide 

```bash
git clone https://github.com/libobjc/SGPlayer.git
cd SGPlayer
git checkout 2.0.1 -B latest

// iOS
./build.sh iOS build

// tvOS
./build.sh tvOS build

// macOS
./build.sh macOS build
```

构建要点时间，构建完成之后， 打开根目录下的 SGPlayer.xcodeproj， CMD+B，编译

编译通过后，`Build Phases` 增加`Run Script`, 内容如下

```sh
FRAMEWORK=$1
echo "Trimming $FRAMEWORK..."

FRAMEWORK_EXECUTABLE_PATH="${BUILT_PRODUCTS_DIR}/${FRAMEWORKS_FOLDER_PATH}/$FRAMEWORK.framework/$FRAMEWORK"

EXTRACTED_ARCHS=()

for ARCH in $ARCHS
do
    echo "Extracting $ARCH..."
    lipo -extract "$ARCH" "$FRAMEWORK_EXECUTABLE_PATH" -o "$FRAMEWORK_EXECUTABLE_PATH-$ARCH"
    EXTRACTED_ARCHS+=("$FRAMEWORK_EXECUTABLE_PATH-$ARCH")
done

echo "Merging binaries..."
lipo -o "$FRAMEWORK_EXECUTABLE_PATH-merged" -create "${EXTRACTED_ARCHS[@]}"
rm "${EXTRACTED_ARCHS[@]}"

rm "$FRAMEWORK_EXECUTABLE_PATH"
mv "$FRAMEWORK_EXECUTABLE_PATH-merged" "$FRAMEWORK_EXECUTABLE_PATH"

echo "Done."
```
再次build即可在项目目录下的`Products`目录下，找到framework了，大概800+M，zip压缩下300M左右

不知道如何搞pod，以后要是用得多的话可以搞个

# 代码使用

SGPlayViewController.h
```obj-c
#import <UIKit/UIKit.h>

@interface SGPlayViewController : UIViewController


@property (nonatomic, copy) NSString *mediaPath;
@property (nonatomic,copy) NSString *tit;

@end
```

SGPlayViewController.m
```obj-c
#import "SGPlayViewController.h"
#import <SGPlayer/SGPlayer.h>

@interface SGPlayViewController ()

@property (nonatomic, assign) BOOL seeking;
@property (nonatomic, strong) SGPlayer *player;

@property (weak, nonatomic) IBOutlet UILabel *stateLabel;
@property (weak, nonatomic) IBOutlet UILabel *durationLabel;
@property (weak, nonatomic) IBOutlet UILabel *currentTimeLabel;
@property (weak, nonatomic) IBOutlet UISlider *progressSilder;
@property (weak, nonatomic) IBOutlet UILabel *titleLabel;

@end

@implementation SGPlayViewController

- (instancetype)init
{
    if (self = [super init]) {
        self.player = [[SGPlayer alloc] init];
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(infoChanged:) name:SGPlayerDidChangeInfosNotification object:self.player];
    }
    return self;
}

- (void)dealloc
{
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    self.player.videoRenderer.view = self.view;
    self.player.videoRenderer.displayMode = SGDisplayModePlane;
    SGOptions *options = [SGOptions sharedOptions];
    options.demuxer.options = @{
            @"rtsp_transport": @"tcp",
    };
    self.player.options = options;
    
    [self.player replaceWithURL:[NSURL URLWithString:_mediaPath]];
    self.titleLabel.text = _tit;
    [self.player play];
}

#pragma mark - SGPlayer Notifications

- (void)infoChanged:(NSNotification *)notification
{
    SGTimeInfo time = [SGPlayer timeInfoFromUserInfo:notification.userInfo];
    SGStateInfo state = [SGPlayer stateInfoFromUserInfo:notification.userInfo];
    SGInfoAction action = [SGPlayer infoActionFromUserInfo:notification.userInfo];
    if (action & SGInfoActionTime) {
        if (action & SGInfoActionTimePlayback && !(state.playback & SGPlaybackStateSeeking) && !self.seeking && !self.progressSilder.isTracking) {
            self.progressSilder.value = CMTimeGetSeconds(time.playback) / CMTimeGetSeconds(time.duration);
            self.currentTimeLabel.text = [self timeStringFromSeconds:CMTimeGetSeconds(time.playback)];
        }
        if (action & SGInfoActionTimeDuration) {
            self.durationLabel.text = [self timeStringFromSeconds:CMTimeGetSeconds(time.duration)];
        }
    }
    if (action & SGInfoActionState) {
        if (state.playback & SGPlaybackStateFinished) {
            self.stateLabel.text = @"完成";
        } else if (state.playback & SGPlaybackStatePlaying) {
            self.stateLabel.text = @"播放中";
        } else {
            self.stateLabel.text = @"已暂停";
        }
    }
}

#pragma mark - Actions

- (IBAction)back:(id)sender
{
    [self.navigationController popViewControllerAnimated:YES];
}

- (IBAction)play:(id)sender
{
    [self.player play];
}

- (IBAction)pause:(id)sender
{
    [self.player pause];
}

- (IBAction)progressTouchUp:(id)sender
{
    CMTime time = CMTimeMultiplyByFloat64(self.player.currentItem.duration, self.progressSilder.value);
    if (!CMTIME_IS_NUMERIC(time)) {
        time = kCMTimeZero;
    }
    self.seeking = YES;
    [self.player seekToTime:time result:^(CMTime time, NSError *error) {
        self.seeking = NO;
    }];
}

#pragma mark - Tools

- (NSString *)timeStringFromSeconds:(CGFloat)seconds
{
    return [NSString stringWithFormat:@"%ld:%.2ld", (long)seconds / 60, (long)seconds % 60];
}

@end
```

感谢 libobjc

最后附上打包好的成品

链接: https://pan.baidu.com/s/1Dp9dcLSgN15lPrs6OcXxvg  密码: m0ki

https://mega.nz/file/GjhFwYqb#zPgW_Y_OVvj7XtlATj4qt4utxxb0nUrt96bcCYIjorM







