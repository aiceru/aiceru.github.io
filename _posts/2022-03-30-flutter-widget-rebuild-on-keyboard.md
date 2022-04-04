---
layout: post
title: '[Flutter] keyboard가 나타날 때 widget rebuild 일어나는 문제 해결'
categories: [Development]
tags: [app, flutter, widget, rebuild, keyboard, textfield, textformfield]
feature_image: 'https://images.unsplash.com/photo-1604246914453-b0d2afe6dcc0?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2940&q=80'
feature_license: Cover picture by <a href="https://unsplash.com/@chrisbriscoe?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Christopher Briscoe</a> on <a href="https://unsplash.com/s/photos/rebuild?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
---

<!-- more -->
Flutter 에서 BLOC pattern 을 이용하여 개발할 때, `statefulWidget` 을 상속받아 화면`route` 를 구현하는데 network 을 통해 data fetch 하는 등의 동작을 [`didChangeDependencies`](https://api.flutter.dev/flutter/widgets/State/didChangeDependencies.html) method 를 Override 하여 넣는다. 이 때 화면의 위젯 트리에 `TextField` 나 `TextFormField` 등이 포함되어 있을 경우 이를 탭하여 SW Keyboard 가 화면에 display 되면 키보드 영역이 route 영역을 밀어올리면서 위젯들이 그려져야 하는 영역이 재계산되어 위젯 트리의 rebuild 가 발생하게 된다.

# 문제 발생
내 경우에는 `didChangeDependencies` method 에서 bloc 의 fetch method (server API 호출 → data fetch 로 이어지는) 들을 호출하고 exception 발생 시 그 내용을 담은 dialog 를 띄우도록 해 두었는데, API call 이 성공적일 경우에는 눈에 보이는 문제가 없다가 exception dialog 를 띄우는 데서 문제가 발생했다. API 호출이 정상적으로 종료되면 화면의 위젯을 갱신하는데 fetch 한 데이터가 이전과 동일할 경우에도 실제 화면에는 변화가 없으니 일단 표면적으로는 문제가 없는 것으로 보였다. (🧨 물론 실제로는 백그라운드에서 필요없는 API 호출이 계속 반복되고 있었지만... 🧨)

Alpha test version 을 배포하고 [개밥을 먹는](https://en.wikipedia.org/wiki/Eating_your_own_dog_food) 도중, 서버에서 발행한 JWT token 이 만료되는 케이스에서 실제로 눈에 띄는 문제가 발생하기 시작했다.
    
1. API 호출에서 `Auth token expired` 에러가 발생
2. Error dialog 가 나타남.
3. 사용자가 close 버튼을 눌렀지만 짧게는 3~4개부터 많게는 10여개에 이르는 error dialog 가 중복해서 나타남.

## Errod dialog 중복 방지
처음에는, 하나의 page(`route`) 에서 2~3개의 API call 을 수행하면서 쌓여있던 error 들에 의해 dialog 가 중복 표시되는 것으로 생각하고 error dialog 를 구현한 class 에 static flag 를 두어 중복 발생을 막는 방법으로 해결하려 했다.

```dart
class ErrorDialog {
  ...
  static bool _isShowing = false;

  Future? _show(BuildContext context, AlertContents alert) {
    if (_isShowing) {
      return null;
    }

    _isShowing = true;
    return showDialog<void>(
      ...
    );
  }
```
확실히 이전보다 중복 dialog 발생이 줄어들긴 했지만, 문제가 완전히 해결되지는 않았다. 주로 `TextField` 나 `TextFormField` 를 탭하여 새로운 내용을 입력하려 할 때 error dialog 의 중복 현상이 나타나는 것을 보고, `didChangeDependencies()` method 가 반복적으로 실행되고 있나 의심되어 서버측 log 를 확인해 보니 아니나다를까 매번 `TextField` 를 탭할 때마다 `didChangeDependencies()` 에 포함된 API 호출이 일어나고 있었다. 심지어 현재 navigation stack 의 가장 상위에 있는 route 뿐만 아니라, 아래쪽에 있는 route 들 역시 동시다발적으로 `didChangeDependencies()` method 가 실행되고 있었다. 🤯 🙀

# 근본적인 대책
서두에서 밝힌 바와 같이, 사용자 입력 widget 을 탭하여 SW 키보드가 나타나면서 화면의 영역을 차지하게 되면, flutter 는 키보드가 차지하는 영역을 제외한 나머지 공간에 widget 들을 재배치하기 위해 전체 widget tree 를 rebuild 한다. 이 때 각 route 들의 `didChangeDependencies()` 가 호출되면서 원치 않는 API 호출과 이에 따른 error dialog 발생이 대량으로 일어난 것이다.

문제를 해결하기 위해서는 keyboard 가 나타날 때 일어나는 widget tree rebuild 를 막아주면 된다. [`Scaffold`](https://api.flutter.dev/flutter/material/Scaffold-class.html) Object 의 `resizeToAvoidBottomInset` property 에 `false` 값을 주면, 기존 위젯 트리가 keyboard 영역에 밀려 올라가는 것이 아니라 keyboard 아래에 그대로 유지되므로 widget tree 의 영역을 재계산하지 않고 rebuild 도 일어나지 않게 된다. 아래는 flutter api document 의 해당 property 에 대한 설명이다.

> **resizeToAvoidBottomInset property Null safety**
> &nbsp;  
> &nbsp;  
> bool? resizeToAvoidBottomInset  
> &nbsp;  
> final  
> &nbsp;  
> If true the body and the scaffold's floating widgets should size themselves to avoid the onscreen keyboard whose height is defined by the ambient MediaQuery's MediaQueryData.viewInsets bottom property.  
> &nbsp;  
> For example, if there is an onscreen keyboard displayed above the scaffold, the body can be resized to avoid overlapping the keyboard, which prevents widgets inside the body from being obscured by the keyboard.  
> &nbsp;  
> Defaults to true.

# 아니 근데 외않되
문제는 모든 route 의 Scaffold 에 위의 property 를 설정해 주고 난 뒤에도 계속되었다. 또 포풍 구글링을 시전한 뒤 flutter 의 issue thread 에서 답을 찾을 수 있었다. scaffold 자체는 키보드 영역의 z축 아래쪽에 위치하여 키보드가 나타나고 사라지는 데 영향을 받지 않지만, 내부의 widget 들 중 [`MediaQuery`](https://api.flutter.dev/flutter/widgets/MediaQuery-class.html) 에 의존성을 가진 부분이 있으면, 키보드 영역의 변화에 의해 MediaQuery 내부에서 재계산이 발생하여 이것이 해당 widget 의 재계산으로 이어지므로 결국 `didChangeDependencies()` 가 호출되고 마는 것. 따라서 widget rebuild 를 방지하려면 MediaQuery 에 의존하는 부분을 제거해 주어야 하고, 내 경우는 [sizer](https://pub.dev/packages/sizer) package 를 이용하여 이를 대체함으로서 완전히 해결하였다.

# References
https://api.flutter.dev/flutter/material/Scaffold/resizeToAvoidBottomInset.html  
https://github.com/flutter/flutter/issues/26004#issuecomment-757326742