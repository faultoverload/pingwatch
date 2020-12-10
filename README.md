# pingwatch
Simple monitoring tool for your Internet connection, pings hosts and reports on availability and trends

## Screen shot

![Screen shot](screenshot.png)

## Building

Prerequisites:

* Go 1.13 or later

```
git clone https://github.com/fazalmajid/pingwatch
cd pingwatch
go build
```

On Linux, run:

```
sudo setcap cap_net_raw=+ep pingwatch
```

this allows `pingwatch` to open raw sockets so it can send ICMP packets.

You can get a list of command-line flags using `pingwatch -h`

#### Docker Install ####

Create a working dir for you to save your docker compose files and database
```
mkdir ~/pingwatch
```
Create a Dockerfile
```
FROM golang:1.13
RUN git clone https://github.com/fazalmajid/pingwatch
WORKDIR ./pingwatch
RUN go mod download
RUN go build
RUN setcap cap_net_raw=+ep pingwatch
CMD ./pingwatch -db /data/pingwatch.sqlite -p "0.0.0.0:8086"
```
Create a docker-compose.yml file
```
version: '2'
services:
  pingwatch:
   container_name: pingwatch
   build:
    context: ./
    dockerfile: Dockerfile
   ports:
    - "8086:8086/tcp"
   volumes:
    - ~/pingwatch/:/data/
   tty: true
```
Build and start up your new docker file
```
docker-compose up --build -d
```
Now we just need to add some networks to ping.
```
docker exec -it pingwatch bash
```

```
./pingwatch -add 8.8.8.8 -db /data/pingwatch.sqlite
```
Finally restart your container for the changes to take effect
```
docker restart pingwatch
```

## Database

### Configuration

The list of hostnames or IPs to ping is in the table `dests`.

To add a destination, run:

```
sqlite3 pingwatch.sqlite << EOF
insert into dests values ('1.1.1.1'), ('8.8.8.8'), ('www.google.com');
EOF
```

After database is initialized you can add or delete destinations as follows:

To add a destination, run:

```
pingwatch -add example.com
```

To remove a destination, run:

```
pingwatch -del example.com
```


### Measurement data

The actual measurements are in the table `pings`:

* *time* timestamp in SQLite julian day format, UTC)
* *host* hostname that was pinged
* *ip* IPv4 or IPv6 address *host* resolved to at ping time
* *rtt* ping round-trip time in milliseconds. A value of `-3600` means the ping timed out

### Housekeeping

You can use sqlite triggers to automatically clear old entries to keep the database small

This trigger will delete entries from table pings which are older than 1 day:

```
sqlite3 pingwatch.sqlite << EOF
CREATE TRIGGER clean1day AFTER INSERT ON pings
BEGIN
	DELETE FROM pings WHERE pings.time < julianday('now', '-1 day');
END
EOF
``` 

Refer to the [julianday() function manual](https://sqlite.org/lang_datefunc.html) for details


## Web user interface

The web user interface can be accessed at http://localhost:8086/ by default (or change it using the `-p` flag).

By convention, a ping time of -100ms means timeout, so downtime can stand out in the graph. In the SQLite database, this is stored as -3600e3

You can zoom in by selecting a time range, scroll using by pressing Shift while moving the mouse or zoom out back to the original view by double-clicking (graphs are courtesy of the awesome [Dygraphs](https://dygraphs.com) library).

The default date range for the graph is 14 days, but you can change that:

* `pingwatch -days 7` to show only the last 7 days
* `pingwatch -display 6h` to display only the last 6 hours
