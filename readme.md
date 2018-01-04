针对该插件无法缓存管理，进行改造，使其兼容react-native-fetch-blob插件，改造点:
【image-viewer.type.ts】
1.添加结构
--------------------------------------------------------------------------------
export interface Progress {
    received: number
    total: number
}
--------------------------------------------------------------------------------
2.修改结构StateDefine，添加属性
--------------------------------------------------------------------------------
   /**
     * 图片信息（被缓存则来自于本地）
     */
    cachedImageUrls?: Array<ImageInfo>
    /**
     * 下载进度内容
     */
    progresses?: Array<Progress>
--------------------------------------------------------------------------------


【image-viewer.component.tsx】
1.修改文件中的init函数添加下载进度state，为进度条做准备
--------------------------------------------------------------------------------
// 给 imageSizes 塞入空数组
        const imageSizes: Array<typings.ImageSize> = [];
        const progresses: Array<typings.Progress> = [];
        nextProps.imageUrls.forEach(imageUrl => {
            imageSizes.push({
                width: imageUrl.width || 0,
                height: imageUrl.height || 0,
                status: 'loading'
            });
            progresses.push({
                received: 0,
                total: 0
            });
        })

        this.setState({
            currentShowIndex: nextProps.index,
            imageSizes,
            progresses
        }
--------------------------------------------------------------------------------
2.在loadImage函数上方添加生成guid的函数
--------------------------------------------------------------------------------
    //生成随机ID：GUID
    GUID(){
        return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
            var r = Math.random() * 16 | 0,
            v = c == 'x' ? r : (r & 0x3 | 0x8);
            return v.toString(16);
        }).toUpperCase();
    }
--------------------------------------------------------------------------------
3.添加保存与获取缓存配置的信息
--------------------------------------------------------------------------------
    //存储文件缓存信息
    saveImageCachePath(url: string, localPath: string,callback?: (error?:Error) => void) {
        AsyncStorage.getItem('imageCache').then((result: string)=>{
            if (null == result) {
                result = '{}';
            }
            //添加图片缓存
            let imgCache = JSON.parse(result);
            imgCache[url] = localPath;
            //保存
            AsyncStorage.setItem('imageCache', JSON.stringify(imgCache), callback);
        });
    }

    //读取全部缓存配置信息
    loadImageCache(callback?: (result: any) => void) {
        AsyncStorage.getItem('imageCache').then((result: any)=>{
            if (null != result) {
                result = '{}';
                //添加图片缓存
                let imgCache = JSON.parse(result);
                callback(imgCache);
            } else {
                callback(null);
            }
        }).catch(()=>{
            callback(null);
        });
    }
--------------------------------------------------------------------------------
4.重新定义私有属性对url进行缓存处理
--------------------------------------------------------------------------------
    private cachedImageUrls = new Array();
--------------------------------------------------------------------------------
5.修改componentWillReceiveProps函数，在loadImage之前实现对图片url进行处理
--------------------------------------------------------------------------------
componentWillReceiveProps(nextProps: typings.PropsDefine) {
        if (nextProps.index !== this.state.currentShowIndex) {
            this.setState({
                currentShowIndex: nextProps.index
            }, () => {
                //进行加载图片操作之前先要对传入的url进行核实，如果已经缓存的话则不再进行网络加载操作
                this.cachedImageUrls = this.props.imageUrls.slice();
                this.loadImageCache((cache) => {
                    if (null != cache) {
                        //对缓存进行检测
                        this.cachedImageUrls.forEach(imageUrl => {
                            let {url, originUrl} = imageUrl;
                            if (cache[url]) imageUrl.url = cache[url];
                            if (cache[originUrl]) imageUrl.originUrl = cache[originUrl];
                        });
                    } 
                        // 立刻预加载要看的图
                    this.loadImage(nextProps.index)

                    this.jumpToCurrentImage()

                    // 显示动画
                    Animated.timing(this.fadeAnim, {
                        toValue: 1,
                        duration: 200
                    }).start()

                    this.setState({ cachedImageUrls: this.cachedImageUrls });
                })
            });
        }
    }
--------------------------------------------------------------------------------
6.对init函数进行处理，内容与前一步相同
--------------------------------------------------------------------------------
this.setState({
            currentShowIndex: nextProps.index,
            imageSizes,
            progresses
        }, () => {
            //进行加载图片操作之前先要对传入的url进行核实，如果已经缓存的话则不再进行网络加载操作
            this.cachedImageUrls = this.props.imageUrls.slice();
            this.loadImageCache((cache) => {
                if (null != cache) {
                    //对缓存进行检测
                    this.cachedImageUrls.forEach(imageUrl => {
                        let {url, originUrl} = imageUrl;
                        if (cache[url]) imageUrl.url = cache[url];
                        if (cache[originUrl]) imageUrl.originUrl = cache[originUrl];
                    });
                } 
                // 立刻预加载要看的图
                this.loadImage(nextProps.index)

                this.jumpToCurrentImage()

                // 显示动画
                Animated.timing(this.fadeAnim, {
                    toValue: 1,
                    duration: 200
                }).start()
                this.setState({ cachedImageUrls: this.cachedImageUrls });
            })
        })
