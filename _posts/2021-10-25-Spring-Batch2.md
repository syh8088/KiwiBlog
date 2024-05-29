---
layout: post
title: "Spring Batch2"
description: "Spring Batch Job, Step 에 대해 알아봅시다."
date: 2021-10-25
tags: [SpringBatch]
writer: syh8088
category: SpringBatch
comments: true
share: true
---


# Spring Batch - Job, Step

이번시간은 Job 에 대해서 자세히 알아보는 시간을 갖겠습니다.

Spring Batch 에서 'Job' 이란 하나의 배치 작업에서 최상위 개념으로 특정 하나의 배치 작업이라고 생각하면 이해하기 쉽습니다.

## Spring Batch 에서 Job 은 어떻게 실행할까?

우선 Spring Batch 에서 Job 은 어떻게 실행 하는지 알아보겠습니다.

이전에 우리가 생성한 'BasicJobConfiguration' 클래스 기준으로 말씀드리겠습니다.

BasicJobConfiguration 클래스는 @Configuration 선언했습니다.

그럼 @bean 으로 선언한것은 Spring Ioc 컨테이너가 관리 하게 될 것 입니다.

프로젝트를 기동 합니다.

### 프로젝트 기동

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/BasicJobConfigurationStart.PNG)

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/baseJob.PNG)

그럼 우리가 선언한 메소드 basicJob 이 실행됩니다. 그리고 Job 이라는 객체를 반환 하게 됩니다.

어떻게 생성 하는지 한번 알아봅시다.

```markdown
public class JobBuilderFactory {

	private JobRepository jobRepository;

	public JobBuilderFactory(JobRepository jobRepository) {
		this.jobRepository = jobRepository;
	}

	public JobBuilder get(String name) {
		JobBuilder builder = new JobBuilder(name).repository(jobRepository);
		return builder;
	}
}
```
get 메소드를 실행해서 우리가 Job 이름을 설정한 'BasicJob' 와  메타데이터 조회 및 저장 전용 'jobRepository' 객체 함께 'JobBuilder' 객체를 생성합니다.

```markdown
public class JobBuilder extends JobBuilderHelper<JobBuilder> {

	public SimpleJobBuilder start(Step step) {
		return new SimpleJobBuilder(this).start(step);
	}
}
```

```markdown
public class SimpleJobBuilder extends JobBuilderHelper<SimpleJobBuilder> {
	public SimpleJobBuilder start(Step step) {
		if (steps.isEmpty()) {
			steps.add(step);
		}
		else {
			steps.set(0, step);
		}
		return this;
	}
}
```

이번에는 start 메소드 입니다. 기존에 생성한 JobBuilder 클래스에 SimpleJobBuilder 클래스를 생성합니다. 동시에 우리가 Step 객체를 생성한 객체도 전달해서

steps 변수에 add 하게 됩니다.

```markdown
public class SimpleJobBuilder extends JobBuilderHelper<SimpleJobBuilder> {
	public SimpleJobBuilder next(Step step) {
		steps.add(step);
		return this;
	}
}
```

start 메소드와 크게 다르지 않습니다. steps 변수에 add 하게 됩니다.

```markdown
public class SimpleJobBuilder extends JobBuilderHelper<SimpleJobBuilder> {

	private List<Step> steps = new ArrayList<>();

	private JobFlowBuilder builder;

	public Job build() {
		if (builder != null) {
			return builder.end().build();
		}
		SimpleJob job = new SimpleJob(getName());
		super.enhance(job);
		job.setSteps(steps);
		try {
			job.afterPropertiesSet();
		}
		catch (Exception e) {
			throw new JobBuilderException(e);
		}
		return job;
	}
}
```

마지막인 build 메소드 입니다. 

가장 중요한 부분은 SimpleJob 객체 생성 입니다. 그외 최상위 인터페이스 중 Job 이라는 객체가 있고 그중 하위 클래스 중 하나가 SimpleJob 입니다.

기존에 steps 변수에 저장했던 Step 객체를 SimpleJob 객체에 저장 하게 됩니다.

