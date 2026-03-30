---
title:  "[Claude Code] Kitty Keyboard Protocol 한/영 전환 문제 해결"
excerpt: "macOS에서 Caps Lock 한/영 전환 시 [57358u 문자열이 누출되는 문제 분석 및 PTY 프록시 래퍼를 이용한 해결 방법"
toc: true
toc_sticky: true
header:
  teaser: /images/dev/claude-code-kitty-protocol.png

categories:
  - Dev
tags:
  - Claude Code
  - macOS
  - Terminal
  - Kitty Keyboard Protocol
  - iTerm2
last_modified_at: 2026-03-30T12:00:00+0900
comments: true
---

# 개요

Claude Code 실행 중 Caps Lock으로 한/영 전환 시 `[57358u` 문자열이 화면에 반복 출력되는 문제를 분석하고, PTY 프록시 래퍼로 해결한 기록이다.

> 환경: macOS (Darwin 25.3.0), zsh, iTerm2, Caps Lock 한/영 전환
> Claude Code 버전: v2.1.84+ (v2.1.83부터 동일 문제 확인)

# 증상

Claude Code 실행 중 Caps Lock으로 한/영 전환 시 `[57358u` 문자열이 화면에 반복 출력된다.
터미널 프롬프트에서도 `0;5u0;5u9;5u9;5u9;5u` 등 깨진 문자가 표시된다.

```
❯ [57358u[57358uasdfasdf[57358u[57358uㄴㅁㅇㄹ
```

# 원인 분석

## Kitty Keyboard Protocol이란

터미널 키 입력을 확장된 형식으로 전달하는 프로토콜이다.
애플리케이션이 `\x1b[>1u` (CSI > 1 u)를 stdout에 출력하면 터미널이 이 모드로 전환되어,
모든 키 이벤트를 `\x1b[{keycode};{modifier}u` 형식으로 전송한다.

이 프로토콜 덕분에 터미널 앱에서 Cmd+V, Ctrl+V 등 modifier 키 조합을 정확히 구분할 수 있다.

## Claude Code의 동작

Claude Code는 raw mode 진입 시 아래 시퀀스를 터미널에 전송한다:

| 시퀀스 | 의미 |
|--------|------|
| `\x1b[?2004h` | Bracketed Paste 활성화 |
| `\x1b[?1004h` | Focus Events 활성화 |
| **`\x1b[>1u`** | **Kitty Keyboard Protocol Push (flags=1)** |
| `\x1b[>4m` | 기타 모드 |

종료 시 역순으로 해제:

| 시퀀스 | 의미 |
|--------|------|
| `\x1b[>4m` | 모드 해제 |
| **`\x1b[<u`** | **Kitty Keyboard Protocol Pop** |
| `\x1b[?1004l` | Focus Events 비활성화 |
| `\x1b[?2004l` | Bracketed Paste 비활성화 |

## 한/영 전환 키와 Keycode 57358

Kitty protocol에서 Caps Lock 키는 keycode **57358**로 매핑된다.
macOS에서 Caps Lock을 한/영 전환으로 사용할 경우:

1. 사용자가 Caps Lock을 누름
2. 터미널이 kitty protocol 모드이므로 `\x1b[57358u`를 전송
3. Claude Code의 키 파서가 57358을 처리하지 못함
4. 인식 불가한 시퀀스가 raw 텍스트로 화면에 출력

```javascript
// Claude Code 키 파서 (minified)
function Blq(H) {
  switch(H) {
    case 9: return "tab";
    case 13: return "return";
    case 27: return "escape";
    case 127: return "backspace";
    case 57399: return "0";  // numpad
    // ... 57400~57415 numpad keys
    default:
      if (H >= 32 && H <= 126)
        return String.fromCharCode(H).toLowerCase();
      return;  // ← 57358은 여기서 undefined → 시퀀스 누출
  }
}
```

# 해결: PTY 프록시 래퍼

## 접근 방식의 변화

### v1.0.x: kitty protocol 전체 비활성화

최초 접근은 Claude Code의 stdout에서 `\x1b[>1u` (kitty push) 시퀀스를 통째로 제거하는 방식이었다.

```python
# v1.0.x
KITTY_PUSH = b'\x1b[>1u'
stdout_buf = stdout_buf.replace(KITTY_PUSH, b'', 1)
```

이 방식은 한/영 문제를 해결하지만, **kitty protocol 자체가 비활성화**되어 터미널이 legacy 모드로 동작한다. 그 결과:

