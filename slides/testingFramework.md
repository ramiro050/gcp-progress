## Testing Framework: `Catch2`

***

## Why `Catch2`

- Simple and expressive
- Header-only
- Good integration with `ctest` and mocking libraries

***

## Testing [`read_runfile`](#/read-runfile/2)

notes:
bugs in runfile parser, very easy to get from testing
- Buffer not advancing (just send runfile and see if you get it back)
- staying stuck in while loop (your test will never terminate
- EOF stuff 

***

#### `tests/smurfdTests/test_SmurfRunfileParse.cc`

```cpp [10|11-17|20-22|24-39|42]
/**
 * This test checks that the function parse_runfile_helper is working
 * as expected.
 *
 * In order to simulate a TCP communication (which is what GCP + SMuRF 
 * will eventually do) this test creates a pipe and a child process, 
 * and the process plays the role of the client (SMuRF) sending the runfile
 * to the parent process representing the server (GCP)
 */
SCENARIO("runfile parser only succeeds on valid sized runfile", "[runfile]") {
  GIVEN("a buffer to write parsed file in and a pipe for communication") {
    REQUIRE(SMURF_RUNFILE_SIZE > 0); // Assume we can write at least one character
    char buf[SMURF_RUNFILE_SIZE] = { '\0' };
    int runfile_size;
    bool success;
    int pipefd[2];
    REQUIRE(pipe(pipefd) == 0);

    pid_t cpid;
    WHEN("runfile is smaller than expected") {
      runfile_size = SMURF_RUNFILE_SIZE - 1;
      double timeout = 100.0; // parser should fail before timeout expires

      THEN("parser will return false") {
        cpid = fork();
        if (cpid == 0) {
          close(pipefd[0]); // child writes to pipe
          for (int i = 0; i < runfile_size; i++) {
            write(pipefd[1], "a", 1);
          }
          close(pipefd[1]);
          exit(0);
        } else {
          close(pipefd[1]); // parent reads from pipe
          success = gcp::smurfd::SmurfRunfileParse::parse_runfile_helper(pipefd[0],
                                                                         buf,
                                                                         timeout);
          close(pipefd[0]);
          wait(NULL); // wait for child
        }

        REQUIRE(!success);
      }
    }
  }
}

```

***

<!-- .slide: data-auto-animate -->

## demo

After compilation, executable `RunSmurfTests` is created

```shell []
badaq@basmurf:~/gcp_build$ ./bin/RunSmurfTests
```

***

<!-- .slide: data-auto-animate -->

## demo

After compilation, executable `RunSmurfTests` is created

```shell []
badaq@basmurf:~/gcp_build$ ./bin/RunSmurfTests
```

There is also integration with `ctest`

```shell []
badaq@basmurf:~/gcp_build/tests$ ctest
```

***

## Current limitation

`GCP` was not designed with unit testing in mind

***

<!-- .slide: data-auto-animate -->

#### Class dependence of subset of `GCP`'s `smurfd`

![img](imgs/gcpGraph.svg "Dependence graph of GCP")

notes:
- What do the arrows mean?
- Example: How does data get an instance of `SmurfRunfileParser`
- All coupled

***

<!-- .slide: data-auto-animate -->

#### Class dependence of subset of `GCP`'s `smurfd`

![img](imgs/gcpGraph.svg "Dependence graph of GCP") <!-- .element: height="600px" -->

- No abstract classes = no mocking

***

<!-- .slide: data-auto-animate -->

#### Class dependence of subset of `GCP`'s `smurfd`

![img](imgs/gcpGraph.svg "Dependence graph of GCP") <!-- .element: height="550px" -->

- No abstract classes = no mocking
- Heavy dependence on `SmurfdMaster`

***

<!-- .slide: data-auto-animate -->

#### Class dependence of subset of `GCP`'s `smurfd`

![img](imgs/gcpGraph.svg "Dependence graph of GCP") <!-- .element: height="500px" -->

- No abstract classes = no mocking
- Heavy dependence on `SmurfdMaster`
- Most of my work is in `Data` and `CommandTask`
