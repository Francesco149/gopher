so it seems like extracting fitgirl repacks on linux always gets stuck on the `version` file for
some reason

let's try to extract and decompile manually with http://innounp.sourceforge.net/

    wine innounp.exe -x -m -dextracted ./setup.exe
    wine rops-3.0.53.935-disasm/disasm.exe extracted/embedded/CompiledCode.bin disasm

now let's take a look at disasm

hmmm the steps for the version file don't look any different, so what could be holding it up?
it runs x5 and creates version and version_ and cmd appears to be stuck in a read loop

this is the command it runs

    x5 version version.x5 version_&&del version.x5&&move version_ version

version.x5 disappears so clearly we are at least getting to the move

if i manually kill the wineconsole process it moves to the next file and gets stuck again

i feel like this has to do with moving the file, because other ones that just removed the unpatched
files work fine

what happens if I try to move a file over another manually like the installer does?

    $ touch test1 test2
    $ wine cmd /c 'move test1 test2'
    Overwrite (...)\test2? (Yes|No)n
    $ echo "move test1 test2" > test.bat
    $ wine cmd
    Microsoft Windows 6.1.7601

    (...)>test.bat

    (...)>move test1 test2
    Overwrite (...)\test2? (Yes|No)n

after googling around, I found [this](https://ss64.com/nt/move.html)

    Under Windows 2000 and above, the default action is to prompt on overwrite unless the command is
    being executed from within a batch script. 

so wine isn't emulating this correctly.

yep, bingo. I'm gonna report this to wine and maybe fix it myself.

while digging around wine code I found this:

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

so we can work around it by starting the setup with a batch file that exports COPYCMD to /Y right?
not so fast, this appears to be yet another bug in wine

    $ export COPYCMD=/Y
    $ echo test1 > test1
    $ echo test2 > test2
    $ wine cmd /c 'move test1 test2'
    File already exists.

    $ cat test2
    test2

# the solution: fixing wine cmd

apply this patch to wine (save it as a .patch file and do `patch -p1 < file.patch` in wine source):

    From 480ac8841539ee3481edec56287ec15a5631bb58 Mon Sep 17 00:00:00 2001
    From: "Francesco Noferi" <lolisamurai@tfwno.gf>
    Date: Wed, 9 Sep 2020 21:33:43 +0200
    Subject: [PATCH] cmd: Fix non-interactive move overwrite behaviour

    Signed-off-by: Francesco Noferi <lolisamurai@tfwno.gf>
    ---
     programs/cmd/builtins.c | 31 ++++++++++++++++---------------
     1 file changed, 16 insertions(+), 15 deletions(-)

    diff --git a/programs/cmd/builtins.c b/programs/cmd/builtins.c
    index 70ccdde..c218186 100644
    --- a/programs/cmd/builtins.c
    +++ b/programs/cmd/builtins.c
    @@ -3026,18 +3026,17 @@ void WCMD_move (void)
         WINE_TRACE("Source '%s'\n", wine_dbgstr_w(src));
         WINE_TRACE("Dest   '%s'\n", wine_dbgstr_w(dest));
     
    -    /* If destination exists, prompt unless /Y supplied */
    +    /* If destination exists, prompt unless called from batch */
         if (GetFileAttributesW(dest) != INVALID_FILE_ATTRIBUTES) {
    -      BOOL force = FALSE;
    +      BOOL force = !interactive;
           WCHAR copycmd[MAXSTRING];
           DWORD len;
     
    -      /* /-Y has the highest priority, then /Y and finally the COPYCMD env. variable */
    -      if (wcsstr (quals, parmNoY))
    -        force = FALSE;
    -      else if (wcsstr (quals, parmY))
    -        force = TRUE;
    -      else {
    +      /* https://ss64.com/nt/move.html
    +       * "Under Windows 2000 and above, the default action is to prompt on overwrite unless the
    +       *  command is being executed from within a batch script. " */
    +
    +      if (!force) {
             static const WCHAR copyCmdW[] = {'C','O','P','Y','C','M','D','\0'};
             len = GetEnvironmentVariableW(copyCmdW, copycmd, ARRAY_SIZE(copycmd));
             force = (len && len < ARRAY_SIZE(copycmd) && !lstrcmpiW(copycmd, parmY));
    @@ -3051,14 +3050,16 @@ void WCMD_move (void)
             question = WCMD_format_string(WCMD_LoadMessage(WCMD_OVERWRITE), dest);
             ok = WCMD_ask_confirm(question, FALSE, NULL);
             LocalFree(question);
    +      } else {
    +        ok = TRUE;
    +      }
     
    -        /* So delete the destination prior to the move */
    -        if (ok) {
    -          if (!DeleteFileW(dest)) {
    -            WCMD_print_error ();
    -            errorlevel = 1;
    -            ok = FALSE;
    -          }
    +      /* So delete the destination prior to the move */
    +      if (ok) {
    +        if (!DeleteFileW(dest)) {
    +          WCMD_print_error ();
    +          errorlevel = 1;
    +          ok = FALSE;
             }
           }
         }
    -- 
    2.28.0


recompile cmd and install it into your wine prefix

    ./configure
    make programs/cmd -j$(nproc)
    cp programs/cmd/cmd.exe.so /path/to/wineprefix/drive_c/windows/syswow64/cmd.exe

everything should work as intended

I have submitted the patch to wine here: https://bugs.winehq.org/show_bug.cgi?id=48396
