## Next Step

Figure out bugs in `SmurfDataParser`

notes:
The data parser has similar issues as the runfile parser

***

## Question

What should be the steps the data parser takes?

***

- Should the sequence of steps the data parser takes be:
  1. Run `mce_start_acq` on `bicepViewer`
  2. `SMuRF` opens connection at port `GCP` is listening on
  3. `GCP` accepts connection
  4. `SMuRF` begins sending data
  5. `SMuRF` closes connection, sending `EOF`
  6. `GCP` reads data until `EOF` is reached
  7. `GCP` closes its side of the connection
     - Currently `GCP` does not close connection when `SMuRF` does

- How was the blocking for reading data done in the past?
