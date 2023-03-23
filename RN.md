# react native ios
1. **清除pod cache**
pod cache clean --all

# Mac
1. **从安卓复制文件到Mac**
 1. adb shell 
 2. cd sdcard/
 3. ls 目录，找到要copy的文件，复制路径
 4. exit
 5. adb pull /sdcard/DCIM/ScreenRecorder/Screenrecorder-2023-03-23-10-01-06-812.mp4
