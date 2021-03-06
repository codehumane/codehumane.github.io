---
layout:     post
title:      docker storage driver의 이해
date:       2016-04-07 12:32:18
summary:    도커 이미지 및 컨테이너의 레이어 구성과 관리, 그리고 선택
---

docker의 storage driver를 파악하기 위해 docker 공식 문서를 살펴보았고 이를 간단한 의견과 함께 요약했다. 참고 문서의 링크는 다음의 2가지이다.

- [Understand images, containers, and storage drivers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers)
- [Select a storage driver](https://docs.docker.com/engine/userguide/storagedriver/selectadriver)


## Layer

가장 먼저 이미지와 컨테이너를 구성하는 layer가 소개 된다. 이미지는 여러 개의 읽기 전용<sup>read-only</sup> 레이어로 이루어진 파일 시스템이며, 컨테이너가 실행될 때는 읽기 쓰기가 가능한 새로운 레이어가 이미지의 레이어 스택 최상단에 추가된다. 컨테이너가 수행되는 동안에 새로 추가되거나 수정된 데이터들은 이 쓰기 가능한 레이어에 기록되고, 컨테이너가 삭제되는 경우는 이 쓰기 가능한 레이어만 삭제되며 이미지의 레이어들은 그대로 남아 있게 된다.

이 구조를 그림으로 보면 다음과 같다.

![그림이 없는 경우 아래 URL을 참고하세요.](https://docs.docker.com/engine/userguide/storagedriver/images/container-layers.jpg "Container based on the Ubuntu 15.04 image")

그림 1. Container based on the Ubuntu 15.04 image (출처: [http://goo.gl/DhlZyD](http://goo.gl/DhlZyD))

도커의 storage driver는 바로 이러한 레이어들을 스택으로 관리하고 하나의 단일화된 뷰로 제공하는 역할을 수행한다.

> The Docker storage driver is responsible for stacking these layers and providing a single unified view.

그리고 storage driver가 무엇이냐에 따라 이 작업들이 어떻게 이뤄지는 지가 결정된다.



## Copy-on-write strategy

도커 이미지와 컨테이너를 관리하는 두 가지 핵심 기술은 이미지 레이어 스택과 copy-on-write(CoW) 전략이라고 한다. 레이어에 대해서는 위에서 다루었고, 이제 CoW에 대해서 살펴볼 차례이다. 문서에서는 CoW에 대한 설명을 다음과 같이 하고 있다.

> In this strategy, system processes that need the same data share the same instance of that data rather than having their own copy. At some point, if one process needs to modify or write to the data, only then does the operating system make a copy of the data for that process to use. Only the process that needs to write has access to the data copy. All the other processes continue to use the original data.

정리하면, 기본적으로 시스템 프로세스들끼리는 동일 데이터를 사용할 때, 데이터의 복사본이 아니라 데이터의 인스턴스를 공유한다. 데이터를 복사하는 경우는 특정 프로세스가 데이터를 수정하려고 할 때 이다. 이 데이터의 복사본은 데이터를 수정하려고 하는 프로세스에게만 접근 가능하다.

도커는 이 CoW 전략을 통해 이미지의 디스크 사용 효율과 컨테이너 실행 속도를 높이는데, 디스크 효율성에 대해서는 아래 이미지를 보면 된다.

![](https://docs.docker.com/engine/userguide/storagedriver/images/saving-space.jpg "Image Layer Sharing")

그림 2. Image Layer Sharing (출처: [http://goo.gl/DhlZyD](http://goo.gl/DhlZyD))

그림에는 changed-ubuntu라는 이름의 이미지와 ubuntu:15.04라는 이름의 이미지가 있는데, changed-ubuntu는 베이스 이미지<sup>base image</sup>를 ubuntu:15.04로 하는 이미지이다. 베이스 이미지가 가진 레이어는 공유를 하고(복사가 아니라), changed-ubuntu의 차이점만을 레이어 스택에 추가하고 있다. changed-ubuntu의 Dockerfile은 다음과 같다.

{% highlight html %}

FROM ubuntu:15.04
RUN echo "Hello world" > /tmp/newfile

{% endhighlight %}

즉, changed-ubuntu의 레이어인 94e6b7d2c720는 RUN 수행 부분임을 알 수 있다. 참고로, 이러한 디스크 효율성은 이미지 빌드 시의 속도 향상으로 나타나기도 한다. Dockerfile의 문장들을 일일이 수행하는 게 아니라, 이미 존재하는 레이어가 있다면 이를 그대로 공유하도록 하는 것이다.

다음은 컨테이너의 효율성과 성능에 관한 이야기인데, 먼저 아래 그림을 보자.

![](https://docs.docker.com/engine/userguide/storagedriver/images/sharing-layers.jpg "Container Layer Sharing")

그림 3. Container Layer Sharing (출처: [http://goo.gl/DhlZyD](http://goo.gl/DhlZyD))

컨테이너는 이미지의 레이어 스택 최상단에 새로운 쓰기 가능한 레이어를 추가하는 형태이므로, 여러개의 컨테이너가 실행되더라도 이미지의 레이어가 공유되어 공간 효율성이 높아진다. 또한, 컨테이너 실행 시 단지 얇고 쓰기 가능한 레이어<sup>thin and writable layer</sup>만 추가하면 되므로 속도도 빠르다(컨테이너 실행 시 마다 매번 이미지의 레이어들을 복사해야 한다고 생각해 보라).

한편, 이러한 구조로 인한 단점도 존재한다. 이미 존재하던 파일을 컨테이너가 수정할 때 storage driver의 copy-on-write 작업을 거치게 되고 이 때 성능 부하가 일어나는 것이다. 도커 문서에서는 이 성능 부하를 눈에 띌만한noticeable 이라는 수식어로 표현하고 있다.

> A copy-up operation can incur a noticeable performance overhead.

이러한 부하는 어떤 storage driver를 사용하냐에 따라 다르며, 따라서 어떤 storage driver를 사용할 것인지 알아두는 것은 중요해 보인다. 참고로, 많은 파일이나 레이어, 깊은 디렉토리 구조로 인한 부하는 이보다 더 크다는 이야기도 하고 있다. 다행히, 파일 수정 시의 copy-on-write는 최초 수정 시에만 발생하고 동일 파일에 대한 이후의 수정 시에는 이미 컨테이너에 있는 파일을 수정한다고 한다.


## storage driver의 역할

이미 storage driver의 역할에 대해서는 언급되었지만 다시 한 번 간단히 요약하면 다음과 같다.

- 레이어들을 스택으로 관리하고 하나의 단일화된 뷰 제공
- 레이어들을 copy-on-write 전략으로 관리
    - 일반적으로 레이어들을 복사가 아닌 공유하여 사용
    - 파일에 수정이 필요한 경우 이를 쓰기 가능한 레이어에 복사

참고로, 데이터 볼륨에 대한 관리는 storage driver의 역할이 아니다. 따라서 이 공간의 파일들을 읽고 쓰는 것은 호스트의 네이티브 속도로 이루어진다.


## storage driver의 종류 및 확인

도커에서 제공하는 storage driver는 여러 가지이며, 각각은 자신만의 방식으로 컨테이너 및 이미지의 레이어를 관리한다. 따라서 리눅스 환경과 사용 목적을 고려하여 적절한 것을 선택해야 안정성과 성능 측면의 이점을 누릴 수 있다. storage driver의 종류는 다음과 같다.

| Technology    | Storage driver name |
|---------------|---------------------|
| OverlayFS     | overlay             |
| AUFS          | aufs                |
| Btrfs         | btrfs               |
| Device Mapper | vfs                 |
| VFS*          | right               |
| ZFS           | zfs                 |

표1. storage driver의 종류 (출처: [http://goo.gl/DhlZyD](http://goo.gl/DhlZyD))

storage driver는 도커 데몬이 실행될 때 결정되며, 아무것도 지정하지 않은 경우에는 시스템 설정을 기반으로 적절한 것이 선택된다(이 때는 안정성<sup>stability</sup>이 중요한 결정 요소가 된다). 도커의 이미지와 컨테이너의 레이어들을 관리하는 것이 storage driver임을 생각할 때, 데몬 인스턴스에 의해 만들어진 컨테이너들에는 모두 동일한 storage driver가 사용됨을 알 수 있다. 실제로도 그러하다.

> As a result, the Docker daemon can only run one storage driver, and all containers created by that daemon instance use the same storage driver.

자신이 쓰고 있는 storage driver가 무엇인지 확인하려면 docker info 명령을 입력하면 된다.

{% highlight html %}

[dockeradmin@build-slave containers]# docker info
Containers: 1
 Running: 1
 Paused: 0
 Stopped: 0
Images: 89
Server Version: 1.10.3
Storage Driver: devicemapper
 Pool Name: docker-253:2-5152905-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 37.58 GB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 37.92 GB
 Data Space Total: 107.4 GB
 Data Space Available: 53.15 GB
 Metadata Space Used: 28.38 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 2.119 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: false
 Deferred Deletion Enabled: false
 Deferred Deleted Device Count: 0
 Data loop file: /home/lib/docker/devicemapper/devicemapper/data
 WARNING: Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.
 Metadata loop file: /home/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.107-RHEL7 (2015-12-01)
Execution Driver: native-0.2
Logging Driver: json-file
Plugins:
 Volume: local
 Network: bridge null host
Kernel Version: 3.10.0-327.10.1.el7.x86_64
Operating System: CentOS Linux 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 16
Total Memory: 15.67 GiB
Name: do-build-slave.terracetech.co.kr
ID: NHFZ:QP7R:NJPP:R7CA:3Q4F:7O7F:KO3V:SFSA:LPXF:XBCA:ZTYT:P37I
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled

{% endhighlight %}

위 정보는 현재 Jenkins의 [Docker Slave](https://wiki.jenkins-ci.org/display/JENKINS/Docker+Plugin)로 활용하는 장비에 대한 것이다. 살펴보면 storage driver는 devicemapper, 파일 시스템은 xfs임을 알 수 있다. 한가지 주목할 만한 것은 WARNING 메시지이다.

> WARNING: Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.

프로덕션 환경에서 사용하지 않기를 강하게 권장<sup>strongly discouraged</sup>한다는 것인데, 다른 대안을 찾는 법에 대해서는 바로 다음에서 설명한다.


## storage driver 선택하기

기본으로 설정되는 storage driver 대신 직접 지정하고 싶다면 어떻게 결정해야 할까? 도커 공식 문서에서는 안정성<sup>stability</sup>과 경험 및 전문성<sup>experience and expertise</sup>을 언급하고 있다. 안정성에 대해서는 아래 2개를 권고한다.

1. Use the default storage driver for your distribution.
2. Follow the configuration specified on the CS Engine compatibility matrix.

storage driver는 도커가 설치될 때 시스템 구성에 따라 적절히 선택되는데, 이는 안정성을 고려한 선택이여서 이 기본값을 벗어나는 경우에는 문제를 겪을 가능성이 커진다고 한다. 다음으로는 CS Engine(도커의 상용 버전)의 호환성 매트릭스 명시된 구성을 따르라고 되어 있다. 마찬가지로 이를 따르지 않을 경우 문제를 겪을 가능성이 커진다.

경험 및 전문성은 단지 자신이 익숙한 것을 쓰는게 도움이 된다는 이야기이다.

마지막으로, 아래 그림은 각 storage driver의 특성을 비교하는 다이어그램이다. 앞에서 [Jenkins Docker Slave](https://wiki.jenkins-ci.org/display/JENKINS/Docker+Plugin) 장비의 docker info 내용에서 봤던 경고 메시지처럼, devicemapper (loop)는 production과 performance에서 빨간색 표시(“If bad for use case”를 가리킴)가 되어 있음을 다시 한 번 확인할 수 있다.

![](https://docs.docker.com/engine/userguide/storagedriver/images/driver-pros-cons.png "storage driver pros and cons")
- 그림 4. storage driver pros and cons (출처: [http://goo.gl/EVJeGI](http://goo.gl/EVJeGI))
