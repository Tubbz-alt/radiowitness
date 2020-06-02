# RadioWitness
Immutable, peer-to-peer archiving and distribution of First Responder radio broadcasts. Authors use [Software-Defined Radios](https://sdr.osmocom.org/trac/wiki/rtl-sdr) and [Dat Archives](https://datproject.org/) to record local Police and Fire radio. Publishers seed archives from Authors and then re-distribute them to larger audiances. Other rolse arise too, such as Studios who synthesize radio archives into archives of audible speech, and Indexers who aggregate metadata on individual radio calls. Stop by [#radiowitness on freenode](https://webchat.freenode.net/) and say hi.

## Download & Build
```
$ git clone https://github.com/RadioWitness/radiowitness
$ cd ./radiowitness
$ docker build -t rwp2p .
$ docker build -t usbreset lib/c/usbreset
```

## Search for Radios
```
$ chmod +x ./bin/rtl_devices.sh
$ docker run $(./bin/rtl_devices.sh) --rm \
    rwp2p search -g 26 -f "851162500,851287500,851137500" 2> /dev/null
> 851162500 counted 0 frames.
> 851287500 counted 36 frames.
> 851137500 counted 43 frames.
$
$ docker volume create rw-author
$ docker run --rm --mount source=rw-author,target=/archive \
    rwp2p author create --lat "30.245016" --lon="-97.788914" --wacn 781833 --sysid 318 --rfssid 1 --siteid 1
> dat://d7a4c7d47b4e9791cae6710b637afc76d462a2aafcf05ba60ce891925c495271
```

## Provision Storage
This technique makes use of the [Minio](https://min.io) object store in order to support multiple cloud storage APIs. Publishers peer with Authors, replicate their archive into minio, then seed their archive from minio. Start minio and create a bucket:
```
$ docker network create rw-net
$ docker run -d --name rw-objects --network rw-net -p 9000:9000 \
    -e MINIO_ACCESS_KEY=abc -e MINIO_SECRET_KEY=12345678 \
    minio/minio server /data
$
$ docker run --rm -it --network rw-net --entrypoint=/bin/sh minio/mc
# mc config host add local http://rw-objects:9000 abc 12345678
# mc mb local/rw-author && exit
> Bucket created successfully `local/rw-author`.
```

## Authoring & Publishing
The [Dat Protocol](https://www.datprotocol.com/) is transport agnostic, meaning Dat peers can communicate over TCP, UDP, any duplex stream really. This flexibility allows Publishers to provide any number of peering strategies to their Authors. In practice a TCP socket on a well-routed VPS is a fine start, add [WireGuard VPN](https://www.wireguard.com/) and you'll be feeling like the su of Nynex. Both Author and Publisher peers replicate via `stdin&stdout` so you've just gotta pipe them together using your transport of choice, in this example a linux fifo:
```
$ mkfifo /tmp/replication
$ docker run $(./bin/rtl_devices.sh) -i --name rw-author --mount source=rw-author,target=/archive \
    rwp2p author --radios 3 --mux 2 -s 1200000 -g 26 -f 851137500 < /tmp/replication | \
      docker run -i --name rw-pub --network rw-net -p 9001:9001 -e WSS=9001 \
        -v /tmp/rw-publish:/archive:rw -e OBJECTS=rw-objects:9000 -e BUCKET=rw-author \
        rwp2p publish dat://d7a4c7d47b4e9791cae6710b637afc76d462a2aafcf05ba60ce891925c495271 > /tmp/replication
```

## Audio Synthesis
Public safety radio is unlike the radio you hear from a car or boombox because it travels through the airwaves in a compressed form, all this means is that we've got to do an extra step to get something audible out of it. This extra step is intentionally left behind by both Author and Publisher peers because the *raw* radio archive has importance in and of itself. The Studio peer reads a radio archive as input and produces an audio archive as output:
```
$ docker volume create rw-studio
$ docker run --rm --mount source=rw-studio,target=/archive \
    --network rw-net -e PUB=rw-pub:9001 \
    rwp2p studio create dat://d7a4c7d47b4e9791cae6710b637afc76d462a2aafcf05ba60ce891925c495271
> dat://9a09cb434fab6bf8c126f3036c4d9a7965e7c9acd3481db7df86bf665355259d
$
$ mc mb rwb2/rw-studio
$ mkfifo /tmp/repl-studio
$
$ docker run -d --name rw-studio --mount source=rw-studio,target=/archive \
    --network rw-net -e PUB=rw-pub:9001 rwp2p studio < /tmp/repl-studio | \
      docker run -d --name rw-pub-studio --network rw-net -p 9002:9002 -e WSS=9002 \
        --mount source=rw-studio,target=/archive
        -v /tmp/rw-pub-studio:/archive:rw -e OBJECTS=rw-objects:9000 -e BUCKET=rw-studio \
        rwp2p publish dat://9a09cb434fab6bf8c126f3036c4d9a7965e7c9acd3481db7df86bf665355259d > /tmp/repl-studio
$
$ docker run --rm -it --network rw-net -e PUB=rw-pub-studio:9002 \
    rwp2p play dat://9a09cb434fab6bf8c126f3036c4d9a7965e7c9acd3481db7df86bf665355259d | \
    play -t raw -b 16 -e signed -r 8k -c 1 -
```

## Audio Indexing
The archives created by Studios are basically giant, ever-growing [.WAV audio files](https://en.wikipedia.org/wiki/WAV) with a handful of extra metadata thrown in. Studio archives are great for streaming live but to really explore years worth of audio some search indexes are needed. Index peers read an audio archive as input and produces a [hyperdb](https://github.com/mafintosh/hyperdb) as output with calls batched by hours-since-unix-epoch at keys with the form `/calls/{epoch-hour}/{callid}`:
```
$ docker volume create rw-index
$ docker run --rm --mount source=rw-index,target=/archive \
    --network rw-net -e PUB=rw-pub-studio:9002 \
    rwp2p index create dat://9a09cb434fab6bf8c126f3036c4d9a7965e7c9acd3481db7df86bf665355259d
> dat://593aae4e3c3870d53961849144a07c4014479656ef127191efcef840804bf9e1
$
$ mc mb rwb2/rw-index
$ mkfifo /tmp/repl-index
$
$ docker run -d --name rw-index --mount source=rw-index,target=/app \
    --network rw-net -e PUB=rw-pub-studio:9002 \
    rwp2p index < /tmp/repl-index | \
      docker run -d --name rw-pub-index --network rw-net -p 9003:9003 -e WSS=9003 \
        -v /tmp/rw-pub-index:/archive:rw -e OBJECTS=rw-objects:9000 -e BUCKET=rw-index \
        rwp2p publish dat://593aae4e3c3870d53961849144a07c4014479656ef127191efcef840804bf9e1 > /tmp/repl-index
```

## Web App
todo: serialize archive keys along with peer list into `dat.json`:
```
$ mkdir -p archive.web
$ ./bin/radiowitness json dat://author.key \
    --studio dat://studio.key --index dat://index.key > archive.web/dat.json
$ dat create -y archive.web
$ ln -f archive.web/dat.json web/dat.json
$ npm run build --prefix web/
$ cp -r web/dist/\* archive.web/
$ dat sync archive.web
```

## Multi-Host Deploy
vpn w/ iptables + WireGuard

## Standards & Conventions
Different peers author different types of data and so we need to introduce a little structure. All peer types operate on [hypercores](https://github.com/mafintosh/hypercore) with the exception of *Index* who's output is a [hyperdb](https://github.com/mafintosh/hyperdb). Hypercores are an append-only log structure and we use record `0` within the log as a sort of header. HyperDB is a magic P2P key-value store and we put a sort-of-header at key `/rw-about`.

### rw-author hypercore record `0`
```
{
  "version" : 1,
  "type" : "rw-author",
  "tags" : {
    "geo" : { "lat" : 4.20, "lon" : 6.66 },
    "network" : { "wacn" : 1, "sysid" : 1, "rfssid" : 1, "siteid" : 1 }
  }
}
```

### rw-studio hypercore record `0`
Notice how `links` is used to reference the input hypercore:
```
{
  "version" : 1,
  "type" : "rw-studio",
  "links" : {
    "author" : [{ "type" : "rw-author", "href" : "dat://abc123" }]
  },
  "tags" : {
    "geo" : { "lat" : 4.20, "lon" : 6.66 },
    "network" : { "wacn" : 1, "sysid" : 1, "rfssid" : 1, "siteid" : 1 }
  }
}
```

### rw-index hyperdb key `/rw-about`
Notice how `links` is used to reference both the original radio hypercore and the audio hypercore produced by a `rw-studio` peer:
```
{
  "version" : 1,
  "type" : "rw-index",
  "links" : {
    "author" : [{ "type" : "rw-author", "href" : "dat://abc123" }]
    "author" : [{ "type" : "rw-studio", "href" : "dat://def456" }]
  },
  "tags" : {
    "geo" : { "lat" : 4.20, "lon" : 6.66 },
    "network" : { "wacn" : 1, "sysid" : 1, "rfssid" : 1, "siteid" : 1 }
  }
}
```

### rw-publisher `dat.json`
The idea here is to build upon the existing Beaker Browser [dat.json spec](https://beakerbrowser.com/docs/apis/manifest) as much as possible. In this way Publishers can use a [Dat Website](https://beakerbrowser.com/docs/how-beaker-works/peer-to-peer-websites) as the front page of their P2P presence while seeding their Author's archives in a structured, discoverable way.
```
{
  "type" : ["website", "rw-publisher"],
  "title" : "Radio Venceremos",
  "description" : "we shall overcome",
  "web_root" : "/",
  "fallback_page" : "/assets/404.html",
  "links" : {
    "publisher" : [
      { "type" : "rw-author", "href" : 'dat://abc123' },
      { "type" : "rw-studio", "href" : 'dat://def456' },
      { "type" : "rw-index", "href" : 'dat://ghi789' }
    ]
  }
}
```

## Development
Making this up as we go... create `index.html` from README.md, rebuild `web/`, and generate root `.datignore` file by concatenating all `.gitignore` files found in subdirectories:
```
$ chmod +x bin/radiowitness && \
    ./bin/radiowitness devstuff
```

## TCP Transport Example
publisher:
```
$ while true; do time ncat -l -p 6666 -c \
    "docker run -i --rm --name rw.pub --mount source=rw.pub,target=/archive rwp2p \
      publisher --capacity 604800 dat://1bf95043c31af876acff4f1c937eacce7de85f21178d7e63fb8b2828809b9c98";
    sleep 5; done;
```

author:
```
$ while true; do time ncat cool.pub.peer 6666 -c \
    "docker run $(./bin/rtl_devices.sh) -i --rm --name rw.author --mount source=rw.author,target=/archive rwp2p \
      author --capacity 604800 --radios 1 --mux 2 -s 1600000 -g 26 -f 851137500";
    sleep 5; done;
```

## Mirror
```
$ mc mb rwb2/rw-mirror
$ curl https://radiowitness.org/dat.json | docker run -d --name rw-mirror \
    --network rw-net -p 9004:9004 -e WSS=9004 -e OBJECTS=rw-objects:9000 -e BUCKET=rw-mirror \
    rwp2p mirror
```

## Http Tuning
```
$ while true; do \
    rm -rf /tmp/rw-*; echo 851287500 | \
      node lib/js/rw-peer/index.js search --period 5000 -d 0 -g 26 | \
        ./bin/radiowitness http -p 8080;
    sleep 5; \
  done;
```

## License
License Zero Reciprocal Public License 2.0.1
