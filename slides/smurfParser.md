## Smurf Runfile Parser

***

<!-- .slide: id="data-cc-file" -->

#### `bin/smurfd/Data.cc`

```cpp [|5-9|15|31-33|41-51]
void Data::serviceMsgQ()
{
  // some variable initialization

  if (!simData_) {
    if (dataBoard_->runfileListenFd() > 0)
      fdSet_.registerReadFd(dataBoard_->runfileListenFd());
    if (dataBoard_->dataListenFd() > 0)
      fdSet_.registerReadFd(dataBoard_->dataListenFd());
  }

  // more initialization

  while(!stop) {
    nready=select(fdSet_.size(), fdSet_.readFdSet(), NULL, NULL, timeOut_.timeVal());

    if(nready > 0) {
      //---------------------------------------------------------------
      // A message on our message queue?
      //---------------------------------------------------------------
      if(fdSet_.isSetInRead(msgqFd)) {
        DBPRINT(true, Debug::DEBUG11, "Call processMsg.");
        processTaskMsg(&stop);
      }

      //---------------------------------------------------------------
      // Now check for data arriving on any of the pipes
      //---------------------------------------------------------------
      //---------------------------------------------------------------
      // New connection
      if (fdSet_.isSetInRead(dataBoard_->runfileListenFd())) {
        dataBoard_->smurfAcceptRunfile();
        fdSet_.registerReadFd(dataBoard_->runfileReadFd());
      }

      if (fdSet_.isSetInRead(dataBoard_->dataListenFd())) {
        dataBoard_->smurfAcceptData();
        fdSet_.registerReadFd(dataBoard_->dataReadFd());
      }

      if(fdSet_.isSetInRead(dataBoard_->runfileReadFd()) ) {
        parse_bufferRunFile();

        // After every attempt to parse, runfile pipe is closed.
        // If there is an error while sending runfile
        // client (SMuRF) needs to reconnect and send again.
        // This is done to make sure server always reads from
        // beginning of runfile and not from left over data
        // on runfile pipe when parse_bufferRunfile() is called
        fdSet_.clearFromReadFdSet(dataBoard_->runfileReadFd());
        dataBoard_->closeRunfileReadFd();
      }
      //---------------------------------------------------------------
      // Run File ready to read?
      if(fdSet_.isSetInRead(dataBoard_->dataReadFd()) ) {

        // parse data
      }
    }
  }
}
```

notes:
- What is the purpose of `serviceMsgQ`?
  - Smurf can get messages from `biceViewer`, `SMuRF`, etc
  - Handles such messages

***

#### Old `bin/smurfd/SmurfRunfileParse.cc`

```cpp [|16-29|32-38]
void SmurfRunfileParse::
parse_runfile (SmurfBoard * smurf_board)
{
  ReportMessage ("Parsing SMURF runfile.");
  char buf[SMURF_RUNFILE_SIZE];
  int res;
  int nready;
  int nread = 0;
  unsigned crate_id, fpga_version;
  FdSet fd_set;
  TimeVal time_out;

  fd_set.registerReadFd(smurf_board->runfileReadFd());
  // Client has 10 seconds to send runfile
  time_out.setTime(10, 0);
  while (nread < SMURF_RUNFILE_SIZE)
    {
      nready = select(fd_set.size(), fd_set.readFdSet(), nullptr, nullptr, time_out.timeVal());
      if (nready <= 0) {
        ReportError("SMuRF runfile read timed out");
        return;
      }
      nread += read(smurf_board->runfileReadFd(), buf, SMURF_RUNFILE_SIZE-nread);

      if (res != 0)
        {
          DBPRINT(true, Debug::DEBUG11, "Error parsing runfile line, but will keep trying.\n");
        }
    }
  DBPRINT(true, Debug::DEBUG11, "Finished parsing SMURF runfile");

  // Grab registers from buffer
  crate_id_ = ntohl(*(uint32_t *)(&buf[0]));

  fpga_version_ = ntohl(*(uint32_t *)(&buf[4]));

  memcpy(commit_, &buf[8], 8);
  commit_[8] = '\0';

  // Now ready to pack runfile frames
  runfile_read_ = true;
}
```