즉 Job 하나는 우리가 설정한 Step 을 저장하고 있다는 의미입니다.

### 배치 실행

기동시 우리가 설정한 Job, Step 은 Spring Batch 관리 하고 등록 되어 있다는것을 알아보았습니다.

그럼 등록했던 Job, Step 을 실행해야 하는데요.

이전에서 실행해봐서 알겠지만 기동과 동시에 Batch 를 실행 한다는 것을 알 수 있었습니다.


```markdown
public class BatchAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnProperty(prefix = "spring.batch.job", name = "enabled", havingValue = "true", matchIfMissing = true)
	public JobLauncherApplicationRunner jobLauncherApplicationRunner(JobLauncher jobLauncher, JobExplorer jobExplorer,
			JobRepository jobRepository, BatchProperties properties) {
		JobLauncherApplicationRunner runner = new JobLauncherApplicationRunner(jobLauncher, jobExplorer, jobRepository);
		String jobNames = properties.getJob().getNames();
		if (StringUtils.hasText(jobNames)) {
			runner.setJobNames(jobNames);
		}
		return runner;
	}	
}
```

우리가 등록한 Job, Step 을 실행하는 주범은 바로 JobLauncherApplicationRunner 입니다.

JobLauncherApplicationRunner 클래스는 Spring 에서 관리하는 ApplicationRunner 구현체 입니다.

즉 JobLauncherApplicationRunner 를 bean 으로 등록하고 최종적으로 SimpleJobLauncher 클래스의 run 메소드를 호출 하게 됩니다.

일단 다시 정리해서 말씀드리자면 Job 을 실행하기에 반드시 'Job' 객체, 'Job Parameter' 이 두가지 입니다. 

여기서 Job 은 실행 할수 있는 객체 단위라고 보면 될것 같습니다. 이후에 Job 객체에 대해 설명 하도록 하겠습니다. 그리고 Job parameter 는 해당 Job 을 실행할때 한번만 실행되지 않을겁니다.

오늘 Job 실행 했으면 내일도 실행 할것 입니다. 앞으로 실행될때 마다 각각의 실행을 분별 하기 용도라고 생각하시면 되겠습니다.

어느 하나의 Job 'TestJob' 이름이 있다면 파라미터는 '2021-06-06' 지정 하고 내일은 '2021-06-07' 로 지정 해서 앞으로 실행될 Job 을 구분짓는 것 입니다.

Job 실행시 'JobLauncher' 가 담당 하게 됩니다.

그럼 JobLauncher 구현체인 SimpleJobLauncher run 메소드를 알아봅시다.

```markdown
public class SimpleJobLauncher implements JobLauncher, InitializingBean {

    @Override
	public JobExecution run(final Job job, final JobParameters jobParameters)
			throws JobExecutionAlreadyRunningException, JobRestartException, JobInstanceAlreadyCompleteException,
			JobParametersInvalidException {

        // .. 생략 ..
	    final JobExecution jobExecution;
        JobExecution lastExecution = jobRepository.getLastJobExecution(job.getName(), jobParameters);
        if (lastExecution != null) {
            if (!job.isRestartable()) {
                throw new JobRestartException("JobInstance already exists and is not restartable");
            }

            // .. 생략 ..
        }

    }

    job.getJobParametersValidator().validate(jobParameters);
    jobExecution = jobRepository.createJobExecution(job.getName(), jobParameters);

    taskExecutor.execute(new Runnable() {

        // .. 생략 ..

        @Override
        public void run() {
            // .. 생략 ..
            job.execute(jobExecution);
        }

    }
}
```

여기서 ``JobExecution lastExecution = jobRepository.getLastJobExecution(job.getName(), jobParameters);``

위에 설명한대로 job 이름, 실행되는 job parameter 를 인자값으로 가져오고 있습니다.

'JobExecution' 이라는 객체를 반환 받는데요. 앞서 설명한 배치 실행에 관리 및 히스토리를 보관하는 메타데이터 통해 JobExecution 객체를 가져옵니다.

즉 JobExecution 는 해당 Job 실행 단위로 보면 되겠습니다. 특정 Job 이라는 뼈대가 있다면 그것을 실행하는 주체가 JobExecution 입니다.

