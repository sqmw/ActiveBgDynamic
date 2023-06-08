# activeBgDynamic
this project is written by tauri

## what is tauri: 
tauri is a fronted framework composed of webView(fronted) and rust(backend)

## application
This project is used to display a dynamic wallpaper which can interact with activeBg
1. 这进程为activeBg的子进程，每隔周期性每隔100毫秒发起一次请求，获取activeBg需要本进程做的task，处理完之后返回结果
2. 这个进程可以通过canvas提取视频的一帧，该视频仅仅是传输过来的链接，提取好一帧之后将这一帧图片编码成base64形式，返回给activeBg

## 目前的功能，支持动态壁纸的展示，但是，可以添加subJoin的js代码，
1. subJoin：表示的是附加的style，对应的是video(也就是动态视频壁纸)或者是和video同级的动态的图片动效壁纸

## 完成部分的功能