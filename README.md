Asphalt
=======

A simple distributed execution server built on Docker with Go.

*Note: This is a work in progress with documentation being written before code. It may or may not be built.*

Build Dependencies
------------
- Docker
- Go

Runtime Dependencies
--------------------
- Docker

Using Asphalt
---------------
```sh
# start a default master node
$ asphalt

# start a master node with a specified port
$ asphalt -port 8080

# create a project
$ curl --data="{'name':'483c8a...', ...}" 127.0.0.1:8080

# create a task
$ curl --data="{'id':'9399dda8c384', ...}" 127.0.0.1:8080/projects/483c8a...

# start a worker node
$ asphalt -master 10.0.1.7:8000
```

Concepts
--------

### Master/Worker Nodes
Each instance can be started in either master or worker mode. A master node is responsible for pushing tasks to workers and has a web interface where users can check task status, re-run a task, and view task output. Once a task is created a master node will send the task details to a queue. All worker nodes pick up tasks from the queue, execute them, and return the results in a result queue. The master pulls from the result queue to store results for future analysis by the user. If post-task actions are needed the master can schedule the action in a reaction queue for a worker to pick up.

Both task messages and reaction messages are picked up by the same workers. They both execute the given task and return the output to a results queue with a specified result type.

*Note: Master nodes start a worker on the same server so it's possible to run without any external workers configured.*

### Queue
Master and worker nodes communicate over a queue. This allows for distributed processing without overwhelming workers and implementing back off strategies. Once a task is created the master will push a message to the queue with the task details. The first available worker will pull from the task queue, execute the task, and return the results in a result queue. The master node can read from the result queue for further task execution and storage.

*Note: Master nodes create the queues on the same server.*

### Jobs
Jobs are the most granular level of execution containing all executional data.
```go
type Job struct {
  id            string            // uuid
  action        string            // job to run (ex: "rake test")
  container     string            // docker container (ex: 'ubuntu', 'ruby')
  env           map[string]string // environment variables for task execution (ex: '{"API_KEY": "483c8add9939", "RAILS_ENV": "test"}')
  startedAt     int               // time the job started
  endedAt       int               // time the job ended
  output        string            // stdout from the action
  status        int               // status code from the job completion (0 for success, 1 for failure)
  timeout       int               // number of seconds to allow the job to run before killing it and triggering a failure
}
```

### Tasks
A task is a collection of jobs.

```go
type Task struct {
    id          string            // id of the task (ex: the current git commit hash -- `git rev-parse HEAD`)
    jobs        map[string]Job    // job to run (ex: '{"main": MainJob, "success": SuccessJob, "failure": FailureJob, "after": AlwaysAfterJob}')
    status      int               // the tasks current status (ex: 0 for 'waiting', 1 for 'running', 2 for 'complete')
}
```

### Projects
A project is a collection of tasks. These can be created to isolate different groups of tasks.
```go
type Project struct {
    id          string // uuid
    name        string // name of the task (ex: "My Great Site")
    tasks       []Task // tasks belonging to the project
}
```

### Receivers
Receivers consume data from outside sources and converts it into jobs and tasks. The HTTP API is currently the only receiver (the master node web interface uses it), but more receivers could be added to handle git pushes, emails, or even go out and fetch the data itself.

### Storage
All results are stored in BoltDB with serialized Gobs. In the future this could be expanded to other storage solutions, but for now if you need your data elsewhere you can integrate the data push in your post result jobs.

### Future ideas
- Give workers "types" so you can target iOS builds to Mac workers, C# builds Windows workers, etc