[goto Data.cc](#/data-cc-file/3)

notes:
Remember: simple errors
1. `res` defined but not used (25)
2. `buf` not advanced (23)
3. What happens if SMuRF closes connection while in loop? (infinite loop)
4. What happens if SMuRF sends complete runfile and then closes connection?
   - `Data.cc` while loop executes, then goes back into `read_runfile` again
   and stays there
5. **NOTE** It's really hard to diagnose this bug by using GCP because GCP is
multithreaded. So most things still work except you can't reconnect and send
new runfile

***

<!-- .slide: id="read-runfile" -->

#### `bin/smurfd/SmurfRunfileParse.cc`

```cpp [|63-67|69-73|1-12|22-33|35-39|41-55]
/**
 * read_runfile reads from FD a total of SMURF_RUNFILE_SIZE characters
 * and writes them into BUF.
 * 
 * Returns true if reading is successful
 * Returns false if FD does not have the expected number of bytes or
 *         it takes longer than TIME_OUT_SECS to read all the bytes needed.
 *         This includes taking too long to send EOF (i.e. close connection)
 */
bool SmurfRunfileParse::read_runfile (const int fd,
                                      char (&buf)[SMURF_RUNFILE_SIZE],
                                      const double time_out_secs) {
  FdSet fd_set;
  fd_set.registerReadFd(fd);

  TimeVal time_out;
  time_out.setTime(time_out_secs);

  int fd_ready = 0;
  int bytes_read = 0; // bytes read after each call to read()
  int total_bytes_read = 0;
  do {
    fd_ready = select(fd_set.size(), fd_set.readFdSet(),
                      nullptr, nullptr, time_out.timeVal());

    if (fd_ready <= 0) {
      ReportError("SMuRF runfile read timed out");
      return false;
    }

    bytes_read = read(fd, buf + total_bytes_read, SMURF_RUNFILE_SIZE - total_bytes_read);
    total_bytes_read += bytes_read;
  } while (bytes_read != 0);

  if (total_bytes_read < SMURF_RUNFILE_SIZE) {
    ReportError("SMuRF runfile smaller than expected. Expected size: "
                << SMURF_RUNFILE_SIZE << " bytes");
    return false;
  }

  // Wait for EOF to be recieved
  fd_ready = select(fd_set.size(), fd_set.readFdSet(),
                    nullptr, nullptr, time_out.timeVal());
  if (fd_ready <= 0) {
    ReportError("SMuRF runfile read timed out");
    return false;
  }

  char small_buf;
  bytes_read = read(fd, &small_buf, 1);
  if (bytes_read != 0) {
    ReportError("SMuRF runfile larger than expected. Expected size: "
                << SMURF_RUNFILE_SIZE << " bytes");
    return false;
  }

  return true;
}

void SmurfRunfileParse::
parse_runfile (SmurfBoard * smurf_board)
{
  ReportMessage ("Parsing SMuRF runfile.");
  char buf[SMURF_RUNFILE_SIZE];
  unsigned crate_id, fpga_version;
  bool success = read_runfile(smurf_board->runfileReadFd(), buf, RUNFILE_PARSER_TIMEOUT);
  if (!success) return;

  // Grab registers from buffer
  crate_id_ = ntohl(*(uint32_t *)(&buf[0]));
  fpga_version_ = ntohl(*(uint32_t *)(&buf[4]));
  memcpy(commit_, &buf[8], 8);
  commit_[8] = '\0';

  // Now ready to pack runfile frames
  runfile_read_ = true;
  ReportMessage ("SMuRF runfile parsed.");
}
```

notes:
1. `res` defined but not used (25)
2. `buf` not advanced (23)
3. What happens if SMuRF closes connection while in loop?
4. What happens if SMuRF sends unlimited data while in loop?
   - both cases always exit because loop is on `bytes_read`
5. If parser fails, GCP closes connection, protecting itself

***

## demo

```python []
>>> from send_runfile import *
>>> s = connecToGCP()
>>> sendData(s, 16); closeConnection(s)
```

notes:
1. success
2. `sendData(s, 10); closeConnection(s)` (error: too small)
2. `sendData(s, 100000)` (error: too long. break pipe)
3. `sendData(s, 16)` (error: time out)
