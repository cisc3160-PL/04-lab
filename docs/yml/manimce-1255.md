# CI: Fix macOS pipeline failure
[ManimCommunity/manim#1255](https://github.com/ManimCommunity/manim/pull/1255)

## Issue
CI macOS pipeline failure due to installation of basictex package that no longer exists.

```
curl: (22) The requested URL returned error: 404 
Error: Download failed on Cask 'basictex' with message: Download failed: http://mirror.ctan.org/systems/mac/mactex/mactex-basictex-20200407.pkg
Error: Process completed with exit code 1.
```

Failing CI workflow:
```yml
- name: Install system dependencies (MacOS)
  if: runner.os == 'macOS'
  run: |
    brew install openssl readline ffmpeg pyenv pyenv-virtualenv
    brew install --cask basictex
    eval "$(/usr/libexec/path_helper -s)"
    sudo tlmgr update --self
    brew install pkg-config
    brew install libffi
    brew install pango
    brew install glib
    sudo tlmgr install standalone preview doublestroke relsize fundus-calligra wasysym physics dvisvgm.x86_64-darwin dvisvgm rsfs wasy cm-super
    echo "/Library/TeX/texbin" >> $GITHUB_PATH
    echo "$HOME/.poetry/bin" >> $GITHUB_PATH
```

## Planning and Implementation
**Resources and Annotations:**
- https://formulae.brew.sh/cask/basictex - Can give details on basictex package (i.e. current version, conflicts, requirements)
- https://docs.brew.sh/Manpage - Manual for brew (useful commands)
- https://www.preining.info/blog/2021/04/tex-live-2021-released/ - News regarding TeX Live 2021 release (major update, breaking changes)

Based on the `curl` error above, the basictex package that's being installed is outdated. The current version (at time of writing) is `2021.0325`, while the pipeline tries to install `2020.0407`. This means we should update brew and the basictex package before installing.

```sh
brew update && brew upgrade basictex && brew cleanup
```

After fixing the basictex error, a different error has occurred:
```
tlmgr install: package dvisvgm.x86_64-darwin not present in repository.
tlmgr: package repository https://mirror.las.iastate.edu/tex-archive/systems/texlive/tlnet (verified)
[1/12, ??:??/??:??] install: cm-super [63050k]
[2/12, 00:14/00:14] install: doublestroke [66k]
[3/12, 00:14/00:14] install: dvisvgm.universal-darwin [2554k]
[4/12, 00:15/00:15] install: dvisvgm [1k]
[5/12, 00:16/00:16] install: fundus-calligra [2k]
[6/12, 00:16/00:16] install: physics [6k]
[7/12, 00:16/00:16] install: preview [7k]
[8/12, 00:16/00:16] install: relsize [6k]
[9/12, 00:17/00:17] install: rsfs [55k]
[10/12, 00:17/00:17] install: standalone [12k]
[11/12, 00:17/00:17] install: wasy [24k]
[12/12, 00:17/00:17] install: wasysym [4k]
tlmgr: action install returned an error; continuing.
running mktexlsr ...
done running mktexlsr.
running updmap-sys ...
tlmgr: An error has occurred. See above messages. Exiting.
done running updmap-sys.
tlmgr: package log updated: /usr/local/texlive/2021basic/texmf-var/web2c/tlmgr.log
Error: Process completed with exit code 1.
```

The texlive manager tries to install `dvisvgm.x86_64-darwin`, but it's not present in repository. However, it has installed `dvisvgm.universal-darwin` instead, so something must have been updated between the time the CI job was last successful and the time it first failed. Looking at the master commit history, this brings the timeframe to right after April 1st, 2021 when CI starts failing.

Searching for news related to `dvisvgm.universal-darwin`, I found out that TeX Live 2021 had been released right at the beginning of April, which matches our timeframe, and gives us important information about MacTeX:
- MacTeX and its new binary folder universal-darwin now require macOS 10.14 or higher (Mojave, Catalina, and Big Sur); the x86_64-darwin binary folder is no longer present. The x86_64-darwinlegacy binary folder, available only with the Unix install-tl, supports 10.6 and newer.
- This is an important year for the Macintosh because Apple introduced ARM machines in November and will sell and support both ARM and Intel machines for many years. All programs in universal-darwin have executable code for both ARM and Intel. Both binaries are compiled from the same source code.

Looks like we found the reason for the macOS CI pipeline failure, so we can remove `dvisvgm.x86_64-darwin` and keep `dvisvgm`.
```diff
- sudo tlmgr install standalone preview doublestroke relsize fundus-calligra wasysym physics dvisvgm.x86_64-darwin dvisvgm rsfs wasy cm-super
+ sudo tlmgr install standalone preview doublestroke relsize fundus-calligra wasysym physics dvisvgm rsfs wasy cm-super
```

After making these changes, the macOS CI pipeline successfully passes:
```yml
- name: Install system dependencies (MacOS)
  if: runner.os == 'macOS'
  run: |
    brew update && brew upgrade basictex && brew cleanup
    brew install openssl readline ffmpeg pyenv pyenv-virtualenv
    brew install --cask basictex
    eval "$(/usr/libexec/path_helper -s)"
    sudo tlmgr update --self
    brew install pkg-config libffi pango glib
    sudo tlmgr install standalone preview doublestroke relsize fundus-calligra wasysym physics dvisvgm rsfs wasy cm-super
    echo "/Library/TeX/texbin" >> $GITHUB_PATH
    echo "$HOME/.poetry/bin" >> $GITHUB_PATH
```