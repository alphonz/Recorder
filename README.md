# Recorder用于html5录音
支持大部分已实现`getUserMedia`的浏览器，包括微信，演示地址：https://xiangyuecn.github.io/Recorder/

录音默认输出mp3格式，另外可选wav格式（此格式录音文件超大）

mp3默认16kbps的比特率，大概2kb每秒，如果使用8kbps可达到1kb每秒，不过音质很渣，没有amr格式的可比性。

~~已内置lamejs依赖用于mp3编码，剥离后核心代码不足300行~~

# 已知问题
*2018-07-22* [mozilla](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaDevices/getUserMedia) 和 [caniuse](https://caniuse.com/#search=getUserMedia) 注明的IOS 11以上Safari是支持调用getUserMedia的，但有用户反馈苹果手机IOS11 Safari和微信都不能录音，演示页面内两个关键指标：获取getUserMedia都是返回false（没有苹果手机未能复现）。但经测试桌面版Safari能获取到getUserMedia。原因不明。

*2018-09-19* [caniuse](https://caniuse.com/#search=getUserMedia) 注明IOS 12 Safari支持调用getUserMedia，经用户测试反馈IOS 12上chrome、UC都无法获取到，部分IOS 12 Safari可以获取到并且能正常录音，但部分不行，原因未知，参考[ios 12 支不支持录音了](https://www.v2ex.com/t/490695)

*2018-12-06* **【已修复】** [issues#1](https://github.com/xiangyuecn/Recorder/issues/1)不同OS上低码率mp3有可能无声，测试发现问题出在lamejs编码器有问题，此编码器本来就是精简版的，可能有地方魔改坏了，用lame测试没有这种问题。已对lamejs源码进行了改动，已通过基础测试，此问题未再次出现。

# 快速使用
在需要录音功能的页面引入js文件代码即可，*对于https的要求不做解释*
``` html
<script src="recorder.js"></script>
```
然后使用，假设立即运行，只录3秒
``` javascript
var rec=Recorder();
rec.open(function(){//打开麦克风授权获得相关资源
	rec.start();//开始录音
	
	setTimeout(function(){
		rec.stop(function(blob){//到达指定条件停止录音，拿到blob对象想干嘛就干嘛：立即播放、上传
			console.log(URL.createObjectURL(blob));
			rec.close();//释放录音资源
		},function(msg){
			console.log("录音失败:"+msg);
		});
	},3000);
},function(msg){//未授权或不支持
	console.log("无法录音:"+msg);
});
```


# 方法文档

### rec=Recorder(set)

拿到`Recorder`的实例，然后可以进行请求获取麦克风权限和录音。

`set`参数为配置对象，默认配置值如下：
```
set={
	type:"mp3" //输出类型：mp3,wav
	,bitRate:16 //比特率 wav:16或8位，MP3：8kbps 1k/s，16kbps 2k/s 录音文件很小
	
	,sampleRate:16000 //采样率，wav格式大小=sampleRate*时间；mp3此项对低比特率有影响，高比特率几乎无影响。wav任意值，mp3取值范围：48000, 44100, 32000, 24000, 22050, 16000, 12000, 11025, 8000
	
	,bufferSize:8192 //AudioContext缓冲大小
					//取值256, 512, 1024, 2048, 4096, 8192, or 16384，会影响onProcess调用速度
	
	,onProcess:NOOP //接收到录音数据时的回调函数：fn(buffer,powerLevel,bufferDuration) 
				//buffer=[缓冲数据,...]，powerLevel：当前缓冲的音量级别0-100，bufferDuration：已缓冲时长
				//如果需要绘制波形之类功能，需要实现此方法即可，使用以计算好的powerLevel可以实现音量大小的直观展示，使用buffer可以达到更高级效果
}
```

### rec.open(success,fail)
请求打开录音资源，如果用户拒绝麦克风权限将会调用`fail`，打开后需要调用`close`。

注意：此方法是异步的；一般使用时打开，用完立即关闭；可重复调用，可用来测试是否能录音。

`success`=fn();

`fail`=fn(errMsg);


### rec.close(success)
关闭释放录音资源，释放完成后会调用`success()`回调

### rec.start()
开始录音，需先调用`open`；如果不支持、错误，不会有任何提示，因为stop时自然能得到错误。

### rec.stop(success,fail)
结束录音并返回录音数据`blob对象`，拿到blob对象就可以为所欲为了，不限于立即播放、上传

`success(blob,duration)`：`blob`：录音数据audio/mp3|wav格式，`duration`：录音时长，单位毫秒

`fail(errMsg)`：录音出错回调

提示：stop时会进行音频编码，音频编码可能会很慢，10几秒录音花费2秒左右算是正常，编码并未使用Worker方案(文件多)，内部采取的是分段编码+setTimeout来处理，界面卡顿不明显。


### rec.pause()
暂停录音。

### rec.resume()
恢复继续录音。

### Recorder.IsOpen()
由于Recorder持有的录音资源是全局唯一的，可通过此方法检测是否有Recorder已调用过open打开了录音功能。

### Recorder.lamejs
lamejs的引用


# 缩小js文件
recorder.js用Uglify压缩一下剩余156kb，不算大

如果不需要mp3格式，可以把lamejs代码全部移除，recorder.js精简到300来行代码，仅仅支持wav格式；mp3编码采用的是`https://github.com/zhuker/lamejs/blob/bfb7f6c6d7877e0fe1ad9e72697a871676119a0e/lame.all.js`这个版本的代码，已对lamejs源码进行了部分改动，用于修复发现的问题。


# 兼容性
对于支持录音的浏览器能够正常录音并返回录音数据；对于不支持的浏览器，引入此js和执行相关方法都不会产生异常，并且进入相关的fail回调。一般在open的时候就能检测到是否支持或者被用户拒绝，可在用户开始录音之前提示浏览器不支持录音或授权。


# 其他音频格式支持办法
``` javascript
//直接在源码中增加代码，比如增加ogg格式支持 (可参看内置的mp3实现)

RecorderFn.prototype.ogg=function(pcmData,successCall){
	//通过ogg编码器把pcm数据转成ogg格式数据，通过this.set拿到传入的配置数据
	... pcmData->oggData
	
	//返回数据
	successCall(new Blob(oggData,{type:"audio/ogg"}));
}

//调用
Recorder({type:"ogg"})
```