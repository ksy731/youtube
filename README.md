![image](https://user-images.githubusercontent.com/63624229/81629985-6bc2cb00-943f-11ea-9aef-b46d48c79244.png)

# 동영상 공유 플랫폼

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [동영상 공유 플랫폼](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)


# 조직

1. 동영상 관리 (video) : Core Domain
2. 채널 관리 (channerl) : Core Domain
3. 고객 관리 (client) : Supporting Domain
4. 정책 관리 (policy) : Supporting Domain


# 서비스 시나리오


기능적 요구사항

1. 고객이 채널을 생성한다
2. 고객이 채널을 수정/삭제 할 수 있다

3. 고객이 생성된 채널에 동영상을 업로드한다
4. 고객이 업로드한 동영상을 수정/삭제 할 수 있다
5. 정책 관리 시스템으로 정책을 등록한다

6. 정책 관리 시스템으로 관리자가 정책을 수정할 수 있다
7. 정책 관리 시스템이 동영상 등록/수정 시 정책 확인 요청을 받게되며 정책을 확인한다
8. 고객이 동영상에 대하여 정책 관리 시스템에 환급 신청한다
9. 정책 관리 시스템이 정책 확인하여 정책 위반 시 동영상 삭제를 요청한다


비기능적 요구사항

1. 트랜잭션
    1. 동영상이 등록/수정 시 정책에 맞는지 확인되어야 한다          Async 호출 
    2. 동영상이 등록/수정/삭제 시 채널 정보 업데이트가 되어야 한다  Async 호출
    3. 고객이 환급신청 시 환급 정책에 맞는지 확인되어야 한다        Async 호출
1. 장애격리
    1. 고객 관리 서비스가 수행되지 않더라도 동영상 서비스는 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    2. 정책 관리 서비스 사용이 과중되면 사용자를 잠시동안 받지 않고 잠시 후에 하도록 유도한다
1. 성능
    1. 정책에 따라 동영상의 상태가 변경될 때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
 ![image](https://user-images.githubusercontent.com/63624229/81755796-b3eff500-94f4-11ea-83dc-aaee8bf6284b.png)

## TO-BE 조직 (Vertically-Aligned)
 ![image](https://user-images.githubusercontent.com/63624229/81755883-04ffe900-94f5-11ea-8e25-a4bd5137f5b0.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://msaez.io/#/storming/kFlEMeCtr1ZQwNanBRR6onPu1Lf1/every/81199d72676bc36939a91135e197c4a2/-M7BG75bxejUqAIUq_dh


### 이벤트 도출
![image](https://user-images.githubusercontent.com/63624229/81628763-7af44980-943c-11ea-9f64-5bd3a577ac04.png)

### 부적격 이벤트 탈락
![20200511_134543](https://user-images.githubusercontent.com/63624229/81533541-5c8d4000-93a1-11ea-9686-68bf31734676.jpg)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        -고객관리 > 로그인됨, 동영상 관리 > 좋아요됨, 동영상 조회됨, 검색됨 : UI 의 이벤트이지,
        업무적인 의미의 이벤트가 아니라서 제외한다


### 액터, 커맨드 부착하여 읽기 좋게
![20200511_140728](https://user-images.githubusercontent.com/63624229/81533574-63b44e00-93a1-11ea-9b96-43acb9b2e9aa.jpg)


### 어그리게잇으로 묶기

![20200511_143718](https://user-images.githubusercontent.com/63624229/81533625-7a5aa500-93a1-11ea-80b8-94e0208674fa.jpg)

   - 고객관리의 회원가입과 채널, 정책관리의 환급, 정책은 그와 연결된 command와 event들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![20200511_142445](https://user-images.githubusercontent.com/63624229/81533585-6747d500-93a1-11ea-96d8-72cfc3c8e75d.jpg)


    - 도메인 서열 분리 
        - Core Domain: video, channel : 없어서는 안될 핵심 서비스이며,
        연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain: client, policy : 경쟁력을 내기위한 서비스이며,
        SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 한다
        - General Domain: pay : 송금 서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)


### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![20200511_143436](https://user-images.githubusercontent.com/63624229/81533594-6dd64c80-93a1-11ea-8296-4fdf2143f947.jpg)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![20200511_150001](https://user-images.githubusercontent.com/63624229/81533663-8e060b80-93a1-11ea-88c7-a09f9ca34cfd.jpg)

### 완성된 1차 모형

![20200511_154304](https://user-images.githubusercontent.com/63624229/81533714-9fe7ae80-93a1-11ea-8a9d-c8f1bdcbe804.jpg)
![20200511_154308](https://user-images.githubusercontent.com/63624229/81533722-a2e29f00-93a1-11ea-9915-83bea5f7825f.jpg)

    - View Model 추가

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/63624229/81630979-e1c83180-9441-11ea-8f8f-dffa0de5b806.png)

    - 고객이 회원 가입한다 (ok)
    - 고객이 채널을 생성한다 (ok)
    - 고객이 채널에 동영상을 등록하면 채널 정보가 업데이트 된다 (ok)
    - 고객이 업로드된 동영상에 대하여 환급 신청을 하면 환급 정책에 따라 환급된다 (ok)

![image](https://user-images.githubusercontent.com/63624229/81648285-6d08ed80-9469-11ea-9354-c10338d0f02d.png)

    - 고객이 동영상을 조회하면 동영상 조회수가 증가한다 (?)
    - 동영상 정보가 수정되면 채널 정보를 업데이트 한다 (?)
    - 채널 정보가 업데이트 되면 고객 관리 정보 중 환급 신청을 위한 전체 조회 수 정보를 업데이트 한다 (?)
    - 고객이 업로드된 동영상에 대하여 환급 신청을 하면 환급 정책에 따라 환급된다 (ok)

### 모델 수정

![image](https://user-images.githubusercontent.com/63624229/81771100-0e9d4700-951d-11ea-95fb-9063afb6a097.png)


    - 수정된 모델은 모든 요구사항을 커버한다
    
    - 고객이 회원 가입을 한다 (ok)
    - 고객이 채널을 생성한다 (ok)
    - 고객이 채널을 수정/삭제 할 수 있다 (ok)
    - 고객이 생성된 채널에 동영상을 업로드한다 (ok)
    - 고객이 업로드한 동영상을 수정/삭제 할 수 있다 (ok)
    - 정책 관리 시스템으로 정책을 등록한다 (ok)
    - 정책 관리 시스템으로 관리자가 정책을 수정할 수 있다 (ok)
    - 정책 관리 시스템이 동영상 등록/수정 시 정책 확인 요청을 받게되며 정책을 확인한다 (ok)
    - 고객이 동영상에 대하여 정책 관리 시스템에 환급 신청한다 (ok)
    - 정책 관리 시스템이 정책 확인하여 정책 위반 시 동영상 삭제를 요청한다 (ok)
    

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/63624229/81632274-d1fe1c80-9444-11ea-9046-7b525d3f2ac1.png)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 모두 inter-microservice 트랜잭션 : 동영상이 등록/수정 시 정책에 맞는지 확인,
        동영상이 등록/수정/삭제 시 채널 정보 업데이트,
        고객이 환급신청 시 환급 정책에 맞는지 확인의 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택한다


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/63624229/81773545-78205400-9523-11ea-8934-1ea0d1bd1699.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 Pub/Sub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리: 각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd client
mvn spring-boot:run

cd channel
mvn spring-boot:run 

cd video
mvn spring-boot:run  

cd policy
mvn spring-boot:run
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 video 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package youtube;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.BeanUtils;
import org.springframework.cloud.stream.messaging.Processor;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.util.MimeTypeUtils;

import javax.persistence.*;
import java.util.Date;

@Entity
@Table(name="VideoService_table")
public class VideoService {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long videoId;
    private Date uploadTime;
    private Long clientId;
    private Long channelId;
    private int viewCount=0;

    @PreUpdate
    public void onPostEdited(){
        EditedVideo editedVideo = new EditedVideo();
        BeanUtils.copyProperties(this, editedVideo);
        editedVideo.publishAfterCommit();

        System.out.println(("**********동영상이 수정되었습니다**********"));
    }

    @PostPersist
    public void onPostUploaded(){
        UploadedVideo uploadedVideo = new UploadedVideo();
        BeanUtils.copyProperties(this, uploadedVideo);
        uploadedVideo.publishAfterCommit();

        System.out.println(("**********동영상이 업로드되었습니다**********"));
    }

    @PreRemove
    public void onPreRemove(){
        DeletedVideo deletedVideo = new DeletedVideo();
        BeanUtils.copyProperties(this, deletedVideo);
        deletedVideo.publishAfterCommit();

        System.out.println(("**********동영상이 삭제되었습니다**********"));
    }

    public Long getVideoId() {
        return videoId;
    }

    public void setVideoId(Long videoId) {
        this.videoId = videoId;
    }

    public Date getUploadTime() {
        return uploadTime;
    }

    public void setUploadTime(Date uploadTime) {
        this.uploadTime = uploadTime;
    }

    public Long getClientId() {
        return clientId;
    }

    public void setClientId(Long clientId) {
        this.clientId = clientId;
    }

    public Long getChannelId() {
        return channelId;
    }

    public void setChannelId(Long channelId) {
        this.channelId = channelId;
    }

    public int getViewCount() {
        return viewCount;
    }

    public void setViewCount(int viewCount) {
        this.viewCount = viewCount;
    }

}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package youtube;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface VideoServiceRepository extends PagingAndSortingRepository<VideoService, Long>{

}


```
- 적용 후 REST API 의 테스트
```

# client 서비스의 회원 가입 처리
http http://localhost:8081/clientSystems clientId=20

# video 서비스의 동영상 업로드 처리
http http://localhost:8083/videoServices videoId=1 clientId=1 viewCount=10

# video 서비스의 동영상 수정 처리
http PATCH http://localhost:8085/videoServices/1 viewCount=1

# video 서비스에서 동영상 수정 처리
http patch http://localhost;8083/videoServices/1 videoId=1 clientId=1 channelId=1 viewCount=20

# video 서비스의 동영상 수정 시 알림 처리
http PATCH http://localhost:8085/videoServices/1 channelId=123

# policy 서비스의 환급 신청 확인 처리
http patch http;//localHost:8081/clientSystems/1 totalView=5000

# policy 서비스의 동영상 삭제 처리
http DELETE localHost:8081/policyManagements/1

# channel 서비스에서 채널 생성 처리
http http://localhost;8082/channelSystems

# channel 서비스에서 채널 수정 처리
http http://localhost;8082/channelSystems totalView=30

```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


고객이 동영상 관리 시스템을 통하여 동영상을 업로드하면 정책 관리 시스템에서 동영상이 정책에 위반되지 않았는지 정책 확인하는 행위는 비동기식으로 처리하여 정책 확인이 블로킹되지 않도록 한다.

이를 위하여 동영상 업로드 이력을 기록으로 남긴 후 동영상 업로드가 완료되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)

--------------

![image](https://user-images.githubusercontent.com/2732669/81760298-cde40480-9501-11ea-949f-b3653deb902e.png)

$ http http://localhost:8085/videoServices viewCount=0

![image](https://user-images.githubusercontent.com/2732669/81761733-a8f19080-9505-11ea-8acc-75395a201fe4.png)


동영상 관리 시스템에서 동영상 등록 (upload), 수정(edit) 발생 시 해당 동영상 정보를 이벤트로 받아서 정책을 확인한다

![image](https://user-images.githubusercontent.com/19707715/81761439-eace0700-9504-11ea-807e-6777c84aab3b.png)

![image](https://user-images.githubusercontent.com/19707715/81761456-f0c3e800-9504-11ea-8627-24e4a94a815e.png)

----------------------------

고객이 동영상 관리 시스템을 통하여 동영상을 수정하면 채널 관리 시스템에서 동영상 조회수를 증가하는 확인하는 행위는 비동기식으로 처리하여 동영상 조회수 증가가 블로킹되지 않도록 한다.

이를 위하여 동영상 조회수 증가 이력을 기록으로 남긴 후 동영상 조회수가 증가되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)

----------------------------

![image](https://user-images.githubusercontent.com/2732669/81761945-519ff000-9506-11ea-8b79-a782257c18f3.png)

ㅇ viewCount를 수정하여 조회수 증가
$ http PATCH http://localhost:8085/videoServices/1 viewCount=1

![image](https://user-images.githubusercontent.com/2732669/81761928-41881080-9506-11ea-822b-6ecf87ffabac.png)


--------------


동영상 관리 시스템에서 동영상 상태가 변경될 때마다 알림을 주는 행위는 비동기식으로 처리하여 알림이 블로킹되지 않도록 한다.

이를 위하여 동영상 상태 변경 이력을 기록으로 남긴 후 동영상 상태 변경이 완료되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)

--------------


![image](https://user-images.githubusercontent.com/2732669/81760397-08e63800-9502-11ea-88aa-50e485f927a4.png)

$ http PATCH http://localhost:8085/videoServices/1 channelId=123

![image](https://user-images.githubusercontent.com/2732669/81762610-0686dc80-9508-11ea-9b94-7d976a6dde22.png)


--------------


고객이 정책 관리 시스템으로 환급 신청을 하는 행위는 비동기식으로 처리하여 환급 신청이 블로킹되지 않도록 처리한다

이를 위하여 환급 신청 이력을 기록으로 남긴 후 환급 신청 완료되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)


![환급신청됨](https://user-images.githubusercontent.com/61961799/81763813-0dfbb500-950b-11ea-9c3f-31abbe6706c5.JPG)

![환급신청](https://user-images.githubusercontent.com/61961799/81763807-0b995b00-950b-11ea-873b-f48d50b9e720.JPG)

고객 관리 시스템에서 환급 신청 (refund) 정책 검토 요청 시 정책 관리 시스템에서 해당 도메인 이벤트로 받아서 정책을 확인한다

![image](https://user-images.githubusercontent.com/19707715/81761424-e3a6f900-9504-11ea-9ecb-a650a7f4c1e7.png)

![결과](https://user-images.githubusercontent.com/61961799/81763819-1227d280-950b-11ea-9a1f-d1ba08f4a1e6.JPG)


---------------event publish-------------------

정책 관리 시스템에서 정책에 위반되는 동영상 삭제를 동영상 관리 시스템에 요청하는 행위는 비동기식으로 처리하여 동영상 삭제 요청을 video 서비스에게 요청한다.
이를 위하여 동영상 삭제 요청 이력을 기록으로 남긴 후 동영상 삭제가 완료되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)


```
 @PostRemove
    public void onPostRemove(){
        DeletedPolicy deletedPolicy = new DeletedPolicy();
        BeanUtils.copyProperties(this, deletedPolicy);
        deletedPolicy.publishAfterCommit();
        System.out.println("###### 정책 위반 ###### video 삭제 요청 event 발생");
        System.out.println("삭제요청 viedo ID : " + deletedPolicy.getDeleteVideoId());
    }
```


![image](https://user-images.githubusercontent.com/19707715/81760657-cb35df00-9502-11ea-9191-ec60de355585.png)


![image](https://user-images.githubusercontent.com/19707715/81760641-c2450d80-9502-11ea-8610-9f30231fbfea.png)



채널 관리 시스템에서 동영상 조회수 증가를 고객 관리 시스템에 요청하는 행위는 비동기식으로 처리하여 동영상 조회수 증가가 블로킹되지 않도록 한다.

이를 위하여 동영상 조회수 증가 요청 이력을 기록으로 남긴 후 동영상 조회수 증가가 완료되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)

--------------
![image](https://user-images.githubusercontent.com/17778248/81759599-1c909f00-9500-11ea-9301-632a40a5ef38.png)
--------------

---------------eventListener-------------------


채널 관리 시스템에서 조회수 변경 발생 시 조회수 증가 반영


![채널변경](https://user-images.githubusercontent.com/61961799/81764040-95e1bf00-950b-11ea-96c9-ba0ca462bd36.JPG)

![채널변경결과](https://user-images.githubusercontent.com/61961799/81764170-e5c08600-950b-11ea-8f01-74c05ef78914.JPG)



채널 관리 시스템에서 채널 삭제 요청이 들어오면 채널을 삭제하고 해당 채널의 동영상 삭제를 동영상 관리 시스템에 요청하는 행위는 비동기식으로 처리하여 동영상 삭제가 블로킹되지 않도록 한다.

이를 위하여 동영상 삭제 요청 이력을 기록으로 남긴 후 동영상 삭제가 완료되었다는 도메인 이벤트를 카프카로 송출한다 (Publish)

--------------

![image](https://user-images.githubusercontent.com/17778248/81759709-64172b00-9500-11ea-9f38-296166dbcd32.png)

--------------

채널관리시스템에서는 동영상 업로드 이벤트에 대해서 이를 수신하여 자신의 정책에서 채널에 동영상을 등록하는  PolicyHandler 를 구현한다

--------------

![image](https://user-images.githubusercontent.com/17778248/81761000-b9087080-9503-11ea-9d2f-92bb77dc1837.png)


결과 시나리오

1. 동영상을 등록한다 (조회수 10)

![image](https://user-images.githubusercontent.com/17778248/81763083-51552400-9509-11ea-9969-611a6cab0622.png)

2. 채널이 신규생성되며 동영상이 등록된다

![image](https://user-images.githubusercontent.com/17778248/81763134-777ac400-9509-11ea-84a6-d1df51dc53cf.png)

3. 동영상을 수정한다 (조회수 20 증가)

![image](https://user-images.githubusercontent.com/17778248/81763177-8eb9b180-9509-11ea-95db-b750f39d99bc.png)

4. 채널이 수정된다 (기존 조회수 10에서 20이 증가하여 30이 된다)

![image](https://user-images.githubusercontent.com/17778248/81763235-bf99e680-9509-11ea-938b-caa26ae87125.png)

--------------


채널관리시스템에서는 동영상 수정(조회수 증가 포함) 이벤트에 대해서 이를 수신하여 자신의 정책에서 채널 정보를 수정하는  PolicyHandler 를 구현한다

--------------
![image](https://user-images.githubusercontent.com/17778248/81761022-c7ef2300-9503-11ea-86b9-7355a9c24235.png)
--------------



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure DevOps를 사용하였으며, Pipeline build script 는 각 프로젝트 폴더 이하에 application.yml 에 작성하였다


## Gateway 설정

gateway 배포 확인

![image](https://user-images.githubusercontent.com/19707715/81765504-f1fa1280-950e-11ea-9f36-20e603c905fe.png)

gateway routes 정보

![image](https://user-images.githubusercontent.com/19707715/81765560-1655ef00-950f-11ea-969f-872be1febc8e.png)


gateway -> policy로 라우팅 되는지 확인

![image](https://user-images.githubusercontent.com/19707715/81765601-35ed1780-950f-11ea-8fe6-61f40f4bfc63.png)



## 장애격리

### livenessProbe, readinessProbe 설정

azure-pipeline.yml을 수정하여 모든 서비스에 livenessprobe, redinessprobe 패스를 재설정하였다

![image](https://user-images.githubusercontent.com/19707715/81764339-4f409480-950c-11ea-8586-a5e32be1c3c5.png)


### 오토스케일 아웃

CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 정책 관리 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 10%를 넘어서면 replica 를 10개까지 늘려준다:

![캡처1](https://user-images.githubusercontent.com/19707715/81761721-a5f6a000-9505-11ea-9477-e686f4b8bb72.PNG)


- 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://policy:8080/policyManagements POST {"deleteVideoId": "1"}'
```

![부하](https://user-images.githubusercontent.com/19707715/81762134-cecb6500-9506-11ea-8dc1-f8bb1458aefd.PNG)


- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:

```
kubectl get deploy policy -w
```

![image](https://user-images.githubusercontent.com/19707715/81762146-d985fa00-9506-11ea-905d-d83209682ea4.png)



## 무정지 재배포

왼쪽은 재배포 수행 시 명령어 호출 화면이고, 오른쪽은 Azure DevOps 에서 재배포되는 화면

![image](https://user-images.githubusercontent.com/19707715/81771712-b2d3bd80-951e-11ea-91d7-31fd46ee2035.png)

## 장애 격리 시나리오
(1)policy pod replic 를 0으로 수정 (장애상황으로 가정)

![현재 디플로이](https://user-images.githubusercontent.com/19707715/81775282-81132480-9527-11ea-941b-b0ab485117b1.PNG)

(2)video 서비스에서 event 5개 발생 (to Policy pod)

```
http video:8080/videoServices videoId=6 clientId=1 channelId=1 viewCount=0
http video:8080/videoServices videoId=7 clientId=1 channelId=1 viewCount=0
http video:8080/videoServices videoId=8 clientId=1 channelId=1 viewCount=0
http video:8080/videoServices videoId=9 clientId=1 channelId=1 viewCount=0
http video:8080/videoServices videoId=10 clientId=1 channelId=1 viewCount=0
```

(3)policy pod replica 를 1로 다시 설정
![레플리카수 늘리기](https://user-images.githubusercontent.com/19707715/81775316-95572180-9527-11ea-8985-5f18f2e5a1a7.PNG)


(4) 서비스가 뜨면서 정상적으로 event listen
![image](https://user-images.githubusercontent.com/19707715/81775241-6e005480-9527-11ea-93f6-91a4713d81e6.png)

# 신규 개발 조직의 추가

![image](https://user-images.githubusercontent.com/63624229/81772614-47d7b600-9521-11ea-954c-84dd60ba640c.png)


## 추천팀의 추가
    - KPI: 기존 고객의 충성도 향상과 신규 고객의 회원가입 수 증대
    - 구현 계획 마이크로 서비스: 기존 search 마이크로 서비스를 인수하여, 고객의 취향에 맞는 동영상 추천 서비스 등을 제공할 예정

## 이벤트 스토밍

 ![image](https://user-images.githubusercontent.com/63624229/81773297-f92b1b80-9522-11ea-86d5-8ad20b723904.png)


## 헥사고날 아키텍처 변화

![image](https://user-images.githubusercontent.com/63624229/81775268-7b1d4380-9527-11ea-9185-80a977ae7095.png)


```
### 고객 생성
http client:8080/clientSystems clientId=1 totalView=0

### 고객이 채널을 생성한다
http channel:8080/channelSystems channelId=1 channelName="channel1" clientId=1 totalView=0

## 고객이 채널 댓글을 등록한다
http comment:8080/commentServices commentId=1 channelId=1 clientId=1 contents="hahahahahaha"

## 고객이 채널 댓글을 수정한다
http PATCH comment:8080/commentServices/1 contents="hello world"

## 고객이 채널 댓글을 삭제한다
http DELETE comment:8080/commentServices/1



### 고객이 생성된 채널에 동영상을 업로드한다
http video:8080/videoServices videoId=1 clientId=1 channelId=1 viewCount=0

### 고객이 업로드한 동영상에 댓글을 등록한다
http comment:8080/commentServices commentId=2 videoId=2 clientId=1 contents="work hard"

### 고객이 업로드한 동영상에 댓글 등록수를 1 증가를 확인한다
http comment:8080/commentServices 






### 고객이 생성된 채널에 동영상을 업로드한다
http localhost:8088/videoServices videoId=1 clientId=1 channelId=1 viewCount=0

### 고객이 업로드한 동영상에 댓글을 등록한다
http localhost:8088/commentServices commentId=1 videoId=1 clientId=1 contents="work hard"
http localhost:8088/commentServices commentId=2 videoId=1 clientId=1 contents="work hard2"
http localhost:8088/commentServices commentId=3 videoId=1 clientId=1 contents="work hard3"
http localhost:8088/commentServices commentId=4 videoId=1 clientId=1 contents="work hard4"



==== 추후

### 고객이 생성된 채널에 동영상을 업로드한다
http video:8080/videoServices videoId=1 clientId=1 channelId=1 viewCount=0

### 고객이 업로드한 동영상에 댓글을 등록한다
http comment:8080/commentServices commentId=2 videoId=2 clientId=1 contents="work hard"

### 고객이 업로드한 동영상을 삭제한다
http DELETE video:8080/videoServices/1

### 동영상에 등록된 댓글 삭제를 확인한다
http comment:8080/commentServices/2




# gateway 확인
http gateway:8080/commentServices
```





## 시연 시나리오

```
### 고객 생성
http client:8080/clientSystems clientId=1 totalView=0


### 고객이 채널을 생성한다
http channel:8080/channelSystems channelId=1 channelName="channel1" clientId=1 totalView=0
http channel:8080/channelSystems channelId=2 channelName="channel2" clientId=1 totalView=0
http channel:8080/channelSystems


### 고객이 채널을 수정/삭제 할 수 있다
http PATCH channel:8080/channelSystems/2 channelName="channel2-1"
http channel:8080/channelSystems
http DELETE channel:8080/channelSystems/2
http channel:8080/channelSystems


### 고객이 생성된 채널에 동영상을 업로드한다
http video:8080/videoServices
http video:8080/videoServices videoId=1 clientId=1 channelId=1 viewCount=0
http video:8080/videoServices videoId=2 clientId=1 channelId=1 viewCount=0
http video:8080/videoServices


### 고객이 업로드한 동영상을 수정/삭제 할 수 있다
http PATCH video:8080/videoServices/2 viewCount=3
http video:8080/videoServices
http DELETE video:8080/videoServices/2
http video:8080/videoServices
http channel:8080/channelSystems


### 정책 관리 시스템으로 정책을 등록한다
http policy:8080/policyManagements policyId=1 refundPolicy=10 deleteVideoId=1


### 정책 관리 시스템으로 관리자가 정책을 수정할 수 있다
http PATCH policy:8080/policyManagements/1 policyId=1 refundPolicy=5


### 정책 관리 시스템이 동영상 등록/수정 시 정책 확인 요청을 받게되며 정책을 확인한다
kubectl logs -f policy-7d474458bb-2mbsq
http video:8080/videoServices videoId=3 clientId=1 channelId=1 viewCount=0


### 고객이 동영상에 대하여 정책 관리 시스템에 환급 신청한다
http client:8080/clientSystems
kubectl logs -f policy-7d474458bb-2mbsq
http PATCH video:8080/videoServices/1 viewCount=100


### 정책 관리 시스템이 정책 확인하여 정책 위반 시 동영상 삭제를 요청한다
kubectl logs -f video-5bf778ffc4-d7rxd
http DELETE policy:8080/policyManagements/1


### gateway 확인
http gateway:8080/videoServices
http gateway:8080/channelSystems
http gateway:8080/policyManagements
```