메타 데이터 통해 JobExecution 가져온 값이 만약이 있다면 여기서 job.isRestartable() false 면 이미 실행한 JobInstance 라고 경고가 나옵니다.

job.isRestartable() 는 옵션 설정인데 Job 재시작 여부 스위치 역활입니다. default 는 false 입니다.

``job.getJobParametersValidator().validate(jobParameters);``

만약 새로운 JobExecution 라면 입력한 파라미터 값 JobParametersValidator 객체 통해 올바른지 체크 하고

``jobExecution = jobRepository.createJobExecution(job.getName(), jobParameters);``

Job Name 하고 Job Parameter 인자값을 통해 새로운 JobExecution 을 생성 하게 됩니다.

SimpleJobLauncher.java
```markdown
	@Override
	public JobExecution createJobExecution(String jobName, JobParameters jobParameters)
			throws JobExecutionAlreadyRunningException, JobRestartException, JobInstanceAlreadyCompleteException {

        JobInstance jobInstance = jobInstanceDao.getJobInstance(jobName, jobParameters);
        if (jobInstance != null) {      
            // .. 생략 ..
            executionContext = ecDao.getExecutionContext(jobExecutionDao.getLastJobExecution(jobInstance));
        } else {
            jobInstance = jobInstanceDao.createJobInstance(jobName, jobParameters);
            executionContext = new ExecutionContext();
        }

		JobExecution jobExecution = new JobExecution(jobInstance, jobParameters, null);
		jobExecution.setExecutionContext(executionContext);
		jobExecution.setLastUpdated(new Date(System.currentTimeMillis()));

		// Save the JobExecution so that it picks up an ID (useful for clients
		// monitoring asynchronous executions):
		jobExecutionDao.saveJobExecution(jobExecution);
		ecDao.saveExecutionContext(jobExecution);

		return jobExecution;
    }
```

JobExecution 생성 하는 메소드 입니다.

``JobInstance jobInstance = jobInstanceDao.getJobInstance(jobName, jobParameters);``

jobName, jobParameters 통해 메타데이터 저장소(우리는 Mysql 로 설정했으니 해당 DB 테이블에서 가져온다.) 에서 JobInstance 를 가져오게 됩니다.

JobInstance 는 Job 의 실행 단위라고 보면 되겠습니다. 그럼 JobExecution 와 차이점이 무엇이냐

* JobInstance: 배치 처리에서 Job 이 실행될 때 하나의 Job 실행 단위
* JobExecution: 해당 JobInstance 대한 한 번의 실행을 나타내는 객체

즉 Job Name 을 'TestJob', Job Parameter 를 '2021-05-05' 통해 실행 한다면 한번의 JobInstance 만들어 지고 하나의 JobExecution 가 만들어 집니다.

그리고 Job Name 을 'TestJob', Job Parameter 를 '2021-05-06' 통해 실행 후 실패 한다고 가정시 

한번의 JobInstance 만들어 지고, 하나의 JobExecution 가 만들어 집니다. 해당 JobInstance 가 실패한다면 다시 실행이 가능한데

여기서 실패한 Job Name 을 'TestJob', Job Parameter 를 '2021-05-06' 통해 다시 실행 한다면

JobInstance 는 이전에 만들어졌으니 생성은 안하지만 JobExecution 는 새롭게 생성 하게 됩니다.

즉 JobInstance (1) vs JobExecution(N) 관계 입니다.

이후 SimpleJobLauncher 클래스로 돌아와서

``job.execute(jobExecution);``

Runnable 통해 run 메소드를 호출 하게 되면 job.execute 가 실행되는데

```markdown
public abstract class AbstractJob implements Job, StepLocator, BeanNameAware,
InitializingBean {

@Override
	public final void execute(JobExecution execution) {

         // .. 생략 ..

				try {
					doExecute(execution);
					if (logger.isDebugEnabled()) {
						logger.debug("Job execution complete: " + execution);
					}
				} 
		 // .. 생략 ..
	}
}
```
execute 메소드 에서는 doExecute 메소드를 호출 하게 됩니다.

