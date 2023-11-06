## Scheduled Jobs
- Phân tích cách EspoCrm triển khai Scheduled jobs

### Câu hỏi
- điều kiện success/fail
- pid, data để làm gì
- cơ chế re-try: attemps

### Config.php
```php
[
    'jobRunInParallel' => false,  //Enabled `jobRunInParallel` parameter requires pcntl and posix extensions.
    'cronMinInterval' => 0, // thời gian tối thiểu giữa các lần chạy cron.php
    'jobPeriodForNotExistingProcess' => 300 // thời gian đánh fail job nếu ko có process chạy
    'jobPeriodForReadyNotStarted' =>  60// thời gian đánh fail job nếu ready mà không khởi động được
    'jobPeriod' => 7800 // thời gian tối đa chạy 1 job
    'jobPeriodForActiveProcess' => 7800 // thời gian tối đa chạy 1 job active

]

```

### Cron.php
- Tạo job mới trước, sau đó run pending

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