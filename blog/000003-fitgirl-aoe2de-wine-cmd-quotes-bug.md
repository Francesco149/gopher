I found yet another bug in wine's cmd.

I've been testing my [protonfit](https://github.com/Francesco149/protonfit) script on various
fitgirl repacks, and Age of Empires II Definitive Edition appears to error out.

`it is not found any file specified for ISArcExtract`

innosetup disasm references this func as 259

    Proc [259]: External Decl: dll:files:ISDone.dll\00ISArcExtract\00\03\01\00\01\00\00\00\00\00\00\00\00\00\00 

call example:

     [5168] PUSHTYPE 17(String) // 22
     [5173] PUSHTYPE 16(Unknown 28) // 23
     [5178] PUSHTYPE 16(Unknown 28) // 24
     [5183] ASSIGN Base[24], ['{src}\fg-03.bin']
     [5213] PUSHVAR Base[23] // 25
     [5219] CALL 40
     [5224] POP // 24
     [5225] POP // 23
     [5226] ASSIGN Base[22], Base[23]
     [5237] POP // 22
     [5238] PUSHTYPE 6(Double) // 23
     [5243] ASSIGN Base[23], [0]
     [5258] PUSHTYPE 9(U32) // 24
     [5263] ASSIGN Base[24], [0]
     [5278] PUSHVAR Base[14] // 25
     [5284] CALL 259 // this is the ISArcExtract call

it appears to error shortly after the message `Compressing files... DO NOT PANIC IF IT LOOKS STUCK`

