# CorpSyncAudit — Reverse Engineering Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | CorpSyncAudit |
| Category | Reverse Engineering |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Final Flag | `HTB{d473_71m3_4nd_64ckd00r5}` |


## 1. Challenge Overview

`CorpSyncAudit` was a reverse engineering challenge based on a Windows executable and multiple audit log files.

The challenge story described a compliance audit tool that appeared to work normally, but had something extra hidden inside it. The logs looked clean, the audit passed, and the software behaved like a legitimate corporate synchronization tool. However, the story hinted that a contractor had inserted a hidden parsing path that was not meant to run during a normal audit.

The provided files were:

```text
CorpSyncAudit.exe
sync_20260412_175646.log
sync_20260412_180346.log
sync_20260412_190917.log
sync_20260412_192054.log
sync_20260412_192225.log
sync_20260412_192334.log
sync_20260412_192364.log
sync_20260413_140142.log
sync_20260413_142119.log
sync_20260413_143314.log
sync_20260413_143324.log
sync_20260413_154447.log
sync_20260413_154456.log
sync_20260413_154518.log
sync_20260413_164254.log
sync_20260413_173108.log
sync_20260413_175649.log
sync_20260413_175725.log
sync_20260413_175748.log
````

The challenge also provided a remote service:

```text
154.57.164.78:30964
```

The goal was to find the crafted log file, understand the hidden parser inside the binary, trigger the backdoor parsing path, and recover the flag.


## 2. Initial File Inspection

I started by identifying the executable type:

```bash
file CorpSyncAudit.exe
```

The file was a Windows PE executable.

Basic static inspection was useful:

```bash
strings -a CorpSyncAudit.exe
rabin2 -I CorpSyncAudit.exe
rabin2 -z CorpSyncAudit.exe
```

Since this was a Windows binary, tools such as Ghidra, IDA Free, x64dbg, Detect It Easy, and PEStudio are also useful for deeper inspection.

The binary appeared to be a normal corporate audit or synchronization tool. It accepted log files, parsed their fields, and produced clean audit results.

However, the challenge description strongly hinted that one log file was crafted to activate a hidden parsing path.


## 3. Reviewing the Logs

There were many log files, so the first step was to compare them and look for anomalies.

Useful commands:

```bash
ls -lh *.log
head -n 20 *.log
grep -RniE "region|date|time|sync|audit|error|warning|failed|unknown" *.log
```

Most logs followed a normal structure.

The suspicious file was:

```text
sync_20260412_192364.log
```

This file stood out because it contained unusual combinations of date/time fields and region values compared to the other logs.

The story mentioned:

```text
a parsing path that was never meant to run during a normal audit
and a log file crafted to wake it up
```

That matched the idea that only one crafted log file would trigger the hidden logic.


## 4. Identifying the Suspicious Log

After comparing all log files, `sync_20260412_192364.log` was the important one.

The other logs behaved like normal audit records. They were likely decoys used to hide the crafted log in a realistic dataset.

The suspicious log contained values that were valid enough to pass normal parsing, but unusual enough to activate a second-stage parser.

This matched the backdoor design:

```text
normal logs       -> normal audit path
crafted log       -> hidden parsing path
hidden parser     -> reconstruct payload
payload           -> contains encoded flag marker
```


## 5. Static Analysis of the Binary

Inside `CorpSyncAudit.exe`, I focused on the log parsing functions.

The interesting code path processed fields such as:

```text
Region
Date
Time
Audit status
Sync metadata
```

At first, the parsing looked like a normal compliance audit routine.

However, one branch used specific values from the log file to build hidden data.

The important recovered behavior was:

```text
Region values     -> hash table index
Date/time fields  -> packed bytes
Packed bytes      -> XOR material
XOR result        -> Windows x64 shellcode
Shellcode string  -> base64 marker
Base64 marker     -> final flag
```

So the binary was not simply checking a password.

It was reconstructing a hidden payload from a crafted audit log.


## 6. Understanding the Hidden Parser

The hidden parser used the log contents as input material.

The `Region=` fields were not only treated as normal text. They were used to select positions through a hash-table style lookup.

The date and time fields were then packed into bytes.

Conceptually, the hidden logic worked like this:

```c
region_index = hash(region_value);
packed_byte  = pack(date, time, region_index);
payload[i]   = packed_byte ^ key[i];
```

This created a decoded byte stream.

That byte stream was not plain text at first. It was Windows x64 shellcode.

The important realization was that the crafted log was acting like a carrier for hidden bytes.

The log looked like audit data, but the parser interpreted selected fields as encoded payload material.


## 7. Payload Reconstruction

After identifying the hidden parser, I reconstructed the payload generation logic.

The high-level process was:

```text
1. Read crafted log file
2. Extract selected Region values
3. Convert Region values into hash table indexes
4. Extract date/time fields
5. Pack selected values into bytes
6. XOR the packed bytes with the recovered key
7. Recover the embedded Windows x64 shellcode
```

This produced shellcode-like bytes.

Instead of executing the shellcode, I analyzed it statically.

This is safer and also enough for a CTF reverse engineering challenge.


## 8. Analyzing the Shellcode

Once the XOR decoding produced the shellcode, I searched the decoded output for printable strings.

Useful commands:

```bash
strings -a decoded_shellcode.bin
xxd -g1 -c16 decoded_shellcode.bin
grep -aE "HTB|SFRC|flag|base64" decoded_shellcode.bin
```

The shellcode contained a base64-looking marker:

```text
SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==
```

The prefix `SFRC` is interesting because base64-decoding `SFRC` gives:

```text
HTB
```

So this was almost certainly the final encoded flag.


## 9. Decoding the Base64 Marker

The recovered marker was:

```text
SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==
```

I decoded it with:

```bash
echo 'SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==' | base64 -d
```

Output:

```text
HTB{d473_71m3_4nd_64ckd00r5}
```

This confirmed the final flag.


## 10. Remote Service Validation

The challenge also provided a spawned remote service:

```text
154.57.164.78:30964
```

Since the service was based on the audit parser, the correct trigger file was the crafted log:

```text
sync_20260412_192364.log
```

The expected interaction was to provide the suspicious log file to the service.

The important point was that not every log triggers the hidden path. The crafted file `sync_20260412_192364.log` wakes up the hidden parser and leads to the same decoded payload.


## 11. Why the Other Logs Were Decoys

The challenge included many log files to make the investigation realistic.

Most of them were normal audit logs.

Their purpose was to hide the crafted file among legitimate-looking records.

This matched the challenge story:

```text
The logs came back clean,
the audits passed,
and nobody looked twice at a report that told them what they wanted to hear.
```

Technically, this means the binary had two behaviors:

```text
Normal log  -> ordinary compliance result
Crafted log -> hidden backdoor parser path
```

The hidden path was only reachable with the right log structure.


## 12. Key Observations

The important observations were:

```text
1. The challenge used a Windows PE executable.
2. The executable parsed audit log files.
3. Most logs were normal decoys.
4. sync_20260412_192364.log contained the crafted trigger data.
5. Region values influenced a hash-table lookup.
6. Date/time fields were packed into bytes.
7. The packed bytes were XOR-decoded into Windows x64 shellcode.
8. The shellcode contained a base64 marker.
9. Base64 decoding revealed the flag.
```


## 13. Solve Chain

The full solve chain was:

```text
Inspect CorpSyncAudit.exe
        ↓
