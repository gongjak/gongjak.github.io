# gongjak.github.io — 프로젝트 지침

> 신규 작성. 상속: L0 `~/.claude/CLAUDE.md` · L1 `~/Developer/CLAUDE.md`.

grade: prototype

> 마크다운 문서를 다듬을 때는 `doc-simplify` 스킬을 따른다 (Global, 자동 발동).

## 무엇인가

GitHub Pages 개인 사이트. Jekyll (Minimal Mistakes 테마).

> [!note] 이름을 바꿀 수 없다
> GitHub Pages 사용자 사이트는 `<username>.github.io` 가 강제된다.
> 저장소 명명 규칙(kebab-case)의 **유일한 예외**다.

## 하드 규칙

1. **삽질로그와 글이 겹치지 않게 한다.** 기술 문제 해결 기록은 `sapjillog` 로 갔다.
   과거에 실제로 중복이 나서 정리한 적이 있다
2. **테마 파일을 직접 고치지 않는다.** `_sass`/`_includes` 오버라이드로 처리한다.
   테마를 올리면 직접 수정분이 날아간다
3. 시크릿·고객사 식별 정보를 넣지 않는다. 공개 사이트다

## 명령

```bash
bundle install
bundle exec jekyll serve --drafts
bundle exec jekyll build
```

CI: `.github/workflows/stale.yml` (이슈 정리용. 빌드는 GitHub Pages 가 한다).

## 구조

```
_posts/ · _pages/ · _data/
_layouts/ · _includes/ · _sass/     테마 오버라이드
_config.yml
assets/
```

## 완료 기준

- `bundle exec jekyll build` 가 오류 없이 끝난다
- 배포 후 **사이트에서 실제로** 확인했다