here are the two instances of it:

     [8418] ASSIGN Base[16], ['Compressing files... DO NOT PANIC IF IT LOOKS STUCK']
     [8484] PUSHTYPE 17(String) // 17
     [8489] PUSHTYPE 16(Unknown 28) // 18
     [8494] PUSHTYPE 16(Unknown 28) // 19
     [8499] ASSIGN Base[19], ['{app}\']
     [8520] PUSHVAR Base[18] // 20
     [8526] CALL 40
     [8531] POP // 19
     [8532] POP // 18
     [8533] ASSIGN Base[17], Base[18]
     [8544] POP // 17
     [8545] PUSHTYPE 17(String) // 18
     [8550] PUSHTYPE 16(Unknown 28) // 19
     [8555] PUSHTYPE 16(Unknown 28) // 20
     [8560] ASSIGN Base[20], ['']
     [8575] PUSHVAR Base[19] // 21
     [8581] CALL 40
     [8586] POP // 20
     [8587] POP // 19
     [8588] ASSIGN Base[18], Base[19]
     [8599] POP // 18
     [8600] PUSHTYPE 17(String) // 19
     [8605] PUSHTYPE 16(Unknown 28) // 20
     [8610] PUSHTYPE 16(Unknown 28) // 21
     [8615] ASSIGN Base[21], ['{tmp}\work\build01.bat']


     [16600] ASSIGN Base[16], ['Compressing files... DO NOT PANIC IF IT LOOKS STUCK']
     [16666] PUSHTYPE 17(String) // 17
     [16671] PUSHTYPE 16(Unknown 28) // 18
     [16676] PUSHTYPE 16(Unknown 28) // 19
     [16681] ASSIGN Base[19], ['{app}\wwise\']
     [16708] PUSHVAR Base[18] // 20
     [16714] CALL 40
     [16719] POP // 19
     [16720] POP // 18
     [16721] ASSIGN Base[17], Base[18]
     [16732] POP // 17
     [16733] PUSHTYPE 17(String) // 18
     [16738] PUSHTYPE 16(Unknown 28) // 19
     [16743] PUSHTYPE 16(Unknown 28) // 20
     [16748] ASSIGN Base[20], ['']
     [16763] PUSHVAR Base[19] // 21
     [16769] CALL 40
     [16774] POP // 20
     [16775] POP // 19
     [16776] ASSIGN Base[18], Base[19]
     [16787] POP // 18
     [16788] PUSHTYPE 17(String) // 19
     [16793] PUSHTYPE 16(Unknown 28) // 20
     [16798] PUSHTYPE 16(Unknown 28) // 21
     [16803] ASSIGN Base[21], ['{tmp}\work\build03.bat']


if i check the temp directory the build03.bat contains this:

    "C:\users\steamuser\Temp\is-DLSPQ.tmp\work\run.exe" " " http://bit.ly/fitgirl-repacks-new-list "C:\users\steamuser\Temp\is-DLSPQ.tmp\work\fitgirl03.txt" 0
    rd /q /s 0
    rd /q /s Campaign.bnk_
    rd /q /s Music.bnk_
    del "C:\users\steamuser\Temp\is-DLSPQ.tmp\work\fitgirl03.txt"
    del "C:\users\steamuser\Temp\is-DLSPQ.tmp\work\build03.bat"

it must be failing before this since it never deletes itself? build01.bat is missing so that passes.
let's see the steps between build01 and build03

well, the very next thing is calling ISArcExtract on 01.fgpack

i noticed that fitgirl01.txt contains this:

    $ cat fitgirl01.txt
    cmd /c "mkdir "_06.fgpack"&&move "06.fgpack" "_06.fgpack"&&cd "_06.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\06.fgpack" 06.fgpack&&cd ..&&rd /q /s "_06.fgpack""
    cmd /c "mkdir "_03.fgpack"&&move "03.fgpack" "_03.fgpack"&&cd "_03.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\03.fgpack" 03.fgpack&&cd ..&&rd /q /s "_03.fgpack""
    cmd /c "mkdir "_07.fgpack"&&move "07.fgpack" "_07.fgpack"&&cd "_07.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\07.fgpack" 07.fgpack&&cd ..&&rd /q /s "_07.fgpack""
    cmd /c "mkdir "_05.fgpack"&&move "05.fgpack" "_05.fgpack"&&cd "_05.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\05.fgpack" 05.fgpack&&cd ..&&rd /q /s "_05.fgpack""
    cmd /c "mkdir "_02.fgpack"&&move "02.fgpack" "_02.fgpack"&&cd "_02.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\02.fgpack" 02.fgpack&&cd ..&&rd /q /s "_02.fgpack""
    cmd /c "mkdir "_01.fgpack"&&move "01.fgpack" "_01.fgpack"&&cd "_01.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\01.fgpack" 01.fgpack&&cd ..&&rd /q /s "_01.fgpack""
    cmd /c "mkdir "_04.fgpack"&&move "04.fgpack" "_04.fgpack"&&cd "_04.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\04.fgpack" 04.fgpack&&cd ..&&rd /q /s "_04.fgpack""
    cmd /c "mkdir "_08.fgpack"&&move "08.fgpack" "_08.fgpack"&&cd "_08.fgpack"&&"C:\users\steamuser\Temp\is-VPS38.tmp\work\fgpack2.exe" -r -o"..\08.fgpack" 08.fgpack&&cd ..&&rd /q /s "_08.fgpack""

this is supposed to create the fgpack files in `C:\Games\...` but for some reason they are not there

and clearly this is the step that fails because it's supposed to remove the underscore fgpack
directories when it's done yet they are still there. so it means this ran, but failed somewhere
in the middle

the quoting looks really weird in these commands, it's probably failing because of that

we can debug cmd by running `PROTON_LOG=1 WINEDEBUG=+cmd SteamGameId=0 protonfit` which will log
to `~/steam-0.log`

here's the relevant part of the log

    023c:trace:cmd:WCMD_move Processing file 'L"06.fgpack"'
    023c:trace:cmd:WCMD_move Source 'L"C:\\Games\\Age of Empires II - Definitive Edition\\06.fgpack"'
    023c:trace:cmd:WCMD_move Dest   'L"C:\\Games\\Age of Empires II - Definitive Edition\\_06.fgpack\\06.fgpack"'
    023c:trace:cmd:WCMD_process_commands Executing command: 'L"cd \"_06.fgpack\"&&\"C:\\users\\steamuser\\Temp\\is-THL8U.tmp\\work\\fgpack2.exe\" -r -o\"..\\06.fgpack\" 06.fgpack"'
    023c:trace:cmd:WCMD_execute command on entry:L"cd \"_06.fgpack\"&&\"C:\\users\\steamuser\\Temp\\is-THL8U.tmp\\work\\fgpack2.exe\" -r -o\"..\\06.fgpack\" 06.fgpack" (0031B2D0)
    023c:trace:cmd:WCMD_execute Command: 'L"cd \"_06.fgpack\"&&\"C:\\users\\steamuser\\Temp\\is-THL8U.tmp\\work\\fgpack2.exe\" -r -o\"..\\06.fgpack\" 06.fgpack"'
    023c:trace:cmd:WCMD_execute param1: L"_06.fgpack", param2: L"&&\"C:\\users\\steamuser\\Temp\\is-THL8U.tmp\\work\\fgpack2.exe\""
    023c:trace:cmd:WCMD_setshow_default Request change to directory 'L"\"_06.fgpack\"&&\"C:\\users\\steamuser\\Temp\\is-THL8U.tmp\\work\\fgpack2.exe\" -r -o\"..\\06.fgpack\" 06.fgpack"'

clearly a bug with wine cmd parsing, it tries to use the whole command as the directory

I've narrowed this bug down to this minimal example:

    cmd /c "cd "windows"&&"system32\notepad.exe""

on real windows 10 cmd, this runs notepad just fine. on wine, it says "path not found" because
it tries to interpret the entire thing as a path

here's the full output with `WINEDEBUG=+cmd`

    C:\>cmd /c "cd "windows"&&"system32\notepad.exe""
    00b8:trace:cmd:WCMD_DumpCommands Parsed line:
    00b8:trace:cmd:WCMD_DumpCommands 00B91598 0 00 00000000 L"cmd /c \"cd \"windows\"&&\"system32\\notepad.exe\"\"" Redir:L""
    00b8:trace:cmd:WCMD_process_commands Executing command: 'L"cmd /c \"cd \"windows\"&&\"system32\\notepad.exe\"\""'
    00b8:trace:cmd:WCMD_execute command on entry:L"cmd /c \"cd \"windows\"&&\"system32\\notepad.exe\"\"" (0077B2E8)
    00b8:trace:cmd:WCMD_execute Command: 'L"cmd /c \"cd \"windows\"&&\"system32\\notepad.exe\"\""'
    00b8:trace:cmd:WCMD_execute param1: L"cd ", param2: L"windows\"&&\"system32\\notepad.exe\"\""
    00b8:trace:cmd:WCMD_run_program Running 'L"cmd /c \"cd \"windows\"&&\"system32\\notepad.exe\"\""' (0)
    00b8:trace:cmd:WCMD_run_program Searching in 'L".;C:\\windows\\system32;C:\\windows;C:\\windows\\system32\\wbem;C:\\windows\\system32\\WindowsPowershell\\v1.0"' for 'L"cmd"'
    00b8:trace:cmd:WCMD_run_program Found as L"C:\\windows\\system32\\cmd.EXE"
    00c0:trace:cmd:wmain Full commandline 'L"cmd /c \"cd \"windows\"&&\"system32\\notepad.exe\"\""'
    00c0:trace:cmd:wmain /c command line: 'L"\"cd \"windows\"&&\"system32\\notepad.exe\"\""'
    00c0:trace:cmd:wmain Set L"=C:" to L"C:\\"
    00c0:trace:cmd:WCMD_DumpCommands Parsed line:
    00c0:trace:cmd:WCMD_DumpCommands 00111D48 0 00 00000000 L"cd \"windows\"&&\"system32\\notepad.exe\"" Redir:L""
    00c0:trace:cmd:WCMD_process_commands Executing command: 'L"cd \"windows\"&&\"system32\\notepad.exe\""'
    00c0:trace:cmd:WCMD_execute command on entry:L"cd \"windows\"&&\"system32\\notepad.exe\"" (0077B2E8)
    00c0:trace:cmd:WCMD_execute Command: 'L"cd \"windows\"&&\"system32\\notepad.exe\""'
    00c0:trace:cmd:WCMD_execute param1: L"windows", param2: L"&&\"system32\\notepad.exe\""
    00c0:trace:cmd:WCMD_setshow_default Request change to directory 'L"\"windows\"&&\"system32\\notepad.exe\""'
    00c0:trace:cmd:WCMD_setshow_default Looking for directory 'L"windows&&system32\\notepad.exe"'
    00c0:trace:cmd:WCMD_setshow_default Really changing to directory 'L"windows&&system32\\notepad.exe"'
    Path not found.

the cmd parsing logic for nested quotes is a nightmare:

    If /C or /K is specified, then the remainder of the command line after
    the switch is processed as a command line, where the following logic is
    used to process quote (") characters:

        1.  If all of the following conditions are met, then quote characters
            on the command line are preserved:

            - no /S switch
            - exactly two quote characters
            - no special characters between the two quote characters,
              where special is one of: &<>()@^|
            - there are one or more whitespace characters between the
              the two quote characters
            - the string between the two quote characters is the name
              of an executable file.

        2.  Otherwise, old behavior is to see if the first character is
            a quote character and if so, strip the leading character and
            remove the last quote character on the command line, preserving
            any text after the last quote character.

I managed to fix this by adding `&` as a character that ends quote-counting here:

    From 0750c27f4f7077b124077dd7c5cea32e51b81eeb Mon Sep 17 00:00:00 2001
    From: "Franc[e]sco" <lolisamurai@tfwno.gf>
    Date: Fri, 11 Sep 2020 05:18:31 +0200
    Subject: [PATCH] cmd: & should also end quote counting

    Signed-off-by: Francesco Noferi <lolisamurai@tfwno.gf>
    ---
     programs/cmd/wcmdmain.c | 3 ++-
     1 file changed, 2 insertions(+), 1 deletion(-)

    diff --git a/programs/cmd/wcmdmain.c b/programs/cmd/wcmdmain.c
    index 97cc607a647..47e43fcd675 100644
    --- a/programs/cmd/wcmdmain.c
    +++ b/programs/cmd/wcmdmain.c
    @@ -1792,7 +1792,8 @@ static BOOL WCMD_IsEndQuote(const WCHAR *quote, int quoteIndex)
     
             /* Quote counting ends at EOL, redirection, space or pipe if current quote is complete */
             else if(((quoteCount % 2) == 0)
    -            && ((quote[i] == '<') || (quote[i] == '>') || (quote[i] == '|') || (quote[i] == ' ')))
    +            && ((quote[i] == '<') || (quote[i] == '>') || (quote[i] == '|') || (quote[i] == ' ') ||
    +                (quote[i] == '&')))
             {
                 break;
             }
    -- 
    2.28.0

not sure this is the correct solution but it works for my purposes
