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



### 7챕터 - ItemReader

-	Step은 Tasklet 단위로 처리되고, Tasklet 중에서 ChunkOrientedTasklet 을 통해 Chunk를 처리하며 이를 구성하는 요소로 ItemReader, ItemWriter, ItemProcessor 가 있다.

-	(데이터) ->(읽기) ItemReader -> ItemProcessor -> ItemWriter ->(쓰기) -> 데이터

-	Spring Batch의 ItemReader는 데이터를 읽어들임. (DB뿐만 아니라 File, XML, JOSN 등 다른 데이터 소스를 배치 처리의 입력으로 사용 가능)

	-	Spring Batch에서 지원하지 않는 Reader가 필요할 경우 직접 Reader를 만들 수 있음

-	ItemStream 인터페이스

	-	주기적으로 상태를 저장하고 오류가 발생하면 해당 상태에서 복원하기 위한 마커 인터페이스
	-	배치 프로세스의 실행 컨텍스트와 연계해서 ItemReader의 상태를 저장하고 실패한 곳에서 다시 실행할 수 있게 해주는 역할을 함

-	Spring Batch 는 2개의 Reader 타입을 지원함

	-	Cursor

		-	JDBC ResultSet의 기본 기능
		-	ResultSet이 open될 때마다 next() 메서드가 호출되어 Database의 데이터가 반환됨.

	-	Paging

		-	페이지라는 Chunk로 Database에서 데이터를 검색
		-	페이지 단위로 한번에 데이터 조회

	-	Cursor 기반 ItemReader 구현체

		-	JdbcCursorItemReader
		-	HibernateCursorItemReader
		-	StoredProcedureItemReader

	-	Paging 기반 ItemReader 구현체

		-	JdbcPagingItemReader
		-	HibernatePagingItemReader
		-	JpaPagingItemReader

-	CursorItemReader

	-	Streaming으로 데이터를 처리

	-	장점

		-	데이터를 Streaming 할 수 있음
		-	read() 메서드는 데이터를 하나씩 가져와 ItemWriter로 데이터를 전달하고, 다음 데이터를 다시 가져옴
		-	reader & processor & writer가 chunk 단위로 수행되고 주기적으로 Commit 됨

	-	주의사항

		-	CursorItemReader를 사용할 때는 Database와 SocketTimeout을 충분히 큰 값으로 설정해야 함
		-	Cursor는 하나의 Connection으로 Batch가 끝날때까지 사용되기 때문에 Batch가 끝나기전에 Database와 어플리케이션의 Connection이 먼저 끊어질 수 있음
		-	Batch 수행 시간이 오래걸리는 경우에는 PagingItemReader를 사용하는게 낫다

-	PagingItemReader

	-	Cursor를 사용하는 대신 여러 쿼리를 실행하여 각 쿼리가 결과의 일부를 가져오는 방법

	-	Spring Batch에서는 SqlPagingQueryProviderFactoryBean을 통해 Datasource 설정값을 보고 위 이미지에서 작성된 Provider중 하나를 자동으로 선택하도록 함

-	JpaPagingItemReader

	-	ORM으로 데이터를 단순한 값으로만 보는게 아닌, 객체로 볼 수 있게 됨
	-	Batch 역시 JPA를 지원하기 위해 JpaPagingItemReader 를 지원함

		-	Querydsl, Jooq 등을 통한 구현체는 지원하지 않음

	-	PagingItemReader 주의 사항

		-	정렬(Order)가 무조건 포함되어있어야 함

-	ItemReader 주의사항

	-	JpaRepository를 ListItemReader, QueueItemReader에 사용하면 안됨
		-	ex) new ListItemReader<>(jpaRepository.findByAge(age))
		-	이렇게 하면 Spring Batch 장점인 페이징 & Cursor 구현이 없어 대규모 데이터 처리가 불가능함 (Chunk 단위 트랜잭션은 됨)
		-	써야한다면 RepositoryItemReader 사용 


	- Hibernate, JPA 등 영속성 컨텍스트가 필요한 Reader 사용 시 fetchSize와 ChunkSize는 같은 값을 유지해야 함


### 8챕터 - ItemWriter

-	Writer는 Reader, Processor와 함께 ChunkOrientedTasklet 구성

	-	Processor 는 선택 (나머진 필수)

-	ItemWriter는 SpringBatch에서 사용하는 출력 기능

	-	ItemWriter는 업데이트 이후 item 하나를 작성하지 않고 Chunk 단위로 묶인 item List 를 다룸

	-	ItemReader는 각 항목을 개별적으로 읽음

	-	즉, Reader와 Processor를 거쳐 처리된 Item을 Chunk 단위 만큼 쌓은 뒤 이를 Writer에 전달

-	JdbcBatchItemWriter

	-	ORM을 사용하지 않는 Writer는 대부분 JdbcBatchItemWriter 사용
	-	쿼리를 모아서 한번에 전송함

	-	JdbcBatchItemWriterBuilder

		-	beanMapped : Pojo 기반으로 Insert SQL 의 Values 매핑
		-	columnMapped : Key,Value 기반으로 Insert SQL 의 Values 매핑

-	JpaItemWriter

	-	ORM을 사용할 수 있는 JpaItemWriter
	-	JdbcBatchItemWriter와 차이
		-	Pay Entity 를 읽어서 Writer에는 Pay2 Entity 전달하는 processor 추가
		-	넘어온 Item을 그대로 테이블에 반영되기 떄문

-	Custom ItemWriter

	-	Reader와 달리 Writer는 커스텀하게 구현해야할 일이 많음
		-	Reader에서 읽어온 데이터를 RestTemplate으로 외부 API에 전달해야할 때
		-	임시저장을 하고 비교하기 위해 싱글톤 객체에 값을 넣어야할 떄
		-	여러 Entity를 동시에 save 해야할 때


---

-	오류메모

	-	The server time zone value ‘KST’ is unrecognized or represents more than one time zone : mysql-connector-java 버전 5.1.X 이후 버전부터 KST 타임존을 인식하지 못하는 이슈

		-	타임존을 명시해준다
		-	jdbc:mysql://localhost:3306/spring_batch?characterEncoding=UTF-8&serverTimezone=UTC

	-	ERROR 1366 (HY000) : incorrect string value : ''\xED\x95\x9C\xEC\x9A\xB0...' for column

		-	mysql 언어 설정 utf 8 로 변경할 것
		-	(workbench 에서 스키마 생성 시 설정 가능)
