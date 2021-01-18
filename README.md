배치 실습 해보기
================

-	https://jojoldu.tistory.com/325?category=902551

-	스프링 배치란?

### 2챕터 - Batch Job 실행해보기

-	@EnableBatchProcessing

	-	Batch 기능 활성화
	-	++

-	Job

	-	Step들
		-	(Tasklet) / (Reader, Processor, Writer) 로 구성

-	Tasklet : Step 안에서 단일로 수행될 커스텀한 기능들을 선언할 때 사용

-	Spring Batch 에선 메타 데이터 테이블들이 필요함

	-	메타 데이터들은 다음을 담고 있음
		-	이전에 실행한 Job이 어떤 것들이 있는지
		-	최근 실패한 BatchParameter가 어떤것들이 있고, 성공한 Job은 어떤것들이 있는지
		-	다시 실행한다면 어디서부터 시작하면 될지
		-	어떤 Job들에 어떤 Step들이 있었고, Step들 중 성공한 Step과 실패한 Step은 어떤 것들이 있는지

-	boot의 기본 DataSource는 Hikari

-	h2 는 필요테이블을 boot 가 실행될 때 자동으로 생성해주지만 mysql은 직접 생성해야함

	-	schema-mysql.sql 검색해서 실행

-	edit run configurations 에서 active 조절해서 실행 가능함

### 3챕터 - 메타 테이블 엿보기

-	BATCH_JOB_INSTANCE

	-	Job Parameter에 따라 생성되는 테이블

		-	Spring Batch가 실행될 때 외부에서 받을 수 있는 파라미터
		-	특정 날짜를 Job Parameter로 넘기면 Spring 배치에서는 해당 날짜 데이터로 조회/가공/입력 등의 작업을 할 수 있음
		-	같은 Batch Job이라도 Job Parameter가 다르면 BATCH_JOB_INSTANCE 에 기록됨.
		-	Job Parameter 가 같으면 기록되지 않음

-	BATCH_JOB_EXECUTION

	-	BATCH_JOB_INSTANCE 와 부모 자식관계
	-	부모 BATCH_JOB_INSTANCE 가 성공/실패 했던 내역을 가지고 있음
	-	동일한 Job Parameter 로 성공한 기록이 있을 때만 재수행이 안됨

### 4챕터 - Job Flow

-	Step은 실제 Batch 작업을 수행하는 역할

-	next()는 순차적으로 Step들 연결시킬 때 사용

-	yml 파일에 설정을 통해 지정한 Batch Job 만 실행하도록 가능

	-	spring.batch.job.names: ${job.name:NONE}

-	on(), to(), from(), end() 활용해서 실패 시 동작 설정가능

-	BatchStatus , ExitStatus

	-	BatchStatus

		-	Job 또는 Step의 실행 결과를 Spring에서 기록할 때 사용하는 Enum

	-	ExitStatus

		-	Step의 실행 후 상태

-	JobExecutionDecider 를 통해 Step과 역할을 분리 하고 분기를 진행할 수 있음

### 5챕터 - Spring Batch Scope & Job Parameter

-	Scope

	-	@StepScope , @JobScope

-	Job Parameter

	-	Spring Batch는 외부 혹은 내부에서 파라미터를 받아 여러 Batch 컴포넌트에서 사용할 수 있게 지원함
	-	Job Parameter를 사용하기 위해서는 항상 Spring Batch 전용 Scope를 선언해야 함

		-	@StepScope

			-	Tasklet이나 ItemReader, ItemWriter, ItemProcessor 에서 사용

		-	@JobScope

			-	Step 선언문에서 사용 가능

