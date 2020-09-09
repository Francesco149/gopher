so it seems like extracting fitgirl repacks on linux always gets stuck on the `version` file for
some reason

let's try to extract and decompile manually with http://innounp.sourceforge.net/

```
wine innounp.exe -x -m -dextracted ./setup.exe
wine rops-3.0.53.935-disasm/disasm.exe extracted/embedded/CompiledCode.bin disasm
```

now let's take a look at disasm

hmmm the steps for the version file don't look any different, so what could be holding it up?
it runs x5 and creates version and version_ and cmd appears to be stuck in a read loop

this is the command it runs

```
x5 version version.x5 version_&&del version.x5&&move version_ version
```

version.x5 disappears so clearly we are at least getting to the move

if i manually kill the wineconsole process it moves to the next file and gets stuck again

```
$ ps aux | grep x5
loli     20337  0.1  0.0 2683220 22384 ?       Ss   06:09   0:00 C:\windows\system32\cmd.exe /C C:\users\loli\Temp\is-101RB.tmp\x5.exe battleplan\scene_mp_2p_02.snapshot.bytes battleplan\scene_mp_2p_02.snapshot.bytes.x5 battleplan\scene_mp_2p_02.snapshot.bytes_&&del battleplan\scene_mp_2p_02.snapshot.bytes.x5&&move battleplan\scene_mp_2p_02.snapshot.bytes_ battleplan\scene_mp_2p_02.snapshot.bytes
```

i feel like this has to do with moving the file, because other ones that just removed the unpatched
files work fine

what happens if I try to move a file over another manually like the installer does?

```
$ touch test1 test2
$ wine cmd /c 'move test1 test2'
Overwrite (...)\test2? (Yes|No)n
$ echo "move test1 test2" > test.bat
$ wine cmd
Microsoft Windows 6.1.7601

(...)>test.bat

(...)>move test1 test2
Overwrite (...)\test2? (Yes|No)n
```

after googling around, I found (this)[https://ss64.com/nt/move.html]

```
Under Windows 2000 and above, the default action is to prompt on overwrite unless the command is
being executed from within a batch script. 
```

so wine isn't emulating this correctly.

yep, bingo. I'm gonna report this to wine and maybe fix it myself.

while digging around wine code I found this:

```
     /* /-Y has the highest priority, then /Y and finally the COPYCMD env. variable */
      if (wcsstr (quals, parmNoY))
        force = FALSE;
      else if (wcsstr (quals, parmY))
        force = TRUE;
      else {
        static const WCHAR copyCmdW[] = {'C','O','P','Y','C','M','D','\0'};
        len = GetEnvironmentVariableW(copyCmdW, copycmd, ARRAY_SIZE(copycmd));
        force = (len && len < ARRAY_SIZE(copycmd) && !lstrcmpiW(copycmd, parmY));
      }

      /* Prompt if overwriting */
      if (!force) {
```

so we can work around it by starting the setup with a batch file that exports COPYCMD to /Y right?
not so fast, this appears to be yet another bug in wine

```
$ export COPYCMD=/Y
$ echo test1 > test1
$ echo test2 > test2
$ wine cmd /c 'move test1 test2'
File already exists.

$ cat test2
test2
```

# the solution

we're just gonna have to mv the file manually from linux and kill the cmd process. here's a script
that automates it specifically for fitgirl installers. run it when it gets stuck

NOTE: this is a really shit and fragile solution, ideally you want to patch wine's cmd to not ask
for confirmation on move. this is already reported on their bug tracker since 2019

```sh
#!/bin/sh

while :; do
  sleep 0.1
  pid="$(ps aux | awk '/cmd[.]exe \/C .*[ &|]move [^&|]+ [^&|]+/ { print $2 }')" || continue
  [ "$pid" ] || continue
  # if x5 is still running, wait for it to terminate
  x5pid="$(ps aux | grep -v 'cmd[.]exe' | awk '/x5[.]exe/ { print $2 }')" && [ "$x5pid" ] &&
    echo "waiting for x5 ($x5pid)..." &&
    tail "--pid=$x5pid" -f /dev/null
  if [ ! "$pfx" ]; then
    pfx="$(tr '\0' '\n' < "/proc/$pid/environ" | awk -F= '/^WINEPREFIX/ { print $2 }')" || continue
    cd "$pfx/drive_c/Games/"* || exit # couldn't find a good way to extract the windows work dir
  fi
  movecmd="$(tr '\0' ' ' < "/proc/$pid/cmdline" | grep -o "move [^&|]\+ [^&|]\+" |
             sed 's/^move/mv -v/g;s|\\|/|g;s|C:|$pfx|g')"
  [ "$(sudo cat "/proc/$pid/stack" | sed 1q | awk -F'[ +]' '{ print $2 }' )" != "pipe_read" ] &&
    continue
  [ "$movecmd" ] || (echo "failed to extract move cmd" && continue)
  $movecmd || continue
  kill -9 "$pid"
  tail "--pid=$pid" -f /dev/null # wait for process to terminate
done
```