```markdown
public class SimpleJob extends AbstractJob {

	@Override
	protected void doExecute(JobExecution execution) throws JobInterruptedException, JobRestartException,
	StartLimitExceededException {

		StepExecution stepExecution = null;
		for (Step step : steps) {
			stepExecution = handleStep(step, execution);
            // .. 생략 ..
		}

        // .. 생략 ..
	}
}
```

기존에 Step 들을 steps 변수에 저장했었는데 for 문 통해 handleStep 메소드를 호출 하게 됩니다.

이후 Step 영역이기 때문에 다음 부분에서 설명을 하도록 하겠습니다.



### Job 객체

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/job.png)

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/job_interface.PNG)

Spring Batch 에서 사용되는 Job 객체에 대해 알아보겠습니다.

#### Job

Spring Batch 'Job' 은 최상위 인터페이지로 구성되어 있습니다.

대표적인 메소드를 보면

* getName(): 설정한 해당 job 의 이름을 가져오는 메소드 입니다.
* execute(): 설정한 해당 job 을 실행 하는 메소드 입니다. 
* getJobParametersValidator(): 해당 job 을 실행 할려면 'job', 'parameter' 인자값 통해 실행합니다. 이 중 두개의 인자값이 이전 실행한 job 과 중복이 일어나면 다시 실행 되지 않습니다.
예를들어 job: 'testJob', parameter: '2021-05-05' 이렇게 파라미터 통해 실행한다면 다음에는 job: 'testJob', parameter: '2021-05-06' 이렇게 다르게 인자값을 달리 해서 호출 해야 합니다.

#### AbstractJob

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/AbstractJob.png)

AbstractJob 입니다. 여기서 관리하는 대표적인 필드만 알아보겠습니다.

* name: 설정한 해당 job 의 이름 입니다.
* restartable: 해당 Job 이 재시작 여부 입니다.
* jobRepository: 앞서 설명한 배치 실행에 관련한 데이터 관리를 하는 메타데이터 저장소 입니다.
* jobParametersIncrementer: Job 실행하기 위해 이전에 실행한 파라미터와 비교해서 기록이 없다면 실행이 가능한데 이것을 가능 하도록 파라미터를 증가시킬수 있는 객체 입니다. 
* jobParametersValidator: 위에 Job 객체를 설명하면서 Job 객체가 실행전 인자값으로 전달받은 해당 파라미터 값을 검증하는 객체 입니다.  
* stepHandler: Job 안에 설정한 Step 을 실행하기 위한 핸드러 입니다.

#### SimpleJob

Spring Batch 여러 Job 객체 중 가장 대표적인 SimpleJob 입니다. 순차적으로 Step 을 실행 시키는 Job 입니다.

#### FlowJob

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/FlowJobDomain.PNG)

SimpleJob 은 순차적으로 실행 된다고 한다면 FlowJob 은 순차적으로 Step 을 실행하다가 어느 조건에 의해 Step2 로 실행 할지

아니면 Step3 로 진행 할지 분기 처리 방식 입니다.


### JobInstance, JobExecution 자세히 알아보자

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/job_instance_execution_domain.PNG)

지금까지 Job 은 어떻게 실행하는지 알아보았습니다. 복잡한 로직 때문에 이해가 어려워 쉽게 이미지로 만들어보았습니다.

우리가 앞서 BasicJobConfiguration 클래스에서 Job 이름을 'basicJob' 이라고 정했습니다.

그럼 Job 이름 basicJob 라고 구조체를 만들고 그 해당 Job 을 실행하면 실행단위을 나타내는 'JobInstance' 객체가 생성 됩니다.

물론 여기서 먼저 JobParameter 객체를 만들고  JobName 와 JobParameter 객체를 가지고 JobInstance 생성 합니다.

그런 다음 생성한 JobInstance 의 하위 객체인 JobExecution 객체가 생성 됩니다. 실행단위을 나타내는 것이 'JobInstance' 라면

JobExecution 는 해당 실행 단위를 더 구체적인 상태정보를 담고 있습니다.

