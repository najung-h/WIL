# TIL — “6시간마다 + 최근 커밋 없으면 스킵, 메일은 안 가게”



## 문제

- 스케줄 실행 때 **커밋이 없으면 실패로 끝나 알림 메일**이 갔다.

- `git-auto-commit-action` 단계에서 **non-fast-forward** 에러(원격과 분기)도 발생했다.

  

## 목표

- **6시간마다(scheduled)** 실행하되, **최근 6시간 내 master 커밋 없으면 성공(Success)으로 조기 종료**.

- 원격과 분기가 나지 않게 안전하게 커밋/푸시.

  

## 해결 요약

1. **Checkout 먼저** 하고(`actions/checkout@v4`)
2. **최근 6시간 master 커밋 유무 체크** → 없으면 **성공 종료(Exit 0)**
3. 이후 스텝은 **조건 실행(`if:`)** 으로 커밋 있을 때만 수행
4. Checkout은 **`fetch-depth: 0`**(전체 히스토리)로 받아와 rebase 가능하게
5. auto-commit 전에 **`git pull --rebase origin master`** 로 분기 정리
6. **concurrency 그룹**으로 동시 실행 충돌 방지

---

즉,

**커밋 없으면 → 바로 성공 종료**

**커밋 있으면 → 산출물 생성 후 master/output 반영**

**분기 에러 안 나게 rebase 처리**

```yml
name: Profile 3d grass and snake   # 워크플로 이름

on:
  schedule:                        # 스케줄 이벤트
    - cron: "0 */6 * * *"          # 6시간마다 (UTC 기준) 실행
  workflow_dispatch:                # 수동 실행도 가능

permissions:
  contents: write                  # output 브랜치에 푸시하려면 contents 권한 필요

concurrency:                        # 동시 실행 충돌 방지
  group: profile-assets             # 동일 그룹 실행 중일 경우 대기
  cancel-in-progress: false         # 기존 실행은 취소하지 않음

jobs:
  build:
    runs-on: ubuntu-latest          # 최신 Ubuntu 환경에서 실행

    steps:
      # 0) 저장소 전체 히스토리 체크아웃 (rebase 가능하도록 fetch-depth: 0)
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0            # 얕은 클론 말고 전체 히스토리 가져오기

      # 1) 최근 6시간 동안 master 브랜치 커밋 수 확인
      - name: Check commits within last 6 hours on master
        id: check                   # 이후 if 조건에서 참조하기 위한 ID
        run: |
          git fetch origin master --depth=1                     # 원격 master 최신 커밋 가져오기
          COUNT=$(git log origin/master --since='6 hours ago' \ # 최근 6시간 커밋 로그 확인
          --oneline | wc -l | tr -d ' ')                        # 커밋 수 세기
          echo "commit_count=$COUNT" >> "$GITHUB_OUTPUT"        # 출력값 저장 (steps.check.outputs.commit_count)

      # 1-1) 최근 커밋이 없으면 조기 종료 (성공 상태) → 실패 메일 안 감
      - name: No recent commits, skip the run
        if: ${{ steps.check.outputs.commit_count == '0' }}      # commit_count가 0일 때만 실행
        run: |
          echo "No commits on master in the last 6 hours. Exiting successfully."
          exit 0                                                 # 성공 코드로 종료

      # 2) (커밋이 있을 때만 실행) dist 폴더 준비
      - name: Prepare dist folder
        if: ${{ steps.check.outputs.commit_count != '0' }}      # commit_count가 0이 아닐 때 실행
        run: mkdir -p dist

      # 3) (커밋이 있을 때만 실행) 잔디(metrics) SVG 생성
      - name: Generate 3d grass
        if: ${{ steps.check.outputs.commit_count != '0' }}
        uses: lowlighter/metrics@latest
        with:
          user: najung-h                       # 사용자 ID
          token: ${{ secrets.METRICS_TOKEN }}  # PAT 토큰 필요
          template: classic                    # 카드 템플릿
          filename: dist/metrics-6m.svg        # dist에 저장
          base: ""                             # 기본 정보 제거
          config_timezone: Asia/Seoul          # 한국 시간대
          plugin_isocalendar: yes              # 달력 플러그인
          plugin_isocalendar_duration: half-year # 6개월 범위

      # 4) (커밋이 있을 때만 실행) Snake 애니메이션 생성
      - name: Generate snake
        if: ${{ steps.check.outputs.commit_count != '0' }}
        uses: Platane/snk@v3
        id: snake-gif
        with:
          github_user_name: najung-h           # 사용자 ID
          outputs: |                           # 여러 파일 형식으로 저장
            dist/github-contribution-grid-snake.svg
            dist/github-contribution-grid-snake-dark.svg?palette=github-dark
            dist/github-contribution-grid-snake.gif

      # 5) (옵션) 상태 확인
      - name: Git status
        if: ${{ steps.check.outputs.commit_count != '0' }}
        run: git status

      # 6) (옵션) dist 폴더 내용 확인
      - name: List dist
        if: ${{ steps.check.outputs.commit_count != '0' }}
        run: ls -alh dist || true

      # 7) (커밋이 있을 때만 실행) output 브랜치로 배포
      - name: Deploy profile assets to output branch
        if: ${{ steps.check.outputs.commit_count != '0' }}
        uses: crazy-max/ghaction-github-pages@v3
        with:
          target_branch: output                # 배포 브랜치
          build_dir: dist                      # 배포할 디렉토리
          commit_message: "chore: update profile assets"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # 8) (커밋이 있을 때만 실행) 원격 master와 rebase 동기화
      - name: Sync master with remote (rebase)
        if: ${{ steps.check.outputs.commit_count != '0' }}
        run: |
          git fetch origin master              # 원격 master 가져오기
          git checkout master                  # master 체크아웃
          git pull --rebase origin master      # rebase로 동기화

      # 9) (커밋이 있을 때만 실행) dist 변경사항 master에 커밋 & 푸시
      - name: Commit dist to default branch
        if: ${{ steps.check.outputs.commit_count != '0' }}
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore: sync dist assets to default branch" # 커밋 메시지
          branch: master                       # 기본 브랜치
          file_pattern: dist/**                # dist 내 파일만 추적
          # 필요시: push_options: --force-with-lease (경쟁 충돌시 임시 해결책)

```



## 포인트/교훈

- **실패 메일을 막으려면 “실패를 만들지 말자”**: 조건 불충족 시 성공 상태로 종료.
- `if:`로 **필요한 스텝만** 실행해 러너 낭비/오류 최소화.
- GitHub Actions에서 `schedule`+`push` **AND 조건은 불가** → **런타임 조건 체크**로 해결.
- `fetch-depth: 0` 없이 rebase 시 분기/충돌 빈발.
- 안전한 강제푸시는 `--force-with-lease` (최후수단).