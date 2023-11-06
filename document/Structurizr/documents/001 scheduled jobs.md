## Scheduled Jobs
- Phân tích cách EspoCrm triển khai Scheduled jobs

### Câu hỏi
- điều kiện success

### Config.php
```php
[
    'jobRunInParallel' => false,  //Enabled `jobRunInParallel` parameter requires pcntl and posix extensions.
    'cronMinInterval' => 0, // thời gian tối thiểu giữa các lần chạy cron.php
    'jobPeriodForNotExistingProcess' => 300 // thời gian đánh fail job nếu ko có process chạy
    'jobPeriodForReadyNotStarted' =>  60// thời gian đánh fail job nếu ready mà không khởi động được
    'jobPeriod' => 7800 // thời gian tối đa delay 1 job so với kế hoạch
    'jobPeriodForActiveProcess' => 7800 // thời gian tối đa delay 1 job so với kế hoạch active
    'jobMaxPortion' => 15 // số lượng job tối đa 1 lần process
]

```

### Cron.php
- Tạo job mới trước, sau đó run pending
- Các status của Job
```php
namespace Espo\Core\Job\Job;
class Status
{
    public const PENDING = 'Pending';
    public const READY = 'Ready';
    public const RUNNING = 'Running';
    public const SUCCESS = 'Success';
    public const FAILED = 'Failed';
}

```

### Điều kiện fail job
1. markJobsFailedByNotExistingProcesses
- tìm các job Running và startAt trước hiện tại 1 khoảng (mặc định 300s).
- Nếu Pid của job không còn active thì đanh fail

2. jobPeriodForReadyNotStarted
- tìm các job Ready và startAt trước hiện tại 1 khoản (mặc định 60s) và đánh fail

3. markJobsFailedByPeriod ($isForActiveProcesses = true)
- tìm các job Running và excuteTime trước hiện tại 1 khoảng (mặc định 2 tiếng) và đánh fail

4. markJobsFailedByPeriod ($isForActiveProcesses = false)
- tìm các job Running và excuteTime trước hiện tại 1 khoảng (mặc định 2 tiếng)
- nếu không có Pid đang active thì đánh fail

### Đánh fail job
1. update status = failed và attempts = 0;
2. ghi log

### Process các job
1. lấy list scheduled job active và running
- active Scheduled Job là các Scheduled Job có status = Active
- running Scheduled Job là các Scheduled Job mà 
    - có Job đang có status = running hoặc ready
    - field job không có trong list metadataProvider->getPreparableJobNameList()
2. Tạo Job cho các Scheduled Job đang active mà chưa thuộc running
- status: pending
- excuteTime: thời gian mong muốn job chạy 

###  Queue Q0 Q1 E0
- Chạy các job được setup bởi ` Espo\Core\Job\JobScheduler `
- đây là tính năng lên lịch các Job mà không thông qua Scheduled Jobs. Có thể đăng ký trực tiếp Class chạy.
Ví dụ ` Espo\Tools\Stream\HookProcessor `
```php
    $this->jobSchedulerFactory
        ->create()
        ->setClassName(ControlFollowersJob::class) // Espo\Tools\Stream\Jobs\ControlFollowersJob
        ->setQueue(QueueName::Q1)
        ->setData([
            'entityType' => $entity->getEntityType(),
            'entityId' => $entity->getId(),
        ])
        ->schedule();
```

- Nếu implements interface Job chứ không phải interface JobDataLess thì sẽ truyền vào hàm run($data) 
    - tham khảo application\Espo\Core\Job\JobRunner.php(212)
    - method của data
    ```php
    array(9) {
        [0]=>
        string(11) "__construct"
        [1]=>
        string(6) "create"
        [2]=>
        string(6) "getRaw"
        [3]=>
        string(3) "get"
        [4]=>
        string(3) "has"
        [5]=>
        string(11) "getTargetId"
        [6]=>
        string(13) "getTargetType"
        [7]=>
        string(12) "withTargetId"
        [8]=>
        string(14) "withTargetType"
    }

    ```

1. Q0
- Chạy ASAP
- limit : Q0_PORTION_NUMBER = 200;

2. Q1
- Chạy mỗi phút
- limit:  Q1_PORTION_NUMBER = 500;

3. E0
- Chạy ASAP gửi Email
- limit:  E0_PORTION_NUMBER = 100;

### Tips & Tricks
1. Truyền dữ liệu
- schedule thủ công hoặc dùng JobScheduler để ghi được data
- Tất cả dữ liệu cần pass ghi vào cột data, bao gồm cả các thông tin của Job
- Khai báo job implement Job để lấy data

2. retry
- Can thiệp vào giá trị attempts > 0

3. Run job manually
```php

    $jobEntity = $this->getEntityManager()->getEntity('Job');

    $jobEntity->set([
        'name' => $data->name,
        'status' => 'Pending',
        'scheduledJobId' => $data->id,
        'executeTime' => $nextDate,
    ]);

    $this->getEntityManager()->saveEntity($jobEntity)

```