--------------------------------------------------------------------------------
7.修改文件中的loadImage函数修改下载，为进度条以及完成后对图片路径的缓存做准备，由于改动较大，所以列出改动前与改动后的内容变更
《改动前》
--------------------------------------------------------------------------------
if (Platform.OS !== 'web' as any) {
            const prefetchImagePromise = Image.prefetch(image.url)
            
            if (image.url.match(/^(http|https):\/\//)) {
                prefetchImagePromise.then(() => {
                    imageLoaded = true;
                    if (sizeLoaded) {
                        imageStatus.status = 'success';
                        saveImageSize();
                    }
                }, () => {
                    imageStatus.status = 'fail';
                    saveImageSize();
                });
            } else {
                // 本地图片
                imageLoaded = true;
                prefetchImagePromise.then(() => { }).catch(() => { });
                if (sizeLoaded) {
                    imageStatus.status = 'success';
                    saveImageSize();
                }
            }
--------------------------------------------------------------------------------
《改动后》
--------------------------------------------------------------------------------
if (Platform.OS !== 'web' as any) {
            //const prefetchImagePromise = Image.prefetch(image.url)
            
            if (image.url.match(/^(http|https):\/\//)) {
                let dirs = RNFetchBlob.fs.dirs;
                RNFetchBlob.config({
                    path: `${dirs.DocumentDir}/ImageCache/${this.GUID()}`
                })
                .fetch('GET',image.url)
                .progress((received:number, total:number) => {
                    //重新设置进度
                    let progress = Object.assign({}, this.state.progresses[index]);
                    progress.received = received;
                    progress.total = total;
                    let progresses = this.state.progresses.slice();
                    progresses[index] = progress;
                    this.setState({ progresses: progresses });
                }).then((res: any) => {
     //将图片缓存路径写入配置
    this.saveImageCachePath(image.url, res.path, (error) => {
         if (null != error) {
             //配置保存完之后对缓存进行更新
             this.cachedImageUrls[index].url = res.path();
         }
     });

                    image.url = res.path();
                    Image.prefetch(image.url).then(() => { }).catch(() => { });
                    imageLoaded = true;
                    if (sizeLoaded) {
                        imageStatus.status = 'success';
                        saveImageSize();
                    }
                }).catch(() => {
                    imageStatus.status = 'fail';
                    saveImageSize();
                });
            } else {
                // 本地图片
                imageLoaded = true;
                Image.prefetch(image.url).then(() => { }).catch(() => { });
                if (sizeLoaded) {
                    imageStatus.status = 'success';
                    saveImageSize();
                }
            }
--------------------------------------------------------------------------------
8.修改getContent函数，获取image信息时由this.props.imageUrls获取变更为this.cachedImageUrls中获取
--------------------------------------------------------------------------------
/**
     * 获得整体内容
     */
    getContent() {
        // 获得屏幕宽高
        const screenWidth = this.width
        const screenHeight = this.height

        const ImageElements = this.cachedImageUrls.map((image, index) => {
            let width = this.state.imageSizes[index] && this.state.imageSizes[index].width
            let height = this.state.imageSizes[index] && this.state.imageSizes[index].height
            const imageInfo = this.state.imageSizes[index]

            // 如果宽大于屏幕宽度,整体缩放到宽度是屏幕宽度
            if (width > screenWidth) {
                const widthPixel = screenWidth / width
                width *= widthPixel
                height *= widthPixel
            }

            // 如果此时高度还大于屏幕高度,整体缩放到高度是屏幕高度
            if (height > screenHeight) {
                const HeightPixel = screenHeight / height
                width *= HeightPixel
                height *= HeightPixel
            }

            if (imageInfo.status === 'success' && this.props.enableImageZoom) {
                return (
                    <ImageZoom key={index}
                        style={this.styles.modalContainer}
                        cropWidth={this.width}
                        cropHeight={this.height}
                        imageWidth={width}
                        imageHeight={height}
                        maxOverflow={this.props.maxOverflow}
                        horizontalOuterRangeOffset={this.handleHorizontalOuterRangeOffset.bind(this)}
                        responderRelease={this.handleResponderRelease.bind(this)}
                        onLongPress={this.handleLongPress.bind(this, image)}
                        onClick={this.handleClick.bind(this)}
                        onDoubleClick={this.handleDoubleClick.bind(this)}>
                        <Image style={Object.assign({}, this.styles.imageStyle, { width: width, height: height })}
                            source={{ uri: image.url }} />
                    </ImageZoom>
                )
            } else {
                switch (imageInfo.status) {
                    case 'loading':
                        return (
                            <TouchableHighlight key={index}
                                onPress={this.handleClick.bind(this)}
                                style={this.styles.loadingTouchable}>
                                <View style={this.styles.loadingContainer}>
                                    {this.props.loadingRender()}
                                </View>
                            </TouchableHighlight>
                        )
                    case 'success':
                        return (
                            <Image key={index}
                                style={Object.assign({}, this.styles.imageStyle, { width: width, height: height })}
                                source={{ uri: image.url }} />
                        )
                    case 'fail':
                        return (
                            <ImageZoom key={index}
                                style={this.styles.modalContainer}
                                cropWidth={this.width}
                                cropHeight={this.height}
                                imageWidth={width}
                                imageHeight={height}
                                maxOverflow={this.props.maxOverflow}
                                horizontalOuterRangeOffset={this.handleHorizontalOuterRangeOffset.bind(this)}
                                responderRelease={this.handleResponderRelease.bind(this)}
                                onLongPress={this.handleLongPress.bind(this, image)}
                                onClick={this.handleClick.bind(this)}
                                onDoubleClick={this.handleDoubleClick.bind(this)}>
                                <TouchableOpacity key={index}
                                    style={this.styles.failContainer}>
                                    <Image source={this.props.failImageSource}
                                        style={this.styles.failImage} />
                                </TouchableOpacity>
                            </ImageZoom>
                        )
                }
            }
        })

        return (
            <Animated.View style={Object.assign({}, this.styles.container, { opacity: this.fadeAnim })}>
                {this.props.renderHeader(this.state.currentShowIndex)}

                <View style={this.styles.arrowLeftContainer}>
                    <TouchableWithoutFeedback onPress={this.goBack.bind(this)}>
                        <View>
                            {this.props.renderArrowLeft()}
                        </View>
                    </TouchableWithoutFeedback>
                </View>

                <View style={this.styles.arrowRightContainer}>
                    <TouchableWithoutFeedback onPress={this.goNext.bind(this)}>
                        <View>
                            {this.props.renderArrowRight()}
                        </View>
                    </TouchableWithoutFeedback>
                </View>

                <Animated.View style={Object.assign({}, this.styles.moveBox, { transform: [{ translateX: this.positionX }] }, { width: this.width * this.props.imageUrls.length })}>
                    {ImageElements}
                </Animated.View>

                {this.props.imageUrls.length > 1 &&
                    this.props.renderIndicator(this.state.currentShowIndex + 1, this.props.imageUrls.length)
                }

                {this.cachedImageUrls[this.state.currentShowIndex].originSizeKb && this.cachedImageUrls[this.state.currentShowIndex].originUrl &&
                    <View style={this.styles.watchOrigin}>
                        <TouchableOpacity style={this.styles.watchOriginTouchable}>
                            <Text style={this.styles.watchOriginText}>查看原图(2M)</Text>
                        </TouchableOpacity>
                    </View>
                }

                {this.props.renderFooter(this.state.currentShowIndex)}
            </Animated.View>
        )
    }
--------------------------------------------------------------------------------
注：对于显示原图的功能插件作者似乎只是显示了按钮并没有完成它，日后如果有需要的话会对插件进行进一步处理以满足需求

9.修改saveToLocal函数，获取image信息时由this.props.imageUrls获取变更为this.cachedImageUrls中获取
--------------------------------------------------------------------------------
    /**
     * 保存当前图片到本地相册
     */
    saveToLocal() {
        if (!this.props.onSave) {
            CameraRoll.saveToCameraRoll(this.cachedImageUrls[this.state.currentShowIndex].url)
            this.props.onSaveToCamera(this.state.currentShowIndex)
        } else {
            this.props.onSave(this.cachedImageUrls[this.state.currentShowIndex].url)
        }

        this.setState({
            isShowMenu: false
        })
    }
--------------------------------------------------------------------------------
【编译内容】
改完之后需要编译内容以生效，终端cd到插件目录《项目路径/node_modules/react-native-image-zoom-viewer》并执行yarn进行编译操作，编译完成后重启react-native才可以生效



### Dependence

Depend on `react-native-image-pan-zoom`: https://github.com/ascoders/react-native-image-zoom
