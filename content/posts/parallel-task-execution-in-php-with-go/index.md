---
title: "Parallel Task Execution in PHP... with Go"
date: 2026-03-07
description: "PHP is a blocking language, and that is fine. Instead of fighting it with complex libraries, Gyproc delegates parallelism to Go and streams each command's output back as tagged JSON events."
tags: ['PHP', 'Go']
---

PHP is a blocking language. When you loop over a batch of images and resize each one sequentially, you wait for every conversion to finish before starting the next. For small batches that is acceptable. For anything larger, it becomes a bottleneck.

The natural instinct is to reach for a parallelism solution inside PHP. But I have come to prefer a different approach: accept that PHP is blocking, and delegate the responsibility of parallelism to a language that does it beautifully.

## The alternatives I considered

The first option that comes to mind is appending `&` to a shell command to push it to the background:

```php
shell_exec('convert input.jpg -resize 800x600 output.jpg &');
```

This works, but you immediately lose track of the process. You have no way to know when it finishes, whether it succeeded, or what it produced as output. For a fire-and-forget task that might be enough. For anything where progress or failure matters, it is not.

The next option is `pcntl`, PHP's process control extension. It can fork processes and run work in parallel, but it is Linux-only. That is a hard constraint that rules it out for development on macOS or any Windows environment.

`spatie/async` is a well-made library that wraps `pcntl` with a cleaner API. But it inherits the same Linux-only limitation, and it adds an external dependency on top. My general preference is to avoid growing my dependency surface for a problem I can solve differently. Unless parallel multitasking is at the heart of my system, in which case PHP would probably not have been the right choice to begin with.

None of these options give you a real-time stream of what is happening inside each process while it runs. You get a result when everything is done, not a window into the work in progress.

## Assuming the blocking nature of PHP

