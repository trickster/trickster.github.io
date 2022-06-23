+++
title = "Scraping with Scala"
description = "Using Akka Streams"
type = "post"
date = 2020-02-29
in_search_index = true
[taxonomies]
tags = ["Scala", "Tools"]
+++

**Step 1**: Download ammonite

```bash
# Download Ammonite
$ curl -L https://github.com/lihaoyi/ammonite/releases/download/2.0.4/2.12-2.0.4-boostrap > amm212 && chmod +x amm212
```

**Step 2**. Start ammonite

```sh
amm212 --class-based
```

**Step 3**. We need `akka-streams`

```scala
import $ivy.`com.typesafe.akka::akka-stream:2.6.3`
// standard imports
import akka.stream._
import akka.stream.scaladsl._
import akka.{ Done, NotUsed }
import akka.actor.ActorSystem
import akka.util.ByteString
import scala.concurrent._
import scala.concurrent.duration._

// Json parsing
import ujson._

// Handle files
import java.nio.file.Paths
implicit val system = ActorSystem("QuickStart")

val ids = scala.io.Source.fromFile("source.txt") // we need to scrape these urls or some ids for an endpoint

def parser(i: Int): (Int, String) = {
      val url = s"ENDPOINT/{i}"
      val urlResponse = Try(ujson.read(requests.get(url).data.toString))
      val result = urlResponse match {
        case Success(res) => "PARSED_RESULT"
        case Failure(e) => "null"
      }
      return (i, result)
    }

```

**Step 4**. We create file sink to store the data (you can print to console as well)

```scala
def lineSink(filename: String): Sink[(Int, String), Future[IOResult]] =
    Flow[(Int, String)].map(s => ByteString(s._1 + "," + s._2 + "\n")).toMat(FileIO.toPath(Paths.get(filename)))(Keep.right)
```

**Step 5**. Introduce throttling behavior because some websites disallow too many requests. Here, we make 25 requests in 10 seconds (5 requests in 2 seconds)

```scala
val resultFile = source.map(_.toInt).throttle(25, 10.second).map(parser).runWith(lineSink("output.txt"))
```

### All done from console with Ammonite shell
