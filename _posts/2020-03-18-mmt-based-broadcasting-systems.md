---
layout: post
title: '[MMT] Service configuration for MMT-base broadcasting systems'
categories: [Development]
tags: [research, mmt, broadcasting]
feature_image: https://upload.wikimedia.org/wikipedia/commons/thumb/b/ba/MPEG_Transport_Stream_HL.svg/800px-MPEG_Transport_Stream_HL.svg.png
last_modified_at: '2022-04-09 23:50:01'
---

<!-- more -->

# 시스템 구조

{% include figure.html image="/assets/img/20200318/fig1.png" position="center" %}

MMT-based broadcasting system 에서 video, audio 등 media components 와, closed captions 등 TV 프로그램을 구성하는 요소들은 MFU (Media fragment unis) / MPU (Media processing units) 단위로 캡슐화되고, 이는 다시 MMTP packet 의 MMTP payload 에 실려 IP packet 의 형태로 전송된다. 또한 여러 개의 방송 채널들은 IP multiplexing scheme 에 따라 (e.g. TLV-Type length value multiplexing) 서로 섞여 전송된다.

MMT 는 또한 EPG (Electronic Program Guide) 등의 TV 프로그램의 구조 혹은 TV 서비스와 관련된 정보들을 MMT-SI (MMT Signalling information) 로 전달하는데, 이 역시 마찬가지로 MMTP packet 에 실려 IP packet 의 형태로 전송되며, 방송 수신기와 방송국 간의 시간 동기화를 위한 UTC 신호 역시 동일한 구조를 통해 전송된다.

# Service 구조

## Services in a broadcasting channel

{% include figure.html image="/assets/img/20200318/fig2.png" position="center" %}
Broadcasting service 란? 일련의(a series of) TV 프로그램의 집합이다. MMT 기반 방송 시스템에서는, 하나의 MMT package 가 하나의 'broadcasting service' 에 해당한다. MMT package 내에서 각각의 개별 프로그램(event 로 표현)들은 start time / end time 을 이용하여 구분된다. ISO/IEC 23008-1 (이하 MMT 표준 이라 칭함) 에서 하나의 media component 는 'Asset' 으로 정의하며, 이는 일련의 MPU 의 집합이다. 하나의 TV 프로그램은 하나 이상의 Asset 과 signalling information 을 포함하는 MMT package 이다. MPT (MMT package table) 는 MMT package (TV 프로그램) 을 구성하는 asset 들에 대한 정보를 담고 있는데, 이는 MMT-SI 의 PA (package access) message 를 이용하여 전달될 수 있다.
여러 개의 IP data flow 는 하나의 layer 2 stream 으로 다중화되어 전송될 수 있으며, layer 2 stream 은 IP packet 을 역다중화하는 데 필요한 signalling information 을 포함한다.

## Services in b/c channels and broadband networks

{% include figure.html image="/assets/img/20200318/fig3.png" position="center" %}
MMT-based 시스템에서는 하나의 program (MMT package) 에 속하는 Asset 들을 broadcast channel 과 broadband network 을 통해 동시에 전달하는 것이 가능하다. broadcast channel 을 통해 전달되는 Asset 들은 모든 수신 단말에 공통적으로 수신되지만, broadband network 을 통한 asset 은 그것을 요청(request)한 특정 단말에만 전달된다. (support hybrid delivery of multimedia content.)