- iTerm2에서 Cmd+V가 "터미널 텍스트 붙여넣기"로만 처리됨
- **Claude Code에 이미지 데이터가 전달되지 않아 Cmd+V 이미지 붙여넣기 불가**
- Ctrl+V로만 이미지 붙여넣기 가능 (불편)

### v1.1.0: CapsLock 키코드만 선택적 필터링 (현재)

kitty protocol은 유지하면서, 문제의 원인인 CapsLock 키코드(`ESC[57358u`)만 정규식으로 필터링한다.

```python
# v1.1.0
CAPSLOCK_PATTERN = re.compile(rb'\x1b\[57358(;\d+)*u')
stdout_buf = CAPSLOCK_PATTERN.sub(b'', stdout_buf)
```

**장점:**
- Cmd+V 이미지 붙여넣기 정상 동작
- 한/영 전환 키코드 누출 차단
- kitty protocol의 다른 기능(modifier 키 구분 등) 모두 유지

## 동작 원리

```
사용자 → stdin → [claude-wrapper] → stdin → [claude binary]
                                              ↓ stdout
사용자 ← stdout ← [57358u만 제거] ← stdout ←┘
                   (kitty protocol 유지)
```

1. PTY fork로 Claude Code를 자식 프로세스로 실행
2. stdin은 그대로 전달 (키 입력 변조 없음)
3. stdout에서 `\x1b[57358u` 패턴만 실시간 필터링
4. kitty protocol push/pop은 그대로 통과

## 래퍼 코드 (v1.1.0)

**파일 위치**: `~/.local/bin/claude-wrapper`

```python
#!/usr/bin/env python3
"""Claude Code wrapper - CapsLock 한/영 전환 키코드 필터.

Changelog:
  v1.0.0 (2026-03-26): kitty keyboard protocol 전체 비활성화 (ESC[>1u 제거)
  v1.0.1 (2026-03-26): PTY 크기 전달 버그 수정 — os.get_terminal_size()
                        rows/cols 순서 뒤바뀜 → ioctl TIOCGWINSZ 기반으로 교체
  v1.0.2 (2026-03-28): SIGWINCH 핸들러 안정화, set_winsize() ioctl 통합
  v1.1.0 (2026-03-30): kitty protocol 유지, CapsLock(57358u) 키코드만 필터링
                        → Cmd+V 이미지 붙여넣기 복원
"""
__version__ = "1.1.0"

import os
import re
import sys
import pty
import signal
import select
import struct
import fcntl
import termios

CAPSLOCK_PATTERN = re.compile(rb'\x1b\[57358(;\d+)*u')


def get_real_claude():
    """실제 claude 바이너리 경로 반환."""
    for path in [
        '/opt/homebrew/bin/claude',
        '/usr/local/bin/claude',
    ]:
        if os.path.exists(path):
            return path
    versions_dir = os.path.expanduser('~/.local/share/claude/versions')
    if os.path.isdir(versions_dir):
        versions = sorted(os.listdir(versions_dir))
        if versions:
            return os.path.join(versions_dir, versions[-1])
    return None


def set_winsize(fd):
    """터미널 윈도우 크기를 pty에 전달."""
    try:
        s = struct.pack('HHHH', 0, 0, 0, 0)
        result = fcntl.ioctl(sys.stdout.fileno(), termios.TIOCGWINSZ, s)
        fcntl.ioctl(fd, termios.TIOCSWINSZ, result)
    except (OSError, AttributeError):
        pass


def main():
    claude_bin = get_real_claude()
    if not claude_bin:
        print("Error: Claude Code binary not found", file=sys.stderr)
        sys.exit(1)

    pid, master_fd = pty.fork()

    if pid == 0:
        os.execvp(claude_bin, [claude_bin] + sys.argv[1:])
        sys.exit(1)

    set_winsize(master_fd)

    def handle_winch(signum, frame):
        set_winsize(master_fd)
        os.kill(pid, signal.SIGWINCH)

    signal.signal(signal.SIGWINCH, handle_winch)

    old_settings = None
    try:
        old_settings = termios.tcgetattr(sys.stdin.fileno())
    except termios.error:
        pass

    try:
        if old_settings:
            import tty
            tty.setraw(sys.stdin.fileno())

        stdout_buf = b''

        while True:
            try:
                rlist, _, _ = select.select(
                    [sys.stdin.fileno(), master_fd], [], [], 0.1
                )
            except (select.error, ValueError):
                break

            if sys.stdin.fileno() in rlist:
                try:
                    data = os.read(sys.stdin.fileno(), 4096)
                    if not data:
                        break
                    os.write(master_fd, data)
                except OSError:
                    break

            if master_fd in rlist:
                try:
                    data = os.read(master_fd, 4096)
                    if not data:
                        break

                    stdout_buf += data

                    if stdout_buf.endswith(b'\x1b'):
                        continue

                    stdout_buf = CAPSLOCK_PATTERN.sub(b'', stdout_buf)

                    os.write(sys.stdout.fileno(), stdout_buf)
                    stdout_buf = b''
                except OSError:
                    break

        if stdout_buf:
            stdout_buf = CAPSLOCK_PATTERN.sub(b'', stdout_buf)
            os.write(sys.stdout.fileno(), stdout_buf)

    finally:
        if old_settings:
            termios.tcsetattr(
                sys.stdin.fileno(), termios.TCSADRAIN, old_settings
            )

        try:
            _, status = os.waitpid(pid, 0)
            sys.exit(os.WEXITSTATUS(status) if os.WIFEXITED(status) else 1)
        except ChildProcessError:
            pass


if __name__ == '__main__':
    main()
```

