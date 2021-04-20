# CI: Caching ffmpeg, tinytex dependencies and poetry venv
[ManimCommunity/manim#1257](https://github.com/ManimCommunity/manim/pull/1257) \
[ManimCommunity/manim#1339](https://github.com/ManimCommunity/manim/pull/1339)

## Issue
Dependency installation for macOS and Linux CI tests take up a considerable amount of time, often pushing each test up to 10+ minutes. There is already a limit of 5 concurrent macOS jobs, so caching will allow the reuse of previously installed dependencies to speed up the time it takes to complete each job.

No caching:
| | py37 | py38 | py39 |
| :---: | :---: | :---: | :---: |
| **Linux** | 8m 2s | 7m 41s | 6m 51s |
| **macOS** | 15m 7s | 14m 15s | 15m 26s |
| **Windows** | 10m 27s | 10m | 11m 18s |

## Planning and Implementation
**Resources and Annotations:**
- https://python-poetry.org/docs/configuration/ - Details on Poetry environment variables and config settings; Poetry venv for Python can be cached and reused to save time on dependency installation
- https://medium.com/ai2-blog/python-caching-in-github-actions-e9452698e98d - Tutorial on caching in GitHub actions
- https://docs.github.com/en/actions/learn-github-actions/managing-complex-workflows#caching-dependencies - Tutorial on caching in GitHub actions
- https://github.com/FedericoCarboni/setup-ffmpeg - GitHub action for FFmpeg installation and caching

Since the goal is to cache as many dependency-heavy packages within the 5GB cache size limit on GitHub Actions, a good place to start would be ffmpeg. The existing CI already had ffmpeg caching for Windows builds, so I decided dive straight in and do the same for macOS, but failed. Luckily, there was an existing action for ffmpeg installation and caching for all OS builds (Windows, macOS, Linux), and I decided to go with that.
```yml
- name: Install and cache ffmpeg (all OS)
  uses: FedericoCarboni/setup-ffmpeg@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
  id: setup-ffmpeg
```

Next, another large package that we could cache was the Poetry venv, which would contain the isolated Python environment with all the dependencies installed. This was significantly better than the existing pip cache, because the pip cache only contained downloaded wheel files used for installation, not the installed dependencies. This would only save time in downloading the files (which was not a lot), but not the installation of these files. Caching the entire Poetry venv would save time on the installation process; plus, the cache size is around the same compared to pip.
```yml
- name: Setup Poetry cache
  uses: actions/cache@v2
  with:
    path: ${{ steps.cache-vars.outputs.poetry-venv-dir }}
    key: ${{ runner.os }}-poetry-${{ env.pythonLocation }}-${{ hashFiles('poetry.lock') }}

# Cache is dependent on Python version and dependencies, so pythonLocation and poetry.lock are used as key. If any of them change, the cache would be updated.
```

Another large package that the software depends on is TinyTeX. The Windows builds already had TinyTeX cached, so I only had to do it for macOS (Linux was fast enough to not need caching). The process was similar, just needed to translate Windows PowerShell to Bash.
```yml
- name: Setup macOS cache
  id: cache-macos
  if: runner.os == 'macOS'
  uses: actions/cache@v2
  with:
    path: ${{ github.workspace }}/macos-cache
    key: ${{ runner.os }}-dependencies-tinytex-${{ hashFiles('.github/manimdependency.json') }}-${{ steps.cache-vars.outputs.date }}

- name: Install system dependencies (MacOS)
  if: runner.os == 'macOS' && steps.cache-macos.outputs.cache-hit != 'true'
  run: |
    tinyTexPackages=$(python -c "import json;print(' '.join(json.load(open('.github/manimdependency.json'))['macos']['tinytex']))")
    IFS=' '
    read -a ttp <<< "$tinyTexPackages"
    oriPath=$PATH
    sudo mkdir -p $PWD/macos-cache
    echo "Install TinyTeX"
    sudo curl -L -o "/tmp/TinyTeX.tgz" "https://ci.appveyor.com/api/projects/yihui/tinytex/artifacts/TinyTeX.tgz?job=image:%20macOS"
    sudo tar zxf "/tmp/TinyTeX.tgz" -C "$PWD/macos-cache"
    export PATH="$PWD/macos-cache/TinyTeX/bin/universal-darwin:$PATH"
    sudo tlmgr update --self
    for i in "${ttp[@]}"; do
    sudo tlmgr install "$i"
    done
    export PATH="$oriPath"
    echo "Completed TinyTeX"
```

After caching (poetry venv, ffmpeg, tinytex):
| | py37 | py38 | py39 |
| :---: | :---: | :---: | :---: |
| **Linux** | 6m 33s | 5m 32s | 5m 15s |
| **macOS** | 6m 23s | 6m 10s | 5m 17s |
| **Windows** | 6m 27s | 6m 36s | 7m 13s |