즉 우리가 구성한 Job (JobName: basicJob) 이 실행하면 이전에 실행한 JobParameter 가 같지 않는 이상  JobInstance 가 생성되고

JobExecution 생성 됩니다. 하지만 여기서 비지니스 로직 수행 중 Exception 일어나고 다시 실행 하게 된다면

기존 JobInstance 는 유지 되고 JobExecution 한번 더 생성 하게 됩니다. 물론 성공 후에는 이전과 같은 JobParameter 라면 다시 Job 이 실행 되지 않습니다.

이러한 상태정보는 후에 자세히 배우게 되는데요 메타데이터 저장소에 모두 관리 하게 됩니다.


## Spring Batch 에서 Step 은 어떻게 실행할까?

이번에는 Job 이러 Step 실행 과정에 대해 알아보겠습니다.

Step 은 다시 말씀드리자면 하나의 배치 작업 즉 Job 이 실행 하는 과정에서 그 안에 독립적으로 구성되고 역활 하는 하나하나가 Step 이라고 생각 하시면 됩니다.

### 프로젝트 기동

```markdown
@Bean
public Step basicStep1() {
    return stepBuilderFactory.get("basicStep1")
            .tasklet(new Tasklet() {
                @Override
                public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
                    System.out.println("basicStep1");
                    return RepeatStatus.FINISHED;
                }
            })
            .build();
}
```

우리가 만들었던 BasicJobConfiguration 클래스 확인 해보면 Step 을 설정하는 곳이 보입니다.

```markdown
public class StepBuilderFactory {

	private JobRepository jobRepository;
	private PlatformTransactionManager transactionManager;
	
    public StepBuilder get(String name) {
        StepBuilder builder = new StepBuilder(name).repository(jobRepository).transactionManager(
                transactionManager);
        return builder;
    }
}
```
StepBuilderFactory 가 StepBuilder 를 생성합니다. 이후 StepBuilder 는 하위 빌더 클래스 중 가장 기본이 되는 TaskletStepBuilder 생성 합니다.

```markdown
public class StepBuilder extends StepBuilderHelper<StepBuilder> {

	public TaskletStepBuilder tasklet(Tasklet tasklet) {
		return new TaskletStepBuilder(this).tasklet(tasklet);
	}
```
TaskletStepBuilder 클래스를 생성하고 우리가 배치 비지니스를 정의한 tasklet 인자로 받아서 

```markdown
public class TaskletStepBuilder extends AbstractTaskletStepBuilder<TaskletStepBuilder> {

	private Tasklet tasklet;
	
	public TaskletStepBuilder tasklet(Tasklet tasklet) {
		this.tasklet = tasklet;
		return this;
	}
}
```
Tasklet 전역변수에 값을 저장 합니다.

```markdown
public abstract class AbstractTaskletStepBuilder<B extends AbstractTaskletStepBuilder<B>> extends
StepBuilderHelper<AbstractTaskletStepBuilder<B>> {

    public TaskletStep build() {

		// .. 생략 ..

		TaskletStep step = new TaskletStep(getName());

		// .. 생략 ..
		
		step.setTasklet(createTasklet());

		// .. 생략 ..

		try {
			step.afterPropertiesSet();
		}
		catch (Exception e) {
			throw new StepBuilderException(e);
		}

		return step;

	}

}
```
마지막으로 build 메소드를 호출 하게 된다면 AbstractTaskletStepBuilder 클래스에서 build 메소드로 호출 하게 됩니다.

그럼 ``TaskletStep step = new TaskletStep(getName());``

getName 을 호출해서 우리가 'basicStep1' 이라고 Step 이름을 지정한것을 불러오고 TaskletStep 클래스를 생성하게 됩니다.

``step.setTasklet(createTasklet());``

createTasklet() 메소드를 통해 바로 전에 TaskletStepBuilder 클래스에서 우리가 만든 비지니스 로직을 tasklet 변수에 저장한것을 

불러오게 되고 생성한 TaskletStep 에 setter 하게 됩니다.