## 설정 방법

`~/.zshrc`에 alias 추가:

```bash
alias claude='python3 ~/.local/bin/claude-wrapper'
```

이렇게 하면:
- `claude` 명령 실행 시 자동으로 래퍼 경유
- 자동 업데이트가 바이너리를 변경해도 영향 없음
- 래퍼가 바이너리 경로를 추적하여 항상 최신 버전 실행

# 추가 조치

## .zshrc — Kitty Protocol Pop (비정상 종료 대비)

Claude Code가 비정상 종료되어 protocol이 해제되지 않았을 때를 대비:

```bash
_disable_kitty_protocol() {
  printf '\e[<u\e[<u\e[<u\e[<u\e[<u'
}
autoload -Uz add-zsh-hook
add-zsh-hook precmd _disable_kitty_protocol
printf '\e[<u\e[<u\e[<u\e[<u\e[<u'
```

# 버그 수정 이력

## v1.0.1: PTY 크기 전달 버그 (2026-03-26)

`os.get_terminal_size()` 반환값은 `(columns, lines)` 순서인 반면, `struct.pack`의 `TIOCSWINSZ`는 `(rows, cols)` = `(lines, columns)` 순서를 기대한다. 튜플 언패킹 순서가 뒤바뀌어 PTY가 200행 × 50열로 설정되는 버그 발생.

**수정**: `os.get_terminal_size()` 대신 `ioctl TIOCGWINSZ`로 직접 현재 터미널 크기를 읽어 PTY에 그대로 전달하는 방식으로 교체.

## v1.0.2: SIGWINCH 핸들러 안정화 (2026-03-28)

`set_winsize()` 함수를 ioctl 기반으로 통합하여, 터미널 리사이즈 시에도 안정적으로 크기가 전파되도록 개선.

# 대안 (미적용)

| 방법 | 장단점 |
|------|--------|
| Caps Lock 대신 Ctrl+Space | 키 변경 필요 — 사용자 거부 |
| 바이너리 패치 | 시퀀스가 런타임 생성이라 불가 |
| VS Code 터미널 설정 | kitty protocol 비활성화 옵션 없음 |
| TERM=dumb | 색상/UI 깨짐 |
| 이전 버전 롤백 | 2.1.83에도 동일 문제 |

# 향후 대응

- Anthropic이 keycode 57358 (및 기타 미처리 modifier key)을 무시 처리하면 래퍼 불필요
- Claude Code 업데이트 후 래퍼 없이 테스트: `command claude` (alias 우회)
- 해결 확인 시 `.zshrc`에서 alias 제거

# 버전 이력

| 버전 | 날짜 | 변경 |
|------|------|------|
| v1.0.0 | 2026-03-26 | kitty keyboard protocol 전체 비활성화 (`ESC[>1u` 제거) |
| v1.0.1 | 2026-03-26 | PTY 크기 전달 버그 수정 — `ioctl TIOCGWINSZ` 기반으로 교체 |
| v1.0.2 | 2026-03-28 | SIGWINCH 핸들러 안정화, `set_winsize()` ioctl 통합 |
| v1.1.0 | 2026-03-30 | kitty protocol 유지, CapsLock `57358u`만 필터링 → Cmd+V 복원 |
