---
layout: post
title: '[Flutter] Bidirectional pull to refresh ListView'
categories: [Development]
tags: [flutter, listview, pull-to-refresh, bidirectional, pull-to-load-more]
feature_image: '/assets/img/posts/20220411/scroll.jpg'
feature_license: 'Photo by <a href="https://unsplash.com/@taypaigey?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Taylor Wilcox</a> on <a href="https://unsplash.com/s/photos/scroll?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>'
use_math: false
published: true
last_modified_at: '2022-04-09 23:50:01'
---

<!-- more -->

앱 화면을 구성할 때, top position 에서 아래로 끌어당겼다가 놓으면 페이지가 새로고침되는 UX 를 많이 이용한다.

또한 List View 를 이용하여 데이터를 렌더링할 경우 대량의 데이터를 한꺼번에 fetch 해 오면 네트워크 호출에 의한 성능 문제가 발생할 수 있어 보통 pagination 기법을 사용한다. 초기 로딩시 일정 page size 의 data 만 fetch 해 두고, list scroll 이 맨 아래에 도달하면 추가로 data fetch 를 수행하는 식이다.

이 두 가지 기능을 하나의 ListView 에서 동시에 사용할 수 있도록 구현해 보자.

# Simple pull(down)-to-refresh

Pull-to-refresh 기능은 단순히 [`RefreshIndicator`](https://api.flutter.dev/flutter/material/RefreshIndicator-class.html) class 를 이용하는 것 만으로도 쉽게 구현이 가능하다.

- RefreshIndicator 를 이용하여 ListView.builder 를 감싸준다.
- RefreshIndicator 의 onRefresh property 에 새로고침 기능을 구현한다.

```dart
List<String> dummies = List.generate(45, (index) => 'Item $index');

class MyListView extends StatefulWidget {
  const MyListView({Key? key}) : super(key: key);

  @override
  State<MyListView> createState() => _MyListViewState();
}

class _MyListViewState extends State<MyListView> {
  late List<String> items;

  @override
  void initState() {
    super.initState();
    items = dummies.sublist(0, 20);
  }

  Future<void> refresh() {
    return Future.delayed(const Duration(seconds: 1));  // 1초 후 리턴
  }

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: () => refresh().then((_) => {
        ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('refresh!!!')) // refresh 완료시 snackbar 생성
        )
      }),
      child: ListView.builder(
          itemCount: items.length,
          itemBuilder: (context, index) {
            return ListTile(
                title: Text(items[index])
            );
          }),
    );
  }
}
```

{% include figure.html image="/assets/img/posts/20220411/simple-refresh.gif" position="center"
width="200" caption="Simple pull-to-refresh" %}

# Pull(up) to 'load more'

dummy data로 45개의 String item을 생성했고, state 생성시 'Item 0' ~ 'Item 19' 의 20개 데이터만 가져와 items 변수에 할당했다. 이제 _**Pull up to load more**_ 기능을 추가해 보자.

1. `ListView` 의 Scrolling position을 감지하기 위해 [`ScrollController`](https://api.flutter.dev/flutter/widgets/ScrollController-class.html) class 를 이용한다.
2. `ListView` 를 렌더링할 때, 약간의 트릭을 적용한다.
  - 더 가져와야 할 data가 남아있을 경우 `itemCount` property 에 data length + 1 을 넘겨주어 맨 아래 하나의 fake item 이 추가로 그려지도록 한다.
  - 더이상 가져와야 할 data가 없다면 `itemCount` 에 data length 를 넘겨준다.

## ScrollController

```dart
class _MyListViewState extends State<MyListView> {
  ...
  // scroll controller
  late ScrollController scrollController;
  ...
  @override
  void initState() {
    super.initState();
    ...

    // scroll controller 초기화, 리스너 추가
    scrollController = ScrollController();
    scrollController.addListener(onScroll);
  }

  ...
  // scroll controller listener
  void onScroll() {
    if (scrollController.position.pixels == scrollController.position.maxScrollExtent) {
      dataFuture = loadmore(items.length, pageSize);
      setState(() {});
    }
  }

  Future<List<String>> loadmore(int offset, int limit) async {
    return Future.delayed(
      // 2초 후에 dummy data 의 sublist (offset ~ offset+limit) 를 반환
      const Duration(seconds: 2), () => getPagedData(offset, limit)
    );
  }

  List<String> getPagedData(int offset, int limit) {
    // 이후에 구현
    print('getPagedData');
    return List.empty();
  }
  ...

  @override
  Widget build(BuildContext context) {
    return FutureBuilder (
        future: dataFuture,
        builder: (context, AsyncSnapshot<List<String>> snapshot) {
          if (snapshot.hasData) {
            ...
            return RefreshIndicator(
              ...
              child: ListView.builder(
                  controller: scrollController,
                  itemCount: items.length,
                  itemBuilder: (context, index) {
                    ...
                    return ListTile(
                        title: Text(items[index])
                    );
                  }
              ),
            );
          }
          return const SizedBox.shrink();
        }
    );
  }
}
```
`ScrollController` 를 이용하여 `ListView` 의 마지막<sub>bottom</sub> 부분 스크롤 이벤트를 캐치해야 한다. `initState` 에서 `ScrollController` 객체를 생성하고, 리스너 함수를 등록해 준다. 리스너 함수 `onScroll` 에서는 스크롤 포지션이 `maxScrollExtent` 에 도달했을 경우 추가 데이터를 로딩하도록 구현한다.
마지막으로 `ListView.builder` 의 `controller` property 에 위에서 생성한 `ScrollController` 를 연결해준댜.

## Rendering ListView

> 전체 소스

```dart
List<String> dummies = List.generate(45, (index) => 'Item $index');

class MyListView extends StatefulWidget {
  const MyListView({Key? key}) : super(key: key);

  @override
  State<MyListView> createState() => _MyListViewState();
}

class _MyListViewState extends State<MyListView> {
  late List<String> items;

  // scroll controller
  late ScrollController scrollController;
  // data 를 받아올 Future 객체
  late Future<List<String>> dataFuture;

  final int pageSize = 20;
  bool hasMoreData = true;

  @override
  void initState() {
    super.initState();
    items = List.empty(growable: true);

    // scroll controller 초기화, 리스너 추가
    scrollController = ScrollController();
    scrollController.addListener(onScroll);
    // 초기 data loading
    dataFuture = loadmore(0, pageSize);
  }

  @override
  dispose() {
    scrollController.dispose();
    super.dispose();
  }

  // scroll controller listener
  void onScroll() {
    if (scrollController.position.pixels == scrollController.position.maxScrollExtent) {
      dataFuture = loadmore(items.length, pageSize);
      setState(() {});
    }
  }

  Future<void> refresh() {
    return Future.delayed(
      // 1초 후에 items 를 리셋 (dummies[0~20])
      const Duration(seconds: 1), () => items = dummies.sublist(0, pageSize)
    );
  }

  Future<List<String>> loadmore(int offset, int limit) async {
    return Future.delayed(
      // 2초 후에 dummy data 의 sublist (offset ~ offset+limit) 를 반환
      const Duration(seconds: 2), () => getPagedData(offset, limit)
    );
  }

  List<String> getPagedData(int offset, int limit) {
    if (offset >= dummies.length) {
      return List.empty();
    } else if (offset + limit >= dummies.length) {
      return dummies.sublist(offset, dummies.length);
    } else {
      return dummies.sublist(offset, offset+limit);
    }
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder (
        future: dataFuture,
        builder: (context, AsyncSnapshot<List<String>> snapshot) {
          if (snapshot.hasData) {
            List<String> got = snapshot.data!;
            // 반환된 list 가 empty 가 아니면
            if (got.isNotEmpty) {
              // items 가 비어있거나, items 의 마지막 element 가 새로 들어온 data list 의 첫 값보다 작으면
              // items 에 새로 들어온 data list 를 전부 추가한다.
              if (items.isEmpty || items.last.compareTo(got.first) < 0) {
                items.addAll(snapshot.data!);
              }
            }
            // items 와 dummies 의 길이를 비교하여 더 받아올 데이터가 남아있는지 판단
            hasMoreData = items.length != dummies.length;
            return RefreshIndicator(
              onRefresh: () => refresh().then((_) => setState(() => {})),
              child: ListView.builder(
                  controller: scrollController,
                  // trick! 더 받아올 데이터가 있으면 [items.length + 1] => 리스트의 마지막에
                  // 하나의 item 을 추가로 렌더링하게 만든다.
                  itemCount: hasMoreData ? items.length + 1 : items.length,
                  itemBuilder: (context, index) {
                    // 렌더링하는 아이템의 index 가 items.length 와 같으면 (마지막 fake item)
                    // Progress indicator 를 렌더링
                    if (index == items.length) {
                      return const Center(
                        child: CircularProgressIndicator(),
                      );
                    }
                    return ListTile(
                        title: Text(items[index])
                    );
                  }
              ),
            );
          }
          return const SizedBox.shrink();
        }
    );
  }
}

```

Data 의 갱신에 따라 `ListView` 가 새로 그려져야 하므로, `FutureBuilder` 를 사용하도록 수정했다. 눈여겨 볼 부분은 `hasMoreData` 변수를 수정하고 이용하는 부분이다. 코드에 comment 로 설명을 붙여 놓았지만, 아래에서 조금 더 자세히 설명한다.

### Data merge

```dart
    ...
    return FutureBuilder (
        future: dataFuture,
        builder: (context, AsyncSnapshot<List<String>> snapshot) {
          if (snapshot.hasData) {
            List<String> got = snapshot.data!;
            // 반환된 list 가 empty 가 아니면
            if (got.isNotEmpty) {
              // items 가 비어있거나, items 의 마지막 element 가 새로 들어온 data list 의 첫 값보다 작으면
              // items 에 새로 들어온 data list 를 전부 추가한다.
              if (items.isEmpty || items.last.compareTo(got.first) < 0) {
                items.addAll(snapshot.data!);
              }
            }
            // items 와 dummies 의 길이를 비교하여 더 받아올 데이터가 남아있는지 판단
            hasMoreData = items.length != dummies.length;
            ...
          }
          ...
        }
      ...
    );
    ...
```
추가로 로드한 data 를 기존의 items 변수에 병합하는 부분이다. 체크해야 할 사항이 의외로 많다.
- Future 에서 반환된 List 가 비어 있다면: 더 이상 추가로 로드할 데이터가 없는 상황.<br/>hasMoreData = false 로 설정한다.
- `items.isEmpty`: 처음으로 데이터가 로딩되는 상황.<br/>로드한 데이터를 items 변수에 전부 추가 (`addAll`) 한다.
- `items.last.compareTo(got.first) < 0`: 새로 로드된 데이터와 기존 데이터를 비교하여 병합 여부를 결정

실제 구현하면서 마지막 case 를 처리해야 하는 것을 발견하기까지 가장 애먹었다. 여기에서는 예제로 `FutureBuilder` 를 사용했지만 프로젝트를 구현하면서는 rxDart 와 `StreamBuilder` 를 사용했는데, 종종 예측하지 못한 widget tree rebuild 가 발생하는 경우가 있었고, 페이지<sub>route</sub> 가 처음으로 로딩될 때에는 꼭 `build` method 가 두 번쌕 실행되어 초기 데이터가 중복으로 렌더링되었다.

> RxDart 의 PublishSubject, BehaviorSubject 등은 snapshot data 를 access/read 해도 stream을 비우지 않는다. 실제로 subject 에 추가된 데이터가 없는 상태에서 widget rebuild 가 일어나 다시 stream 에 접근할 경우 가장 최근에 가져왔던 데이터를 다시 (중복하여) 가져오게 된다.

오랜 검색 끝에 내린 결론은,
1. UI layer 에서는 현재 상태<sub>state</sub> 에 관여하지 않도록.
2. 같은 상태에서는 widget tree 가 몇 번을 rebuild 되어도 같은 결과가 나오도록.


구현하는 것이 해결책이었다. 예제에서는 코드가 너무 길어지는 것을 피하기 위해 하나의 클래스에 구현했지만, 실제 프로젝트에서는 BLoC 패턴 등을 사용하여 데이터 fetch 로직과 UI 로직을 완전히 분리시키는 것이 좋다.

### Rendering

```dart
...
return RefreshIndicator(
  onRefresh: () => refresh().then((_) => setState(() => {})),
  child: ListView.builder(
      controller: scrollController,
      // trick! 더 받아올 데이터가 있으면 [items.length + 1] => 리스트의 마지막에
      // 하나의 item 을 추가로 렌더링하게 만든다.
      itemCount: hasMoreData ? items.length + 1 : items.length,
      itemBuilder: (context, index) {
        // 렌더링하는 아이템의 index 가 items.length 와 같으면 (마지막 fake item)
        // Progress indicator 를 렌더링
        if (index == items.length) {
          return const Center(
            child: CircularProgressIndicator(),
          );
        }
        return ListTile(
            title: Text(items[index])
        );
      }
  ),
);
...
```
마지막으로, 화면에 list item 들을 rendering 하는 부분이다. `ListView` 의 `itemCount` property 에 넣어 주는 값이 상황에 따라 다르다. 추가로 로딩할 데이터가 더 남아있는 경우 items.length 에 1을 더해 추가로 fake item 이 렌더링되도록 한다. 아래 `itemBuilder` 함수의 첫 번째 if 문에서 렌더링하는 item 의 index 가 items 의 길이와 같을 경우 Progress indicator 를 출력한다. 여기서 조금 더 위로 스크롤이 올라가면 처음에 추가한 `ScrollController` 의 리스너가 호출되어 추가로 데이터를 로딩하게 된다.

{% include figure.html image="/assets/img/posts/20220411/load-more.gif" position="center"
width="200" caption="pull-up to load more" %}
