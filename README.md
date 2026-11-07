# Investigating a Forensic Image of Windows RAM with Redline and Volatility

A hands-on memory forensics lab analyzing a Windows RAM dump with two separate tools, Redline and the Volatility framework, tracing a suspicious network connection on a known exploitation-tool port back through process ancestry to confirm an active, live system compromise.

## Scenario
A firm suspected a data breach and needed to determine whether malicious processes were actually running on its systems. Disk forensics only shows what's stored; memory forensics shows what was actually running, connected, and executing at the moment a RAM image was captured. This lab analyzed `Windows_RAM.mem` with both Redline and Volatility to confirm the same findings independently.

## What I did

### Part 1: Redline
- Set up a new analysis session pointed at `Windows_RAM.mem`; analysis took roughly an hour to complete
- Reviewed the full **Processes** list as a starting point (not conclusive on its own)
- Opened **Ports** and found `rundll32.exe` connected to a remote IP on **port 4444** — a well-known default port associated with Metasploit's Meterpreter payload
- Found a second instance of the same process (different PID, 1896) with the **same remote connection in an ESTABLISHED/active state**
- Checked **Hierarchical Processes**: `rundll32.exe`'s parent was `spoolsv.exe` (Windows print spooler), and `rundll32.exe` had spawned `cmd.exe`, which in turn launched `notepad.exe` and `calc.exe`
- Reviewed **Timeline** to confirm the chronological sequence of these events

### Part 2: Volatility (independent confirmation)
- Used `imageinfo` and `kdbgscan` plugins to identify the correct memory profile, selecting `Win2008R2SP0x64`
- Ran `netscan` — independently reproduced the same finding: `rundll32.exe` with PIDs 1972 (closed) and 1896 (active), connected to the same suspect address
- Ran `pslist` and located `rundll32.exe` in the full process listing
- Ran `handles -p 1896 -t file` and `-t key` to list the specific files and registry keys that process instance had open — deeper behavioral evidence beyond just confirming the process existed
- Ran `pstree` and confirmed the exact process ancestry: `spoolsv.exe → rundll32.exe → rundll32.exe → cmd.exe → calc.exe`, matching Redline's hierarchical view precisely

## Key takeaways
- Memory forensics answers "what was actually running" at a specific moment — a different and complementary question to disk forensics, which only shows what was stored
- Recognizing a known port (4444, associated with Metasploit's default Meterpreter payload) turns a vague "this looks unusual" into a specific, documented indicator of compromise
- Process ancestry (parent/child relationships) tells a story a flat process list can't — no single process name here was individually alarming; the suspicious chain (a background service spawning a code-execution utility spawning a shell spawning unrelated apps) is what made it detectable
- Confirming findings independently with a second tool (Volatility via command line, after Redline's guided GUI) protects against a single tool's blind spots or presentation bias
- File and registry handle data for a specific PID goes beyond confirming a process exists — it starts to show what that process was actually doing
- Multiple PIDs for the same process name over time (one closed, one active) can indicate a compromise relaunched more than once, itself worth further investigation

## Tools
Redline, Volatility 2.6 (standalone)

---
*Completed in an isolated training lab environment using a provided Windows RAM forensic image.*
