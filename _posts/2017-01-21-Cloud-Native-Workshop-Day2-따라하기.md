---
layout:     post
title:      Cloud Native Workshop Day2 따라하기
date:       2017-01-21 15:32:18
summary:    Making a Spring Boot application Production Ready
---

# Making a Spring Boot application Production Ready

Josh Long의  '[Cloud Native Java Workshop](https://github.com/joshlong/cloud-native-workshop#2-making-a-spring-boot-application-production-ready)' 중 2일차인 '[Making a Spring Boot application Production Ready](https://github.com/joshlong/cloud-native-workshop#2-making-a-spring-boot-application-production-ready)' 과정을 따라해 보았다. 간단히 기억할 만한 것들을 문서로 함께 기록함.

## 과정 간단 소개

> In this lab, we'll look at how Spring Boot is optimized for the continuous delivery of applications into production.
>
> - 출처: [Making a Spring Boot application Production Ready](https://github.com/joshlong/cloud-native-workshop#2-making-a-spring-boot-application-production-ready)

코드의 완성과 프로덕션 반영은 완전히 다르며, 이 사이의 여정은 어느 누구의 예상보다 길다고 한다. 이 과정에서는 Spring Boot가 프로덕션 환경에서의 지속적 배포를 어떻게 돕는지 살펴본다.

## 전체 절차

- [x] `org.springframework.boot:spring-boot-starter-actuator` 추가
- [x] 사용자가 정의한 `HealthIndicator`를 통해 `HealthEndpoint` 커스터마이징
- [x] `./bin/graphite.sh` 실행
- [x] 2개의 환경 변수 `GRAPHITE_HOST` (`export GRAPHITE_HOST="$DOCKER_IP"`) and `GRAPHITE_PORT` (`2003`) 설정 (변수 설정 후 IDE를 재시작해야 할지도 모름)
- [x] `GraphiteReporter` @Bean 추가
- [x] `io.dropwizard.metrics:metrics-graphite` 추가
- [x] '완전히 실행 가능한' `.jar` 만들기
- [x] HAL 브라우저로 Actuator endpoint 살펴보기
- [x] Resoure Filtering 적용
- [x] Git commit ID 플러그인 적용
- [x] `@RepositoryEventHandler`와 `CounterService`로 Graphite에게 메트릭 보내기

## 참고 리소스

아래 3가지 리소스를 기반으로 학습을 진행함

1. [Cloud Native Workshop Hands-On Youtube](https://youtu.be/fxB0tVnAi0I?t=2350)
2. [Cloud Native Workshop Github repository](https://github.com/joshlong/cloud-native-workshop/tree/master/labs/2/reservation-service)
3. [Spring Boot Actuator: Production-ready features](http://docs.spring.io/spring-boot/docs/current/reference/html/production-ready.html)

## Actuator Endpoints 추가

- 어플리케이션과의 상호 작용 및 모니터링을 위한 방법을 제공함
- 자세한 설명은 [Spring Boot Endpoints](http://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)를 참고
- 이 기능을 쓰고 싶다면 `build.gradle`의 `dependencies` 항목에 아래 라인을 추가
    + `compile('org.springframework.boot:spring-boot-starter-actuator')`
    + classpath를 추가한 것만으로 사용이 가능함
- 서버를 시작한 후, URL 경로로 `/metrics`, `/health` 등의 endpoint 접근이 가능해짐을 확인
- 접근 가능한 모든 endpoint를 보고 싶다면, `/actuator` 경로로 접근
- 주요 endpoint 몇 가지에 대한 설명은 다음과 같다.

endpoint | 설명 | 기본값
-------- | --- | ----
actuator | 다른 모든 endpoint를 HATEOAS 방식으로 보여줌 | true
autoconfig | 모든 자동 설정<sup>auto-configuration</sup> 후보들을 적용/미적용 이유와 함께 보여줌 | true
health | 시스템 헬스<sup>health</sup> 정보를 보여줌 | true
trace | 시스템 트레이스(마지막 100개의 HTTP 요청) 정보 | true

## HealthIndicator 커스터마이징

- 위에서 소개된 Endpoint들은 커스터마이징 가능하며, 자신만의 Endpoint를 등록할 수도 있다.
- 2가지 방법이 존재함
    + 설정 파일의 수정
    + @Bean 등록
- 여기서는 Actuator 접근 경로와 Health Endpoint의 내용을 바꾸어본다.
- 먼저, endpoint 접근 prefix를 `/admin`으로 설정하는 작업
    + `src/main/resources/application.properties` 파일 열기
    + `management.context-path=/admin` 설정 추가
- 다음으로, 사용자 정의 `HealthIndicator`를 등록
    + `org.springframework.boot.actuate.health.HealthIndicator` 구현체를 `@Bean`으로 등록

```java
@Bean
HealthIndicator healthIndicator() {
    return new HealthIndicator() {
        @Override
        public Health health() {
            return Health.status("health.status.custom").build();
        }
    };
}
```

- 어플리케이션을 재시작하면 아래 2가지 변경 사항이 확인됨
    + endpoint 경로가 `/health` 대신 `/admin/health` 경로로 바뀜
    + `healthIndicator`의 `status` 항목 값이 `health.status.custom`로 바뀜

## Graphite 실행

- 아래의 shell 스크립트를 작성

```shell
#!/bin/bash
docker run --name cna-graphite -p 80:80 -p 2003:2003 -p 8125:8125/udp hopsoft/graphite-statsd
```

- shell이 하는 일을 살펴보면,
    + `cna-graphite`라는 컨테이너명으로 실행
    + 컨테이너와 호스트의 포트를 tcp에 대해서는 80:80 및 2003:2003, udp에 대해서는 8125:8125으로 대응
    + 실행할 이미지명은 `hopsoft/graphite-statsd`
    + 이미지가 로컬에 없으니 dockerhub로부터 가져오리라 예상
- graphite에 대한 설명은 [여기](https://matt.aimonetti.net/posts/2013/06/26/practical-guide-to-graphite-monitoring/)에 잘 나와있음(Spring Boot 문서에서도 이 글을 언급함). 설명 하나를 발췌하면,

> Graphite is used to store and render time-series data. In other words, you collect metrics and Graphite allows you to create pretty graphs easily.

- 즉, 시계열 데이터를 저장하고 보여줌. 또한 메트릭스를 수집하고 예쁜 그래프를 쉽게 만들도록 도와줌.
- [graphite 홈페이지](http://graphiteapp.org/)도 함께 참고
- 각 포트에 대한 설명은 아래 표 참고

포트  | 설명
---- | ---
80   | 그래프를 보여주는 웹 화면 접속에 사용됨
2003 | Carbon의 [plaintext protocol](http://graphite.readthedocs.io/en/latest/feeding-carbon.html#the-plaintext-protocol)로 데이터를 받기 위한 포트
8125 | [StatsD](https://github.com/etsy/statsd/blob/master/docs/server.md)가 보내는 데이터를 받기 위한 포트

## GRAPHITE_HOST 및 GRAPHITE_PORT 환경 변수 설정

- 다음 단계에 등장하는 GraphiteReporter @Bean을 위해 graphite 접근 정보를 환경 변수에 설정
- 우선 `$DOCKER_IP`를 구함
    + 이게 뭘까를 잠시 고민. graphite에 접근하기 위한 IP로 예상되며, 아래 명령어를 통해 IP 획득
    + `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} cna-graphite`
    + `172.17.0.2`인 것을 확인
    + 하지만 다시 생각해 보면, docker 실행 시 포트 매핑을 사용하므로 `localhost(127.0.0.1)`을 지정해 주면 됨
- 그리고 나서 `GRAPHITE_HOST` 및 `GRAPHITE_PORT`를 설정
    + Zsh를 사용하고 있으므로 `vi ~/.zshrc`로 파일 열기
    + 아래 두 라인 추가

```shell
export GRAPHITE_HOST=localhost
export GRAPHITE_PORT=2003
```

- 환경 변수 반영을 위해 shell 세션과 IntelliJ 재시작

## GraphiteReporter @Bean 추가

- GraphiteReporter는 graphite로 데이터를 보내주는 @Bean
- 우선, gradle에 아래 의존성 추가
    + `compile('io.dropwizard.metrics:metrics-graphite')`
    + 원래의 Cloud Native Workshop에서는 이 작업이 GraphiteReporter 이후의 별도 단계로 빠져 있음
- 아래와 같이 @Bean 추가

```java
@Bean
GraphiteReporter graphiteReporter(
        MetricRegistry registry,
        @Value("${graphite.host}") String host,
        @Value("${graphite.port}") int port) {

    GraphiteReporter reporter = GraphiteReporter.forRegistry(registry)
            .prefixedWith("reservations")
            .build(new Graphite(host, port));
    reporter.start(2, TimeUnit.SECONDS);
    return reporter;
}
```

- 코드를 간단히 살펴보면,
    + `GraphiteReporter`는 Graphite에게 측정<sup>metric</sup> 값들을 보내주는 보고자
    + `MetricRegistry`는 Spring Boot의 측정 값들이 등록되는 곳
        * 이 `MetricRegistry`의 측정값들이 `/metrics` endpoint를 통해 노출됨
        * [Spring Boot 레퍼런스 Dropwizard Metrics](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html#production-ready-dropwizard-metrics) 참조
    + `Graphite`는 `GraphiteReporter`의 보고 대상
    + 그리고 `GraphiteReporter`는 보고를 2분 간격으로 하도록 설정되어 실행됨
    + 참고로, host와 port의 값은 `@Value`를 통해 각각 `GRAPHITE_HOST`와 `GRAPHITE_PORT`의 값으로 할당됨
        * 설정값이 @Value에 할당되는 과정은 [Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html) 문서를 참고

## '완전히 실행 가능한' `.jar` 만들기

- `executable` 설정 플래그을 이용하면, '완전히 실행 가능한<sup>fully executable</sup>` jar를 만들 수 있음
- 즉, `java -jar ${jar명}` 명령어 대신, `./${jar명}`으로 어플리케이션이 실행 가능해짐
- 자세한 내용은 [Installing Spring Boot applications](http://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html) 참고
- gradle 사용시에는 `build.gradle`에 아래 내용을 추가

```gradle
springBoot {
    executable = true
}
```

- 설정을 추가한 후 `gradle build` 명령어로 빌드 수행
- `find . -name '*.jar'` 명령어로 찾은 jar 실행
    + `./build/libs/cloud-native-workshop-DAY2.jar`
    + 실제로 실행됨을 확인할 수 있음

## HAL 브라우저로 Actuator endpoint 살펴보기

- Spring Data REST 문서의 [The HAL Browser](http://docs.spring.io/spring-data/rest/docs/current/reference/html/#_the_hal_browser0) 설명

> The developer of the HAL spec has a useful application: the HAL Browser. It’s a web app that stirs in a little HAL-powered JavaScript. You can point it at any Spring Data REST API and use it to navigate the app and create new resources.
>
> Instead of pulling down the files, embedding them in your application, and crafting a Spring MVC controller to serve them up, all you need to do is add a single dependency.

- 보다 자세한 내용은 아래 리소스 참고
    + [HAL Brwoser](https://github.com/mikekelly/hal-browser)
    + [HAL Specification](http://stateless.co/hal_specification.html)
- 이제 직접 설정해보자.
- 가장 먼저 그리고 끝으로, classpath에 `org.springframework.data:spring-data-rest-hal-browser`추가
- Boot 재시작 후 `localhost:8080` 접근하여 웹앱을 확인
- 기본 설정은 루트 경로에서 HAL Browser를 제공함
- Explorer 항목에 `/admin`, `/admin/health` 등을 넣어보며 Actuator endpoint를 확인할 수 있음

## Resource Filtering

- 여기서부터는 이해를 돕기 위해 Cloud Native Workshop과는 절차를 조금 다르게 진행하려 하려함
- 우선은 Resource Filtering부터 시작할건데, 이는 원래 과정의 다음 2개 단계에 해당함
    + Configure Maven resource filtering and the Git commit ID plugin in the `pom.xml` in all existing and subsequent `pom.xml`s, or extract out a common parent `pom.xml` that all modules may extend.
    + Add `info.build.artifact=${project.artifactId}` and `info.build.version=${project.version}` to application.properties.
    + 하지만 git commit ID 플러그인에 대해서는 다음 단계에서 소개할 예정
- Resource Filtering 소개는 아래를 참고

> Variables can be included in your resources. These variables, denoted by the ${...} delimiters, can come from the system properties, your project properties, from your filter resources and from the command line.Variables can be included in your resources. These variables, denoted by the ${...} delimiters, can come from the system properties, your project properties, from your filter resources and from the command line.
>
> \- [Apache Maven Resource Plugin - Filtering](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)

- Gradle에서는 Resource Filtering이 없음
- 대신, `processResources`라는 task가 존재함
    + [Migrating a Maven Build to Gradle](https://gradle.org/migrating-a-maven-build-to-gradle/https://gradle.org/migrating-a-maven-build-to-gradle/) 참고
- 상세한 설명이 담긴 문서는 [여기](https://dzone.com/articles/resource-filtering-gradle)를 참고
- 확인을 위해 `build.gradle` 파일을 먼저 아래와 같이 작성

```gradle
import org.apache.tools.ant.filters.ReplaceTokens
processResources {
    filter ReplaceTokens, tokens: [
            "projectName": project.name,
            "projectVersion": project.version
    ]
}
```

- `tokens`에서 콜론 좌측은 토큰명이고, 우측은 대체할 값을 가리킴
- `processResources`의 기본 경로에 포함되는 `resources/application.properties` 파일을 아래와 같이 작성

```properties
process.resources.project.name=@projectName@
process.resources.project.version=@projectVersion@
```

- 터미널에서 `gradle processResources`을 수행해 보면 `/build` 경로의 `application.properties`가 다음과 같이 바뀌어 있음을 확인

```properties
process.resources.project.name=cloud-native-workshop
process.resources.project.version=day2
```

- 이는 또한 Actuator를 통해서도 확인 가능함
    + endpoint를 `admin/env`로 접근하여 `applicationConfig` 항목을 보면 값을 확인할 수 있음

## Git commit ID 플러그인 적용

- ['Git commit id plugin' Git Repository](https://github.com/ktoso/maven-git-commit-id-plugin/blob/master/README.md)에 가보면 이 플러그인의 가치와 사용 사례가 잘 나와 있음. 배포된 어플리케이션의 git revision 정보 제공은 꼭 필요한 일인데, 이 플러그인은 이를 설정만으로 가능하게 해줌
- [gradle plugin com.gorylenko.gradle-git-properties](https://plugins.gradle.org/plugin/com.gorylenko.gradle-git-properties) 문서를 따라 아래 내용을 `build.gradle`에 추가
    + [Limitations of the plugins DSL](https://docs.gradle.org/current/userguide/plugins.html#plugins_dsl_limitations)을 참고하여, 순서를 비롯한 몇 가지 제약사항이 있음에 유의

```gradle
plugins {
    id "com.gorylenko.gradle-git-properties" version "1.4.17"
}
```

- 확인을 위해 터미널에서 `gradle generateGitProperties` 수행
- 으헛, 안된다. `gradle generateGitProperties --debug --stacktrace`로 다시 실행하여 오류 내용을 확인. 그 중에서도 관련 있는 부분을 보면,

```
[ERROR] [org.gradle.BuildExceptionReporter] Caused by: org.eclipse.jgit.errors.RepositoryNotFoundException: repository not found: /Users/codehumane/dev/github/codehumane/learning/cloud-native-workshop/day2
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.eclipse.jgit.lib.BaseRepositoryBuilder.build(BaseRepositoryBuilder.java:581)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.eclipse.jgit.api.Git.open(Git.java:116)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.eclipse.jgit.api.Git.open(Git.java:99)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.eclipse.jgit.api.Git$open.call(Unknown Source)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.ajoberstar.grgit.operation.OpenOp.call(OpenOp.groovy:84)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.ajoberstar.grgit.operation.OpenOp.call(OpenOp.groovy)
[ERROR] [org.gradle.BuildExceptionReporter] 	at java_util_concurrent_Callable$call.call(Unknown Source)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.ajoberstar.grgit.util.OpSyntaxUtil.tryOp(OpSyntaxUtil.groovy:45)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.ajoberstar.grgit.Grgit$__clinit__closure1.doCall(Grgit.groovy:193)
[ERROR] [org.gradle.BuildExceptionReporter] 	at com.gorylenko.GitPropertiesPlugin$GenerateGitPropertiesTask.generate(GitPropertiesPlugin.groovy:76)
[ERROR] [org.gradle.BuildExceptionReporter] 	at org.gradle.internal.reflect.JavaMethod.invoke(JavaMethod.java:75)
[ERROR] [org.gradle.BuildExceptionReporter] 	... 64 more
```

- 이 플러그인의 [Git Repository](https://github.com/n0mer/gradle-git-properties)로 가서 [에러나는 코드 부분](https://github.com/n0mer/gradle-git-properties/blob/master/src/main/groovy/com/gorylenko/GitPropertiesPlugin.groovy#L73)을 열어보면 대략 다음과 같은 코드가 보임

```gradle
def repo = Grgit.open(dir: project.gitProperties.gitRepositoryRoot ?: project.rootProject.file('.'))
```

- 현재 프로젝트의 최상위 경로가 git repository의 root가 아니어서 발생하는 문제임을 예상할 수 있음
    + 좀 더 자세한 원인은 `jgit`의 [`BaseRepositoryBuilder`](https://github.com/spearce/jgit/blob/master/org.eclipse.jgit/src/org/eclipse/jgit/lib/BaseRepositoryBuilder.java) 참고
- `build.gradle`에 아래 내용을 추가한다.

```gradle
gitProperties {
    gitRepositoryRoot = new File('../../../')
}
```

- 다시 한 번 터미널에서 `gradle generateGitProperties` 수행
- `cat ./build/resources/main/git.properties`을 통해 git 정보가 생성되어 있음을 확인할 수 있음
- 이 내용은 `info` entrypoint에서도 확인 가능하다. 해서, `admin/info`을 접근하면 관련 내용을 확인할 수 있음
- 만약 actuator에서 `git.properties`의 모든 내용을 확인하고 싶다면 `application.properties`에 아래 내용 추가

```properties
management.info.git.mode=full
```

## `@RepositoryEventHandler`와 `CounterService`로 Graphite에게 메트릭 보내기

- [`Entity`의 이벤트를 처리하는 방법](http://docs.spring.io/spring-data/rest/docs/2.0.x/reference/html/events-chapter.html) 중의 하나로 `@RepositoryEventHandler` 애노테이션이 존재함
- 이벤트의 종류는 6가지임
    + BeforeCreateEvent
    + AfterCreateEvent
    + BeforeSaveEvent
    + AfterSaveEvent
    + BeforeLinkSaveEvent
    + AfterLinkSaveEvent
    + BeforeDeleteEvent
    + AfterDeleteEvent
- [자신만의 메트릭을 기록](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html)하기 위해 [`CounterService`](https://github.com/spring-projects/spring-boot/blob/v1.4.3.RELEASE/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/metrics/CounterService.java)나 [`GaugeService`](https://github.com/spring-projects/spring-boot/blob/v1.4.3.RELEASE/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/metrics/GaugeService.java)를 이용할 수 있음
- 메트릭의 이름으로는 어느 것이나 사용가능하지만, 메트릭을 보내는 툴의 가이드라인을 따르는 것이 좋음
    + Graphite는 여기 [가이드라인](https://matt.aimonetti.net/posts/2013/06/26/practical-guide-to-graphite-monitoring/)을 참고
- day1에서 작성했던 `Resrevation`의 생성과 소멸을 Graphite로 전송하기 위해 아래 코드를 작성

```java
@Component
@RepositoryEventHandler
public static class ReservationEventHandler {

    @Autowired
    private CounterService counterService;

    @HandleAfterCreate
    public void create(Reservation p) {
        count("reservations.create", p);
    }

    @HandleAfterSave
    public void save(Reservation p) {
        count("reservations.save", p);
        count("reservations." + p.getId() + ".save", p);
    }

    @HandleAfterDelete
    public void delete(Reservation p) {
        count("reservations.delete", p);
    }

    protected void count(String evt, Reservation p) {
        this.counterService.increment(evt);
        this.counterService.increment("meter." + evt);
    }
}
```

- 부트 서버를 실행하고 HAL 브라우저를 통해 `reservations` 리소스 하나를 수정
- Graphite의 좌측 트리메뉴에 `reservations`의 `save` count 항목이 생기고 그래프가 변화됨을 확인
