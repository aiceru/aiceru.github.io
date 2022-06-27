---
layout: post
title: '[Cycling] Drift Ghost XL 먹통현상'
categories: [Cycling]
tags: [blackbox, dashcam, drift, ghost xl, firmware, freezing]
feature_image: '/assets/img/posts/20220627/feature.png'
published: true
last_modified_at: '2022-06-27 11:54:11'
---

<!-- more -->
# 📌 Drift Ghost XL 펌웨어 업그레이드 후 오류 현상
라이딩할 때 블랙박스 용도로 Drift 사의 Ghost XL 을 구입하여 잘 사용하고 있습니다. 후기들을 찾아보면 녹화중 프리징 현상이 자주 발생한다 하는데, 다행인지 아니면 저는 아주 초기에 구매하여 그런지 몰라도 양품을 받아 아무런 문제 없이 사용하고 있었습니다. 혹시나 모를 용량 부족을 방지하기 위해 라이딩을 다녀올 때마다 WIFI로 모바일 앱과 연결하여 메모리 카드 포맷을 해 주는데, 일주일 전쯤부터 새로운 펌웨어 업데이트가 있다는 알림이 뜹니다. 블루투스 커넥션을 개선한 내용이라는데, 저는 리모컨을 사용하지 않아 쓸모는 없지만 그래도 항상 최신 펌웨어를 유지하는 편이라 업데이트를 실행했습니다.

<div style="display:flex">
{% include figure.html image="/assets/img/posts/20220627/fw_upgrade_in_app_01.png" caption="업데이트 알림" %}
{% include figure.html image="/assets/img/posts/20220627/fw_upgrade_in_app_02.png" caption="업그레이드 중" %}
{% include figure.html image="/assets/img/posts/20220627/fw2043.png" caption="Firmware v2043" %}
</div>

그런데 업데이트 이후, 전원 버튼을 눌러도 Auto recording 도 작동하지 않고, 심지어는 전원이 잘 켜지지도 않는 현상이 발생합니다.
{% include video.html id="tqH24oVqmes" %}

전원 버튼을 몇십 초 이상 꾹 눌러도 켜지지 않다가도, 또 어느 정도 가만 놔뒀다가 전원 버튼을 3초가량 길게 누르면 간헐적으로 켜지기도 합니다. "알리발" 이라는 인식 때문인가, 내구성이 다해서 드디어 보내줄 때가 되었나 싶다가도, 아니 아무리 그래도 이렇게 갑자기? 라는 생각도 들고... 또 아예 안켜지는 것도 아니고 어쩌다 한번씩 켜지기만 하면 정상 작동을 하니...

아무래도 펌웨어의 문제다 싶어서, Drift 사의 공식 홈페이지를 방문하니 최신 펌웨어의 날짜가 *2021년 4월 30일* 입니다. 모바일 앱에서 펌웨어 업데이트 알림이 뜬 건 최근의 일이라 혹시나 해서 microSD 카드를 이용한 펌웨어 덮어쓰기가 가능할지 시도해 봤습니다.

1. [Drift 공식 홈페이지](https://driftinnovation.com/blogs/support/ghost-xl)에 가서 "Version 2.0.2.9 (30/04/2021)" 의 링크를 클릭해 GHOST_XL.bin 펌웨어 파일을 다운로드합니다. 
2. 어떻게든 🤦‍♀️ 전원을 켭니다. 제 경우 microSD 카드를 뺐다꼈다, 전원버튼을 30초이상 눌렀다 뗐다 다시누르기, PC와 USB 케이블로 연결/해제 반복, 혹은 몇 분 동안 가만 놔뒀다가 다시 전원버튼 누르기 등등 다양한 방법을 시도했는데, 하다 보면 대략 10분에 한번 정도씩은 전원이 들어옵니다. 전원이 들어오면 그대로 전원을 끄지 말고 PC 에 연결하여 USB 모드로 진입시킵니다.
3. microSD 카드의 루트에 위에서 다운로드한 GHOST_XL.bin 파일을 복사해 줍니다.
{% include figure.html image="/assets/img/posts/20220627/sdcard.png" position="center" caption="sdcard root" %}
4. PC 에서 안전하게 USB 저장소를 연결 해제 한 후, 케이블을 뺍니다. 이후 기기가 자동으로 재시작하고 빨간색 LED 에 "FIRMWARE UPDATING" 메시지가 표시되면서 펌웨어를 덮어씁니다.
<div style="display:flex">
{% include figure.html image="/assets/img/posts/20220627/fw_upgrading.jpg" caption="updating firmware" %}
{% include figure.html image="/assets/img/posts/20220627/fw2029.png" caption="Firmware v2029" %}
</div>
1. 덮어쓰기가 완료되면 자동으로 한두 번 더 재시작이 되고, 이후로는 모든 기능이 정상 작동합니다.
{% include video.html id="x_iiLKCQW9s" %}

결론은 2043 버전 펌웨어의 문제였던 듯 합니다. 2029 버전의 펌웨어로 덮어쓰고 나서는 문제가 완전히 해결되어 자동 녹화 기능 (전원 꺼진 상태에서 전원 버튼을 짧게 누르면 바로 녹화 시작)도 잘 동작하고, 3초간 눌러서 전원을 켜는 동작도 문제없이 동작합니다. 혹시나 최근에 앱에서 펌웨어 업데이트를 하셨다가 먹통이 된 케이스가 있다면 펌웨어를 이전 버전으로 덮어써 보시기 바랍니다.