-	@StepScope, @JobScope

	-	Spring Bean의 기본 Scope는 singileton 임
	-	그러나 Spring Batch 컴포넌트(Tasklet, ItemReader, ItemWriter, ItemProcessor 등)에 @StepScope를 사용하게 되면 Spring Batch가 Spring 컨테이너를 통해 지정된 Step의 실행 시점에 해당 컴포넌트를 Spring Bean으로 생성함
	-	@JobScope 는 Job 실행 시점에 Bean이 생성됨

	-	Bean의 생성 시점을 지정된 Scope가 실행되는 시점으로 지연시킴

	-	실행 시점을 지연시킴으로 얻는 장점

		-	Job Parameter의 Late Binding이 가능

			-	Job Parameter가 StepContext 또는 JobExecutionContext 레벨에서 할당시킬 수 있음
			-	Application이 실행되는 시점이 아니더라도 Controller나 Service와 같은 비즈니스 로직 처리 단계에서 Job Parameter를 할당시킬 수 있음

		-	동일한 컴포넌트를 병렬 혹은 동시에 사용할 때 유용함

			-	@StepScope 없이 Step을 병렬로 실행시키게 되면 서로 다른 Step에서 하나의 Tasklet을 두고 마구잡이로 상태를 변경하려고 함
			-	@StepScope가 있다면 각각의 Step에서 별도의 Tasklet을 생성하고 관리하기 때문에 서로의 상태를 침범할 일이 없음

-	Job Parameter는 Scope Bean을 생성할 때만 Step, Tasklet, Reader 등 Batch 컴포넌트 생성 시점에 호출할 수 있음

-	Job Parameter 대신 시스템 변수를 쓰면?

	-	java jar application.jar -D파라미터 처럼..

	-	시스템 변수를 사용할 경우 Spring Batch의 Job Parameter 관련 기능을 못씀

		-	같은 Job Parameter로 같은 Job을 실행하지 않음 (시스템변수는 이 기능 동작안함)
		-	Spring Batch에서 자동으로 관리해주는 Parameter 관련 메타 테이블이 전혀 관리되지 않음

	-	Command line이 아닌 다른 방법으로 Job을 실행하기 어려움

		-	실행해야한다면 전역 상태 (시스템 변수 혹은 환경변수)를 동적으로 계속해서 변경시킬 수 있도록 Spring Batch를 구성해야함
		-	동시에 여러 Job을 실행하려는 경우 또는 테스트 코드로 Job을 실행해야할 때 문제가 발생할 수 있음
		-	Late Binding 불가

### 6챕터 - Chunk 지향 처리

-	Chunk

	-	데이터 덩어리로 작업할 때 각 커밋 사이에 처리되는 row 수

-	Chunk 지향 처리

	-	한 번에 하나씩 데이터를 읽어 Chunk라는 덩어리를 만든 뒤, Chunk 단위로 트랜잭션을 다루는 것

	-	실패할 경우 Chunk 만큼 롤백됨

	-	Chunk 지향 처리의 전체 로직을 다루는 것은 ChunkOrientedTasklet 클래스

-	Page Size vs Chunk Size

	-	PagingItemReader를 사용할 때 Page Size와 Chunk Size를 같은 의미로 오해할 수 있음
	-	Chunk Size : 한 번에 처리될 트랜잭션 단위
	-	Page Size : 한 번에 조회할 item의 양

	-	ex) Page Size 가 10이고, ChunkSize가 50이면 ItemReader에서 Page 조회가 5번 일어나면 1번의 트랜잭션이 발생하여 Chunk가 처리됨

	-	2개 값을 일치하는 것이 보편적으로 좋은 방법임 (성능)


---

-	오류메모

	-	The server time zone value ‘KST’ is unrecognized or represents more than one time zone : mysql-connector-java 버전 5.1.X 이후 버전부터 KST 타임존을 인식하지 못하는 이슈

		-	타임존을 명시해준다
		-	jdbc:mysql://localhost:3306/spring_batch?characterEncoding=UTF-8&serverTimezone=UTC

	-	ERROR 1366 (HY000) : incorrect string value : ''\xED\x95\x9C\xEC\x9A\xB0...' for column

		-	mysql 언어 설정 utf 8 로 변경할 것
		-	(workbench 에서 스키마 생성 시 설정 가능)