BasicJobConfiguration.java
```markdown
@Bean
public Job basicJob() {
    return this.jobBuilderFactory.get("BasicJob")
            .start(basicStep1())
            .next(basicStep2())
            .build();
}
```
그 다음 이전에 설명한대로 최종적으로 Job 객체를 생성하기 위한 build() 메소드를 호출 하게 됩니다. 이 단계는 위 Job 단계에서 설명해드렸습니다.

### 배치 실행

```markdown
public class SimpleJob extends AbstractJob {

	@Override
	protected void doExecute(JobExecution execution) throws JobInterruptedException, JobRestartException,
	StartLimitExceededException {

		StepExecution stepExecution = null;
		for (Step step : steps) {
			stepExecution = handleStep(step, execution);
            // .. 생략 ..
		}

        // .. 생략 ..
	}
}
``` 

상단에 SimpleJob 클래스가 doExecute 메소드까지 호출한다고 설명해드렸습니다.

여기서 Step 역활로 바뀌는데요 

```markdown
public class SimpleStepHandler implements StepHandler, InitializingBean {

	@Override
	public StepExecution handleStep(Step step, JobExecution execution) throws JobInterruptedException,
	JobRestartException, StartLimitExceededException {
	
        // .. 생략 ..
        
        JobInstance jobInstance = execution.getJobInstance();

        StepExecution lastStepExecution = jobRepository.getLastStepExecution(jobInstance, step.getName());
        
        // .. 생략 ..
        
        StepExecution currentStepExecution = lastStepExecution;

        if (shouldStart(lastStepExecution, execution, step)) {

            currentStepExecution = execution.createStepExecution(step.getName());

             // .. 생략 ..

            jobRepository.add(currentStepExecution);

            // .. 생략 ..
            try {
                step.execute(currentStepExecution);
                currentStepExecution.getExecutionContext().put("batch.executed", true);
            }
            // .. 생략 ..

            jobRepository.updateExecutionContext(execution);

             // .. 생략 ..

        }

        return currentStepExecution;

	}

}
```

Job 에서 생성한 JobInstance 를 가져와서 

``StepExecution lastStepExecution = jobRepository.getLastStepExecution(jobInstance, step.getName());``

StepExecution 가져옵니다. 만약 메타데이터 저장소에 해당 Step 이 이전에 실패한 Step 이라면 가져 옵니다.

``currentStepExecution = execution.createStepExecution(step.getName());``

이전에 실행한 Step 이 아니라면 새로 생성을 합니다.

``step.execute(currentStepExecution);``

마지막으로 step 을 실행 하는 메소드로 진행 하게 됩니다.


```markdown
public abstract class AbstractStep implements Step, InitializingBean, BeanNameAware {

    @Override
    public final void execute(StepExecution stepExecution) throws JobInterruptedException,
    UnexpectedJobExecutionException {

         // .. 생략 ..

        stepExecution.setStartTime(new Date());
        stepExecution.setStatus(BatchStatus.STARTED);
        
        Timer.Sample sample = BatchMetrics.createTimerSample();
        getJobRepository().update(stepExecution);

      // .. 생략 ..
        try {
            getCompositeListener().beforeStep(stepExecution);
            // .. 생략 ..

            try {
                doExecute(stepExecution);
            }
    
            exitStatus = ExitStatus.COMPLETED.and(stepExecution.getExitStatus());

        }
        catch (Throwable e) {
             // .. 생략 ..
        }
        finally {

            try {
                // Update the step execution to the latest known value so the
                // listeners can act on it
                exitStatus = exitStatus.and(stepExecution.getExitStatus());
                stepExecution.setExitStatus(exitStatus);
                exitStatus = exitStatus.and(getCompositeListener().afterStep(stepExecution));
            }
              // .. 생략 ..
            try {
                getJobRepository().updateExecutionContext(stepExecution);
            }
             // .. 생략 ..

           
            stepExecution.setEndTime(new Date());
            stepExecution.setExitStatus(exitStatus);
            
             // .. 생략 ..
            try {
                getJobRepository().update(stepExecution);
            }
             // .. 생략 ..

            try {
                close(stepExecution.getExecutionContext());
            }
              // .. 생략 ..

            doExecutionRelease();

             // .. 생략 ..
        }
    }
}
```
``        stepExecution.setStartTime(new Date());
          stepExecution.setStatus(BatchStatus.STARTED);``
          
