#用户自定义拓展功能



- **开始**



1.dart侧实现，集成```IFairPlugin```，实现```getRegisterMethods（），注册需要暴露给JS侧的方法```



```

class FairPhotoSelector extends IFairPlugin {

  static final FairPhotoSelector _photoSelector = FairPhotoSelector._internal();



  FairPhotoSelector._internal();



  factory FairPhotoSelector() {

    return _photoSelector;

  }



  Future<dynamic> getPhoto(dynamic map) async {

    // {

    //   'className': 'PhotoSelector#getPhoto',

    //   'funcName': 'invokePlugin',

    //   'pageName': '页面名称',

    //   'args': {

    //     //用户自定义的参数

    //   }

    // }



    if (map == null) {

      return;

    }

    var req;

    bool isDart;

    if (map is Map) {

      isDart = true;

      req = map;

    } else {

      isDart = false;

      req = jsonDecode(map);

    }

    var pageName = req['pageName'];



    var args = req['args'];

    var callId = args['callId'];

    var successCallback = args['success'];

    var failureCallback = args['failure'];

    var type = args['type'];

    //用户可以自定义参数，通过参数确定图片的选择方式

    var photoPath;

    if ('photo' == type) {

      photoPath = await _onImageButtonPressed(ImageSource.camera);

    } else if ('album' == type) {

      photoPath = await _onImageButtonPressed(ImageSource.gallery);

    } else {

      //no-op

    }

    //需要判断发起方的请求是dart端还是js端

    if (isDart) {

      if (photoPath != null) {

        successCallback?.call(photoPath);

      } else {

        failureCallback?.call();

      }

    } else {

      var resp = {

        //pageName必传，因为每个pageName代表不同页面的作用域

        'pageName': pageName,

        'args': {

          'callId': callId,

          'photoPath': photoPath,

        }

      };

      return Future.value(jsonEncode(resp));

    }



    return Future.value();

  }



  Future<String> _onImageButtonPressed(ImageSource source,

      {BuildContext context}) async {

    final pickedFile = await _getPicker().getImage(

      source: source,

    );

    //获取照片路径成功

    return Future.value(pickedFile?.path);

  }



  ImagePicker _picker;



  ImagePicker _getPicker() {

    _picker ??= ImagePicker();

    return _picker;

  }



  @override

  Map<String, Function> getRegisterMethods() {

    var functions = <String, Function>{};

    //注册需要暴露的方法，名称对应js侧请求的方法

    functions.putIfAbsent('getPhoto', () => getPhoto);

    return functions;

  }

}



```



2.JS侧实现



```

let FairPhotoSelector = function () {

    return {

        getPhoto: function (req) {

            //准备需要发送的消息

               ...

            let reqMap = {

                pageName: req.get('pageName'),

                funcName: 'invokePlugin',

                //对应dart侧的类名以及需要调用的方法名

                'className':'FairPhotoSelector#getPhoto',

                args: {

                    callId: selectorId,

                    type: args.get('type'),

                }

            };

            //调用该方法，等待dart侧数据的返回

            invokeFlutterCommonChannel(JSON.stringify(reqMap), function (resp) {

                //处理dart端返回的请求结果

                //消费完之后及时移除回调

                FairPhotoSelectorCallback[respCallId] = null;

            });

        },



    };

};

```



- **使用**

1. 文件位置

将js文件放入assets目录下， 并在fair_basic_config.json文件中配置对应路径以及名称

eg:



```

{

  "coreJs": {

    "version": 1.0,

    "path": "assets/basic/fair_core.js"

  },

  "plugin": {

        ...

    "fair_photo": "assets/plugin/sample_fair_photo_selector.js"

  }

}

```



2.main函数中注册



```

void main() {

  WidgetsFlutterBinding.ensureInitialized();



  FairApp.runApplication(

    _getApp(),

    plugins: {

      'FairNet': FairNet(),

       //注册用户自定的插件名称

       //key值对应JS侧发送数据的name

      'WBPermission': WBPermission(),

      'FairPhotoSelector': FairPhotoSelector(),

    },

  );

}

```



**demo在assets目录下，可查看具体流程**