Go handles concurrency natively through [goroutines](https://gobyexample.com/goroutines). Spawning thousands of concurrent tasks is idiomatic and lightweight. Rather than trying to graft that capability onto PHP, why not write a small Go tool that does the parallel coordination, and have PHP consume its output?

That is [Gyproc](https://github.com/maximegosselin/gyproc), which I wrote for exactly this purpose.

Gyproc takes a list of commands, runs them in parallel using goroutines, and multiplexes their output into a single newline-delimited JSON stream. It is around 200 lines of Go. You can read the entire source in a few minutes. Building it requires nothing beyond the Go compiler:

```bash
git clone https://github.com/maximegosselin/gyproc
cd gyproc
go build .
```

It runs on Linux, macOS, and Windows. Any client that can read a stream of JSON lines can consume it. PHP is a natural fit.

The `--limit` flag caps the number of commands running simultaneously, which is useful when you want to avoid overloading the system with too many concurrent processes.

## The key advantage: `seq`

When multiple processes run in parallel, their output arrives interleaved. Without a way to attribute each line to its source, the stream is noise.

Gyproc solves this by tagging every event with a `seq` identifier, an integer assigned to each command in the order it was received. Every line of output, every status change, every failure is tied to a specific `seq`. The client can group events by `seq` and track each command independently, regardless of the order in which output arrives. None of the alternatives discussed earlier provide this. You either get no output at all, or a single result at the end.

The event types are straightforward:

| Event  | When                                      |
|--------|-------------------------------------------|
| `ack`  | Command received and queued               |
| `run`  | Command started                           |
| `out`  | Process wrote a line to stdout or stderr  |
| `exit` | Command finished with an exit code        |
| `fail` | Command could not be started              |

## A PHP client implementation

The implementation is intentionally minimal. It delegates all parallelism and process management to Gyproc, and exposes a generator that produces one raw JSON line per iteration. Parsing and handling those events is left to the caller.

```php
<?php

declare(strict_types=1);

use Generator;

class Gyproc
{
    public function run(array $commands, int $limit = 0): Generator
    {
        $limit = max(0, $limit);
        $tempFile = tempnam(sys_get_temp_dir(), 'gyproc-');
        file_put_contents($tempFile, implode(PHP_EOL, $commands));

        $handle = popen(sprintf('gyproc --file=%s --limit=%d 2>&1', $tempFile, $limit), 'r');
        while ($line = fgets($handle)) {
            yield $line;
        }
        pclose($handle);
        unlink($tempFile);
    }
}
```

## A concrete example: resizing images in parallel

ImageMagick's `convert` command supports a `-monitor` flag that writes progress to stderr as it works. Each line follows this format:

```
resize image[photo.jpg]: 600 of 800, 75% complete
```

Gyproc captures both stdout and stderr, so each `-monitor` line becomes an `out` event tagged with its `seq`. Each image can be tracked independently as the batch progresses.

Here is a complete example:

```php
$images = glob('/path/to/images/*.jpg');

// Map each image to a convert command
// seq corresponds to the position in this array (1-indexed),
// so seq 1 = $images[0], seq 2 = $images[1], and so on
$commands = array_map(fn($path) => sprintf(
    'convert %s -monitor -resize 800x600 %s',
    escapeshellarg($path),
    escapeshellarg(str_replace('.jpg', '-resized.jpg', $path))
), $images);

// Index images by seq for easy lookup
$index = array_combine(range(1, count($images)), $images);

$gyproc = new Gyproc();

// Run with a concurrency limit of 2 to avoid overloading the system
foreach ($gyproc->run($commands, limit: 2) as $line) {
    $event = json_decode(trim($line), true);

    match ($event['event']) {
        'out'  => printf("[%s] %s\n", basename($index[$event['seq']]), $event['message']),
        'exit' => $event['code'] === 0
                    ? printf("[%s] done\n", basename($index[$event['seq']]))
                    : printf("[%s] failed with exit code %d\n", basename($index[$event['seq']]), $event['code']),
        'fail' => printf("[%s] could not start: %s\n", basename($index[$event['seq']]), $event['message']),
        default => null,
    };
}
```

Gyproc's raw JSON stream for 3 images with `--limit 2`:

```json
{"seq":1,"event":"ack","command":"convert 'photo-01.jpg' -monitor -resize 800x600 'photo-01-resized.jpg'","time":"2026-03-05T10:00:00.000000000Z"}
{"seq":2,"event":"ack","command":"convert 'photo-02.jpg' -monitor -resize 800x600 'photo-02-resized.jpg'","time":"2026-03-05T10:00:00.000000000Z"}
{"seq":3,"event":"ack","command":"convert 'photo-03.jpg' -monitor -resize 800x600 'photo-03-resized.jpg'","time":"2026-03-05T10:00:00.000000000Z"}
{"seq":1,"event":"run","pid":5678,"time":"2026-03-05T10:00:00.010000000Z"}
{"seq":2,"event":"run","pid":5679,"time":"2026-03-05T10:00:00.010000000Z"}
{"seq":1,"event":"out","pid":5678,"message":"resize image[photo-01.jpg]: 400 of 800, 50% complete","time":"2026-03-05T10:00:00.500000000Z"}
{"seq":2,"event":"out","pid":5679,"message":"resize image[photo-02.jpg]: 400 of 800, 50% complete","time":"2026-03-05T10:00:00.600000000Z"}
{"seq":2,"event":"out","pid":5679,"message":"resize image[photo-02.jpg]: 800 of 800, 100% complete","time":"2026-03-05T10:00:00.900000000Z"}
{"seq":2,"event":"out","pid":5679,"message":"save image[photo-02-resized.jpg]: 600 of 600, 100% complete","time":"2026-03-05T10:00:00.950000000Z"}
{"seq":2,"event":"exit","pid":5679,"code":0,"time":"2026-03-05T10:00:01.000000000Z"}
{"seq":3,"event":"run","pid":5680,"time":"2026-03-05T10:00:01.010000000Z"}
{"seq":1,"event":"out","pid":5678,"message":"resize image[photo-01.jpg]: 800 of 800, 100% complete","time":"2026-03-05T10:00:01.200000000Z"}
{"seq":1,"event":"out","pid":5678,"message":"save image[photo-01-resized.jpg]: 600 of 600, 100% complete","time":"2026-03-05T10:00:01.250000000Z"}
{"seq":1,"event":"exit","pid":5678,"code":0,"time":"2026-03-05T10:00:01.300000000Z"}
{"seq":3,"event":"out","pid":5680,"message":"resize image[photo-03.jpg]: 400 of 800, 50% complete","time":"2026-03-05T10:00:01.500000000Z"}
{"seq":3,"event":"out","pid":5680,"message":"resize image[photo-03.jpg]: 800 of 800, 100% complete","time":"2026-03-05T10:00:01.800000000Z"}
{"seq":3,"event":"out","pid":5680,"message":"save image[photo-03-resized.jpg]: 600 of 600, 100% complete","time":"2026-03-05T10:00:01.850000000Z"}
{"seq":3,"event":"exit","pid":5680,"code":0,"time":"2026-03-05T10:00:01.900000000Z"}
```

After processing by the PHP client:

```
[photo-01.jpg] resize image[photo-01.jpg]: 400 of 800, 50% complete
[photo-02.jpg] resize image[photo-02.jpg]: 400 of 800, 50% complete
[photo-02.jpg] resize image[photo-02.jpg]: 800 of 800, 100% complete
[photo-02.jpg] save image[photo-02-resized.jpg]: 600 of 600, 100% complete
[photo-02.jpg] done
[photo-01.jpg] resize image[photo-01.jpg]: 800 of 800, 100% complete
[photo-01.jpg] save image[photo-01-resized.jpg]: 600 of 600, 100% complete
[photo-01.jpg] done
[photo-03.jpg] resize image[photo-03.jpg]: 400 of 800, 50% complete
[photo-03.jpg] resize image[photo-03.jpg]: 800 of 800, 100% complete
[photo-03.jpg] save image[photo-03-resized.jpg]: 600 of 600, 100% complete
[photo-03.jpg] done
```

Each line is attributed to its source. Failures are surfaced immediately.

## Conclusion

Gyproc is the tool I reach for when I need parallel task execution in PHP. In my own projects, I use it to rebuild projections in event sourcing applications built with [Backslash](https://backslashphp.github.io/docs/maintenance/rebuilding-projections/), an open-source PHP event sourcing library. Each projector runs as an independent PHP process and streams its progress back in real time. PHP stays blocking. Go handles the concurrency. The two collaborate over a simple JSON stream.