여기서 stepExecution 을 배치 실행 시간과 상태값을 변경 합니다. (나중에 메타데이터 저장소에 저장 하게 됩니다.)

``doExecute(stepExecution);``

doExecute 메소드를 호출하게 됩니다. 여기서 stepExecution 인자값으로 전달하는데 앞서 설명한 JobExecution Job 실행단위 관련된 상태 저장 관리 이고

stepExecution 도 배치 시작 날짜 및 상태값 등 실행 단위 관련된 데이터 관리 한다고 생각하시면 되겠습니다.

```markdown
public class TaskletStep extends AbstractStep {

    @Override
	protected void doExecute(StepExecution stepExecution) throws Exception {
        // .. 생략 ..

		stepOperations.iterate(new StepContextRepeatCallback(stepExecution) {

			 // .. 생략 ..
		});

	}

}
```

AbstractStep 구현체인 TaskletStep 클래스 doExecute 메소드로 호출하게 됩니다.

``stepOperations.iterate(new StepContextRepeatCallback(stepExecution) {``

여기서 iterate 통해 호출하게 되고 반복 수행하는데 여기서 TaskletStep 클래스인 doInTransaction 메소드를 실행 하게 되는 것으로 보입니다.

사실 어떻게 doInTransaction 메소드로 호출 되는 경로는 파악이 되지 않는 상태 입니다. 만약 아신다면 피드백 부탁드립니다.


```markdown
public class TaskletStep extends AbstractStep {
    @Override
    public RepeatStatus doInTransaction(TransactionStatus status) {
        // .. 생략 ..
        result = tasklet.execute(contribution, chunkContext);
        // .. 생략 ..
    }
}
```

``result = tasklet.execute(contribution, chunkContext);``

초기 우리가 BasicJobConfiguration 클래스에서 Step 에 tasklet 정의한 것을 execute 메소드 호출하는데요.

즉 이것은 Job 에 설정한 Step 에서 Tasklet 구현체에 호출 합니다.

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/basicStep.PNG)

우리가 Step 의 Tasklet 에 비지니스 로직을 구현한 곳에 가게 됩니다.

여기서 우리가 로직을 구성한것을 수행하게 됩니다.

## Step 객체

Spring Batch 에서 사용되는 Step 인터페이스 대해 알아보겠습니다.

### Step

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/step.png)
![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/step_interface.PNG)

### Step

Spring Batch 'Step' 은 최상위 인터페이지로 구성되어 있습니다.

대표적인 메소드를 보면

* execute(): 설정한 Step 을 실행 하는 메소드 입니다.

### AbstractStep

AbstractStep 입니다. 대표적인 필드만 알아보겠습니다.

* name: 설정한 해당 Step 의 이름 입니다.
* startLimit: Step 실행 제한 횟수 입니다.
* allowStartIfComplete: 해당 Step 이 실행 완료 후 재 실행 여부 입니다.
* jobRepository: Step 메타데이터 저장소 입니다.

### TaskletStep

Step 구현체 중 가장 기본이 되는 객체가 TaskletStep 입니다.


## TaskletStep 정리


Job 에서 TaskletStep 을 실행 한다면 BatchStatus 값은 'STARTED', ExitStatus 값은 'EXECUTING' 로 변경됩니다.

여기서 우리가 비지니스 로직을 구현한 Tasklet 을 수행하게 되는데요.

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/step_business_logic.PNG)
우리가 만든 비지니스 로직이 수행 하는데 중요한것은 loop 를 돌면서 수행한다는 점 입니다. 반드시 어느 시점에 loop 를 빠져나가고 싶으면

``RepeatStatus.FINISHED`` 값을 return 해야 합니다. 참고로 null 로 return 해도 됩니다.

동시에 RepeatStatus 값은 'FINISHED' 로 변경 됩니다.

최종 해당 Step 완료 되면 BatchStatus 값은 'COMPLETED' 로 변경 됩니다.



