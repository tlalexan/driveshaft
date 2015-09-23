# Driveshaft - Gearman Worker Manager

# Getting Started

# Install
## Dependencies
* cmake 2.8.7 or later
* libgearman(-devel)
* log4cxx(-devel)
* libcurl(-devel)
* boost(-devel) 1.48 or later
* gcc 4.8 or later

## Build
```
$ cmake .
$ make
```

## Install
```
$ sudo make install
```

# Configuration
driveshaft takes two arguments:
```
$ driveshaft
Allowed options:
  --help                produce help message
  --jobsconfig arg      jobs config file path
  --logconfig arg       log config file path
```

## jobsconfig
A simple jobsconfig file looks like this:
```json
{
    "gearman_server_timeout": 5000,
    "gearman_servers_list":
    [
        "localhost"
    ],
    "pools_list":
    {
        "ShopStats":
        {
            "job_processing_uri": "http://localhost/job.php",
            "worker_count": 0,
            "jobs_list":
            [
                "ShopStats"
            ]
        },
        "Newsfeed":
        {
            "job_processing_uri": "http://localhost/job.php",
            "worker_count": 0,
            "jobs_list":
            [
                "Newsfeed"
            ]
        },
        "Regular":
        {
            "job_processing_uri": "http://localhost/job.php",
            "worker_count": 0,
            "jobs_list":
            [
                "Sum3",
                "Sum",
                "Sum2"
            ]
        }
    }
}
```

### Jobs Config Options
* `gearman_server_timeout` - timeout in milliseconds to wait for a job from gearmand.
After this timeout the thread checks if it should exit, if not it tries gearmand again.
* `gearman_servers_list` - addresses of gearmand servers
* `pools list` - a list of named pools and corresponding configuration for every pool:
    * `worker_count` - Number of workers to reserve for jobs in this pool
    * `jobs_list` - Names of jobs that should be ran on the workers in this pool
    * `job_processing_uri` - the uri to send the job payload to for execution

## logconfig
An [example log config is
included](https://github.com/keyurdg/driveshaft/blob/master/logconfig.xml) in
the repository. For more information, see
[the log4cxx documentation](https://logging.apache.org/log4cxx/usage.html).

# Design
1. Jobs are grouped into pools and every pool has a `worker_count` setting in order
to define the maximum concurrency.
2. For every pool, it registers the jobs in `jobs_list` with gearmand and maintains
`worker_count` threads with persistent connections to fetch jobs and submit back
results.
3. When the config changes, it signals the appropriate pool threads to die. Any
currently running job on that thread has a 24 hour window to finish, otherwise
the job is considered failed, gearmand is updated and the thread is closed. Note
that the job may keep on running on the HTTP endpoint. New pool threads are
created as needed to match the configuration.
4. Jobs are run via the HTTP endpoint `job_processing_uri` defined in the config. The endpoint
will receive the class name and all the args and will have to do the right thing and
return SUCCESS/FAILURE along with any response text. The thread that is
processing the job blocks waiting for a response.

By reusing connections and not re-registering with gearmand on every job completion,
Driveshaft saves gearmand a lot of work that impacts enqueue latency.

And by using an HTTP endpoint to actually do the heavy lifting, we get the
benefits of a clean-sandbox and Opcache (and can even use HHVM!).