+++
title = '如何调试 chrome 崩溃日志（MAC）'
date = 2024-12-20T20:15:13+08:00
draft = false
tags = ['崩溃', '调试']
categories = ['前端']
+++
# 引言
在使用 Chrome 浏览器的过程中，偶尔会遇到浏览器崩溃的情况。为了找出崩溃的原因并修复问题，我们需要对崩溃后的 .dmp 文件进行详细分析。本文将详细介绍如何从用户的系统中获取崩溃日志文件，使用 minidump_stackwalk 查看浏览器版本信息，下载对应的 symbols 文件，并使用 LLDB 进行详细分析。
# 第一步：获取崩溃日志文件
1.1 收集 .dmp 文件
当 Chrome 浏览器崩溃时，系统通常会生成一个 .dmp 文件，该文件包含了崩溃时的堆栈信息。用户可以在 Chrome 的崩溃报告目录中找到这些文件。路径通常为：
- Windows: C:\Users\<Username>\AppData\Local\Google\Chrome\User Data\Crash Reports
- macOS: ~/Library/Application Support/Google/Chrome/Crash Reports
- Linux: ~/.config/google-chrome/Crash Reports
# 第二步：使用 minidump_stackwalk 查看浏览器版本信息
## 2.1 安装 minidump_stackwalk
确保你的系统上安装了 minidump_stackwalk。你可以通过以下命令安装：
```shell
# macOS 和 Linux
brew install breakpad
```
或者从源码编译安装：
```shell
git clone https://chromium.googlesource.com/breakpad/breakpad
cd breakpad
./configure
make
sudo make install
```
## 2.2 使用 minidump_stackwalk 查看信息
使用 minidump_stackwalk 工具查看 .dmp 文件中的浏览器版本信息：
```shell
minidump_stackwalk your_crash.dmp /path/to/symbols
```
该命令会输出崩溃的详细信息，包括浏览器版本、模块列表等。根据输出的版本信息，
# 第三步：下载 symbols 文件
## 3.1 访问 Chrome 官方网站
根据 minidump_stackwalk 输出的版本信息，访问 Chrome 官方下载页面 下载对应的 symbols 文件。通常，symbols 文件可以在 Chrome 的开发者资源页面找到。
github 上有一个辅助下载脚本(download_symbols.py)，有些处理过时了，略作修改如下所示：
```python
#!/usr/bin/env python3
# Copyright 2021 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.
"""Downloads symbols for official, Google Chrome builds.

Usage:
    ./download_symbols.py -v 91.0.4449.6 -a x86_64 -c stable -o /dest/path

This can also be used as a Python module.
"""

import argparse
import csv
import os.path
import platform
import subprocess
import sys
import urllib.request as request

# 这个地址已经失效了，channel 需要手动指定（比如 【stable】，可以在 https://chromiumdash.appspot.com/releases?platform=Mac 中查看）
_OMAHAPROXY_HISTORY = 'https://omahaproxy.appspot.com/history?os=mac&format=json'

_DSYM_URL_TEMPLATE = 'https://dl.google.com/chrome/mac/{channel}/dsym/googlechrome-{version}-{arch}-dsym.tar.bz2'


def download_chrome_symbols(version, channel, arch, dest_dir):
    """Downloads and extracts the official Google Chrome dSYM files to a
    subdirectory of `dest_dir`.

    Args:
        version: The version to download symbols for.
        channel: The release channel to download symbols for. If None, attempts
                 to guess the channel.
        arch: The CPU architecture to download symbols for.
        dest_dir: The location to download symbols to. The dSYMs will be
                  extracted to a subdirectory of this directory.

    Returns:
        The path to the directory containing the dSYMs, which will be a
        subdirectory of `dest_dir`.
    """
    if channel is None:
        channel = _identify_channel(version, arch)
        if channel:
            print('Using release channel {} for {}'.format(channel, version),
                  file=sys.stderr)
        else:
            print('Could not identify channel for Chrome version {}'.format(
                version),
                  file=sys.stderr)
            return None

    extracted_dir = _download_and_extract(version, channel, arch, dest_dir)
    if not extracted_dir:
        print('Could not find dSYMs for Chrome {} {}'.format(version, arch),
              file=sys.stderr)
    return extracted_dir


def get_symbol_directory(version, channel, arch, dest_dir):
    """Returns the parent directory for dSYMs given the specified parameters."""
    _, dest = _get_url_and_dest(version, channel, arch, dest_dir)
    return dest


def _identify_channel(version, arch):
    """Attempts to guess the release channel given a Chrome version and CPU
    architecture."""
    # First try querying OmahaProxy for the release.
    with request.urlopen(_OMAHAPROXY_HISTORY) as release_history:
        history = csv.DictReader(
            release_history.read().decode('utf8').split('\n'))
        for row in history:
            if row['version'] == version:
                return row['channel']

    # Fall back to sending HEAD HTTP requests to each of the possible symbol
    # locations.
    print(
        'Unable to identify release channel for {}, now brute-force searching'.
        format(version),
        file=sys.stderr)
    for channel in ('stable', 'beta', 'dev', 'canary'):
        url, _ = _get_url_and_dest(version, channel, arch, '')
        req = request.Request(url, method='HEAD')
        try:
            resp = request.urlopen(req)
            if resp.code == 200:
                return channel
        except:
            continue

    return None


def _get_url_and_dest(version, channel, arch, dest_dir):
    """Returns a the symbol archive URL and local destination directory given
    the format parameters."""
    args = {'channel': channel, 'arch': arch, 'version': version}
    url = _DSYM_URL_TEMPLATE.format(**args)
    dest_dir = os.path.join(dest_dir,
                            'googlechrome-{version}-{arch}-dsym'.format(**args))
    return url, dest_dir


def _download_and_extract(version, channel, arch, dest_dir):
    """Performs the download and extraction of the symbol files. Returns the
    path to the extracted symbol files on success, None on error.
    """
    url, dest_dir = _get_url_and_dest(version, channel, arch, dest_dir)
    if not os.path.isdir(dest_dir):
        os.mkdir(dest_dir)

    print('download from {}'.format(url), file=sys.stderr)
    print(url, file=sys.stderr)

    try:
        with request.urlopen(url) as symbol_request:
            print('Downloading and extracting symbols to {}'.format(dest_dir),
                  file=sys.stderr)
            print('This will take a minute...', file=sys.stderr)
            if _extract_symbols_to(symbol_request, dest_dir):
                return dest_dir
    except:
        print('load failed!!!', file=sys.stderr)
        pass
    return None


def _extract_symbols_to(symbol_request, dest_dir):
    """Performs a streaming extract of the symbol files.

    Args:
        symbol_request: The HTTPResponse object for the symbol URL.
        dest_dir: The destination directory into which the files will be
                  extracted.

    Returns: True on successful download and extraction, False on error.
    """

    proc = subprocess.Popen(['tar', 'xjf', '-'],
                            cwd=dest_dir,
                            stdin=subprocess.PIPE,
                            stdout=sys.stderr,
                            stderr=sys.stderr)
    while True:
        data = symbol_request.read(4096)
        if not data:
            proc.stdin.close()
            break
        proc.stdin.write(data)
    proc.wait()

    return proc.returncode == 0


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--version',
                        '-v',
                        required=True,
                        help='Version to download.')
    parser.add_argument(
        '--channel',
        '-c',
        help='Chrome release channel for the version. The channel will be ' \
             'guessed if not specified.'
    )
    parser.add_argument(
        '--arch',
        '-a',
        help='CPU architecture to download, defaults to that of the current OS.'
    )
    parser.add_argument('--out',
                        '-o',
                        required=True,
                        help='Directory to download the symbols to.')
    args = parser.parse_args()

    arch = args.arch
    if not arch:
        arch = platform.machine()

    if not os.path.isdir(args.out):
        print('--out destination is not a directory.', file=sys.stderr)
        return False

    return download_chrome_symbols(args.version, args.channel, arch, args.out)


if __name__ == '__main__':
    main()
```
## 3.2 放置 symbols 文件
将下载的 symbols 文件放置在你指定的目录，例如 /path/to/symbols。
# 第四步：使用 LLDB 进行详细分析
## 4.1 安装 LLDB
确保你的系统上安装了 LLDB。大多数现代 macOS 和 Linux 发行版都自带了 LLDB，Windows 用户可以通过 Visual Studio 安装。
## 4.2 使用 LLDB 分析 .dmp 文件
打开终端，使用以下命令加载 .dmp 文件并进行分析：
```shell
lldb
(lldb) settings set target.exec-search-paths /symbol/file/path
(lldb) target create --core "/dmp/file/path"
(lldb) thread backtrace all
```
确保将 /path/to/executables 替换为你的 symbols 文件路径。
## 4.3 查看堆栈跟踪
使用以下命令查看崩溃时的堆栈跟踪：

```(lldb) bt```
这将显示详细的堆栈跟踪信息，帮助你定位崩溃的具体位置。之前定位过一个问题如下图所示：
{{< figure src="../../../imgs/9b1488c7a49f43beb6203111f7e6dc39.png" alt="crash信息截图" caption="crash信息截图" >}}
可以看出发生了 OOM，内存占用过大导致崩溃。
## 4.4 进一步调试
根据堆栈跟踪信息，你可以进一步调试，例如：
- 检查变量值
- 打印内存内容
- 断点调试
```shell
(lldb) frame variable
(lldb) memory read --size 16 --format x --count 10 <address>
```
# 结论
这一过程有助于深入理解崩溃的原因，并采取相应的修复措施。希望本文能帮助你有效地分析 Chrome 浏览器的崩溃问题。如果你有任何疑问，欢迎一起探讨。