Identify Windows PE executable
        ↓
Compare all sync logs
        ↓
Find suspicious crafted log
        ↓
Reverse log parsing routine
        ↓
Follow hidden parser path
        ↓
Use Region values as hash indexes
        ↓
Pack date/time fields into bytes
        ↓
XOR-decode hidden payload
        ↓
Recover Windows x64 shellcode
        ↓
Extract base64 marker
        ↓
Decode marker
        ↓
Recover final flag
```

Suspicious log:

```text
sync_20260412_192364.log
```

Recovered base64 marker:

```text
SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==
```

Final flag:

```text
HTB{d473_71m3_4nd_64ckd00r5}
```


## 14. Solver Snippet

A simplified representation of the final decoding step is:

```python
import base64

marker = "SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ=="

flag = base64.b64decode(marker).decode()
print(flag)
```

Output:

```text
HTB{d473_71m3_4nd_64ckd00r5}
```

A more complete solver would include:

```text
log parsing
Region hash lookup
date/time packing
XOR decoding
shellcode extraction
base64 decoding
```

But once the marker was recovered from the decoded shellcode, the final step was a simple base64 decode.


## 15. Lessons Learned

`CorpSyncAudit` was a good reverse engineering challenge because it combined binary analysis with crafted input analysis.

The important lesson was that not all malicious logic is triggered by command-line arguments or obvious strings. Sometimes the payload is hidden inside data files that look normal.

In this challenge, the log file was not just input. It was a carrier for encoded shellcode.

The main lesson was:

```text
When a binary processes many similar input files, compare the files and look for the one that changes the execution path.
```

Another important point was that the shellcode did not need to be executed. Static extraction was enough.

The safe approach was:

```text
decode payload
extract strings
identify base64 marker
decode marker
recover flag
```


## 16. Final Summary

`CorpSyncAudit` was a Windows PE reverse engineering challenge involving a hidden parser inside an audit synchronization tool.

By comparing the provided logs, I identified `sync_20260412_192364.log` as the crafted trigger file. Reversing the binary showed that Region values and date/time fields were used to reconstruct an XOR-encoded Windows x64 shellcode payload. The decoded shellcode contained a base64 marker:

```text
SFRCe2Q0NzNfNzFtM180bmRfNjRja2QwMHI1fQ==
```

Decoding the marker revealed the final flag:

```text
HTB{d473_71m3_4nd_64ckd00r5}
```

